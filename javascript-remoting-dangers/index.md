---
title: JavaScript Remoting Dangers
author: petko-d-petkov
guest: billy-hoffman
date: Tue, 30 Jan 2007 21:31:08 GMT
template: this/views/post.jade
category: fucked
---

> This month our guest blogger is [Billy Hoffman](http://www.memestreams.net/users/Acidus), the lead researcher at [SPI Dynamics](http://www.spidynamics.com/) and author of [StripeSnoop](http://stripesnoop.sourceforge.net/), an application which analyzes data on magnetic stripes. Billy has been a frequent speaker at industry events including Interz0ne, Toorcon, Black Hat Federal, PhreakNIC, FooCamp, O'Reilly Media Emerging Technology Conference and ShmooCon. In this post Billy is taking us on a quite interesting trip into the dangerous world of JavaScript remoting.

I'd like to thank pdp for giving me the opportunity to write a blog post. I'd like to use this post to discuss the various methods JavaScript can use to make HTTP requests. Each method has its own pros and cons that lend themselves to be used in different situations. We will ignore using JavaScript coupled with extra technologies such as Java applets or Flash and focus entirely on native JavaScript objects (or objects provided through ActiveX) and the DOM environment. I'm sure there are others and I hope this post can serve as a starting point for a larger project to collect information and techniques about how JavaScript is used to perform various malware tasks like propagating worms and instigating data theft.

One method JavaScript can use to send HTTP requests is creating HTML tags using the `document.createElement` function. There are all sorts of HTML tags/attributes pairs that will issue an HTTP request when rendered by the browser. There are the obvious ones like IMG+src, but also obscure ones like TABLE+background or INPUT+src. All of these tags issue a so-called "blind" request, meaning that JavaScript is not capable of seeing the response. This method can also be used only to issue GET requests. This makes the method is a good way to send information in the query string one way to a 3rd party such as a 3rd party site to collect stolen cookies. However, blind GETs are limited in the amount of data they can send to a 3rd party through the query string by the allowed length of a URL. While there are no explicit limits defined in any RFC, anything more than 2K to 4K is pushing it. This method can also be used as an XSRF attack vector. JavaScript could be used with a timer to create multiple blind GET requests that are necessary to walk through a multiple stage process such as a money transfer.

JavaScript can also use the `document.createElement` method to dynamically create FORM and INPUT tags and use them to send POSTs to arbitrary domains. If the domain the JavaScript is POSTing to is the same as the domain it came from, JavaScript will be able to access the response. This is a handy method to use when you need JavaScript to send a very large amount of data to a 3rd party that would exceed the amount of data you can safely put in the query string of a URL.

In addition to using HTML tags, JavaScript has a native object which can issue HTTP requests. The Image object is supplied by the DOM environment and allows JavaScript to dynamically request images from any web server. As soon as you set the src property on the image object, a blind GET is issued to the URL you assigned. This tends to be a better method to send blind GETs than creating HTML tags because you avoid the overhead of instantiating DOM elements. Also, when using an Image object the response is not entirely blind. Events can be set to trap when the image has finished loading and what the size of the image is. This creates a side channel for JavaScript to communicate with certain 3rd party hosts using the dimensions of the image. In practice, XBM images tend to work best because you can specify arbitrary lengths and widths up to a 15bit integer without actually needing an image of that size. You can also use the time domain as a side channel by varying when an image loads. Both these methods have several limitations and it is better to use an alternate method such as remote scripting to exchange certain information between 3rd party sites that the JavaScript author controls.

Finally, there are IFRAME remoting and XmlHttpRequest. These methods are interesting because they allow JavaScript to access the response (if sent to the domain the JavaScript came from). Specifically, they allow JavaScript to proactively fetch pages asynchronously and in a hidden fashion. Because the browser will automatically add the appropriate cookies and HTTP authentication info to the requests (as is does for all requests), these methods are very useful for malware authors to actively steal specific data from a user that is stored on a website.

So which method to use? Many people believe XmlHttpRequest doesn't help XSS malware authors. They argue attacker have already been able to proactively fetch content using IFRAME remoting.

There are several problems with IFRAME remoting which XmlHttpRequest solves. First of all, IFRAMEs were never designed to be used in the fashion, making this a fairly clunky procedure. Remember that an attacker doesn't have control of the content that's been returned by the IFRAME. This means that the response that populates the IFRAME is not going to contain JavaScript which will execute a callback function in the attacker's payload which is inside of the parent document for the IFRAME. The response that is in the IFRAME is simply whatever information the attacker is trying to steal. To extract any data from the IFRAME an attacker would create the IFRAME and then hook the onload event, as shown in the following code:

```html
var iframe = document.createElement('IFRAME');
//have the IFRAME download the content we want to steal
iframe.src = 'http://site.com/AddressBook.php);
//make the IFRAME invisible
iframe.style="width:0px; height:0px; border: 0px"
//set our function to call when the IFRAME is done loading
iframe.onload = callbackFunction;
//now add the IFRAME to the DOM.
Document.body.appendChild(iframe);
```

In the above code, the attacker is dynamically creating an IFRAME whose SRC attribute points to AddressBook.php. AddressBook.php is a web page which contains all the email addresses in a user's address book. The attacker also styles the IFRAME sothat it takes up no visible space and does not have a border surrounding it. This styling renders the IFRAME invisible to the user.

So how does the attacker access the data they wish to steal? As we noted before the markup of AddressBook.php is not going to contain code that would notify the attacker or pass the contents of AddressBook.php to the attacker. This only makes sense as AddressBook.php is not accessed in this way when the application is functioning normally. Instead, the attacker has to register a JavaScript function to execute once the IFRAME has finished loading the content specified in its SRC attribute. Here lies the problem. The attacker is only interested in stealing the email addresses. These email addresses will appear as text inside of the HTML that is returned by AddressBook.php. The HTML will also link to images, external style sheets, JavaScript files, Flash objects and other resources that are so common place on today's Internet. Unfortunately for the attacker, the IFRAME does not call the function specified in its onload attribute until after all the external resources referenced by the document in the IFRAME have been downloaded and instantiated.

How bad is this for the attacker? Consider CNN's website http://www.cnn.com/. Using the View Dependencies extension for Firefox, we see that to display CNN fully, an astonishing 363 kilobytes (KB) of data must be downloaded to a user's machine. Only 23 KB of this, about 6% of the total data, is the HTML representing the text on the web page. Since the attacker is trying to extract text they care about downloading the HTML. However, because of the way IFRAMEs and the onload event work, the entire page must be downloaded before the attacker can extract data. Let's put this in prospective. Downloading 363KB of data over a 1 Mbps connection takes approximately 3 seconds. Downloading 23KB over the same link takes 0.17 seconds, or is 15 times faster. While 3 seconds may not seem like a whole lot of time you should focus on the 15 times figure. In this scenario, an attacker could request and siphon data from 15 pages using the XmlHttpRequest method for every 1 page retrieved using IFRAME.

In the interest of fairness, it should be noted that the entire 363 KB of CNN are not downloaded each and every time. CNN implements caching to ensure that the certain files do not need to be requested again. However, the browser does still send conditional GET requests. That is, the browser sends a complete GET request which includes an HTTP header telling CNN to return only the resource if it is newer than the version the browser has cached locally. Even if the browser's resource is still valid, the server must respond with an HTTP 304 message. This tells the browser to use its local copy. Even if all the local copies of the resources are fresh, some amount of network traffic still has to occur. From the attacker's point of view all of this traffic is a waste because they cannot use it and don't care about it. The bottom line is using IFRAME remoting to extract content from a web page is always slower than using an XmlHttpRequest. Thus, while IFRAME remoting can be used to siphon confidential data from a user without their knowledge, XmlHttpRequest makes this a much more realistic attack vector. Add on the fact the XmlHttpRequest allows access to response headersa and (some) modification of the request headers and XmlHttpRequest is the clear winner over IFRAME remoting for an attackers toolkit when used for data theft. This may be why nearly all the XSS malware to date uses XmlHttpRequest instead of IFRAME remoting to propagate.

Remember that JavaScript has many different ways to send HTTP requests. If you are in a situation where some are not allowed or filtered, chances are there is another method you can use. Even if all methods are explicated filtered, JavaScript is so dynamic you can probably get around it. For example, if the function `document.createElement` cannot be used, assemble your function call character by character in a string and then `eval` it. If `eval` is not allowed, use a `setTimeout`, which can also eval strings. JavaScript is far more powerful and people are always underestimating it. Much like Perl, there are many way to do the same thing.

_Thanks again to pdp for providing this chance, and happy hacking._

To help you out, here is a table summarizing the pros and cons for the different methods JavaScript can use to send HTTP requests.

<table>
<thead>
<tr>
<th>Method</th><th>HTTP methods</th><th>Can access response</th><th>Can see response headers</th><th>Can communicate with any domain</th>
</tr>
</thead>
<tbody>
<tr>
<td>Dynamically created HTML Tags</td><td>GET</td><td>No</td><td>No</td><td>Yes</td>
</tr>
<tr>
<td>Dynamically built FORM tag</td><td>GET, POST</td><td>Only if FORM posts to same domain as JavaScript code</td><td>No</td><td>Yes</td>
</tr>
<tr>
<td>Image Object</td><td>GET</td><td>Yes, but only image dimensions</td><td>No</td><td>Yes</td>
</tr>
<tr>
<td>Remote Scripting</td><td>GET</td><td>Yes, if response is JavaScript</td><td>No</td><td>Yes</td>
</tr>
<tr>
<td>iFrame Remoting</td><td>GET</td><td>Only if iframe src is in same domain as JavaScript code</td><td>No</td><td>Yes</td>
</tr>
<tr>
<td>XmlHttpRequest Object</td><td>Any</td><td>Yes</td><td>Yes</td><td>No</td>
</tr>
</tbody>
</table>
