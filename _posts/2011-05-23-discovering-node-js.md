---
layout: post
tags: [JavaScript]
title: Discovering Node.js
---
Sometime last year I came across
[Node.js](http://nodejs.org), the server-side port of
[Google's V8](http://code.google.com/p/v8/)
[ECMAScript](http://en.wikipedia.org/wiki/ECMAScript) engine. At first
I was interested in it mostly as a novelty to allow things like form validation
library portability from client to server side.  After trying it out and
reading through the [documentation](http://nodejs.org/docs/latest/api/) in more
detail and watching Ryan's
[presentation](http://developer.yahoo.com/yui/theater/video.php?v=dahl-node),
I became a lot more excited about it.

It turns out that
[Ryan Dahl](http://tinyclouds.org/) has taken the task of developing a server-side
JavaScript environment, which would seem at first blush to be something
[unholy](http://en.wikipedia.org/wiki/JScript),
[bulky](http://en.wikipedia.org/wiki/Rhino_(JavaScript_engine)) and perhaps even
[passe](http://en.wikipedia.org/wiki/Netscape_Enterprise_Server"), and reframed
it in a way that really showcases JavaScript's strengths and builds on the
single-threaded event model pioneered by the
[DOM](https://developer.mozilla.org/en/DOM) and familiar to browser-savvy
developers the web over. Some might find the lack of threads a complete put-off,
but having worked with [Boa](http://www.boa.org/) in the past, I was immediately
attracted to this idea.

Since Node.js is essentially just a set of bindings for
V8 to allow it to interact with a server type environment instead of the more
familiar DOM, Ryan had complete freedom to implement all of the core I/O API
calls in a way that relies exclusively on callbacks to
[avoid blocking](http://en.wikipedia.org/wiki/Asynchronous_I/O). It is this
clever trick that allows Node.js to perform extremely well in unexpected roles
such as
[webserver](http://posterous.ngspinners.com/benchmarking-static-file-servers-lightnode-li),
while at the same time avoiding the complex
[select event loop](http://linux.die.net/man/2/select) logistics usually
associated with single-threaded daemons. This also sidesteps the entire can of
[thread-synchronization](http://java.sun.com/docs/books/jls/second_edition/html/memory.doc.html)
worms usually associated with high-performance network service.

To show how this
works, here is a naive script which implements a
[memcached](http://memcached.org/)-like network memory cache in just a handful of
lines of code:

{% highlight js %}
#!/usr/bin/node
/*
 * A simple mem cache server in ECMAScript for Node.js
 * Connect to the socket and issue a command in the form:
 *   foo=bar
 * or
 *   foo
 * (The former gets the old value of the key and sets a new value,
 * and the latter just gets the old value without setting anything.)
 */
var cache = {};
var net = require('net');
var server = net.createServer(function (stream) {
    var buffer = '';
    var host = stream.remoteAddress + ':' + stream.remotePort;
    stream.setEncoding('utf8');
    stream.on('data', function (data) {
        buffer = buffer + data;
        var lines = buffer.split(/\r?\n/);
        if(lines.length > 1) {
            buffer = lines[1];
            console.log('Request from ' + host + ': ' + lines[0]);
            var parts = lines[0].split(/=/);
            stream.write(parts[0] + '=' + cache[parts[0]] + '\n');
            if(parts[1] != null) cache[parts[0]] = parts[1];
        }
    });
});
{% endhighlight %}

Here is what it looks like when you telnet to it:

{% highlight text %}
$ telnet localhost 8888
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
foo=bar
foo=undefined
foo
foo=bar
^]
telnet> quit
Connection closed.
$
{% endhighlight %}
    
And this is what the server log looks like:

{% highlight text %}
Connection made from 127.0.0.1:60704
Request from 127.0.0.1:60704: foo=bar
Request from 127.0.0.1:60704: foo
Connection closed from 127.0.0.1:60704
{% endhighlight %}

Another really great thing about Node.js is that it brings exposure to
JavaScript as a general purpose language. Most developers, myself included,
were first exposed to JavaScript via its inclusion in
[Netscape Navigator](http://en.wikipedia.org/wiki/Netscape_Navigator) and were
frustrated by its limited access to useful components of the document and the
incompatibilities with Microsoft's JScript. Blind we were to the the features
of the language itself for these frustrations. But now, with ECMAScript
standardization, the modern DOM, fast implementations like V8 and SpiderMoney
(and now [JaegerMonkey](https://wiki.mozilla.org/JaegerMonkey)), and a strong
community of developers building reusable code, it is finally ready to be seen
for what it is:
[a prototype-based object-oriented language](http://www.crockford.com/javascript/javascript.html)

Most classically-trained developers are used to the phrase "object-oriented
language" meaning [classes](http://www.bfoit.org/itp/JavaClass.html) in the
sense of
[Java](http://pages.cs.wisc.edu/~cs368-1/JavaTutorial/NOTES/Java-Classes.html)
or [C++](http://www.cplusplus.com/doc/tutorial/classes/), but others who cut
their teeth on like likes of [Perl](http://perldoc.perl.org/perlobj.html) or
[Lua](http://www.lua.org/pil/16.1.html) will understand this phrase to also
include other mechanisms of object creation and inheretence. JavaScript falls
into the latter category, and this along with its
[first-class functions](http://www.joelonsoftware.com/items/2006/08/01.html)
and [closures](http://en.wikipedia.org/wiki/Closure_(computer_science)) gives
it some really interesting
[potential](http://javascript.crockford.com/private.html). To demonstrate
this, here is the same contrived example as above, but with a portion
restated in a more object-oriented manner:

{% highlight js %}
#!/usr/bin/node
/*
 * Define a constructor for "lineParser" objects.
 * These objects will maintain an internal buffer to which they will
 * append incoming data, and whenever a complete line has been
 * received, they will dispatch a call to the specified callback
 * function.
 */
var lineParser = function(callback) {
    this.callback = callback;
    this.buffer = '';
};
lineParser.prototype.addData = function(data) {
    this.buffer = this.buffer + data;
    var lines = this.buffer.split(/\r?\n/);
    if(lines.length > 1) {
        this.buffer = lines[1];
        this.callback( lines[0] );
    }
};

/*
 * The rest is mostly the same as last time but uses the lineParser
 * constructor to create objects for chunking the incoming stream into
 * lines.
 */
var cache = {};
var net = require('net');
var server = net.createServer(function(stream) {
    var host = stream.remoteAddress + ':' + stream.remotePort;
    stream.setEncoding('utf8');
    var lp = new lineParser(function(line) {
        console.log('Request from ' + host + ': ' + line);
        var parts = line.split(/=/);
        stream.write(parts[0] + '=' + cache[parts[0]] + '\n');
        if(parts[1] != null) cache[parts[0]] = parts[1];
    });
    stream.on('data', function(data) { lp.addData(data); });
});
{% endhighlight %}

Of course, there are many more things you can do with prototype style
inheritance, but this demonstrates that it can be used to emulate the classical
class mechanism and also demonstrates the use of a closure to bind the local
"host" variable to the callback provided to the lineParser constructor.