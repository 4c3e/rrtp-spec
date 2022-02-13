# Reticulum Internet Protocol Spec 0.1

Reticulum Internet Protocol (RIP) is a Client/Server internet protocol based on Gemini and Http. RIP allows for the bi-directional transfer of Reticulum [Resources](https://markqvist.github.io/Reticulum/manual/reference.html#resource) over the Reticulum [Link API](https://markqvist.github.io/Reticulum/manual/reference.html#link). 

# 1.1 RIP Transactions

Currently there is one main type of RIP transaction, essentially a GET request. In the future, RIP will allow for "POST" requests to be made, allowing clients to send resources to the server. One additional proposed feature that could be interesting, is the ability for the client and/or server to request the RIP connection revert back to a "raw" Link connection, which would allow for something similar to http's Server-Side Events or websockets. 

[[CLIENT AND SERVER FIRST ESTABLISH A RETICULUM LINK]] (Details explained [here](https://markqvist.github.io/Reticulum/manual/understanding.html#link-establishment-in-detail))

C:   Sends request (one CRLF terminated line) (see section 2)
	

S:   Sends response header (one CRLF terminated line), closes Link under non-success conditions (see 3.1 and 3.2)
	

S:   Sends response body (Reticulum Resource containing text or binary data) (see 3.3)
	

C:   Handles response (see 3.4)


S:   If request timeout is reached, Link is severed. Note that this timeout is different from the Reticulum Link upkeep pings.


C:   If Client requests a page on a seperate RNS destination, or exits, Client sends Kill request


S:   Recieves Kill request and severs Link.

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
RIP requests are a single CRLF-terminated line with the following structure:

`<URL><CR><LF>`

`<URL>` is a UTF-8 encoded absolute URL, including a scheme, of maximum length 1024 bytes.  The request MUST NOT begin with a U+FEFF byte order mark.

This request system is identical to Gemini, the only difference being that the RNS Destination segment of the URL can be replaced with a "\*" if users want to preserve bandwidth, this is because Link's already imply a single RNS destination. It is important to still include the scheme prefix, so RIP servers can seamlessly handle multiple different protocols.

# 3 RIP Responses
RIP responses consist of a single CRLF-terminated header line, optionally followed by a response body.

# 3.1 Response Headers
RIP response headers look like this (exactly the same as Gemini Headers):
	

`<STATUS><SPACE><META><CR><LF>`


`<STATUS>` is a two-digit numeric status code, as described below in 3.2 and in Appendix 1.


`<SPACE>` is a single space character, i.e. the byte 0x20.


`<META>` is a UTF-8 encoded string of maximum length 1024 bytes, whose meaning is `<STATUS>` dependent.


The response header as a whole and `<META>` as a sub-string both MUST NOT begin with a U+FEFF byte order mark.


If `<STATUS>` does not belong to the "SUCCESS" range of codes, then the server MUST close the connection after sending the header and MUST NOT send a response body.


If a server sends a `<STATUS>` which is not a two-digit number or a `<META>` which exceeds 1024 bytes in length, the client SHOULD close the connection and disregard the response header, informing the user of an error.
	
	
	

	



