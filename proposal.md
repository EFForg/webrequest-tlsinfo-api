# webRequest TLS Introspection API Extension Propsal

This document outlines a proposed extension to the webRequest browser extension API that would add TLS introspection capabilities that are currently missing from all major browsers (exlucding the soon to be removed XPCOM/XUL Firefox APIs).

## The Problem

While the existing webRequest API provides basically carte blanche access to the request life cycle the one thing it is missing is any form of introspection into the TLS connection state other than knowing that a user may have requested a HTTPS URL.

The lack of any kind of introspection prevents the development of a number of security focused browser extensions such as:

1. extensions that feed third-party telemetry systems like EFF's DSO
2. extensions that want to replicate functionality of things like SSL Labs security scoring capabilities
3. extensions that want to provide lower level/more prominent cert chain introspection than provided by existing cert viewer UIs
4. extensions that want to attempt MITM detection
5. extensions that help debug various TLS deployment and configuration issues

## Proposed API

We propose that a limited set of TLS connection stateinformation be presented in a new object as part of the `onCompleted` request callback. Providing this information only on the completion of a request would prevent extensions from attempting to meddle with the browsers internal validation engine and would simply provide a window into the outcome of the validation procedures. In cases where the connection fails to complete, such as when Chrome shows validity warning interstitials, the `onCompleted` event is never actualy fired preventing extensions from attempting to subvert various invalidity states.

### `details` object extension

This new TLS state object should have the key `tlsInfo` and be included in the `details` object passed to the callback method for the `onCompleted` event when a request is made using HTTPS and the information is available (in certain response caching situations this information may not be available when requesting a HTTPS url and may be omitted) and contain at least the following information:

* Built + sent chain (raw DER)

  ```
  builtChain: array of arrays of raw DER
  sentChain: array of arrays of raw DER
  ```

  Both built and sent chain are useful for knowing which path the validation engine considers canonical and for knowing if the server is sending any superfluous certificates in the chain for both third-party telemetry gathering and PageSpeed/SSL Labs style extensions. No further processing needs to be done beyond providing the raw DER as these can be trivially parsed by extensions and third-party services.

* Built chain validity

  `chainValid: bool`
  
  This is the only way to know if a user clicked through a cert warning and is useful for third-party telemetry where the third-party doesn't want to attempt to reconstruct the behavior of various validation engines and is hard to replicate via simple crawling. It is okay for this to just be a simple boolean instead of providing the explicit reasoning as to why the chain is considered invalid.

* Protocol

  `protocol: enum of [TLS, QUIC]`
  
  There are certain cases where additional information defined below may not be present because the protocol being used is not actually TLS (mainly just QUIC), this information allows extensions to know whether that extra information should be expected to be present or not.

* Cipher Suite

  `cipherSuite: int (optional)`

  There is no way of extracting this from the chain, useful for determining feature deployment and understanding which suite a server/browser decided on. There is no real way of replicating this information via crawling without replicating the various cipher suite lists that are provided by browsers on an ongoing basis. Field should contain the uint16 assigned to the used cipher suite as defined in the IETF [TLS cipher suite registry](https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4). Field should not be present if `protocol` contains the value `QUIC`.

* TLS version

  `tlsVersion: string (optional)`

  Again there is no way of extracting this from the chain, useful for determining version deployment and it is hard to determine with crawling without replicating the same protocol version support that the browser has on an ongoing basis. Format should be a basic string containing the specific protocol and the RFC standardized version (i.e. `TLS 1.2` or `SSL 3.0`). Field should not be present if `protocol` contains the value `QUIC`.

## Open Questions

* Should including this information require an extra manifest permission or should the existing `webRequest` permission be enough?
* Does exposing information about the sent chain create a privacy risk when it may expose enterprise level MITMs which may be personally (or organizationally) identifying?
