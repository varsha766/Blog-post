## Securing JWT in Production: What Matters Beyond RS256

In my previous article, I explained how to implement **asymmetric JWT authentication using RS256.**
While choosing an asymmetric algorithm is an important step, **JWT security does not end with key pairs and signature.**

In real-world systems, JWT-related vulnerabilities usually happen because of poor claim design, long token lifetimes, weak validation, or insecure storage.

In this article, we'll focus on how to make JWT truly secure in production — beyond just signing it correctly.

### Why Asymmetric JWT Alone Is Not Enough

Using RS256:
- Tokens are signed with a private key
- Tokens can be verified using a public key
- Secrets don't need to be shared across services

However, even with RS256:
- A stolen token is still valid until it expires
- Incorrectly data in the payload can still be exposed
**Security depends on how JWT is designed, issued, stored, and validated — not just the algorithm.**

### Security Goals of a JWT
Before defining claims or expiry, it's important to understand what a secure JWT should guarantee:

**1. Integrity** - the token must not be tampered with

**2. Authenticity** - the issuer must be trusted
**3. Limited lifetime** - tokens should expire quickly
**4. Controlled usage** - tokens should only work where intended
Every JWT design decision should support these goals.
### Mandatory JWT Claim You Should Always Validate
A signed JWT is useless if its claims are not validated properly.

`iss`**(Issuer)**

Identifies **who issued the token.**

Why it matters:
- Prevents token issued by another system from being accepted
- Especially important in multi-environemt setups

Always validate `iss` against a known value.

`aud`**(Audience)**

Defines **who the token is intended for**.
Why it matters:

- Prevents token reuese across different APIs or services
- Critical in microservices architectures

Never skip audience validation.

`exp`**(Expiration Time)**

Defiens when the token becomes invalid.

Why it matters:

 - Limits damage if a token is leaked
 - Reduce replay attacks

 **Short expiry is non-negotiable.**

 `sub`(Subject)

Represents the identity of the user or entity.

Best practice:

- Use a stable iternal identifier(userId)
- Avoid emails or mutable identifiers

`iat`**(Issued At) and** `jti`**(JWT Id)**

- `iat` helps detect old or replayed tokens
- `jti` uniquely identifies a token and enables revocation or blacklisting

These claims become very useful in advanced security scenarios.

### Designing Token Expiry the Right Way

One of the most common JWT mistakes is **long-lived access tokens.**

#### Recommended Approach
- **Access Token**: 5-15 minutes
- **Refresh Token**: LOnger-lived, stored securely

Why short-lived access tokens?
- Limits the impact of token leakage
- Reduces the need of complex revocation logic

JWTs are stateless — expiry is strong defense.

### Refresh Tokens and Token Rotation Strategy

Since asymmetric JWTs are hard to revoke immediately, **refresh tokens are essential.**

Best practices:

- Store refresh tokens in a database or Redis
- Rotate refresh tokens on every use
- Invalidate old refresh tokens immediately
- Never expose refresh tokens to JavaScript if possible

This gives you even in a stateless authentication system.

### Key Management in Asymmetric JWT

Asymmetric JWT security heavily depends on **key management.**

**Private Key**
- Must never leave the authentication server
- Should be stored securely(env vars or secret manager)

**Public Key**

- Can be shared safely
- Often exposed via a **JWKS endpoint**

`kid`**(Key ID)**
- Helps identify which key was used to sign the token
Enables seamless key rotation without downtime

Key rotation should be planned from day one.

### Correct JWT Validation on the Server

JWT validation must be strict and explicit.

Always:
- Verify the signature
- Whitelist allowed algorithms(never trust the token header)
- Validate `iss`, `aud`, and `exp`
- Handle clock skew safely
- Reject malformed or partially valid tokens

A token that is mostly valid is still **invalid.**

### Secure JWT Storage on the Client Side

Even a perfectly signed JWT can be stolen if stored incorrectly.

**Avoid**
- LocalStorage
- SessionStorage

These are vulnerable to XSS attacks.

**Recommended**
- HttpOnly cookies
- Secure and SameSite flags enabled

### Common JWT Security Mistakes(Even with RS256)

The mistakes are surprisingly common:

- Accepting both `HS256` and `RS256`
- Not validating `aud`
- Very long token expiry
- Putting sensitive data in the payload
- Assuming `signed` means `encrypted`

### Final Thoughts

JWT is not insecure by design — incorrect implementation is.

While asymmetric algorithms like RS256 address key-sharing and trust boundaries, true JWT security comes from careful claim design, short token lifetimes, strict validation, secure storage, and disciplined key management. A signed token alone does not guarantee safety.

Treat JWT as a security boundary, not just a convenient authentication mechanism. When used thoughtfully and validated rigorously, JWT can scale securely across modern, distributed production systems.


