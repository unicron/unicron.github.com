---
layout: post
title: Simple WebDAV Part 2
categories: [java, .net, webdav]
---

**Part 2 **of this **[series][1]** is about the client-side extension used to open the remote file locally.  Normally, clicking a link to a file causes the browser to prompt the user or OS for what to do with it, and then pull it down as a stream.  This behavior can sometimes be modified depending on the HTTP headers sent along with the server response (attachment specification, MIME type, etc.), and browser plugins, but in nearly all cases it is still the browser interacting with the file as a stream and opening it directly.  Since the browser is not a WebDAV client, it doesn’t implement the protocols we need.

What we need to do is open the file directly using a WebDAV client, such as Microsoft Word.  We can accomplish this one of several ways:

1. Write our own COM component which we can instantiate from browser javascript.  
2. Use the the SharePoint library which is built-in to Office 2000-2007.  
3. Provide only the link and have the user open Word and the file location interactively.

**Write our own COM Component**:  
This is the initial approach I took, and it can be a good one for several reasons.  First, it allows you to control how the file is opened in the client application.  You can have full access to the application using Office Automation, which can be powerful if you need to set things up a certain way for the user.  I wrote my implementation using a combination of C# and VB.NET, but I am confident it could be accomplished using other 4GLs.

For my component, I created the interface and implementation in C# and wrote the client interaction in VB.NET.  This was more personal preference than anything else because I prefer C#, but have had better luck doing Office Automation using VB.  The key here is to also implement the IObjectSafety interface in order to avoid an annoying security popup every time a user clicks on your links.

You can see how simple this is (remember to create new GUIDs for yourself!):

*WebEditCS.cs*:

{% highlight c# %}
    namespace WebEdit.ActiveX.CS {
    
        [
            Guid("B6E9F230-F7C6-4686-9DB1-6C0E8C33FBA2"),
            InterfaceType(ComInterfaceType.InterfaceIsDual),
            ComVisible(true)
        ]
        public interface IWebEditCS {
            [DispId(1)]
            bool DoWebEdit(int app, string url);
        };
    
        [
            Guid("01453349-18FB-434f-AB73-BFA3F73D9967"),
    
            // This is basically the programmer friendly name
            // for the guid above. We define this because it will
            // be used to instantiate this class. I think this can be
            // whatever you want. Generally it is
            // [assemblyname].[classname]
            ProgId("WebEdit.ActiveX"),
    
            // No class interface is generated for this class and
            // no interface is marked as the default.
            // Users are expected to expose functionality through
            // interfaces that will be explicitly exposed by the object
            // This means the object can only expose interfaces we define
            ClassInterface(ClassInterfaceType.None),
    
            // Set the default COM interface that will be used for
            // Automation. Languages like: C#, C++ and VB
            // allow to query for interface's we're interested in
            // but Automation only aware languages like javascript do
            // not allow to query interface(s) and create only the
            // default one
            ComDefaultInterface(typeof(IWebEditCS)),
            ComVisible(true)
        ]
        public class WebEditCS : IObjectSafetyImpl, IWebEditCS {
    
            public bool DoWebEdit(int app, string url) {
                return WebEditVB.DoWebEditVB(app, url);
            }
        };
    }
{% endhighlight %}    

This is just an example, and could be modified a lot to open different applications with different settings.   This particular version also uses a single copy of each application by getting a reference to the running object (if it exists).  While this is probably the safest way to do Office Automation, it isn’t the only one (you can certainly open a new copy every time a link is opened…).   Experiment and find what works best for your situation.

*WebEditVB.vb*:

{% highlight vb %}
    Namespace WebEdit.ActiveX.VB
        Public Class WebEditVB
    
            Private Shared Function GetInstance(ByVal appName As String) As Object
                Dim application As Object = Nothing
    
                Try
                    application = Marshal.GetActiveObject(appName)
                    If application Is Nothing Then
                        application = CreateObject(appName)
                    End If
                Catch ex As Exception
                    application = CreateObject(appName)
                End Try
    
                Return application
            End Function
    
            Public Shared Function DoWebEditVB(ByVal app As Integer, ByVal url As String) As Boolean
    
                Dim application As Object = Nothing
                Dim document As Object = Nothing
    
                Try
                    If app = 0 Then
                        application = GetInstance("Word.Application")
                        application.Visible = True
                        document = application.Documents.Open(url)
                        document.Activate()
    
                    ElseIf app = 1 Then
                        application = GetInstance("PowerPoint.Application")
                        application.Visible = True
                        document = application.Presentations.Open(url)
    
                    ElseIf app = 2 Then
                        'user cannot have an open spreadsheet with a cell currently being edited!
                        application = GetInstance("Excel.Application")
                        application.Visible = True
                        document = application.Workbooks.Open(url)
                        document.Activate()
                    End If
    
                Catch ex As Exception
                    Try
                        Dim logName As String = "C:Program FilesNLRBQuickEdit-ErrorLog.txt"
                        Dim fs As FileStream = New FileStream(logName, FileMode.Append, FileAccess.Write)
                        Dim sw As StreamWriter = New StreamWriter(fs)
                        sw.WriteLine(DateTime.Now.ToString())
                        sw.WriteLine(ex.ToString())
                        sw.Close()
                        fs.Close()
                    Catch
                    End Try
    
                    Return False
                End Try
    
                Return True
            End Function
    
        End Class
    End Namespace
{% endhighlight %}    

**Use the SharePoint Library**:  
After developing my own control, I discovered that there was an even easier way to accomplish the same thing.  I found out that Microsoft ships a (probably similar) control as part of each Office installation starting with Office 2000.  This control seems to work pretty much the same as my version, except it doesn’t implement IObjectSafety.   The advantage is that you do not have to deploy any kind of client-side control because it is already on every machine with Office!  In many cases, even though it is not as nice for your users (who will have to click ‘Yes’ on a dialog every time), it is often the only viable option in an enterprise (which often won’t allow you to push ActiveX/COM controls out to users).

As you can see above, I don’t have any specialized logic for opening these applications or files, so using the library that ships with Office is great for me.  I implemented this so that it tries to use each version of the library (there is a different version for each Office version…) in turn.

{% highlight js %}
    function webedit(id) {
		if (window.ActiveXObject) {
			var ed;
			try {
				//Office 2003
				ed = new ActiveXObject('SharePoint.OpenDocuments.2');
			} catch (err1) {
				try {
					//Office 2000/XP
					ed = new ActiveXObject('SharePoint.OpenDocuments.1');
				} catch (err2) {
					try {
						//Office 2007
						ed = new ActiveXObject('SharePoint.OpenDocuments.3');
					} catch (err3) {
						window.alert('Unable to create an ActiveX object to open the document. This is most likely because of the security settings for your browser.');
						return false;
					}
				}
			}
			if (ed) {
				ed.EditDocument('/webdav/--'+id);
				return false;
			} else {
				window.alert('Cannot instantiate the required ActiveX control to open the document. This is most likely because you do not have Office installed or you have an older version of Office.');
				return false;
			}
		} else {
			window.alert('Internet Explorer is required to use this feature.');
		}
		return false;
	}
{% endhighlight %}    

**Provide only the Link**:  
As a last resort, you can always provide/display a link for the user to open manually.  While this sounds silly, users still gain the benefit of WebDAV actions from within their local apps, meaning they still save directly against the repository.  This can be helpful for people who work on a few files a day for a longer period of time rather than those who open lots and lots of documents daily.

 [1]: 2009-09-21-simple-webdav-part-1 "Simple WebDAV Part 1"  