---
layout: post
title: One-way Hashing for C# and Java
categories: [.net, java]
---

I was recently faced with the need to create some sort of verification measure for URLs.  Basically, public users would be able to access an internal system using a generated URL, and I needed to make sure that these links were very difficult to *guess* or even *reverse-engineer*.

I decided to use an SHA1 hash, which creates a reproducible series of bytes that can then be authenticated by comparison at a later time.  This is a pretty common way of checking to make sure your message has not been tampered with and, while it can be hacked, it is very, very difficult (and time-consuming) to do so.  For my use, it is a reasonable solution.

The trick here is that I needed to generate the hash in Java and then check it in C#.  This sounds easy (and it is), but I did spend some time tracking down how to get **the same output** from these two algorithms when converting back to a string.  Unfortunately,  you can’t just pass the bytes around in a URL.

Let’s take a look at the Java version:

{% highlight java %}
    public static String generateSHA1Hash(String input) throws Exception {

		MessageDigest messagedigest = MessageDigest.getInstance("SHA");
		messagedigest.update(input.getBytes("UTF-8"));
		byte digest[] = messagedigest.digest();

		StringBuffer s = new StringBuffer(digest.length * 2);
		int length = digest.length;
		for (int n = 0; n &lt; length; n++) {
			int number = digest[n];
			number = (number &lt; 0) ? (number + 256) : number;
			s.append(Integer.toString(number, 16));
		}

		return s.toString();
	}
{% endhighlight %}      

The input is an ID string plus a *salt *(another, static string).  You need the salt in order to prevent someone from guessing how the hash  is generated.  The key is that second part which is taking each byte and converting it into a character (so it can be easily displayed and passed around).  This is where the difficulty presented itself.  I could get the same output in byte form, but the character conversion was funky.  I feel like I am probably doing something wrong, because I had to actually just drop any space ” ” characters in the C# version:

{% highlight c# %}
    private bool checkHash(message) {

		SHA1 sha = new SHA1Managed();
		UTF8Encoding ue = new UTF8Encoding();
		byte[] data = ue.GetBytes(message);
		byte[] digest = sha.ComputeHash(data);

		StringBuilder newHash = new StringBuilder();
		int length = digest.Length;
		for (int n = 0; n &lt; length; n++) {
			//remove spaces to match java
			newHash.Append(String.Format("{0,2:x}", digest[n]).Replace(" ", String.Empty));
		}

		return (docReq.Hash.Equals(newHash.ToString()))
	}
{% endhighlight %}      

In this example, message is the incoming text to hash (in this case, the ID string along with the *same salt* used in generating the original), and docReq.Hash is the existing hash to compare against.  As you can see, the code is not quite the same (but very similar).  Can anyone point out the mistake I made and tell me why the C# version produces ” ” characters, but the Java version does not?

Either way, these **do** produce the same comparable result.  Just remember to *salt *your inputs!  