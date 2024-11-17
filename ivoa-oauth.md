# Proposed ivoa-oauth challenge


## Aims

1.  Allow use of existing off-the-shelf OAuth 2.x/OIDC authentication
    providers

    a.  This implies minimal changes to any of the standard flows/grants

2.  Must work for non-browser clients

    a.  Non-browser clients should not be required to call out a
        browser, nor run a web server for redirects (this covers
        situations where clients are running on a remote server or are
        otherwise unable to contact a browser directly)

3.  Fit in with SSO-next

    a.  This could either go into SSO-next directly, or be published as
        a separate note

## Definitions

-   **OAuth 2 Authorization Server** (from RFC 6749): The server issuing
    access tokens to the client after successfully authenticating the
    resource owner and obtaining authorization.

-   **OAuth 2 Resource Server** (from RFC 6749): The server hosting the
    protected resources, capable of accepting and responding to
    protected resource requests using access tokens.

    -   In the VO context, this would be a TAP/SCS/etc. service.

-   **OAuth 2 Client** (from RFC 6749): An application making protected
    resource requests on behalf of the resource owner and with its
    authorization.

    -   In the VO contact, this would be Aladin, pyvo, TOPCAT and
        similar clients (as would be expected).

-   [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749): The OAuth
    2.0 standard. Note that usage of `Bearer` is covered in
    [RFC 6750](https://datatracker.ietf.org/doc/rfc6750/). There are numerous
    extensions/additions/modifications on top of this. OAuth 2.1 is still in
    draft form, but merges in most of these extensions.

-   [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html):
    This is an extension of RFC 6749 which adds profile/identity related
    information. Usually this is abbreviated as **OIDC** (for **O**pen **ID**
    **C**onnect).

-   [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html):
    Allows for finding out what the endpoints are of an OIDC provider.

-   [OpenID Connect Dynamic Client Registration 1.0](https://openid.net/specs/openid-connect-registration-1_0.html):
    Allows for acquiring a `client_id` and `client_secret`.

-   [RFC 8628](https://datatracker.ietf.org/doc/rfc8628/): The Device
    Authorization Grant.

-   [RFC 8414](https://datatracker.ietf.org/doc/rfc8414/): Authorization
    Server Metadata. This is the OAuth 2.x version of OpenID Connect Discovery,
    and aims to be compatible with it.

-   [RFC 7591](https://datatracker.ietf.org/doc/rfc7591/): This is the OAuth 2.x
    version of OpenID Connect Dynamic Client Registration. There is a companion
    RFC [RFC 7592](https://datatracker.ietf.org/doc/rfc7592/).

-   [RFC 8252](https://datatracker.ietf.org/doc/rfc8252/): OAuth 2.0 for Native
    Apps.

    -   As will be discussed, this is not written for apps outside the
        two mobile platforms.

-   **VO Discovery Service**: A new service which aims to be a minimal
    bridge between OAuth 2.x/OIDC Authorization servers and VO Clients
    that are not browser-based.

## ivoa-oauth challenge (normative)

The `ivoa-oauth` challenge proceeds in 4 phases:

1.  The `www-authenticate` challenge containing the required information to find
    the appropriate VO Discovery Service from the OAuth 2.x resource server.

2.  The acquisition of the `client_id` and `client_secret` along with metadata
    from the VO Discovery Service.

3.  The acquisition of the `access_token` from the OAuth 2.x authorization
    server.

4.  The use of the `access_token` with the original OAuth 2.x resource server.

### The www-authenticate challenge from the OAuth 2.x resource server

Following the framework outline in SSO-next, OAuth 2.x resource servers
supporting the `ivoa-oauth` challenge should return the following
header:

`www-authenticate: ivoa-oauth discovery_url="https://<url of VO discovery service>"`

Multiple services can use the same VO discovery service.

The URL used MUST be the discovery endpoint of the VO discovery service:
redirects should be avoided as much as possible (validators should warn
if a redirect is used).

`discovery_url` must be provided, there is no default discovery URL.

Multiple `ivoa-oauth` challenges may be provided, though if a
resource server does provide multiple challenges, it is up to the
resource server to disambiguate the `access_tokens`, clients are
free to choose any of the VO discovery services provided.

### The VO discovery service

The VO discovery service must provide two endpoints:

1.  A discovery endpoint, only accepting GET requests

2.  A registration endpoint, only accepting POST requests

#### The discovery endpoint

The discovery endpoint MUST return a JSON document with the following
keys:

-   `registration_url`: the registration endpoint for this VO
    discovery endpoint

-   `allowed_domains`: the list of domains that any `access_token` that is
    acquired can be used against (as the use of cookies in responses lacks
    standardisation in the OAuth 2 space).

-   `supported_grant_types`: the list of supported grant types
    from
    https://www.iana.org/assignments/oauth-parameters/oauth-parameters.xhtml
    or as otherwise defined.
    `urn:ietf:params:oauth:grant-type:device_code` MUST be in that list.

-   `device_authorization_endpoint`: As per RFC 8414, the endpoint to get the
    device code.

-   `token_endpoint`: As per RFC 8414, the endpoint to get the access token.

The following additional keys may be provided:

-   `oauth2_discovery_url`: The URL of the authorisation servers
    OAuth 2 discovery endpoint (this should be provided when additional
    grant types are supported).

-   `oidc_discovery_url`: The URL of the authorisation servers OIDC discovery
    endpoint (this should be provided when additional grant types are
    supported).

-   `allow_bearer`: A boolean representing whether `Bearer` may be used for
    `access_tokens` against resource which are managed by this service. This
    exists solely to simply code that uses off-the-shelf libraries to handle
    OAuth2/OIDC (**COMMENTARY: we may want to drop this anyway, if services
    would find this painful to support?**).

### The registration endpoint

The client must POST a JSON document with the following keys to the
registration endpoint:

-   `client_name`: A human readable string to identify the client.

-   `grant_types`: A list containing a subset of the `supported_grant_types`
    that the returned `access_token` will be used for.

Additional keys may be specified as per RFC 7591 and OpenID Connect
Dynamic Client Registration. We note that `redirect_uris` is
optional given that redirect flows are problematic for the target
clients, but that developers of VO discovery services should verify the
behaviour of their systems where redirect flows are attempted using
clients registered through their service.

The VO discovery service will return JSON with at least the following
keys (as per RFC 7591 and OpenID Connect Dynamic Client Registration):

-   `client_id`

Additional keys may be returned as per RFC 7591 and OpenID Connect
Dynamic Client Registration.

#### Acquiring the access token

Using the `client_id` and the `device_authorization_endpoint` and
`token_endpoint`, RFC 8628 (Device Authorization Grant) should be followed to
get the access token. No VO specific steps are required.

#### Using the access token

Following the SSO-next recommendation, the access token should be provided in
the header with `authorization` field like `authorization: ivoa-oauth <token>`
where `<token>` is replaced by the token. If `allow_bearer` was specified, then
`Bearer` as per RFC 6750 can be used, in order simplify how clients manage the
access and refresh tokens.

## Discussion and commentary

### The name ivoa-oauth

This name could be changed (would `ivoa-oauth2` work in clients, currently none
of the other standard challenges have numbers in their name), but `ivoa-bearer`
was not used because it is plausible that other challenges might also use bearer
tokens, and to avoid there being ambiguity around what the aim of the challenge
was.

### RFC 8252

RFC 8252 would sound like it would address the needs of the VO, but as can be
seen by reading it, the focus of the RFC is to push back on the usage of
web-views on mobile platforms, and there is minimal applicable advice to the VO
(though VO client implementers may which to read it to understand the "vibe" of
the groups providing commercial authentication servers).

### Avoiding use of existing discovery and client registration protocols

Given that there is substantial overlap with the existing discovery and
client registration protocols, why create our own. There are two main
reasons, driving mainly by the aim to allow existing authentication
servers:

1.  The existing providers may not implement one or both of the existing
    discovery protocols, and neither of the discovery protocols cover
    all the information that the VO Discovery Service does (and
    requiring the modification of an existing provider to also include
    the required information also has the same problem)

2.  The existing providers may not implement one or both of the client
    registration protocols, and existing providers generally have very
    basic policy options (if any) around client registration. By
    specifying a minimal registration protocol, we avoid possible issues
    with existing providers.

### Requiring the availability of the Device Authorization Grant

Looking to the future of OAuth 2.x, where the Implicit Grant and the
Resource Owner Password Credentials Grant are being removed, the Device
Authorization Grant is the only standardised grant that works outside a
browser. Hopefully new standardised grants are developed and supported
which address the need to authenticate without a browser while being
both secure and usable.

### Use of JSON

Given the pre-existing use of JSON in the OAuth/OIDC ecosystem, JSON was
chosen as the format used to avoid bringing in any additional
dependencies.

### Not requiring more fields in discovery endpoint

The aim is to put as minimal requirements on both clients and service
providers as possible. Only the required information to perform the
Device Authorization Grant is required, and if additional grants would
be used, using an off-the-shelf library which accepts the
`client_id` and the discovery endpoint (either the OAuth2 or OIDC
URL) is a better path to go down, rather than something bespoke.
