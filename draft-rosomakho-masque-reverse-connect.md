---
title: "Reverse HTTP CONNECT for TCP and UDP"
category: std

docname: draft-rosomakho-masque-reverse-connect-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: ""
workgroup: "Multiplexed Application Substrate over QUIC Encryption"
keyword:
 - quic
 - http
 - proxy
 - reverse
venue:
  group: "Multiplexed Application Substrate over QUIC Encryption"
  type: ""
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "yaroslavros/draft-masque-reverse-connect"
  latest: "https://yaroslavros.github.io/draft-masque-reverse-connect/draft-rosomakho-masque-reverse-connect.html"

author:
 -
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com

normative:
  H1:
    =: RFC9112
    display: HTTP/1.1
  H2:
    =: RFC9113
    display: HTTP/2
  H3:
    =: RFC9114
    display: HTTP/3

informative:


--- abstract

This document specifies an extension to the HTTP CONNECT method, enabling a proxy client to accept inbound TCP and UDP sessions proxied through HTTP/1.1, HTTP/2, or HTTP/3. This mechanism allows the client to dynamically advertise available local or internal network services and expose them through a HTTP proxy without reliance on IP routing.


--- middle

# Introduction

Reverse HTTP CONNECT for TCP {{!TCP=RFC0793}} and UDP {{!UDP=RFC0768}} extends the traditional CONNECT method (see {{Section 9.3.6 of !HTTP=RFC9110}}) by enabling a proxy client to accept inbound TCP and UDP sessions proxied through HTTP/1.1 {{H1}}, HTTP/2 {{H2}}, or HTTP/3 {{H3}}. In contrast to the traditional CONNECT method, which establishes outbound sessions to remote servers, this extension facilitates inbound sessions to the proxy client, allowing access to local or internal services.

Unlike Proxying IP in HTTP {{!CONNECT-IP=RFC9484}}, this approach simplifies deployment in dynamic or constrained network environments by removing reliance on IP routing. By eliminating the need for  routing configurations, this approach reduces operational complexity and allows for easier integration in scenarios where traditional IP routing is impractical. On top of that, Reverse HTTP CONNECT reduces overhead as it does not carry IP or transport layer headers.

Since Reverse HTTP CONNECT is built on top of existing HTTP transport, it can be efficiently combined with outbound CONNECT and other HTTP communications.

The primary use cases for this extension include:

- **Access to Local Services**: A proxy client can expose its own local services by accepting inbound TCP or UDP sessions from an HTTP server. For example, this can enable remote management  or access to local application servers that can only be accessed through an outbound connection to the proxy.

~~~aasvg
+---------+                                 +------------+
|       --+----Outbound HTTP connection-----+--.         |
| Proxy  <+-------Inbound TCP session-------+-- \ Proxy  |
| Client <+-------Inbound UDP session-------+-- / Server |
|       --+---------------------------------+--'         |
+---------+                                 +------------+
~~~
{: #local-services title="Accessing local client services"}

- **Gateway to Internal Networks**: A proxy client acts as a gateway, facilitating access to internal network services provided over TCP or UDP sessions. In this scenario, the proxy client forwards incoming session originating from an HTTP server to specific internal services, enabling secure communication with private network resources.

~~~aasvg
                    +--------+
.----------.        | Proxy  |
| .--------+-.      | Client |                            +------------+
| | .--------+-.    |    ----+--Outbound HTTP connection--+--.         |
'-+ | Internal |<---+----<---+-----Inbound TCP session----+-- \ Proxy  |
  '-+ Services |<---+----<---+-----Inbound UDP session----+-- / Server |
    '----------'    |    ----+----------------------------+--'         |
                    +--------+                            +------------+
~~~
{: #internal-services title="Accessing internal services through client"}

As explained in the specification both use cases can be combined on the same proxy client.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Client Configuration

To use a Reverse Connect proxy HTTP clients are configured with two URI Templates {{!TEMPLATE=RFC6570}}:

* Listener Control Channel described in {{listener-template}}
* Accepting Connection Requests described in {{accepting-connection}}.

# Listener Control Channel

The Listener Control Channel enables the HTTP server to request the establishment of inbound TCP or UDP sessions on the proxy client. This is necessary because the HTTP protocol inherently expects the client to initiate connections. To address this, the Listener Control Channel uses the Capsule Protocol (see {{Section 3.2 of !HTTP-DGRAM=RFC9297}}) to carry connection requests from the server to the client.

Additionally, the Listener Control Channel can carry optional Service Discovery Capsules, which allow the proxy client to advertise available local or internal services to the HTTP server. This feature facilitates dynamic service discovery and ensures that the server has up-to-date information about the services accessible through the proxy client.

## Listener Control Channel URI Template {#listener-template}

The Listener Control Channel template is similar to the one defined in {{Section 3 of CONNECT-IP}} and it MAY contain two variables: "target" and "ipproto". These variables are used to define the scope of network services that the client accepts connectivity for.

Examples are shown below:

~~~
https://example.org/.well-known/masque/listen/{target}/{ipproto}/
https://proxy.example.org:4443/masque/listen?t={target}&i={ipproto}
https://proxy.example.org:4443/masque/listen{?target,ipproto}
https://masque.example.org/?user=bob
~~~
{: #fig-listen-template-examples title="Listener Control Channel URI Template Examples"}

All template requirements listed in {{Section 3 of CONNECT-IP}} apply here.

If set, the variable "target" MUST contain one of the following values:

* A hostname according to {{Section 4.6 of CONNECT-IP}}
* IPv4 or IPv6 prefix according to {{Section 4.6 of CONNECT-IP}}
* "\*" that does not limit scope
* A wildcard: a domain name prefixed by "\*.". This implies that client MAY accept inbound sessions for any host in a given domain
* "." which means that the client will accept sessions to local services only and will not forward to other hosts

If set, the variable "ipproto" MUST contain one of the following values:

* "6" which means that the client will only accept TCP sessions.
* "17" which means that the client will only accept UDP sessions.
* "\*" which means that the client will accept both TCP and UDP sessions.

As with {{CONNECT-IP}}, some client configurations for Reverse Connect proxies will only allow
the user to configure the proxy host and proxy port. Clients with such
limitations MAY attempt to access proxying capabilities using the default
template, which is defined as:
"https://$PROXY_HOST:$PROXY_PORT/.well-known/masque/listen/{target}/{ipproto}/",
where $PROXY_HOST and $PROXY_PORT are the configured host and port of the
proxy, respectively. Reverse Connect proxy deployments SHOULD offer service at this location
if they need to interoperate with such clients.

## Establishing Listener Control Channel over HTTP/1.1

When establishing the Listener Control Channel using HTTP/1.1 {{H1}}, the client follows the requirements from {{Section 4.2 of CONNECT-IP}} but uses "connect-listen" as the value for the Upgrade header.

For example, the client configured with the default URI Template "https://example.org/.well-known/masque/listen/{target}/{ipproto}/" can establish the Listener Control Channel for local TCP and UDP services by sending the following request:

~~~ http-message
GET https://example.org/.well-known/masque/listen/./*/ HTTP/1.1
Host: example.org
Connection: Upgrade
Upgrade: connect-listen
Capsule-Protocol: ?1
~~~
{: #fig-req-h1 title="Example HTTP/1.1 Request"}

The server confirms successful registration of the client as described in {{Section 4.3 of CONNECT-IP}}, but with "connect-listen" for the value for Upgrade header.

An example of the server confirming successful registration of the client is provided below:

~~~ http-message
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: connect-listen
Capsule-Protocol: ?1
~~~
{: #fig-resp-h1 title="Example HTTP/1.1 Response"}

## Establishing Listener Control Channel over HTTP/2 and HTTP/3

When establishing Listener Control Channel using HTTP/2 {{H2}} or HTTP/3 {{H3}}, the client follows requirements from {{Section 4.4 of CONNECT-IP}}, but uses "connect-listen" as the value for :protocol pseudo-header.

For example, the client configured with the default URI Template "https://example.org/.well-known/masque/listen/{target}/{ipproto}/" can establish the Listener Control Channel for local TCP and UDP services by sending the following request:

~~~ http-message
HEADERS
:method = CONNECT
:protocol = connect-listen
:scheme = https
:path = /.well-known/masque/listen/./*/
:authority = example.org
capsule-protocol = ?1
~~~
{: #fig-req-h2 title="Example HTTP/2 or HTTP/3 Request"}

The server indicates a successful registration of the client as described in {{Section 4.5 of CONNECT-IP}}.

Example of server confirming successful registration of the client is provided below:

~~~ http-message
HEADERS
:status = 200
capsule-protocol = ?1
~~~
{: #fig-resp-h2 title="Example HTTP/2 or HTTP/3 Response"}

## Service Discovery {#service-discovery}

The Reverse Connect client MAY advertise offered TCP and UDP services to the proxy server through the AVAILABLE_SERVICES Capsule over the established Listener Control Channel. This information serves as a hint for the proxy server and does not constrain the server from attempting to establish sessions with services that were not advertised. The presence of a service in the AVAILABLE_SERVICES list does not guarantee actual service availability, as the client may not be able to track the health of the service.

Capsule format for AVAILABLE_SERVICES is the following:

~~~
AVAILABLE_SERVICES Capsule {
  Type (i) = 0xTBD,
  Length (i),
  Service (..) ...,
}
~~~
{: #available-services-format title="AVAILABLE_SERVICES Capsule Format"}

The AVAILABLE_SERVICES capsule contains a sequence of zero or more Services.

~~~
Service {
  Destination Type (8),
  Destination (..),
  Protocol (8),
  Port (16),
}
~~~
{: #service-format title="Service Format"}

Each service record represents potential connection destination and contains the following fields:

Destination Type:

: A single byte value defining type and format of Destination field. Valid destination types are:
- 0: Destination is local for the client.
- 1: Destination is specified by a hostname.
- 4: Destination is specified by an IPv4 address.
- 6: Destination is specified by an IPv6 address.

Destination:

: Content of the Destination field is determined by the Destination Type. For local destination type destination is omitted. IPv4 destination carries 32 bits of IPv4 address. IPv6 destination carries 128 bits of IPv6 address. Hostname destination (destination type 1) is provided in the following format:

~~~
Hostname_Destination {
  Length (i),
  Hostname (..),
}
~~~
{: #hostname-destination-format title="Hostname Destination Format"}

Length is encoded as a variable-length integer and Hostname contains the hostname string.

Protocol:

: A single byte value of 6 for TCP and 17 for UDP.

Port:

: Two bytes value of target TCP or UDP port.

AVAILABLE_SERVICES messages are not incremental. A new message completely overwrites the previous hint.

If any of the capsule fields are malformed upon reception, the server MUST follow the error-handling procedure defined in {{Section 3.3 of HTTP-DGRAM}}.

# Connection Lifecycle

## Requesting Connection

To request a new outbound connection from the client for an inbound TCP or UDP session, the server sends a CONNECTION_REQUEST Capsule. The CONNECTION_REQUEST Capsule is defined as follows:

~~~
CONNECTION_REQUEST Capsule {
  Type (i) = 0xTBD,
  Length (i),
  Request ID (i),
  Service (..),
}
~~~
{: #connection-request-format title="CONNECTION_REQUEST Capsule Format"}

Request ID:
: A unique request identifier, encoded as a variable-length integer. The Request ID MUST be unique for a given Listener Control Channel.

Service:
: Session destination as described in {{service-discovery}}.

If any of the capsule fields are malformed upon reception, or if the Request ID has been previously used on the same Listener Control Channel, the client MUST follow the error-handling procedure defined in {{Section 3.3 of HTTP-DGRAM}}.

If the client declines the connection request it responds with a CONNECTION_REQUEST_DECLINED Capsule in the following format:

~~~
CONNECTION_REQUEST_DECLINED Capsule {
  Type (i) = 0xTBD,
  Length (i),
  Request ID (i),
}
~~~
{: #connection-request-declined-format title="CONNECTION_REQUEST_DECLINED Capsule Format"}

If any of the capsule fields are malformed upon reception, or if the  Request ID does not correspond to an outstanding CONNECTION_REQUEST on the same Listener Control Channel, the server MUST follow the error-handling procedure defined in {{Section 3.3 of HTTP-DGRAM}}.

## Accepting Connection {#accepting-connection}

To accept the inbound session, the client sends a new outbound request to the server. This request uses an Accepting Connection URI Template {{TEMPLATE}}. The URI Template MUST contain "request_id" variable.

Examples are shown below:

~~~
https://example.org/.well-known/masque/accept/{request_id}/
https://proxy.example.org:4443/masque/accept?id={request_id}
https://proxy.example.org:4443/masque/accept{?request_id}
https://masque.example.org/?user=bob&request_id={request_id}
~~~
{: #fig-accepting-template-examples title="Accepting Connection URI Template Examples"}

The following requirements apply to the Accepting Connection URI Template:

* The URI Template MUST be a level 3 template or lower.
* The URI Template MUST be in absolute form and MUST include non-empty scheme, authority, and path components.
* The path component of the URI Template MUST start with a slash "/".
* All template variables MUST be within the path or query components of the URI.
* The URI Template MUST contain variable "request_id" and MAY contain other variables.
* The URI Template MUST NOT contain any non-ASCII Unicode characters and MUST only contain ASCII characters in the range 0x21-0x7E inclusive (note that percent-encoding is allowed; see {{Section 2.1 of !URI=RFC3986}}.
* The URI Template MUST NOT use Reserved Expansion ("+" operator), Fragment Expansion ("#" operator), Label Expansion with Dot-Prefix, Path Segment Expansion with Slash-Prefix, nor Path-Style Parameter Expansion with Semicolon-Prefix.

Clients SHOULD validate the requirements above; however, clients MAY use a general-purpose URI Template implementation that lacks this specific validation.  If a client detects that any of the requirements above are not met by a URI Template, the client MUST reject its configuration and abort the request without sending it to the IP proxy.

As with {{CONNECT-IP}}, some client configurations for Reverse Connect proxies will only allow
the user to configure the proxy host and proxy port. Clients with such
limitations MAY attempt to access proxying capabilities using the default
template, which is defined as:
"https://$PROXY_HOST:$PROXY_PORT/.well-known/masque/accept/{request_id}/",
where $PROXY_HOST and $PROXY_PORT are the configured host and port of the
proxy, respectively. Reverse Connect proxy deployments SHOULD offer service at this location
if they need to interoperate with such clients.

### Accepting Connection over HTTP/1.1

When accepting an inbound session over HTTP/1.1 {{H1}}, the client sends to the proxy a new request as follows:

* The method SHALL be "GET"
* The request SHALL include a single "Host" header field containing the origin of the proxy.
* The request SHALL include a "Connection" header field with the value "Upgrade" (note that this requirement is case-insensitive as per {{Section 7.6.1 of HTTP}}).
* The request SHALL include an Upgrade header field with value "connect-accept".

A request that does not match the above restrictions is considered malformed. A proxy server receiving a malformed request MUST respond with an error and SHOULD use the 400 (Bad Request) status code. If `request_id` template variable is not matching an outstanding connection request for given client, proxy server MUST respond with an error and SHOULD use the 404 (Not Found) status code.

The proxy SHALL indicate a successful response by replying with the following requirements:

* The HTTP status code on the response SHALL be 101 (Switching Protocols).
* The response SHALL include a Connection header field with value "Upgrade" (note that this requirement is case-insensitive as per {{Section 7.6.1 of HTTP}}).
* The response SHALL include a single Upgrade header with value "connect-accept".
* The response SHALL meet the requirements of HTTP responses that start the Capsule Protocol according to {{Section 3.2 of HTTP-DGRAM}}.

A response that does not match these requirements MUST be treated as failed and corresponding connection MUST be aborted by the client.

For example, the client configured with the URI Template "https://example.org/.well-known/masque/accept/{request_id}/" can accept the inbound Connection as a response to a connection request with Request ID 1:

~~~ http-message
GET https://example.org/.well-known/masque/accept/1/ HTTP/1.1
Host: example.org
Connection: Upgrade
Upgrade: connect-accept
Capsule-Protocol: ?1
~~~
{: #fig-req-accept-h1 title="Example HTTP/1.1 Request Accepting Connection"}

The proxy server could respond with the following:

~~~ http-message
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: connect-accept
Capsule-Protocol: ?1
~~~
{: #fig-resp-accept-h1 title="Example HTTP/1.1 Response Accepting Connection"}

### Accepting Connection over HTTP/2 and HTTP/3

When accepting an inbound sessions over HTTP/2 {{H2}} or HTTP/3 {{H3}}, the client sends to the proxy a request on a new Stream of the connection that carries the Listener Control Channel as follows:

* The :method pseudo-header field SHALL be "CONNECT".
* The :protocol pseudo-header field SHALL be "connect-accept".
* The :authority pseudo-header field SHALL contain the authority of the proxy.
* The :path and :scheme pseudo-header fields SHALL contain the path and scheme of the request URI derived from the Accepting Connection URI template for the proxy.

A request that does not match the above restrictions is considered malformed. A proxy server receiving a malformed request MUST respond with an error and SHOULD use the 400 (Bad Request) status code. If "request_id" template variable is not matching an outstanding connection request for given client, proxy server MUST respond with an error and SHOULD use the 404 (Not Found) status code.

The proxy SHALL indicate a successful response by replying with the following requirements:

* The HTTP status code on the response SHALL be in 2xx (Successful) range.
* The response SHALL meet the requirements of HTTP responses that start the Capsule Protocol according to {{Section 3.2 of HTTP-DGRAM}}.

A response that does not match these requirements MUST be treated as failed and corresponding connection MUST be aborted by the client.

For example, the client configured with the URI Template "https://example.org/.well-known/masque/accept/{request_id}/" can accept the inbound session as a response to a connection request with Request ID 1:

~~~ http-message
HEADERS
:method = CONNECT
:protocol = connect-accept
:scheme = https
:path = /.well-known/masque/accept/1/
:authority = example.org
capsule-Protocol = ?1
~~~
{: #fig-req-accept-h2 title="Example HTTP/2 or HTTP/3 Request Accepting Connection"}

The proxy server could respond with the following:

~~~ http-message
HEADERS
:status = 200
capsule-Protocol = ?1
~~~
{: #fig-resp-accept-h2 title="Example HTTP/2 or HTTP/3 Response Accepting Connection"}

## Establishing Connection

Once the client receives a successful response from the proxy for the request accepting the connection it establishes the requested TCP or UDP socket to a local or internal destination. If the socket failed to establish, the client MUST immediately close the request stream.

For TCP flows, communicating parties carry TCP payload data in the payload of DATA Capsules over the established stream according to {{Section 8.3 of !Template-TCPCONNECT=I-D.ietf-httpbis-connect-tcp}}. Either party MAY close the stream if TCP connection is terminated.

UDP data is carried in DATAGRAM capsules or encoded in QUIC DATAGRAM frames as explained in {{Section 5 of !CONNECT-UDP=RFC9298}}. The lifetime of the UDP socket is tied to the request stream as described in {{Section 3.1 of CONNECT-UDP}}.

# Security Considerations

Proxy servers providing Reverse HTTP Connect services MUST restrict their services to authenticated users. An unauthorized user accepting an inbound session from the proxy server may cause availability risks and compromise the wider systems by misrouting communications. Clients that advertise services through the AVAILABLE_SERVICES Capsule may inadvertently expose sensitive or private services. Proxy servers MUST verify that advertised services comply with organizational security policies.

To avoid exposing local or internal services to unauthorized parties, clients MUST confirm identity of the proxy service that they connect to and allow inbound sessions only to services that should be accessible from the proxy.

Proxies providing Reverse HTTP Connect service over HTTP/1.1 MUST consistently authenticate clients on both Listener Control Channel and individual Connection Acceptance Requests. To minimize the risks of misrouting connection request to a rogue client, HTTP/1.1 Reverse Connect proxies SHOULD generate Request ID randomly instead of using a predictable sequence.

# IANA Considerations

## HTTP Upgrade Tokens

IF APPROVED, IANA is requested to add the following entries to the HTTP Upgrade Token Registry:

| Value | Description | Reference |
| ----- | ----------- | --------- |
| "connect-listen" | Listening for inbound connection requests | (This document) |
| "connect-accept" | Accepting an inbound connection request | (This document) |

## New MASQUE Default Templates

IF APPROVED, IANA is requested to add the following entries to the MASQUE URI Suffixes Registry:

| Value | Description | Reference |
| ----- | ----------- | --------- |
| "listen" | Listening for inbound connection requests | (This document) |
| "accept" | Accepting an inbound connection request | (This document) |

## New HTTP Capsule Types

IF APPROVED, IANA is requested to add the following entries to the HTTP Capsule Types Registry:

| Value | Capsule Type |
| ----- | ------------ |
| (TBD) | AVAILABLE_SERVICES |
| (TBD) | CONNECTION_REQUEST |
| (TBD) | CONNECTION_REQUEST_DECLINED |

All of these new entries use the following values for these fields:

Status: permanent
Reference: (this document)
Change Controller: IETF
Contact: MASQUE
Notes: None

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
