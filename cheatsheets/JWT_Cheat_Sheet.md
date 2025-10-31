# JWT Cheat Sheet

## Introduction

**JSON Web Token** (JWT) is a standard format ([RFC 7519](https://tools.ietf.org/html/rfc7519))
for cryptographically secured tokens. It can be used for many a wide range of usages such as:

* transporting information about the end-user identity and attributes in [OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) (ID token);
* representing authorizations for accessing a an API in an access token (eg. [RFC 9068](https://tools.ietf.org/html/rfc9068));
* proving possesion of a private key (eg. [RFC 9449](https://tools.ietf.org/html/rfc9449));
* authenticating a workload (eg. [JWT-SVID](https://github.com/spiffe/spiffe/blob/main/standards/JWT-SVID.md)).

JWT can provide **authenticity** (JWS) and/or **confidentiality** (JWE) to the token content:

* An authenticated JWT (JWS) is protected against tampering (**authenticity**).
  It includes some proof of authenticity which can be used by the
  recipient to verify that it has not been tampered with
  and has not been forged altogether.
  In most applications, you want the token to be authenticated.
* The content of an encrypted JWS (JWE) is protected such that only its recipient
  should be able to inspect its content (**confidentiality**).
  This is desirable if the token is passed to a third party
  which should not be able to inspect the token content.

In addition, the JWT specification allows the usage of unsecure JWTs (`"alg":"none"`).
These JWTs do not provide ANY form of authenticity protection
and should usually not be used.
They are not discussed here.

JWT is a profile of the more general
JOSE format ([RFC 7515](https://tools.ietf.org/html/rfc7515), [RFC 7516](https://tools.ietf.org/html/rfc7516)).
While this cheat sheet is focused on JWTs,
a large part of what is discussed here is more generally applicable to JOSE messages in general.
Conversely, [CWT](https://datatracker.ietf.org/doc/html/rfc8392), and more generally [COSE](https://datatracker.ietf.org/doc/rfc9052/), have a very similar design
and many of the things discussed might be applicable to CWT and COSE as well.

## Concepts

### Actors

**Issuer:** the issuer of the JWT is the party which created the token.
The issuer is generally communicated in the issuer claim (`iss`):
this claim can be used by the audience to find the relevant keys to process the token
and apply the correct policy.

**Audience:** the audience of the JWT is the actor which is supposed to verify its authenticity,
decrypt it (if necessary) and validate its content. In order to prevent against audience confusion attacks,
where a JWT intended for one audience is sent by a malicious actor to an unintended audience,
the audience is generally communicated in the audience claim (`aud`)
and MUST be validated by the audience.

**Presenter/Holder:**
the presenter (resp. holder) of the token is the actor which presents the token to the audience the token (resp. holds the token).
In some cases, the presenter of the token is the issuer
but in many cases, the issuer gives the JWT to another presenter.
Depending on the application, this third-party may for example be identified by the authorized party (`azp`) of client identifier (`client_id`) claims.

TODO, add some examples?

### Type of tokens

**Bearer token:** TODO

**Proof-of-posession token:** TODO

## Authenticity (JWS)

An authenticated JWT (JWS) is protected against tampering (**authenticity**).
It includes some proof of authenticity which can be used by the
recipient to verify that it has not been tampered with
and has not been forged altogether.
This can be done either using:

* a digital signature (public-key cryptography, using a public/private key pair);
* a MAC (using a shared secret).

In most applications, you want the token to be authenticated
and the consumer of the token MUST vaidate the authenticity of the token.

### Structure of a signed JWT

The following elements are present in signed JWTs:

* **Protected Header:** the JWT header contains some information about the token such as the type of token (IANA media type)
and the cryptographic algorithms used to protect the token.
* **Claims:** the content JWT is a list of claims (usually about the subject). See the [JWT IANA Registry](https://www.iana.org/assignments/jwt/jwt.xhtml) for a list of standard claims.
* **Signature**, a signature in JWT is either a public-key digital signature (using a public/private key pair) or a MAC (using a shared secret). The signatures protects both the protected headers and the claims.

An authenticated JWT has the following format:

~~~
{base64url(json(header))}.{base64url(json(claims))}.{base64ur(signature)}
~~~

The following example is taken from [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)
(with line breaks added for presentation purpose):

~~~
eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJIUzI1NiJ9
.
eyJpc3MiOiJqb2UiLA0KICJleHAiOjEzMDA4MTkzODAsDQogImh0dHA6Ly9leGFt
cGxlLmNvbS9pc19yb290Ijp0cnVlfQ
.
dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
~~~

The decoded protected header is:

~~~json
{
  "typ": "JWT",
  "alg": "HS256"
}
~~~

The decoded claims are:

~~~json
{
  "iss": "joe",
  "exp": 1300819380,
  "http://example.com/is_root": true
}
~~~

### Signature vs. MAC

Recommendation: use a digital signature scheme when possible?

A JWE can be authenticated using either a digital signature or a MAC:

* When using a digital signature, the issuer of the token uses a private key to generate a signature.
  The audience of the token can use the associated public key to verify the authenticity of the token.
  Whereas the private key must only be known by the issuer, the public key can be public.
* When using a MAC, a shared secret is shared between the issuer and the audience.
  The same shared secret is used by the issuer to generate the token and by the audience
  to verify the authenticity of the token.
  The issuer MUST not use the same secret for different audiences
  and, more generally, the same secret MUST only be shared between these two participants.

Using a MAC may be interesting in the following cases:

* The issuer of the token is the sole audience of the token.

#### Public-key signature

Benefits of using a digital signature:

* The issuer can reuse the same public key for many different audiences.
* The consumer (audience) of the token only need public information to validate the token authenticity
  which reduces the risk of secret leakage by the consumer.
* Because the public key does not need to be secret, it can easily be distributed (eg. by publishing it at a public HTTPS URI).
* This makes key rotation simpler as well.

Recommended signature algorithms in order of preference:

1. EdDSA (Ed25519, Ed448);
2. ECDSA (ES256, ES384, ES512);
3. RSASSA-PSS (PS256, PS384, PS512).

The following algorithms are NOT recommended:

* RSASSA-PKCS1-v1_5 (RS256, RS384, RS512).

TODO, ES256K?

Notes:

* Support for EdDSA in JWT implementations may be currently somewhat limited.
* Generating ECDSA signatures may be dangerous on embedded systems where the quality of the randomness may be problematic. In this case, you must only use ECDSA signature if you make sure than the signature implementation uses deterministic signatures as defined in [RFC 6979](https://datatracker.ietf.org/doc/html/rfc6979).

TODO, Post-quantum signatures. ML-DSA (ML-DSA-44, ML-DSA-65, ML-DSA-87)? very large signature, probably not great justified at the moment for short-lived signatures. ML-DSA is designed to be resistant against quantum computers. However its support in JWT implementation is currently very limited. The size of if the signatures in ML-DSA is much larger than in ECDSA and EdDSA, resulting into very large JWTs.

#### MAC

The following MAC algorithms are recommended:

* HMAC with SHA-2 (HS256, HS384, HS512)

Secret management:

* Do not reuse the same secret for another purpose (eg. for encryption).
  * Using the same key for authenticating different types of JWTs or JOSE is fine as long as this does not introduce a risk of token type confusion.
* Do not reuse the same secret with another audience.
* Do not reuse the same secret with another issuer.
* Do not use a password as MAC secret.
* The secret must be generated using a local, cryptographically secure secret generator.
* The secret must have at least the same size as the output (eg. 256, 384 and 512 bits respectively for HS256, HS384 and HS512).
* Do not publish your secret key!

Valid HMAC secret generation example:

~~~python
import secrets
secret_for_hs256 = secrets.token_bytes(256//8)
~~~

Invalid HMAC secret generation:

~~~python
import random

# Using a password/passphrase is not OK.:
bad_secret1_for_hs256 = "MyProject2025!"

# Not a secure randomness source:
bad_secret2_for_hs256 = random.randbytes(256//8)

# Not enough entropy:
bad_secret3_for_hs256 = random.randbytes(128//8)

# Not enough entropy for HS512:
bad_secret_for_hs512 = random.randbytes(256//8)
~~~

## Protected headers

TODO

### Token Media Type

The `typ` header field may be used to indicate the media type of the token. The  `application/jwt` type is a generic type for JWTs. However, specific applications of JWTs define more specific media types of the form `application/*+jwt` such as:

* `application/at+jwt` for access tokens;
* `application/dpop+jwt` for [DPoP proofs](https://datatracker.ietf.org/doc/html/rfc9449) (proof-of-possession of a private key);
* etc.

It is recommended to use a specific media type for specific applications instead of using the generic `application/jwt` type. This makes it possible to

Using a specific media type in your tokens and validating this specific media type can be used to prevent cross-application JWT

Notes:

* you can and should omit the `application/` prefix in the `typ` header (eg. `"typ:"at+jwt"`);
* media types are case insensitive.

An an issuer,

* you SHOULD include a specific token type in the generated tokens;
* use a standard one if applicable (see the [Media Types IANA registry](https://www.iana.org/assignments/media-types/media-types.xhtml));
* use a private one otherwise (eg. `application/myorganisation-myapplication+jwt`) and document its usage.

As a consumer,

* you SHOULD validate that the token type included in the JWT is the one expected in the current context if such a type has been defined;
* in some cases, you may need to accept `application/jwt` for retrocompatibility with older issuers which did not include a specific toke type;
* you SHOULD reject other unexpected token types.

## Claims

See the [JWT IANA Registry](https://www.iana.org/assignments/jwt/jwt.xhtml) for a list of standard claims.
Some importants claims are discussed in this section.

### Validity

TODO, `iat` and `nbf`

### Audience

TODO, `aud`

A JWT can include more than one audience:

~~~json
{"aud": ["audience1","audience2"]}
~~~

If these token represent unrelated entities, this might present unrelated (and possibly distrusting entitied). When receiving the JWT from its presente r`"audience1"` could forward the token to `"audience2"` and impersonate the subject and/or presented on `"audience2"`. Even if the different audiences trust each other one audience could be compromised. If the JWT has multiple audiences representing different entities (as opposed to different endpoint of the same entity), the token should be a proof-of-possession token (not a bearer token).

TODO, audience ambiguity/etc. eg. when the audience is chosen by another party.

### Issuer

TODO, `iss`

### Presenter

TODO, `client_id`, `azp`

### Metadata

TODO, `jti` and `iat`

### Subject

TODO, `sub`

TODO, `act`, `may_act`

### Authorizations

TODO, `scope`

## Confidentiality (JWE)

Usually, you want a token which provides authenticity (JWS). In some cases, you might want to have confidentiality as well. This might be important if the claims contain some sensitive information (such as PII) that should not be exposed to the presenter (for example). This is achieved by using a Nested JWT: this is usually done by first signing the claims and then encrypting the resulting token.

You usualy don't want to have a JWT which provides confidentiality only: when using an encrypted JWT, you usually want to provide authenticity as well.

The correct handling of encryption introduces additional requirements such as:

* lack of forward secrecy (in general);
* susceptible to Harvest now, decrypt later (HNDL), especially when using non-post-quantum public key encryption.

For these reasons, these considerations are currently not addessed in this Cheat Sheet but will be addressed in a upcoming version.

## Key publishing

TODO

Recognizing a public key from a private key in JWK format:

| Key types           | kty        | Public key fields | Private key fields 
|---------------------|------------|-------------------|-------------------
| ML-DSA              | `AKP`      | `alg`, `pub`      | `priv`
| EC (eg. ECDSA)      | `EC`       | `crv`, `x`, `y`   |
| RSA                 | `RSA`      | `n`, `e`          | `d`, `p`, `q`, `dp`, `dq`, `qi`
| EdDSA, X25519, X448 | `OKP`      | `crv`, `x`        | `d`

TODO, add missing `alg`

TODO, other algorithms

TODO, JSON Web Key Use `use` and JSON Web Key Operations `key_ops`

## Attacks on JWT

For more details, see [RFC 8725](https://datatracker.ietf.org/doc/html/rfc8725).

TODO, align with RFC 8725

### Accepting Unsecured JWT

Some JWT libraries, [used to accept unsecured JWTs by default](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/) (`"alg":"none"`). In this case, an attacker would be able forge his own JWTs: depending on the application, he might be able to impersonate arbitrary users, obtains arbitrary authorizations, etc.

This issue should now be fixed in JWT libraries.

### Cross-JWT Confusion

TODO

## Recommendations

For more details, see [RFC 8725](https://datatracker.ietf.org/doc/html/rfc8725).

### General

* Do not log JWTs if theyr are intended to be secret. You can log specific claims however (if their are not considered sensible). The `jti` claim, associated with the `iss` claim, can be used to identify a specific token.

### Key Management

TODO

Key generation:

* Do not reuse the same key pair for another purpose (eg. for public-key encryption, for TLS authentication, for WebAuthn/Passkey, etc.). Using the same key for authenticating different types of JWTs or JOSE is fine as long as this does not introduce a risk of token type confusion or token audience confusion.
* The issuer should generate its own private keys. Don't use a private key generated by another agent (such as the consumer): your private key would not be private; this would increase the number of actors having access to your private key.
* If possible, store the private keys on dedicated hardware (such as a  smart card, a TPM) or a dedicated service.

Key distribution:

* The issuer can publish its public keys.
* This is typically done using the JWKS format over HTTPS.
* Make sure you do not publish the private keys by mistake! This is especialy important when publishing in JWK format as a private key in JWK format may be interpreted as a public key.

### Issuer

* Verify the issued token are not vulnerable to token type confusion.
* Include a specific token type claim (`typ`) depending on intended usage of the token in order to protect against token type confusion.
* TODO

### Verifier

* Rely on a trusted library for JWT verification.
* Validate important claims
* TODO

## Alternatives to JWT and JOSE

Depending on the application, some alternatives to JWT and JOSE might be:

* opaque tokens;
* [CBOR Object Token](https://datatracker.ietf.org/doc/html/rfc8392) (CWT) and [CBOR Object Signing and Encryption](https://datatracker.ietf.org/doc/html/rfc8152) (COSE);
* [PASETO](https://paseto.io/);
* [Eclipse Biscuit](https://www.biscuitsec.org/);
* [Fernet](https://github.com/fernet/spec/blob/master/Spec.md);
* [Security Assertion Markup Language (SAML)](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html) and [XML signature](https://www.w3.org/TR/xmldsig-core2/).

Critique of JWT and JOSE:

* [No Way, JOSE! Javascript Object Signing and Encryption is a Bad Standard That Everyone Should Avoid](https://paragonie.com/blog/2017/03/jwt-json-web-tokens-is-bad-standard-that-everyone-should-avoid)

## Further Reading

Main JWT and JOSE specifications:

- [RFC 7515](https://datatracker.ietf.org/doc/html/rfc7515), JSON Web Signature (JWS)
- [RFC 7516](https://datatracker.ietf.org/doc/html/rfc7516), JSON Web Encryption (JWE)
- [RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517), JSON Web Key (JWK)
- [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519), JSON Web Token (JWT)
- [RFC 8725](https://datatracker.ietf.org/doc/html/rfc8725), JWT Best Practices

IANA registries:

- [JSON Object Signing and Encryption (JOSE) IANA Reguistry](https://www.iana.org/assignments/jose/jose.xhtml)
- [JSON Web Token IANA Reguistry (JWT)](https://www.iana.org/assignments/jwt/jwt.xhtml)

Attacks on JWT and JOSE:

- [{JWT}.{Attack}.Playbook](https://github.com/ticarpi/jwt_tool/wiki) - A project documents the known attacks and potential security vulnerabilities and misconfigurations of JSON Web Tokens.
- [JWT.io Discussion Forum](https://community.auth0.com/c/jwt/8) (Hosted by [Auth0](https://auth0.com/))

Other useful links:

* [JSON Web Token (JWT) Debugger](https://jwt.io/)

