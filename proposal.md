# webRequest TLS Introspection API Extension Propsal

This document outlines a proposed extension to the webRequest browser extension API that would add TLS introspection capabilities that are currently missing from all major browsers.

## The Problem

While the existing webRequest API provides basically carte blanche access to the request life cycle the one thing it is missing is any form of introspection into the TLS connection state other than knowing that a user may have requested a HTTPS URL.

The lack of any kind of introspection prevents the development of a number of security focused plugins such as:

a. plugins that feed third-party telemetry systems like EFF's DSO
b. plugins that want to replicate functionality of things like SSL Labs security scoring
c. plugins that want to provide lower level/more prominent cert chain introspection than provided by existing cert viewer
d. plugins that want to attempt MITM detection services
e. plugins that help debug various TLS deployment issues

## Proposed API

We propose that a limited set of TLS state information be presented in a new object as part of the `onCompleted` request callback. Providing this information only on the completion of a request would prevent plugins from attempting to meddle with any the browsers internal validation engine and would simply provide a window into the outcome of the validation procedures.

This object should have the key `tlsInfo` and be included in the `details` object passed to the `onCompleted` callback method when a request is made using HTTPS and the information is available (in certain caching situations this information may not be available when requesting a HTTPS url) and contain the following information:

* Built + sent chain (raw DER)

  `builtChain: array of arrays of raw DER`
  `sentChain: array of arrays of raw DER`

  Both built and sent chain are useful for knowing which path the validation engine considers canonical + for knowing if the server is sending superfluous certificates in the chain for both third-party telemetry gathering and PageSpeed style extensions. No further processing needs to be done beyond providing the raw DER as these can be trivially parsed by extensions and third-party services

* Built chain validity

  `chainValid: bool`
  
  Only way to know if a user clicked through a cert warning, useful for third-party telemetry where third-party doesn't want to attempt to reconstruct behavior of specific validation engine (+ hard to replicate via crawling). Fine with this just being a binary var instead of providing reasoning as to _why_ the chain is considered invalid.

* Protocol

  `protocol: enum of [TLS, QUIC]`
  
  There are certain cases where additional information defined below may not be present because the protocol being used is not actually TLS (mainly just QUIC), this information allows plugins to know whether that extra information shall be present or not.

* Ciphersuite

  `cipherSuite: string (optional)`

  No way of extracting from chain, useful for determining feature deployment + understanding which suite a server/browser decided on, no real way of replicating via crawling without replicating the various suite lists provided by browsing on an ongoing basis. String should be RFC style format names (i.e. `TLS_RSA_WITH_AES_256_GCM_SHA384`). Field should not be present if `protocol` contains the value `QUIC`.

* TLS version

  `tlsVersion: string (optional)`

  Again no way of extracting from chain, useful for determining version deployment and hard to determine with crawling without replicating the same protocol version support that the browser has on an ongoing basis. Format should be a basic string containing the specific protocol and the standardized version (i.e. `TLS 1.2` or `SSL 3.0`). Field should not be present if `protocol` contains the value `QUIC`.
  
* SCT embeded in handshake

  `embededSCT: array of array of raw der (optional)`
  
  In the case where a certificate doesn't use an embeded SCT this allows plugins to display information about present SCTs and which CT logs a certificate may be present in by parsing the raw DER.

* OCSP embeded in handshake

  `embededOCSP: array of array of raw der (optional)`
  
  Provides plugins with information about stapled OCSP responses, in the case where one or more SCTs are delivered in a OCSP response this also allows plugins to provide information about CT logs the certificate may be present in. 

## Open Questions

* Should including this information require an extra manifest permission or should the existing `webRequest` permission be enough?
* Does exposing information about the sent chain create a privacy risk when it may expose enterprise level MITMs which may be personally (or organizationally) identifying?
