---
layout:     post
title:      "A hack to expose RESTful services via SOAP"
date:       2017-02-08 07:12:47
summary:    "Use SOAP as an invisible transport layer for accessing REST API"
categories: hacks services rest soap
---

Working in a corporation you sometimes face constraints that would seem strange and artificial to someone working in a small company.
However, overcoming these constraints is both challenging and fun.

One of such cases is described below.

## The challenge
We were building a new REST API.
A good part of it was already done when we discovered a limitation in provided infrastructure: our Enterprise Service Bus (ESB) could only expose SOAP services.

REST support was promised to come soon, but we had to expose our functionality to the clients before that.

The easy solution would have been to build a throwaway SOAP API and ask our clients to upgrade to REST later.
However, adding an unwelcome SOAP API did not feel right.

## The solution

What felt right was solving this challenge in a generic way that would require almost no extra work on both service provider and service consumer sides.

The idea was to use SOAP messages for transferring request and response data, yet make them invisible to both service providers and consumers.

If direct access to a REST API looked like this:

![Diagram of REST]({{ site.url }}/assets/rest.png)
{: .center}

Then this is the *REST via SOAP* solution:

![Diagram of REST via SOAP]({{ site.url }}/assets/rest-via-soap.png)
{: .center}

#### How does it work?

Let's say we have a REST API that allows its clients to create Greeting instances.

1. The client then sends a `POST` request with `{ "content" : "Hello!" }` body to a remote `/api/greetings` endpoint.
1. *REST Interceptor* captures this outgoing HTTP request, transforms it into a SOAP request seen below and sends it to a remote *SOAP Proxy for REST API* (e.g. `/soap-api`).
{% highlight xml %}
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns="http://g.rimsa.lt/rest-via-soap/" >
  <soapenv:Header />
  <soapenv:Body>
    <request method="POST" path="/api/greetings">
      { "content" : "Hello!" }
    </request>
  </soapenv:Body>
</soapenv:Envelope>
{% endhighlight %}

{:start="3"}
1. *SOAP Proxy* parses the incoming SOAP message, rebuilds the REST request and forwards it to the *REST API*. 
1. *REST API* responds with HTTP status `200` and response body `{ "id" : 123 }`.
1. *SOAP Proxy* wraps this response into a SOAP message:
{% highlight xml %}
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/" xmlns="http://g.rimsa.lt/rest-via-soap/">
  <SOAP-ENV:Header/>
  <SOAP-ENV:Body>
    <response status="200">
      { "id" : 123 }
    </response>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
{% endhighlight %}

{:start="6"}
1. *REST Interceptor* unwraps it and the client receives a regular HTTP response with status `200` and `{ "id" : 123 }` as the body, same as via a direct call to *REST API*.


## The implementation
When this *REST via SOAP* idea was born, I set out to write a generic Servlet Filter to act as the *SOAP Proxy for REST API*.

It turned out well and you can now find the code in [*REST via SOAP* project on GitHub](https://github.com/grimsa/rest-via-soap).

However, as we ended up not needing it in the project we were doing at the time, it was not tested in a real-world setting and the *REST Interceptor* part remains to be implemented.