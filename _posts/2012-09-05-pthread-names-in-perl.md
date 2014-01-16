---
layout: post
tags: [Perl]
title: pthread Names and IDs from Perl
---
Modern versions of Perl provide support for threads.  On *nix systems,
this is implemented via the system's `pthreads` (AKA "POSIX threads")
support.  This means that each thread looks to the operating system
like its own lightweight process.  POSIX threads on linux can run on
separate cores, can have separate process ID and names, and can receive
separate signals.  Most of the existing Perl documentation isn't very clear
on how to manage these special attributes from Perl.

First of all, as per the
[perlvar](http://perldoc.perl.org/perlvar.html#%240) manpage, it is possible
to set the process name by modifying `$0`.  In modern versions of Perl
(>5.8), this affects two separate system properties: the process command-line,
and the process name.  The difference between these two properties is
important to understand.

The process command-line is part of the original memory block the OS
allocates when creating a process.  This is usually used initially to pass
the command-line options into the process, and is what populates `@ARGV`.
After the process is started, it can overwrite this area with whatever it
likes, and the result will be displayed by certain process-management tools
such as `ps` and `top`.  The size of this data is limited to whatever space
the OS originally allocated for the `*argv` array.  Because all of the threads
in a single process share the same `*argv` area, there can be only one
command-line across all threads of the process.

Additionally, Linux keeps some internal kernel metadata about each process,
and part of this is a "process name".  This "process name" is a short string
(<=16 bytes) which is taken from the filename of the executable from which the
process was created.  For Perl processes, this is the name of the perl
interpreter binary itself ("perl").  This is the value displayed by default in
top (you can switch to display the command-line instead by pressing "c"), and
the value used by the killall command.  Because each POSIX thread is its own
lightweight process in Linux, each thread will have its own PID and process
name.

When you change the value of the `$0` variable in Perl, prior to version 5.10,
it would only set the command-line not the process name, but in version 5.10
and later, it sets both (truncating the process name to 16 bytes as
necessary).  This means, that you can use the `$0` variable within threads to
define the process name for that thread, but this also repeatedly clobbers the
shared command-line.  In order to work around this, you can temporarily set
the `$0` variable in the main thread to the desired process name for a child
thread, spawn the child thread, and then restore the value in the main thread.

{% highlight perl %}
use 5.010;
use threads;
my $child;
{
    local $0 = 'mythread';
    $child = async { sleep 60; };
}
$child->join();
{% endhighlight %}

This example will create a child thread with the name `mythread`, while the
parent will retain its original settings.

Secondly, obtaining the process ID belonging to a thread in Perl is not
straightforward.  The `threads->tid()` function will simply return Perl's own
internal thread ID, which is an integer starting from "0" for the main thread,
and the special variable `$$` will always return the main process ID, not the
process ID of the thread.  In order to obtain the actual OS value of the
thread's PID, a bit of hackery is required.  The Linux-specific `gettid()`
call is what we want, and it is available via the `syscall` interface using a
constant defined in `syscall.ph`.  The below code demonstrates using this:

{% highlight perl %}
require 'syscall.ph';
use threads;
print "parent PID: $$\n";
async {
    print "thread PID: ".syscall(&amp;SYS_gettid)."\n";
}->join();
{% endhighlight %}

If for some reason syscall.ph isn't available on your Linux system, you can
generate it using `h2ph -a syscall.h`