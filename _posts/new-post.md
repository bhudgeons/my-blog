title: Handling Non-Standard URLs in Dispatch
date: 2013-28-06
author: Brandon Hudgeons
tags: [scala, ]

#Handling Non-Standard URLs in Dispatch
##a code adventure

*tl;dr: Need to use Dispatch 0.10.1 to access a non-standard URL? Use this custom async-http-client RequestBuilder to set your URL the way you want it.*

Let's say that we're using the excellent [Dispatch](http://dispatch.databinder.net/Dispatch.html) library (this post uses 0.10.1) for our HTTP client to access a web service from SillyCo, and there are no alternatives to SillyCo.  And, let's say that documentation for SillyCo's web service specifies a URL that looks something like this:

`http://api.sillyco.com/service?blue&yellow`

Notice anything wrong with that URL?  If not, you will in a minute.

So, we fire up Dispatch and make our stuff like the docs tell us to:

	import dispatch._, Defaults._
	val svc = url("http://api.sillyco.com/service?blue&yellow")
	val silly = Http(svc OK as.String)
	
But when we execute the call, we get a response that we would have gotten for `http://api.sillyco.com/service`, without the query string.  It's like we didn't even ask for blue and yellow.  What happened?

Let's see:

	scala> import dispatch._, Defaults._
	import dispatch._
	import Defaults._
	
	scala> val svc = url("http://api.sillyco.com/service?blue&yellow")
	svc: com.ning.http.client.RequestBuilder = com.ning.http.client.RequestBuilder@62831806
	
	scala> svc.url
	res0: String = http://api.sillyco.com/service?blue=&yellow=
	
Ah hah!  Do you see it?

Our original URL query string was `?blue&yellow`, which is not a valid parameter string.  Dispatch helpfully tries to fix it for us, turning it into `?blue=&yellow=`.  Unfortunately, SillyCo's API doesn't like this.

So, we call SillyCo and inform them that their API documentation requires a non-standard URL query string.  The reply: "Huh.  Interesting."  We ask if there's a way to access the API using a valid URL.  "Um, no."  We ask them if they will fix this.  "Um, no."

Okay.  Surely there's a way that we can fool Dispatch into using the bad URL.  Let's try.

	scala> def sillyHost = host("api.sillyco.com")
	sillyHost: com.ning.http.client.RequestBuilder
	
	scala> def myRequest = sillyHost / "service?blue&yellow"
	myRequest: com.ning.http.client.RequestBuilder
	
	scala> myRequest.url
	res1: String = http://api.sillyco.com/service%3Fblue&yellow

Bah!  It URL-encoded the '?'.  Of course, SillyCo's API doesn't like that either.  Keep trying…

	scala> def myRequestWithParams = myRequest.addQueryParameter("blue").addQueryParameter("yellow")
	<console>:15: error: not enough arguments for method addQueryParameter: (x$1: String, x$2: String)com.ning.http.client.RequestBuilder.
	Unspecified value parameter x$2.
       def myRequestWithParams = myRequest.addQueryParameter("blue").addQueryParameter("yellow")
       
	scala> def myRequestWithParams = myRequest.addQueryParameter("blue","").addQueryParameter("yellow","")
	myRequestWithParams: com.ning.http.client.RequestBuilder

	scala> myRequestWithParams.url
	res2: String = http://api.sillyco.com/service?blue=&yellow=

	scala> def myRequestWithParams = myRequest.addQueryParameter("blue",null).addQueryParameter("yellow",null)
	myRequestWithParams: com.ning.http.client.RequestBuilder
	
	scala> myRequestWithParams.url
	res3: String = http://api.sillyco.com/service?blue=&yellow=

	scala> def myRequestWithParams = myRequest <<? Map("blue" -> null, "yellow" -> null)
	myRequestWithParams: com.ning.http.client.RequestBuilder

	scala> myRequestWithParams.url
	res4: String = http://api.sillyco.com/service?blue=&yellow=

Hmmm… all of those calls return type `com.ning.http.client.RequestBuilder`.  Maybe we can instantiate our own RequestBuilder and set the URL directly, instead of letting Dispatch do it for us, and then it will work.

	scala> import com.ning.http.client.RequestBuilder
	import com.ning.http.client.RequestBuilder
	
	scala> def requestBuilder = new RequestBuilder()
	requestBuilder: com.ning.http.client.RequestBuilder
	
	scala> val myRequest = requestBuilder.setUrl("http://api.sillyco.com/service?blue&yellow")
	myRequest: com.ning.http.client.RequestBuilder = com.ning.http.client.RequestBuilder@77c4fb91
	
	scala> myRequest.url
	res5: String = http://api.sillyco.com/service?blue=&yellow=

BAAAAAAAH!  OK, now at least we know that it isn't [@n8han](https://github.com/n8han) doing this to us.  Let's just look at how RequestBuilder sets the URL … then we'll just override that functionality and set the URLs how we want.  

We figure out that our version of Dispatch uses [async-http-client](https://github.com/AsyncHttpClient/async-http-client) version 1.7.11, so the code for that version of RequestBuilder actually relies on RequestBuilderBase, the 1.7.11 code for which is [here](https://github.com/AsyncHttpClient/async-http-client/blob/776ec568065197cf31995ff7eb833ae37aca1663/src/main/java/com/ning/http/client/RequestBuilderBase.java).  We find the setUrl method.  It looks like this: (java)

    public T setUrl(String url) {
        request.url = buildUrl(url);
        return derived.cast(this);
    }
	    
Ha!  A quick look at the code shows that it is the buildUrl method that is doing all of the URL parameter stuff!  Let's override it!  Here's what we need to override:

	private String buildUrl(String url) {
	...

BAAAAAAAAAAAH! Damn `private`.  Okay, we can't override that one, so let's override setURL instead.

	scala> val requestBuilder = new RequestBuilder {
	     | override def setUrl(url:String) = {
	     | 	val toreturn = super.setUrl(url) // get the derived.cast
	     | 	request.url = url // skip the buildUrl method
	     | 	toreturn // return the derived.cast thing
	     | 	}
	     | }
	<console>:20: error: value url is not a member of com.ning.http.client.RequestBuilderBase.RequestImpl
	       request.url = url // skip the buildUrl method


What?  Let's look at the implementation of RequestImpl in use here:

	private static final class RequestImpl implements Request {
	        private String method;
	        private String url = null;
	        . . .
	        
So, `RequestImpl` is a `private static final class`, and `request.url` is a `private` instance variable.  We are screwed, but we are also bull-headed, and we are not giving up.

Let's just re-implement our own whole damn RequestBuilder object and re-write the setUrl method the way we want to.  Moreover, since we don't really want to add a Java compilation step just for this one class, let's port the thing to Scala.

The full code for the class is here.  I've re-written the setUrl method like so:

	  def setUrl[T](url: String) = {
	    myrequest.asInstanceOf[MyRequestImpl].setUrl(url) // just set the URL like we passed it in. Skip buildUrl!
	    this
	  }

And now:

	scala> import at.atsoft.util.MyRequestBuilder
	import at.atsoft.util.MyRequestBuilder
	
	scala> val requestBuilder = new MyRequestBuilder
	requestBuilder: at.atsoft.util.MyRequestBuilder = at.atsoft.util.MyRequestBuilder@1d5551a8

	scala> val myRequest = requestBuilder.setUrl("http://api.sillyco.com/service?blue&yellow")
	myRequest: at.atsoft.util.MyRequestBuilder = at.atsoft.util.MyRequestBuilder@1d5551a8
	
	scala> myRequest.url
	res0: String = http://api.sillyco.com/service?blue&yellow
	
BWAAAAHAHAHAHAHA!!!

And now:

	val silly = Http(myRequest.GET OK as.String)

WORKS.  

It is time for a scotch.
