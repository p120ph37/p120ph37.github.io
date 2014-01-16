---
layout: post
tags: [JavaScript]
title: XHR, localStorage and Images
---
Traditionally, website resource management has been handled more or less
opaquely by the browser.  A webpage would declare a number of linked resources
via `src` or `href` attributes on various HTML tags, and these would be fetched
by the browser as it saw fit.  Later, some control of resource caching was
provided via server-side HTTP headers, but the browser remains the mastermind.

There are three main problems with this way of doing things.  First, the
caching of resources is not easily modified - once the browser has fetched a
resource with a
[Cache-Control](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9)
header defined, there is no telling if or when it will decide to refresh its
copy short of the pre-defined TTL.  Second, the order and progress of the
resource fetching cannot be easily monitored or controlled.  Finally, if you
have used `Cache-Control: no-cache`, even though
[HTTP 304](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3.5)
responses may save you the bandwidth of re-downloading some large files, the
browser still needs to fire off a separate request for every resource.  Of
course there are hackish work-arounds, such as
[appending a nonce](http://www.developphp.com/view_lesson.php?v=275) to the
query string to force a refresh, or attaching
[onLoad](http://www.w3schools.com/jsref/event_img_onload.asp) events to all
img tags on the page, or using a
[spritesheets](http://css-tricks.com/css-sprites/) to bulk images together
into a single HTTP request, but the usefulness of these is limited.

With recent technology developments, this no longer needs to be the case.
For dynamic websites, the JavaScript
[XMLHttpRequest](http://en.wikipedia.org/wiki/XMLHttpRequest) object provides
the capability to fetch text-based resources from a server programmatically,
and the
[localStorage](http://en.wikipedia.org/wiki/Web_storage#Local_and_session_storage)
object provides a to programmatically store data client-side.  By combining
these two features, a better resource cache could be created.

It might seem at first that the XHR object is only useful for fetching
text-based resources such as XML or JavaScript files, there are ways to use it
to fetch images too.  First, you have to convince it to fetch binary data,
then you need to have a way to inject the fetched data into an img tag.
Fetching binary data can be done via
[either the "overrideMimeType" hack or the "responseType='arraybuffer'" feature](https://developer.mozilla.org/en/using_xmlhttprequest#Handling_binary_data)
(though both of these have
[issues in IE](http://miskun.com/javascript/internet-explorer-and-binary-files-data-access/))
Injecting the data into an img tag can be done by
[translating it](http://jsperf.com/encoding-xhr-image-data/16) into a
[data URI](http://en.wikipedia.org/wiki/Data_URI_scheme).  Because of browser
issues with these tricks, if IE compatibility is an issue, translating the
data into Base64 data-URIs might be best done on the server-side (if
[compression of the HTTP stream](http://en.wikipedia.org/wiki/HTTP_compression)
is enabled, this introduces minimal bandwidth overhead).

So, putting together these pieces, a clinet-side managed cache could look
something like this:

*   Define a list of resources
*   Check local storage to see which resources exist and their age
*   Submit a single request to a web service, listing the required resources
    and whether a local copy exists yet
*   Receive a single packed response from the server containing only the
    changed resources
*   Update the local storage with the changed resources
*   Further JavaScript on the page can now make use of the locally-stored
    resources rather than fetching them

Stay tuned for more ideas on how this idea can be combined with the HTML5
`manifest` attribute to build an offline-capable, self-updating iPhone app
that can be installed directly from the web (no app-store!).