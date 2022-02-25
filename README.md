# Reticulum Internet Protocol Spec 0.1

Reticulum Internet Protocol (RIP) is a Client/Server internet protocol based on Gemini and Http. RIP allows for the bi-directional transfer of Reticulum [Resources](https://markqvist.github.io/Reticulum/manual/reference.html#resource) over the Reticulum [Link API](https://markqvist.github.io/Reticulum/manual/reference.html#link). 

# 1.1 RIP Transactions

Currently there is one main type of RIP transaction, essentially a GET request. In the future, RIP will allow for "POST" requests to be made, allowing clients to send resources to the server. One additional proposed feature that could be interesting, is the ability for the client and/or server to request the RIP connection revert back to a "raw" Link connection, which would allow for something similar to http's Server-Side Events or websockets. 

[[CLIENT AND SERVER FIRST ESTABLISH A RETICULUM LINK]] (Details explained [here](https://markqvist.github.io/Reticulum/manual/understanding.html#link-establishment-in-detail))

C:   Sends request (msgpacked RNS.Packet object) (see section 2)
	

S:   Sends response (msgpacked RNS.Packet or RNS.Resource object), closes Link under non-success conditions (see 3.1 and 3.2)
		

C:   Handles response (see 3.4)


S:   If request timeout is reached, Link is severed.


C:   If Client requests a page on a seperate RNS destination, or exits, Client tears down the link.


S:   Recieves Client disconnect.

# 1.2 RIP URL Scheme
Resources hosted via RIP protocol are requested with the general structure of: 

`rip://<Reticulum Destination Hash String>/optional/sub/paths`

Some notes:
- Actual destinations are not sandwiched in "<>"
- Port numbers are never used because they do not exist in RNS
- Generic URL parameters schemes are supported
- Empty paths are equivilant to the "/" path
- Spaces in the path should be encoded as %20

A note on human-readable names:

Reticulum destinations are excellent as a decentralized global naming scheme, but they are not great for creating human-readable names. The easiest approach to this problem is to allow clients to create "Local Address Books" which locally store mappings between RNS destinations and "traditional" URL names. These files could then be shared between pairs and grassroot consensuses could begin to be formed in communities.

# 2 RIP Requests
RIP requests are a single msgpacked Packet or Resource with the following structure (before being packed):

[time.time(), request_path_hash, data]

`time.time()` is the current time TODO:What time representation?, 

`request_path_hash` is the hash of the request path, using `RNS.Identity.truncated_hash()` implementation.

`data` is raw data, this input can be used to post content to the server. 

The actual object being sent looks like this in the reference python implementation:

```python
unpacked_request = [time.time(), request_path_hash, data]
packed_request   = umsgpack.packb(unpacked_request)

# Sent as packet it size is < RNS MTU
request_packet   = RNS.Packet(self, packed_request, RNS.Packet.DATA, context = RNS.Packet.REQUEST)
packet_receipt   = request_packet.send()


# Sent as resource if > RNS MTU
request_id = RNS.Identity.truncated_hash(packed_request)
request_resource = RNS.Resource(packed_request, self, request_id = request_id, is_response = False, timeout = timeout)
```                

# 3 RIP Responses
RIP responses consist of a single CRLF-terminated header line, optionally followed by a response body. For simplicity, message headers and bodies are bundled together in the following method:
`[header,body]`

Here is how RIP Responses are constructed in the reference python implementation:

```python
packed_rip_respond = umsgpack.packb([header,body])

packed_response = umsgpack.packb([request_id, packed_rip_response])

if len(packed_response) <= Link.MDU:
	RNS.Packet(self, packed_response, RNS.Packet.DATA, context = RNS.Packet.RESPONSE).send()
else:
	response_resource = RNS.Resource(packed_response, self, request_id = request_id, is_response = True)
```

# 3.1 Response Headers
RIP response headers look like this (exactly the same as Gemini Headers):
	

`<STATUS><SPACE><META>`


`<STATUS>` is a two-digit numeric status code, as described below in 3.2 and in Appendix 1.


`<SPACE>` is a single space character, i.e. the byte 0x20.


`<META>` is a UTF-8 encoded string of maximum length 1024 bytes, whose meaning is `<STATUS>` dependent.


The response header as a whole and `<META>` as a sub-string both MUST NOT begin with a U+FEFF byte order mark.


If `<STATUS>` does not belong to the "SUCCESS" range of codes, then the server MUST close the Link after sending the header and MUST NOT send a response body.


If a server sends a `<STATUS>` which is not a two-digit number or a `<META>` which exceeds 1024 bytes in length, the client SHOULD close the Link and disregard the response header, informing the user of an error.

# 3.2 Status codes

RIP uses two-digit numeric status codes. Related status codes share the same first digit. Importantly, the first digit of RIP status codes do not group codes into vague categories like "client error" and "server error" as per HTTP. Instead, the first digit alone provides enough information for a client to determine how to handle the response. By design, it is possible to write a simple but feature complete client which only looks at the first digit. The second digit provides more fine-grained information, for unambiguous server logging, to allow writing comfier interactive clients which provide a slightly more streamlined user interface, and to allow writing more robust and intelligent automated clients like content aggregators, search engine crawlers, etc.

The first digit of a response code unambiguously places the response into one of six categories, which define the semantics of the <META> line.

## 3.2.1 1x (INPUT)

Status codes beginning with 1 are INPUT status codes, meaning:

The requested resource accepts a line of textual user input. The <META> line is a prompt which should be displayed to the user. The same resource should then be requested again with the user's input included as a query component. Queries are included in requests as per the usual generic URL definition in RFC3986, i.e. separated from the path by a ?. Reserved characters used in the user's input must be "percent-encoded" as per RFC3986, and space characters should also be percent-encoded.
	
## 3.2.2 2x (SUCCESS)

Status codes beginning with 2 are SUCCESS status codes, meaning:

The request was handled successfully and a response body will follow the response header. The <META> line is a MIME media type which applies to the response body.
	
## 3.2.3 3x (REDIRECT)
	
Status codes beginning with 3 are REDIRECT status codes, meaning:
	
The server is redirecting the client to a new location for the requested resource. There is no response body. <META> is a new URL for the requested resource. The URL may be absolute or relative. If relative, it should be resolved against the URL used in the original request. If the URL used in the original request contained a query string, the client MUST NOT apply this string to the redirect URL, instead using the redirect URL "as is". The redirect should be considered temporary, i.e. clients should continue to request the resource at the original address and should not perform convenience actions like automatically updating bookmarks. There is no response body.
	
## 3.2.4 4x (TEMPORARY FAILURE)

Status codes beginning with 4 are TEMPORARY FAILURE status codes, meaning:	

The request has failed. There is no response body. The nature of the failure is temporary, i.e. an identical request MAY succeed in the future. The contents of <META> may provide additional information on the failure, and should be displayed to human users.

## 3.2.5 5x (PERMANENT FAILURE)
	
Status codes beginning with 5 are PERMANENT FAILURE status codes, meaning:

The request has failed. There is no response body. The nature of the failure is permanent, i.e. identical future requests will reliably fail for the same reason.  The contents of <META> may provide additional information on the failure, and should be displayed to human users. Automatic clients such as aggregators or indexing crawlers should not repeat this request.
	
## 3.2.6 6x (CLIENT IDENTITY REQUIRED)

Status codes beginning with 6 are CLIENT IDENTITY REQUIRED status codes, meaning:	

The requested resource requires a client identity to access. If the request was made without a remote peer confirming a valid identity, it should be repeated with one. If the request was made with a identity, the server did not accept it and the request should be repeated with a different identity. The contents of <META> (and/or the specific 6x code) may provide additional information on identity requirements or the reason a identity was rejected.	

## 3.2.7 Notes
	
Note that for basic interactive clients for human use, errors 4 and 5 may be effectively handled identically, by simply displaying the contents of <META> under a heading of "ERROR". The temporary/permanent error distinction is primarily relevant to well-behaving automated clients. Basic clients may also choose not to support client-identity authentication, in which case only four distinct status handling routines are required (for statuses beginning with 1, 2, 3 or a combined 4-or-5).

The full two-digit system is detailed in Appendix 1. Note that for each of the six valid first digits, a code with a second digit of zero corresponds is a generic status of that kind with no special semantics. This means that basic servers without any advanced functionality need only be able to return codes of 10, 20, 30, 40 or 50.

The RIP status code system has been carefully designed so that the increased power (and correspondingly increased complexity) of the second digits is entirely "opt-in" on the part of both servers and clients.
	
# 3.3 Response bodies
	
Response bodies only accompany responses whose header indicates a SUCCESS status (i.e. a status code whose first digit is 2). For such responses, <META> is a MIME media type as defined in RFC 2046.
	
Internet media types are registered with a canonical form. Content transferred via RIP MUST be represented in the appropriate canonical form prior to its transmission except for "text" types, as defined in the next paragraph.
	
RIP allows the transport of text media with plain LF alone (but NOT a plain CR alone) representing a line break when it is done consistently for an entire response body. RIP clients MUST accept CRLF and bare LF as being representative of a line break in text media received via Gemini.
	
If a MIME type begins with "text/" and no charset is explicitly given, the charset should be assumed to be UTF-8. Compliant clients MUST support UTF-8-encoded text/* responses. Clients MAY optionally support other encodings. Clients receiving a response in a charset they cannot decode SHOULD gracefully inform the user what happened instead of displaying garbage.
	
If <META> is an empty string, the MIME type MUST default to "text/micron; charset=utf-8". The text/micron media type is defined in section 5.	

# 3.4 Response body handling	

Response handling by clients should be informed by the provided MIME type information. Minimalistic clients might adopt a strategy of printing all text/* responses to the screen without formatting and saving all non-text responses to the disk. Clients for unix systems may consult /etc/mailcap to find installed programs for handling non-text types.

# 4 Identity Authentication
	
TODO: I think using the Link.identify() method should do most of this, but still need to figure out a proper way of doing this
	
# 5 Micron Text Specification
	
Refer to the spec found in [Nomadnet](https://github.com/markqvist/nomadnet)
	


	
	
	

	



