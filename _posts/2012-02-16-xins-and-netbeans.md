---
layout: post
tags: [Java]
title: XINS and NetBeans
---
There are many ways to build
[Web Services](http://en.wikipedia.org/wiki/Web_service)
in Java.  Probably the most common approach is to use
[JAX-WS](http://en.wikipedia.org/wiki/Java_API_for_XML_Web_Services)
annotations to
[build a SOAP web service](http://docs.oracle.com/javaee/5/tutorial/doc/bnayn.html)
within a
[J2EE](http://en.wikipedia.org/wiki/Java_Platform,_Enterprise_Edition)
container.  Besides the [reference implementation](http://jax-ws.java.net/) of
[JSR-224](http://jcp.org/en/jsr/detail?id=224), the two web service frameworks
from the Apache project, [Axis2](http://axis.apache.org/axis2/java/core/) and
[CXF](http://cxf.apache.org/), are the most well-known.  When creating web
services with any of these,
[WSDL](http://en.wikipedia.org/wiki/Web_Services_Description_Language)
documents are an important part of the process.  They may either be
[generated from the API code](http://cxf.apache.org/docs/java-to-ws.html)
[after it is written](http://cxf.apache.org/docs/defining-contract-first-webservices-with-wsdl-generation-from-java.html)
(bottom-up), or they may be created first and used to
[generate a skeleton](http://cxf.apache.org/docs/wsdl-to-java.html) from which
the API code development may begin (top-down).  WSDL files are
[intended to be machine-readable](http://oreilly.com/catalog/webservess/chapter/ch06.html),
so although they provide an easy way to
[inform an application](http://www.service-repository.com/client/start) about
[available](http://www.xmethods.com/ve2/Directory.po)
[web service](http://www.webservicex.net/WS/default.aspx)
[calls](http://www.service-repository.com/) and parameters, they do a poor job
of
[enlightening systems integrators](http://msdn.microsoft.com/en-us/library/aa480506.aspx)
as to the actual functionality behind the interface.  In other words, a WSDL
alone does not serve as adequate
[documentation](http://code.google.com/p/wsdl-viewer/).  WSDL files are also
tedious to construct by hand, so top-down approaches that start with a WSDL
file are not always as nice as they sound.  One other drawback to some
WebService frameworks is that they focus only on the
[SOAP protocol](http://en.wikipedia.org/wiki/SOAP), which works well for
communication from purpose-built client applications, but is awkward
[from browser javascript](http://www.guru4.net/articoli/javascript-soap-client/en/)
([JSON](http://json.org/) or
[plain-old-XML](http://en.wikipedia.org/wiki/Plain_Old_XML) are the
[usual](http://en.wikipedia.org/wiki/Ajax_(programming)) preferred formats
there).

So if a WSDL file, documentation, and multiple communication formats all need
to be supported, and if these are all simply different presentations of the
same intent, why not generate all of them automatically from a concise
[DSL](http://en.wikipedia.org/wiki/Domain-specific_language)?  This is exactly
the approach taken by the [XINS framework](http://xins.sourceforge.net/): a
few [Ant](http://ant.apache.org/) tasks and config file edits, and you have a
WSDL file, comprehensive API documentation and test forms,
[several access methods](http://xins.sourceforge.net/docs/ar01s23.html)
(Including SOAP, POX, and JSON[P]), input and output parameter contract
enforcement, and stubbed-out, ready-to-run Java implementation classes.  The
only initial drawback I found with this framework is that Eclipse has trouble
figuring out how to deal with source file auto-generation and with atypical
source and output directory structures.  This prompted me to look into other
[IDEs recommended for use with XINS](http://xins.sourceforge.net/docs/ar01s29.html),
and so I decided to try [NetBeans](http://netbeans.org/).

As I am somewhat new to the modern Java IDE world, I hadn't really looked at
NetBeans yet, and after downloading and installing it, I was pleasantly
surprised at how clean, intuitive, well-designed, and easily customizable it
was in comparison to Eclipse.  Sure Eclipse has an immense number of
configurable options, and yes you can write your own plugin modules if you
want to teach it new tricks, but for the most part I found the overabundance
of configuration options tended to get in the way of finding the ones I
actually wanted to set.  I was especially happy with the clean NetBeans
integration with Ant - since I have to maintain Ant scripts to handle
deployment anyway, it makes perfect sense to use Ant scripts as the primary
means to customize the IDE's actions as well.

So I began trying to use NetBeans with the XINS framework.  All of the
existing XINS documentation is written from an IDE-agnostic standpoint, and
the brief mention of IDE integration is not very clear, so it was a bit of a
learning curve to figure out not only how to use the framework itself, but
how to use it effectively with an IDE.  However, now that I've finished
setting it up with NetBeans, it works nicely, and I thought I'd share this
short tutorial for the benefit of others.

-   First, download and install the Java EE version of
    [NetBeans](http://netbeans.org/downloads/index.html).
-   Open NetBeans and add a few filename associations (in Preferences -> Misc.
    -> Files) for "fnc", "typ", "rcd" and "cat" as "XML Files (text/xml)".
-   Then close NetBeans and
    [download XINS](http://sourceforge.net/projects/xins/files/latest/download?source=files).
-   Pick some reasonable location to unzip it.  I chose to place mine inside
    `.netbeans` in my user directory ( `~/.netbeans/` ).
-   Edit the (possibly non-existant) file `~/.netbeans/7.1/etc/netbeans.conf`
    (of course the "7.1" part may be different if you installed a newer
    version) and add a line which specifies the XINS location like this:  
    `export XINS_HOME=~/.netbeans/xins-2.3`
-   Re-open NetBeans and in Tools -> DTDs and XML Schemas add the OASIS
    catalog (note, if you put the XINS installation inside your `~/.netbeans`
    folder, you will not be able to browse to it, but you can type in the full
    path manually.  Also, you cannot use the "~" macro here - the path must
    be absolute.):  
    `file:/Users/awirtz/.netbeans/xins-2.3/src/dtd/xinsCatalog.xml`

Now XINS and NetBeans are installed and configured and you can start a new project.

-   In a terminal window, use "export XINS_HOME=~/.netbeans/xins-2.3" or the
    "set" command (Windows) to set the XINS home directory for this session.
    If you wanted, you could make this a system-wide setting, but I personally
    prefer not to unnecessarily clutter my environment variables.
-   Follow the tutorial instructions found
    [here](http://xins.sourceforge.net/primers/primer.html), except for the
    adjustments listed below.
-   Before step 4:
    +   Now that you have the skeleton of an API created, you can configure it
        so the rest of the steps can be done in the IDE.
    +   Unpack [these files](https://github.com/p120ph37/XINS-NB/zipball/master)
        into your apis/myapi directory, and then open that directory in NetBeans
        as a project.
-   Right-click on the nbbuild.xml file and select the "update-nb-files" target.
-   Note on step 4: You can find and edit the "fnc" file via the "Files" tab in
    the NetBeans project pane.
-   Instead of steps 5-7, you can right-click on the project and select "Generate
    Specdocs" and the docs should build and the browser should pop up automatically.
-   Instead of step 11, you can right-click on the project and select "Generate
    Javadocs".
-   Instead of steps 12-13, you can simply run the project.
-   Instead of step 16, you can click the red square (stop) icon next to the console
    output in NetBeans to stop the app.
-   Again, instead of step 18, you can simply run the project.