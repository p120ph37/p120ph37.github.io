---
layout: post
tags: [Apache]
title: Forcing HTTPS via mod_rewrite
---
A fairly typical question for Apache administration is how to force access to
a certain set of resources to occur exclusively via
[HTTPS](http://en.wikipedia.org/wiki/HTTP_Secure). This can be accomplished via
[browser-side JavaScript](http://stackoverflow.com/questions/4723213/detect-http-or-https-then-force-https-in-javascript)
in a pinch, but the more canonical (and
[MitM](http://en.wikipedia.org/wiki/Man-in-the-middle_attack)-resistant)
approach is to enforce this from the server side.  Fortunately, this is fairly
simple to accomplish using
[mod_rewrite rules](http://httpd.apache.org/docs/current/mod/mod_rewrite.html)
in httpd.conf or an [.htaccess file](http://davidwalsh.name/force-secure-ssl-htaccess).
For example, the below `.htaccess` file can be dropped into a directory, and
assuming `mod_ssl`, `mod_rewrite`, and `.htaccess` files are all properly
enabled, all requests for resources in this directory or its subdirectories
will be forcibly redirected to the HTTPS version of the URL.

{% highlight apache %}
RewriteEngine On

# redirect HTTP connections to HTTPS based on internal mod_ssl state
RewriteCond %{HTTPS} !=on
RewriteRule .? https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
{% endhighlight %}

This doesn't quite work in some more complex situations though.  For example,
in a configuration where the SSL layer is handled by a
reverse-proxy/load-balancer like an [F5](http://www.f5.com/)
[Big-IP](http://www.f5.com/products/big-ip/), the built-in Apache HTTPS
mechanism is useless because all connections arrive at the Apache box as HTTP.
In this case, the HTTPS forcing could be accomplished with a
[rule](http://devcentral.f5.com/tabid/1082202/Default.aspx) on the proxy
itself, but this can be complex to maintain since it places the configuration
further from the data in question.  Alternately, if the proxy has an option to
enable the [X-Forwarded-Proto](http://drupal.org/node/313145) header, you can
still do the redirect at the Apache layer using an
[.htaccess](http://en.wikipedia.org/wiki/Htaccess) file like this:

{% highlight apache %}
RewriteEngine On

# redirect HTTP connections to HTTPS based on external proxy headers
RewriteCond %{HTTP:X-Forwarded-Proto} =http
RewriteRule .? https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
{% endhighlight %}

Keep in mind that this configuration
[assumes the X-Forwarded-Proto header is trustworthy](http://forums.tsohost.co.uk/feature-requests/1351-loadbalancer-anti-header-spoofing-measures.html).
This can be true if the proxy in question is the only means to access the
Apache server from the Internet and if it overwrites any Internet-supplied
content in this header, but if other direct access is possible or if the proxy
passes existing values through, the header may be untrustworthy and susceptible
to MitM attacks.

Also of note is the fact that the above examples do not rely on the assumption
that all accesses to the same resource will occur via the same domain name.
In other words, because of the use of the `HTTP_HOST` and `REQUEST_URI`
variable, the redirect will mirror whatever domain name was used in the initial
request rather than imposing a hard-coded domain name as seen in some other
examples.

Further, in some configurations, various other rewrite rules may have been
applied to the request before the HTTPS-forcing rule is encountered.  In such
cases, the
[gory details of the rewritten URL](http://www.mroodles.com/wordpress/hacking/mod_rewrite-displaying-original-url/)
may not be something you want sent back to the user's browser.  So the thing
to do would be to issue the redirect based on the original requested URL +
query string, not the already-rewritten URL + query string.  Since the
`REQUEST_URI` variable available within Apache contains the rewritten URI
minus the query string, we need to choose a different variable.  Confusingly,
an environment variable of the same name is provided to CGI scripts containing
the value we are actually looking for (the original pre-rewrite request
including original query string), but that value is not available to
`mod_rewrite` rules.

In researching how this variable is set by Apache for CGIs, I found this piece
of code in the Apache source:

{% highlight c %}
AP_DECLARE(void) ap_add_cgi_vars(request_rec *r)
{
    apr_table_t *e = r->subprocess_env;

    apr_table_setn(e, "GATEWAY_INTERFACE", "CGI/1.1");
    apr_table_setn(e, "SERVER_PROTOCOL", r->protocol);
    apr_table_setn(e, "REQUEST_METHOD", r->method);
    apr_table_setn(e, "QUERY_STRING", r->args ? r->args : "");
    apr_table_setn(e, "REQUEST_URI", original_uri(r));
{% endhighlight %}

So the CGI environment variable called `REQUEST_URI` is set by a function called
`original_uri`.  Well, that sounded like what I wanted, so I located that function:

{% highlight c %}
/* Obtain the Request-URI from the original request-line, returning
 * a new string from the request pool containing the URI or "".
 */
static char *original_uri(request_rec *r)
{
    char *first, *last;

    if (r->the_request == NULL) {
        return (char *) apr_pcalloc(r->pool, 1);
    }

    first = r->the_request;     /* use the request-line */

    while (*first &amp;&amp; !apr_isspace(*first)) {
        ++first;                /* skip over the method */
    }
    while (apr_isspace(*first)) {
        ++first;                /*   and the space(s)   */
    }

    last = first;
    while (*last &amp;&amp; !apr_isspace(*last)) {
        ++last;                 /* end at next whitespace */
    }

    return apr_pstrmemdup(r->pool, first, last - first);
}
{% endhighlight %}

From this I discovered that even the Apache source itself does not have a
useful pre-computed value for what I was looking for. Instead, the value
is derived programmatically from the `the_request` value (e.g.
`GET /cgi-bin/printenv.pl?abc=123 HTTP/1.1`), which it just so happens is
available to `mod_rewrite` as `%{THE_REQUEST}`.  And the parsing logic is
fairly simple - simple enough that the regex engine in mod_rewrite can be
employed to perform the same steps.

Finally, we need to consider two possible cases:  the original request may
have a query string included, or it may have no query string.  If a query
string is present and we place this original request value into a
`RewriteRule` as-is, it will override any other previously rewritten query
string (remember, we are trying to get back to the original as-requested URL
to change just the protocol portion of it), but if a query string is absent
from the original request, the `RewriteRule` will try to carry over the
existing rewritten query string.  To suppress this, we need to include the `?`
in the `RewriteRule` in all cases, so we need to parse the original request
into its two possible halves - the base request, and the query string, if any
\- and then re-assemble them in the RewriteRule while including a `?` in all
cases so as to force overwriting of any previous query string.

And here is the final masterpiece:

{% highlight apache %}
RewriteEngine On

RewriteCond %{HTTP:X-Forwarded-Proto} =http
RewriteCond %{THE_REQUEST} ^\S*\s*(.*?)(\?(.*?))?(\s|$)
RewriteRule .? https://%{HTTP_HOST}%1?%3 [R=301,L]
{% endhighlight %}