---
layout: post
title: Simple WebDAV Part 1
categories: [documentum, java, webdav]
---

This is **part 1 of a [2-part][1] series** detailing how I implemented [WebDAV][2] support for a [Documentum][3] repository.  It is very basic and does not implement all of the methods that WebDAV supports, but it gets the job done.  The end result is a type of "Web Edit" functionality that allows single-click checkout, edit, and checkin for Microsoft Office files (but the idea isn't restricted in any way to MS Office types).

**The first goal is the actual WebDAV server**, which was actually much easier to put together than I expected.  Some of you may already know, but I was surprised to learn that Apache Tomcat 5.5+ has WebDAV support **built in** as a standalone servlet.  I simply copied this "WebdavServlet" and stuck my code in where it made sense.  WebdavServlet uses the filesystem as its repository, which is a common and flexible way to handle opening and editing files.  This works fine for us, since we are working with physical files as well (they just don't happen to be in this particular location, yet).  **Our second goal is to get those files to behave as if they are local**.

Let's take a look at WebdavServlet.java (obtained via [Subversion][4] @ http://svn.apache.org/repos/asf/tomcat/trunk/java/org/apache/catalina/servlets/WebdavServlet.java).  In addition to the DefaultServlet's **GET** and **PUT** methods, it implements two methods specific to the WebDAV protocol that we are interested in: **LOCK** and **UNLOCK**.  We can drop our custom code into these methods to push and pull files from Documentum (or any other system, really):

**LOCK:** Interestingly, there are two steps to perform here.  First, you must get the file from Documentum and move it to a local path (I am just moving it to the root of the application context).  You should do this right after the path is retrieved from the request object (look for this line in doLock():

<pre class="prettyprint">
String path = getRelativePath(req);

...

Content contentObj = d.getContents().get(0);
// Get the Content of this file
if (contentObj.canGetAsFile()) {
	File f = contentObj.getAsFile();
	File to = new File(getServletContext().getRealPath("/")+ path);
	if (f.renameTo(to))
		log("get successful.");
	else {
		if (f.exists() &amp;&amp; f.canWrite())
			f.delete();
		log("file move failed.");
	}
} else {
	log("get unsuccessful.");
}
</pre>

The second step is to checkout the document in Documentum.  You can do this pretty much wherever you want in the method, but I prefer to wait until the very end of doLock() to make sure that the file was downloaded successfully.  Now you have the local file for Tomcat to serve with GET (which is called after LOCK), and it is locked for editing in your repository.

**GET:** This one is the easiest!  We actually don't have to modify this at all.  This will serve our files from the filesystem like it normally would.

**PUT:** As you can probably guess, we will use this method to save the file back to Documentum.  Just place your logic at the end of the method, and do as you normally would when saving files to the repository from disk.  A slight catch is that you should also checkout the file again, because you can't be sure that the user is finished editing.  Keep in mind that any save operation will fire a PUT, so the user should maintain the lock.  Once the WebDAV client is ready to release the lock, it will call UNLOCK.

**UNLOCK:** All you do here is cancel the checkout in the repository.  That's it.  Well, I also delete it from the filesystem just to be sure it doesn't get stepped on later:

<pre class="prettyprint">
//remove the file from the filesystem
File f = new File(getServletContext().getRealPath("/")+ path);
if (f.exists() &amp;&amp; f.canWrite()) {
	f.delete();
	log("file deleted.");
}
</pre>

Now, you may be wondering "hey, what about security?"  This is a very good question and is something that, fortunate for me, I did not have to worry about since this is not a public system.  Tomcat's WebDAV default web.xml does allow for the normal Tomcat authentication methods (using conf/tomcat-users.xml and the like) so this is still a possibility.  I pass the username in the URL to the file, so I know who to perform repository operations as, in the form 

	http://<domain>/<application context>/<username>–i_chronicle_id.<file extension>.

Your WebDAV server implementation should now be complete!  But how will you provide the single-click "Web Edit" capability that many organizations ask for, and many other products integrate?  [**In part 2**][1], I'll show you how to create a COM component using .Net which can start an application and load WebDAV URLs.

Cheers!

 [1]: http://unicron.github.com/.net/webdav/2009/10/15/simple-webdav-part-2.html "Simple WebDAV Part 2"
 [2]: http://www.webdav.org/
 [3]: http://www.emc.com/products/family/documentum-family.htm
 [4]: http://subversion.tigris.org/  