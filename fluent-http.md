---
layout: default
---

##Fluent HTTP

Flurl allows you to perform many common HTTP tasks directly off the fluent URL builder chain. Barely under the hood is [HttpClient](http://blogs.msdn.com/b/henrikn/archive/2012/02/11/httpclient-is-here.aspx) and related classes. As you'll see, Flurl enhances HttpClient with convenience methods and fluent goodness but doesn't try to abstract it away completely.

````c#
using Flurl;
using Flurl.Http;
````

Get a strongly-typed poco from a JSON API:

````c#
T poco = await "http://api.foo.com".GetJsonAsync<T>();
````

The non-generic version returns a dynamic:

````c#
dynamic d = await "http://api.foo.com".GetJsonAsync();
````

Get a string:

````c#
string s = await "http://api.foo.com".GetStringAsync();
````

Download a file:

````c#
// filename is optional here; it will default to the remote file name
var path = await "http://files.foo.com/image.jpg"
    .DownloadFileAsync("c:\\downloads", filename);
````

Post some JSON data:

````c#
await "http://api.foo.com".PostJsonAsync(new { a = 1, b = 2 });
````

Simulate an HTML form post:

````c#
await "http://api.foo.com".PostUrlEncodedAsync(new { a = 1, b = 2 });
````

The Post methods above return a `Task<HttpResponseMessage>`. You may of course expect some data to be returned in the response body:

````c#
T poco = await url.PostJsonAsync(data).ReceiveJson<T>();
dynamic d = await url.PostUrlEncodedAsync(data).ReceiveJson();
string s = await url.PostUrlEncodedAsync(data).ReceiveString();
````

###Configuring Client Calls

There are a variety of ways to set up HTTP calls without breaking the fluent chain.

Set request headers:

````c#
// one-off:
await url.WithHeader("someheader", "foo").GetJson();
// multiple:
await url.WithHeaders(new { header1 = "foo", header2 = "bar" }}).GetJson();
````

Authenticate with Basic Authentication:

````c#
await url.WithBasicAuth("username", "password").GetJson();
````

Authenticate with an OAuth bearer token:

````c#
await url.WithBasicAuth("username", "password").GetJson();
````

Specify a timeout:

````c#
await url.WithTimeout(10).DownloadFile(); // 10 seconds
await url.WithTimeout(TimeSpan.FromMinutes(2)).DownloadFile();
````

Get at the underlying HttpClient directly if none of the other convenience methods meet your needs:

````c#
await url.ConfigureHttpClient(http => /* do whatever you need! */).GetJson();
````

`ConfigureHttpClient` is a nice hook for extending Flurl with your own extension methods.

###Exceptions

Flurl deviates from HttpClient conventions a bit when it comes to excpetions. HttpClient doesn't throw exceptions upon receiving unsuccessful HTTP response codes; Flurl does.

````c#
try {
    T poco = await "http://api.foo.com".GetJsonAsync<T>();
}
catch (FlurlHttpTimeoutException) {
    LogError("Timed out!");
}
catch (FlurlHttpException ex) {
    if (ex.Call.Response != null)
        LogError("Failed with response code " + call.Response.StatusCode);
    else
        LogError("Totally failed before getting a response! " + ex.Message);
}
````

<a name="httpcall"></a>`FlurlHttpException.Call` is an instance of `HttpCall`, which wraps a variety of helpful diagnostic information:

````c#
public class HttpCall
{
    public HttpRequestMessage Request { get; set; }
    public string RequestBody { get; set; }
    public HttpResponseMessage Response { get; set; }
    public DateTime StartedUtc { get; set; }
    public DateTime? EndedUtc { get; set; }
    public TimeSpan? Duration { get; set; }
    public Exception Exception { get; set; }
    public bool ExceptionHandled { get; set; }
}
````
`RequestBody` is particularly useful in POST and PUT scenarios to work around the [forward-only, read-once nature](http://stackoverflow.com/questions/12102879/httprequestmessage-content-is-lost-when-it-is-read-in-a-logging-delegatinghandle) of `HttpRequestMessage.Content`. It is important to do a null check on `RequestBody` however, as it is typically only populated when using Flurl's Post* and Put* methods.

(Note: `HttpCall` is also used in global callbacks and testing. In the context of a `FlurlHttpException`, `Exception` and `ExceptionHandled` isn't very useful, as you already have the exception and are already handling it.)
