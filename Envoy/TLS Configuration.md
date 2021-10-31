TLS [overview](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/security/ssl#tls) from Envoy docs. Mozilla [recommendations](https://github.com/mozilla/server-side-tls/blob/gh-pages/Server_Side_TLS.mediawiki).

## Certificates

Certifcates are delivered over SDS. The certificate is in PEM format and serialized in the [TlsCertificate](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/transport_sockets/tls/v3/common.proto#extensions-transport-sockets-tls-v3-tlscertificate) protobuf. The actual field is called `certificate_chain`. Envoy processes this as a stream of X509 certificates in PEM format. The first certificate is the server certificate; the remaining certificates are the CA chain.

To actually configure ECC and RSA certificate support in Envoy, you need to somehow feed Envoy two `TlsCertificate` objects. They would each contain a certificate for the same name, but with different keys. The Envoy code to do this iterates TLS certificate configurations (roughly `TlsCertificate` objects) and has some checks to ensure global key type uniqueness.

This is not supported with SDS, [#13226](https://github.com/envoyproxy/envoy/issues/13226). Taking a quick look at the current SDS code, I don't see an obvious restriction. Looks like this was fixed in [#16606](https://github.com/envoyproxy/envoy/pull/16605).

## Session Tickets

Why Mozilla recommends against TLS session tickets [mozilla/server-side-tls#135](https://github.com/mozilla/server-side-tls/issues/135)).

Additional commentary on the difficulty of forward security with session tickets [here](https://www.imperialviolet.org/2013/06/27/botchingpfs.html).

Envoy  session keys can be served from SDS and are an array, where the first is used for encryption and the remainder are used for decryption, [ref](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/transport_sockets/tls/v3/common.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-tlssessionticketkeys). If the control plane doesn't explicitlt disable session resumption, Envoy will enable it with an internal key.

## Cipher Suites

Per [Envoy docs](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/transport_sockets/tls/v3/common.proto#extensions-transport-sockets-tls-v3-tlsparameters), FIPS mode disables Chacha/Poly cipher suites. Would be interesting to benchmark and see whether those suites still have a performance advantage on modern hardware.

A EC2 t2.2xlarge instance running Linux 5.11.12 shows AES support in CPU info. Presumably BoringSSL can detect and use that.

Only use ciphers with forward secrecy in TLS 1.2.

## OCSP Stapling

Envoy supports stapling an OCSP response. It looks like the management server need to query the OCSP responder to get the response to staple and then Envoy simply consumes that.

The OCSP response is delivered over SDS as part of the [TlsCertificate](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/transport_sockets/tls/v3/common.proto#extensions-transport-sockets-tls-v3-tlscertificate).

Commentary from Ryan Sleevi [here](https://gist.github.com/sleevi/5efe9ef98961ecfb4da8).

## Strict-Transport-Security

Mozilla recomends this [ref](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security). For wildcard domains, the `includeSubDomains` field probably should be set.