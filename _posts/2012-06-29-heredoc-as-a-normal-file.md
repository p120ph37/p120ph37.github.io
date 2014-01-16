---
layout: post
tags: [Bash]
title: HEREDOC as a Normal File
---
The Bash `HEREDOC` feature is quite useful when you need to script the stdin
input to a command, however not all commands can be coerced into reading
their input from stdin.  Some commands require that you supply filenames
from which to read some of their input.  In these cases, your Bash script
could create a temporary file on-disk, write some content to it, execute the
command, and then delete the temporary file afterword.  This works of course,
but wouldn't it be nifty if there was a way to do this all at once via a
`HEREDOC`?  In fact there is, but it is not immediately obvious.

The standard `HEREDOC` syntax uses a double-left-angle-bracket to direct input
to stdin.  Like any other Bash redirection, this is just the standard form with
the assumed target filehandle (0 aka stdin in this case).  You can specify a
different filehandle if you like.  Filehandles 1 and 2 are stdout and stderr
respectively and are thus not useful for input.  But what about 3?  Filehandle
3 doesn't normally exist at process creation, but Bash can create it if you ask.

So we have something like this:

{% highlight sh %}
mycommand 3<<END
my data here
END
{% endhighlight %}

Well, that isn't very useful in itself because the command probably isn't aware
that it should look at file descriptor 3 for input, but that's where the next
trick comes in:  There is a directory on Linux under the `/dev` hierarchy which
allows access to the current process' file descriptors by number as if they
were files.  So `/dev/fd/3` would be the pseudo-filename for the file which
backs file descriptor 3. (which technically in our case would be the
Bash-created transient temp-file with the heredoc data in it).

So, assuming `mycommand` expects an option `input_filename` to specify a file
from which to read input, the following command would feed input from a heredoc
instead:

{% highlight sh %}
mycommand --input_filename=/dev/fd/3  3<<END
my data here
END
{% endhighlight %}

Clever, no?