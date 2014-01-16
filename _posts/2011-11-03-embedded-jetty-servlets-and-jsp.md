---
layout: post
tags: [Java]
title: Embedded Jetty Servlets and JSP
---
Recently I had been using [Tomcat](http://tomcat.apache.org/) as a
[Java Servlet](http://en.wikipedia.org/wiki/Java_Servlet) container for a
project at work.  This works well in the context of a tightly-integrated set
of servlets and [JSP](http://en.wikipedia.org/wiki/JavaServer_Pages) pages
like a typical website, but the project I was working on is intended to be a
self-contained module which presents an HTTP API and should be easily
deployable without worrying too much about
[shared settings](http://tomcat.apache.org/tomcat-6.0-doc/config/context.html#Resource_Links)
and
[shared libraries](http://tomcat.apache.org/tomcat-6.0-doc/class-loader-howto.html)
on the target box.  I also wanted the ability to
[profile](http://www.eclipse.org/tptp/home/documents/tutorials/profilingtool/profilingexample_32.html),
[debug](http://www.vogella.de/articles/EclipseDebugging/article.html), and run
[jUnit tests](http://www.junit.org/) on the component from within
[Eclipse](http://www.eclipse.org/), without requiring an additional, separate
deployment to a Tomcat server (even if it is
[a local Tomcat server on my dev machine](http://www.mulesoft.com/tomcat-eclipse).

So I decided to switch to a design involving embedding a
[Jetty](http://www.eclipse.org/jetty/) container into a plain Java app.  This
would give me the ability to have a clean deployable artifact with no external
dependencies, and also to launch the servlet container and the
[HTTP tests](http://download.eclipse.org/jetty/stable-8/apidocs/org/eclipse/jetty/testing/HttpTester.html)
against it from the same jUnit script.  Sadly, the documentation for
[embedding Jetty](http://docs.codehaus.org/display/JETTY/Embedding+Jetty) is
not the most comprehensive.

In order to keep things as tight and straightforward as possible, I opted to
instantiate the
[individual parts of Jetty](http://wiki.eclipse.org/Jetty/Reference/Dependencies#Dependency_Tree)
directly rather than just using the
[WebApp context](http://download.eclipse.org/jetty/stable-8/apidocs/org/eclipse/jetty/webapp/WebAppContext.html)
[as shown in the examples](http://download.eclipse.org/jetty/stable-8/xref/org/eclipse/jetty/embedded/OneWebApp.html).
(using the WebApp context would involve additional dependencies and overhead
of dealing with [WAR files](http://en.wikipedia.org/wiki/WAR_file_format_(Sun))
and [web.xml](http://docs.codehaus.org/display/JETTY/jetty-web.xml), etc.)
Getting a
[basic servlet context](http://download.eclipse.org/jetty/stable-8/xref/org/eclipse/jetty/embedded/OneServletContext.html)
running and
[serving static content](http://download.eclipse.org/jetty/stable-8/apidocs/org/eclipse/jetty/servlet/DefaultServlet.html)
was fairly simple, but I couldn't find any good documentation or example on
setting up the JSP servlet without using the WebApp wrapper.  After some
[source-reading](http://download.eclipse.org/jetty/stable-8/xref/org/eclipse/jetty/webapp/StandardDescriptorProcessor.html#251)
and a lot of experimentation, I got it working, but I ran into quite a few issues along the way.

*   The
    [DefaultServlet](http://download.eclipse.org/jetty/stable-8/apidocs/org/eclipse/jetty/servlet/DefaultServlet.html)
    class serves static content, but not JSP pages.

    Solution: use
    [JspServlet](http://tomcat.apache.org/tomcat-6.0-doc/api/org/apache/jasper/servlet/JspServlet.html).
    
*   Jetty uses parts of the
    [Glassfish JSP](http://jsp.java.net/) project and parts of the
    [Tomcat Jasper](http://tomcat.apache.org/tomcat-6.0-doc/jasper-howto.html)
    engine, and the Jetty 8.0.3 version tried to mix and match incompatible
    library versions of these.  This results in
    [`java.lang.AbstractMethodError: javax.servlet.jsp.PageContext.getELContext()javax/el/ELContext;`](http://stackoverflow.com/questions/2534883/jetty-7-hightide-distribution-jsp-and-jstl-support)

    Solution: use
    [Jetty 8.0.4](http://download.eclipse.org/jetty/stable-8/dist/)

*   When trying to compile JSP pages on the fly, the JspServlet is unable to
    find the Jetty JARs even though they are in the application's classpath,
    resulting in `PWC6033: Error in Javac compilation for JSP,PWC6199:
    Generated servlet error:string:///index_jsp.java:3: package javax.servlet
    does not exist`

    Solution: set the
    [classpath](http://tomcat.apache.org/tomcat-6.0-doc/jasper-howto.html#Configuration)
    [InitParam](http://download.eclipse.org/jetty/stable-8/apidocs/org/eclipse/jetty/servlet/Holder.html#setInitParameter(java.lang.String,%20java.lang.String)
    [based on the context's ClassLoader](http://download.eclipse.org/jetty/stable-8/apidocs/org/eclipse/jetty/server/handler/ContextHandler.html#getClassPath()),
    which in turn should be set from the
    [current thread](http://download.oracle.com/javase/6/docs/api/java/lang/Thread.html#getContextClassLoader()).

After fighting through all of these issues, I finally arrived at the clean,
programmatically-configured embedded Jetty solution, which I present to you
here:

{% highlight java %}
import org.apache.jasper.servlet.JspServlet;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.servlet.DefaultServlet;
import org.eclipse.jetty.servlet.ServletContextHandler;
import org.eclipse.jetty.servlet.ServletHolder;

class RunServer {
    public static void main(String args[]) {

        System.out.println("Initializing server...");
        final ServletContextHandler context =
          new ServletContextHandler(ServletContextHandler.SESSIONS);
        context.setContextPath("/");
        context.setResourceBase("webapp");

        context.setClassLoader(
            Thread.currentThread().getContextClassLoader()
        );

        context.addServlet(DefaultServlet.class, "/");

        final ServletHolder jsp =
          context.addServlet(JspServlet.class, "*.jsp");
        jsp.setInitParameter("classpath", context.getClassPath());

        // add your own additional servlets like this:
        // context.addServlet(JSONServlet.class, "/json");

        final Server server = new Server(8080);
        server.setHandler(context);

        System.out.println("Starting server...");
        try {
            server.start();
        } catch(Exception e) {
            System.out.println("Failed to start server!");
            return;
        }

        System.out.println("Server running...");
        while(true) {
            try {
                server.join();
            } catch(InterruptedException e) {
                System.out.println("Server interrupted!");
            }
        }
    }
}
{% endhighlight %}