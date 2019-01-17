+++
title = "Proxy Fundamentals"
docpage = true
[menu.l5d2docs]
  name = "Proxy Fundamentals"
  weight = 2
+++

As Linkerd's proxy layer is configured automatically by the control plane,
detailed knowledge of the proxy's internals is not necessary to use and
operate it. However, a basic understanding of the high-level level principles
behind the proxy can be valuable for avoiding some pitfalls. This document will
highlight some of the core principles behind the proxy's operation, and discuss
how they can interact with your application. Additionally, we'll shine some
light on the design decisions behind why the proxy works the way it does.

____

## Protocol Detection

The Linkerd proxy is *protocol-aware* --- when possible, it proxies traffic
at the level of application layer protocols (HTTP/1, HTTP/2, and gRPC), rather
than forwarding raw TCP traffic at the transport layer. This protocol awareness
unlocks functionality such as intelligent load balancing, protocol-level
telemetry, and routing.

> **Why is it important for a proxy to operate at the application layer, rather
> than at the transport layer?**
> A number of Linkerd 2's primary features inherently require a protocol-aware
> proxy. For example:
>
> + **Request/response telemetry**: In order to expose protocol-specific
>   telemetry (such as HTTP requests and responses, HTTP status codes, gRPC
>   status codes, et cetera), the proxy must understand those protocols.
> + **Latency-aware load balancing**: Similarly, for the proxy to use load
>   balancing algorithms that incorporate latency data, it must be aware of
>   protocol-level requests and responses so that latencies can be measured.
>   Furthermore, the load balancer must be aware of the target origin of a
>   request, in order to find servers that are capable of serving that request.

### How it Works

Since Linkerd 2 is designed to require as little configuration by the user as
possible, it automatically detects protocols. The proxy does this by peeking at
the data received on an incoming connection until it finds a pattern of bytes
that uniquely identifies a particular protocol. For example, all HTTP/2
messages begin with the bytes `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`.

If no protocol was identified after peeking up to a set number of bytes, the
connection is treated as raw TCP traffic.

### What This Means

The primary impact of the proxy's protocol detection is that it can interfere
with *server-speaks-first protocols*. These are protocols where clients
open connections to a server, but wait for the server to send the first bytes
of data. Because the Linkerd proxy detects protocols by peeking at the first
bytes of data sent by the client before opening a connection to the server,
it will fail to proxy data for these protocols.

Among the most common server-speaks-first protocols are MySQL and SMTP.
When using their default ports, Linkerd's protocol detection is disabled by
default. For other server-speaks-first protocols, or MySQL or SMTP traffic
on other ports, Linkerd has to be configured to disable its protocol detection.
See the ["Adding Your Service"] section of the documentation for more information.

["Adding Your Service"]: /adding-your-service#server-speaks-first-protocols

____

## Request Routing

Linkerd 2's proxy routes HTTP requests based on their target [*authority*] –
typically a hostname or IP address and optional port.

For HTTP/2 requests, the canonical authority is defined as the value of the
[`:authority` pseudo-header field], which is present on all HTTP/2 requests.

 For HTTP/1 requests, however, determining
the request's canonical authority is somewhat more complex:

+ If an [absolute-form] URI is received, it must replace the host header
  (in accordance with [Section 5.4 of RFC7230]).
+ If the request URI is not in absolute form, it is rewritten to contain
  the authority given in the `Host:` header, or, failing that, from the
  request's original destination according to `SO_ORIGINAL_DST`.

Once the authority is extracted from the request, Linkerd will perform service
discovery to determine the request's final destination.

[*authority*]: https://tools.ietf.org/html/rfc3986#section-3.2
[`:authority` pseudo-header field]: https://tools.ietf.org/html/rfc7540#section-8.1.2.3
[absolute-form]: https://tools.ietf.org/html/rfc7230#section-5.3.2
[Section 5.4 of RFC7320]: https://tools.ietf.org/html/rfc7230#section-5.4
