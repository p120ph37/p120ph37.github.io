---
layout: post
tags: [JavaScript]
title: JavaScript Function Chaining
---
From time to time when working with JavaScript, I come across situations where
I want to extend some existing function.  Sometimes this function is defined in
a file which I cannot or do not wish to change - it is often a bad idea to fork
a local modified copy of a library which is maintained by someone else upstream
who might release a new version. You would then be forced you to manually port
your hack into their new version of the source code (though a good
[source control mechanism](http://www.youtube.com/watch?v=4XpnKHJAok8) can be
very helpful there).  In these cases, function chaining is an option.

Though it may not appear so initially, JavaScript actually excels at this sort
of thing - and that is good since JavaScript is often employed to patch
incremental functionality on top of existing code.  Lets disregard for the
moment the advent of newer, more advanced
[DOM](https://developer.mozilla.org/en/DOM) mechanisms like
[addEventListener](https://developer.mozilla.org/en/DOM/element.addEventListener)
and hark back to ye olde days of
[window.onload](https://developer.mozilla.org/en/DOM/window.onload). 
Something like this perhaps:

{% highlight js %}
/*
 * prettytitle.js - Makes the titlebar into an image.
 */
// wait until the document has finished loading.
window.onload = function() {
    // get the titlebar element and replace its content with an image tag.
    document.getElementById('titlebar').innerHTML =
        '<img src="titlebar.png" />';
};
{% endhighlight %}

Then say this website needs to add another, separate progressive-enhancement
script.  Or two.  Or more.  As long as you had only one script, you could just
have the script set window.onload to its main function and be done with it, but
as soon as you have multiple scripts this breaks.

Enter function chaining.  If each of these scripts chains onto the end of the
existing window.onload function, you can stack them ad infinitum, or at least
to the practical limits of memory and computational power.  A first attempt to
apply this to the above example might look something like this:

{% highlight js %}
/*
 * prettytitle.js - Makes the titlebar into an image.
 */
// remember what the old document.onload function was before replacing it.
var old_onload = window.onload;
// wait until the document has finished loading.
window.onload = function() {
    // execute the previous onload function (if it exists) before our own code.
    if(old_onload != null) old_onload();
    // get the titlebar element and replace its content with an image tag.
    document.getElementById('titlebar').innerHTML =
        '<img src="titlebar.png" />';
};
{% endhighlight %}

Unfortunately, this has a few problems: First, it uses a
[global variable](http://en.wikipedia.org/wiki/Global_variable) to store the
old onload function, so if another script happens to use the same variable name,
the value will [get clobbered](http://en.wikipedia.org/wiki/Naming_collision).
Here is similar example but using the onclick event for ease of illustration - try
running this in the [FireBug](http://getfirebug.com/) console and then clicking
on the document to see for yourself:

{% highlight js %}
document.onclick = null; // clear any previous mess you made

var old_onclick = document.onclick;
document.onclick = function() {
    if(old_onclick != null) old_onclick();
    console.log('Function 1 sees a click!');
};

var old_onclick = document.onclick;
document.onclick = function() {
    if(old_onclick != null) old_onclick();
    console.log('Function 2 sees a click!');
};
{% endhighlight %}

Second, although the original function is invoked, it does not have any of the
expected arguments passed to it (for example, the
[event](https://developer.mozilla.org/en/DOM/event") argument) as seen in this
console example:

{% highlight js %}
document.onclick = null; // clear any previous mess you made

var old_onclick1 = document.onclick;
document.onclick = function(e) {
    if(old_onclick1 != null) old_onclick1();
    console.log('Function 1 sees a click!');
    console.log('Click coordinates: '+e.clientX+','+e.clientY);
};

var old_onclick2 = document.onclick;
document.onclick = function() {
    if(old_onclick2 != null) old_onclick2();
    console.log('Function 2 sees a click!');
};
{% endhighlight %}

And third, the value of "this" is not consistent as can bee seen with this
example:

{% highlight js %}
document.onclick = null; // clear any previous mess you made

var old_onclick1 = document.onclick;
document.onclick = function() {
    if(old_onclick1 != null) old_onclick1();
    console.log('Function 1 sees a click!');
    console.log('this is: '+this);
};

var old_onclick2 = document.onclick;
document.onclick = function() {
    if(old_onclick2 != null) old_onclick2();
    console.log('Function 2 sees a click!');
    console.log('this is: '+this);
};
{% endhighlight %}

So how do you resolve all of these issues? Lambda expressions, the "arguments"
variable, and the "apply" method, of course! If this terse and snarky answer
doesn't immediately answer your questions, read on for more details...

Because JavaScript
[lexical scoping](http://en.wikipedia.org/wiki/Scope_(computer_science)#Lexical_scoping)
rules rely on function boundaries, employing a
[lambda expression](http://en.wikipedia.org/wiki/Anonymous_function)
gives us function boundaries for the sake of scoping without polluting the
namespace with the function name.

For the argument passing issue, we could manually pass some fixed number of
arguments to the chained function, but a more elegant solution would work with
any number of arguments without knowing in advance how many will arrive. The
"[arguments](http://www.seifi.org/javascript/javascript-arguments.html)"
array is present inside any function and contains the complete set of arguments
in the form of an array. But since this is an array, if you just pass it in a
normal function call, it will be sent as a single array object, not as the
multiple individual arguments it once was.

This issue and the final problem with "this" are both solved by the
"[apply](https://developer.mozilla.org/en/JavaScript/Reference/Global_Objects/Function/apply)"
method. This is a method available on all function objects (remember, everything
is an object, even functions!). By using this method to invoke the function
rather than the usual idiom, a value for "this" can be specified and an array
object can be exploded into individual arguments.

So this leads us to the final ideal JavaScript function chaining version of the
example:

{% highlight js %}
/*
 * prettytitle.js - Makes the titlebar into an image.
 */
function(){ // Lambda expression for scope
    var old_onload = window.onload; // local variable
    window.onload = function() {
        if(old_onload != null) {
            // invoke the old handler, passing this and arguments properly.
            old_onload.apply(this, arguments);
        }
        document.getElementById('titlebar').innerHTML =
        '<img src="titlebar.png" />';
    };
}(); // End of lambda expression and immediate invocation</pre>
{% endhighlight %}