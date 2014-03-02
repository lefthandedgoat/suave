Introduction.
=============

Suave is inspired in the simplicity of Happstack and born out of the necessity of embedding web server capabilities in my own applications. 
Still in its early stages Suave supports HTTPS, multiple TCP/IP bindings, Basic Access Authentication, Keep-Alive and HTTP compression.

Suave also takes advantage of F# asynchronous workflows to perform non-blocking IO. In fact, Suave is written in a **completely non-blocking** fashion throughout. Suave **runs on Linux**, OS X and Windows flawlessly.

What follows is a tutorial on how to create Suave applications. Scroll past the tutorial to see detailed function documentation.

NuGet
-----

To install Suave, run the following command in the [Package Manager Console](http://docs.nuget.org/docs/start-here/using-the-package-manager-console)

{% highlight dosbatch %}
PM> Install-Package Suave
{% endhighlight %}

Tutorial: Hello World!
----------------------

The simplest Suave application is a simple HTTP server that greets all visitors with the string `"Hello World!"`

{% highlight fsharp %}
web_server default_config (OK "Hello World!")
{% endhighlight %}

The above statement will start a web server on default port 8083. `web_server` takes a configuration record and the webpart `(OK "Hello World")` 

Webparts are functions with the following type: `HttpContext -> Option<Async<unit>>`. For every request `web_server` will evaluate the webpart, if the evaluation succeeds it will execute the resulting asynchronous computation. `OK` is a combinator that always succeed and writes its argument to the underlying response stream.

Tutorial: Composing bigger programs.
------------------------------------

Logic is expressed with the help of different combinators built around the `option<HttpContext>` type. We build webparts out of functions of type `HttpContext -> HttpContext option` and the operator `>>=` in the following way.

{% highlight fsharp %}
let simple_app _ = url "/hello" >>= OK "Hello World" ;
{% endhighlight %}

To select between different routes or options we use the function choose; for example:

{% highlight fsharp %}
let complex_app _ = 
  choose
    [ Console.OpenStandardOutput() |> log >>= never
    ; url "/hello" >>= never >>= OK "Never executes"
    ; url "/hello" >>= OK "Hello World" ]
{% endhighlight %}

The function `choose` accepts a list of webparts and execute each webpart in the list until one returns success. Since choose itself returns a webpart we can nest them for more complex logic.

{% highlight fsharp %}
let nested_logic _ =
  choose
    [ GET >>= choose 
        [ url "/hello" >>= OK "Hello GET"
        ; url "/goodbye" >>= OK "Good bye GET" ]
    ; POST >>= choose 
        [ url "/hello" >>= OK "Hello POST"
        ; url "/goodbye" >>= OK "Good bye POST" ] ]
{% endhighlight %}

To gain access to the underlying `HttpRequest` and read query and http form data we can use the `request` combinator.

{% highlight fsharp %}
let http_form _ = 
  choose
    [ GET  >>= url "/query" >>= request(fun x -> OK ("Hello " + (opt <| x.query ? name)))
    ; POST >>= url "/query" >>= request(fun x -> OK ("Hello " + (opt <| x.form  ? name)))
    ; NOT_FOUND "Found no handlers" ]
{% endhighlight %}

To protect a route with HTTP Basic Authentication the combinator `authenticate_basic` is used like in the following example.

{% highlight fsharp %}
let requires_authentication _ = 
  choose
    [ GET >>= url "/public" >>= OK "Hello anonymous"
    //access to handlers bellow this one will require authentication
    ; authenticate_basic (fun x -> x.Username.Equals("foo") && x.Password.Equals("bar"))
    ; GET >>= url "/protected" >>= request(fun x -> OK ("Hello " + x.Username)) ]
{% endhighlight %}

Typed routes
------------

{% highlight fsharp %}
let testapp : WebPart = 
  choose
    [ url_scan "/add/%d/%d" (fun (a,b) -> OK((a + b).ToString()))
    ; NOT_FOUND "Found no handlers" ]
{% endhighlight %}

Multiple bindings and SSL support
---------------------------------

Suave supports binding the application to multiple TCP/IP addresses and ports combinations. It also supports HTTPS.

{% highlight fsharp %}
let sslCert = new X509Certificate("suave.pfx","easy");
let cfg = 
  { default_config with
      bindings =
        [ { scheme = HTTP
          ; ip     = IPAddress.Parse "127.0.0.1"
          ; port   = 80us }
        ; { scheme = HTTPS sslCert
          ; ip     = IPAddress.Parse "192.168.13.138"
          ; port   = 443us } ]
    ; timeout = TimeSpan.FromMilliseconds 3000. }
choose 
  [ Console.OpenStandardOutput() |> log >>= never // log to standard output
  ; url "/hello" >>= OK "Hello World"
  ; NOT_FOUND "Found no handlers" ]
|> web_server cfg
{% endhighlight %}

API
===

Default-supported HTTP Verbs
----------------------------

See "RFC 2616":http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html.

These applicatives match on HTTP verbs.

{% highlight fsharp %}
let GET     (x : HttpRequest) = meth0d "GET" x
let POST    (x : HttpRequest) = meth0d "POST" x
let DELETE  (x : HttpRequest) = meth0d "DELETE" x
let PUT     (x : HttpRequest) = meth0d "PUT" x
let HEAD    (x : HttpRequest) = meth0d "HEAD" x
let CONNECT (x : HttpRequest) = meth0d "CONNECT" x
let PATCH   (x : HttpRequest) = meth0d "PATCH" x
let TRACE   (x : HttpRequest) = meth0d "TRACE" x
let OPTIONS (x : HttpRequest) = meth0d "OPTIONS" x
{% endhighlight %}

Server configuration
--------------------

The first argument to `web_server` is a configuration record with the following signature.

{% highlight fsharp %}
/// The core configuration of suave. See also Suave.Web.default_config which
/// you can use to bootstrap the configuration:
/// <code>{ default_config with bindings = [ ... ] }</code>
type SuaveConfig =
  /// The bindings for the web server to launch with
  { bindings       : HttpBinding list

  /// An error handler to use for handling exceptions that are
  /// are thrown from the web parts
  ; error_handler  : ErrorHandler

  /// Timeout for responses to be generated
  ; timeout        : TimeSpan

  /// Timeout to wait for the socket bind to finish
  ; listen_timeout : TimeSpan

  /// A cancellation token for the web server. Signalling this token
  /// means that the web server shuts down
  ; ct             : CancellationToken

  /// Buffer size used for socket IO, default value is 8192 (8 Kilobytes)
  ; buffer_size    : int

  /// Maximun number of accept, receive and send operations the server can handle simultaneously
  /// default value is 100
  ; max_ops        : int

  /// MIME types, registry of file extensions Suave can serve and if they can be compressed
  ; mime_types_map : string -> MimeType option

  /// Home or root directory, the directory where content will be served from
  /// if no directory is specified the directory where suave.dll lives is assumed the home directory
  ; home_folder    : string option

  /// Folder for temporary compressed files, directory where to cache compressed static files
  /// if no directory is specified the directory where suave.dll lives is assumed
  ; compressed_files_folder : string option }
{% endhighlight %}

With `Protocol` , `HttpBinding` and `MimeType` defined like follows:

{% highlight fsharp %}
/// Gets the supported protocols, HTTP and HTTPS with a certificate
type Protocol =
  /// The HTTP protocol is the core protocol
  | HTTP
  /// The HTTP protocol tunneled in a TLS tunnel
  | HTTPS of X509Certificate

/// A HTTP binding is a protocol is the product of HTTP or HTTP, a DNS or IP binding 
/// and a port number
type HttpBinding =
  /// The scheme in use
  { scheme : Protocol
  /// The host or IP address to bind to. This will be interpreted by the operating system
  ; ip     : IPAddress
  /// The port for the binding
  ; port   : uint16 }

type MimeType = 
  /// The name of the mime type, i.e "text/plain"
  { name         : string
  /// If the server will compress the file when clients ask for gzip or 
  /// deflate in the `Accept-Encoding` header
  ; compression  : bool }
{% endhighlight %}

## Serving static files, HTTP Compression and MIME types

Suave supports **gzip** and **deflate** http compression encodings.  Http compression is configured via the MIME types map in the server configuration record. By default Suave does not serve files with extensions not registered in the mime types map.

The default mime types map `default_mime_types_map` looks like this.

{% highlight fsharp %}
let default_mime_types_map = function
    | ".css" -> mk_mime_type "text/css" true
    | ".gif" -> mk_mime_type "image/gif" false
    | ".png" -> mk_mime_type "image/png" false
    | ".htm"
    | ".html" -> mk_mime_type "text/html" true
    | ".jpe"
    | ".jpeg"
    | ".jpg" -> mk_mime_type "image/jpeg" false
    | ".js"  -> mk_mime_type "application/x-javascript" true
    | _      -> None
{% endhighlight %}

You can register additional MIME extensions by creating a new mime map in the following fashion.

{% highlight fsharp %}
// Adds a new mime type to the default map
let mime_types x =
  default_mime_types_map 
    >=> (function | ".avi" -> mk_mime_type "video/avi" false | _ -> None)
{% endhighlight %}



## Overview

A request life-cycle begins with the `HttpProcessor` that takes an `HttpRequest`
and the request as bytes and starts parsing it. It returns an `HttpRequest
option` that, if Some, gets run against the WebParts passed.

### The WebPart

A web part is a thing that executes on a HttpContext, asynchronously, maybe executing
on the request.

{% highlight fsharp %}
type WebPart = HttpContext -> Async<unit> option
{% endhighlight %}

### The ErrorHandler

An error handler takes the exception, a programmer-provided message, a request
(that failed) and returns an asynchronous workflow for the handling of the
error.

{% highlight fsharp %}
type ErrorHandler = Exception -> String -> HttpContext -> Async<unit>
{% endhighlight %}
