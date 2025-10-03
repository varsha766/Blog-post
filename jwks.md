## A Step-by-Step Guide to Asymmetric JWT Authentication
A JWT (JSON Web Token) is an open standard for securely transmitting information between parties as a JSON object. It contains claims — data about the user or system — that are digitally signed. This digital signature ensures the token’s content is verifiable and tamper-proof, making JWTs a popular choice for API authentication, single sign-on (SSO), and authorization in modern web applications.
### Understanding JWT Authentication Model
JWTs(JSON Web Tokens) provide a stateless way to authenticate users and authorize access to APIs. Unlike traditional session-based authentication, where the server stores session information, JWTs store all necessary information inside the token itself.
#### How JWT Authentication Works
##### User Login
- The user provides credentials(username/password) to the server.
- The server verifies the credentials against the database.
##### Token Generation
- If the credentials are valid, the server generates a JWT.
- The JWT contains a payload(claims) such as:
    ```js
    {
    “userId”: 123,
    “email”: “user@example.com”,
    “role”: “admin”,
    “iat”: 1696286400,
    “exp”: 1696290000
    }
    ```
   `iat` → issued at timestamp
   `exp` → expiration timestamp
- The server signs the token using either a symmetric secret (HS256) or asymmetric private key (RS256/ES256)
##### Token Transmission
- The server sends the JWT to the client.
- The client stores it locally (localStorage, sessionStorage, or HTTP-only cookies).
##### Client Request with JWT
- For subsequent requests, the client attaches the JWT in the HTTP header:
  `Authorization: Bearer <JWT_TOKEN>`
##### Token Verification

- The server verifies the token’s signature to ensure authenticity.
- If valid, the server extracts user information from the token and authorizes the request.
- If invalid or expired, the request is rejected.
### Symmetric vs. Asymmetric JWT: What’s the Difference?
JWTs can be signed using symmetric or asymmetric algorithms, and understanding the difference is key to choosing the right approach for application.

|Feature|Symmetric JWT (HS256)|Asymmetric JWT (RS256 / ES256)|
|-----|-----|-----|
|Signing Key|Same secret key is used for both signing and verification|Private key signs the token; public key verifies it.|
|Verification|Requires the same secret key on all services that verify the token|Only the public key is needed for verification; private key stays secure|
|Security Risk|If the secret is leaked, anyone can both sign and verify tokens|Public key can be shared safely; private key stays secure, reducing risk|
|Use Case|Simple apps or single-server setups|Microservices, distributed systems, or third-party integrations
|Key Management|Easier but less secure|Slightly more complex but more secure and scalable|

### Step-by-Step Implementation
#### Step 1: Generate a Key Pair
You’ll need a private key to sign tokens and a public key to verify them. Use OpenSSL to generate them:
```js
// Generate a 2048-bit RSA private key
  openssl genrsa -out private.key 2048
// Generate the corresponding public key
  openssl rsa -in private.key -pubout -out public.key
```
**Tips:**
- Store the private key securely (never expose it in your frontend or public repos).
- The public key can be shared with any service that needs to verify tokens.

#### Step 2: Signing a JWT
Use the private key to sign tokens. Here’s a Node.js  example using `jsonwebtoken`
   ```js
       const jwt = require('jsonwebtoken');
       const fs = require('fs');
       // Read the private key
       const privateKey = fs.readFileSync('./private.key');
       // Define the payload (claims)
       const payload = {
          userId: 123,
          email: 'user@example.com',
          role: 'admin'
        };
      // Sign the JWT with RS256 algorithm
      const token = jwt.sign(payload, privateKey, { algorithm: 'RS256', expiresIn: '1h' });
      console.log('Generated JWT:', token)
  ```
**Explanation:**
- RS256 indicates RSA with SHA-256 hashing.
expiresIn sets the token expiration time.
- You can include any custom claims needed by your application.

#### Step 3: Verifying a JWT
Now that tokens are being signed with your private key, you need a way for your API to verify incoming tokens using the public key.
Instead of manually writing verification logic for every route, we’ll use the express-jwt middleware along with jwks-rsa to keep things simple and secure.
```js
const express = require("express");
const { expressjwt } = require("express-jwt");
const jwksClient = require("jwks-rsa");
const morgan = require("morgan");
const app = express();
app.use(express.json());
app.use(morgan("dev"));

// Middleware to verify JWT using the public key from JWKS
app.use(
  expressjwt({
    secret: jwksClient.expressJwtSecret({
      jwksUri: "http://localhost:4000/jwks.json", // JWKS endpoint serving your public key
      cache: true,       // Cache keys to improve performance
      rateLimit: true    // Prevents excessive requests to the JWKS endpoint
    }),
    algorithms: ["RS256"], // Must match the algorithm used to sign the token
  }).unless({ path: ["/"] }) // Make the root route public
);

app.get("/", (req, res) => {
  res.send({ message: "This is a public route" });
});

app.get("/protected", (req, res) => {
  res.send({ message: "You have accessed a protected route!" });
});

app.listen(3000, () => console.log("API running on port 3000"));
```
🔑 **How it works:**
- `jwks-rsa` fetches the public key from your JWKS endpoint (/jwks.json).
- `Express-jwt` automatically verifies every incoming request’s   Authorization header:
    `Authorization : Bearer <JWT_TOKEN>`
- If the token is valid and signed with your private key, the request is allowed to continue.
- Invalid, expired, or missing tokens result in a 401 Unauthorized error.
          
#### Step 4:  Using Asymmetric JWTs in APIs
  Once your API is set up with the express-jwt middleware (from Step 3), clients can include the JWT in each request to access protected routes.
**Client-side request example:**

```js
 Get /protected HTTP/1.1
 Host: localhost:3000
Authorization: Bearer <JWT_TOKEN>
```
- The token is sent in the `Authorization` header as
 Authorization: Bearer <JWT_TOKEN>
- On the server, the middleware automatically checks the token’s signature, expiry, and claims before letting the request reach the route handler
#### Step 5: Security Best Practices
- Rotate keys periodically
- Protect your private key (use environment variables or secure vaults).
- Use short-lived access tokens and refresh tokens for better security.
- Only share the public key with services that need to verify tokens.

### Why Choose Asymmetric JWT Over Symmetric
While symmetric JWTs are simpler to implement, asymmetric JWTs offer significant advantages, especially in distributed systems or applications with multiple services.

1. **Enhanced Security**
   - **Private key never exposed**: Only the private key signs tokens, while the public key is used for verification.
   - **Harder to forge tokens**: Even if the public key is known, an attacker cannot create valid tokens.
   - **Reduced risk in distributed systems**: You can safely share the public key across services without compromising security.
2. **Ideal for Microservices & Distributed Systems**
   - Multiple services can verify tokens independently using the public key.
   - No need to share a secret key among all services (as required in symmetric JWTs).
   - Simplifies scaling and maintaining security in large applications.
3. **Easier Management**
   - The private key is stored securely on the authentication server.
   - Public keys can be rotated or updated without invalidating all existing tokens, depending on implementation.
   - Supports public/private key rotation strategies for enhanced security.
4. **Better for Third-Party Integrations**
   - If you provide an API for external clients or partners, you can share the public key so they can verify tokens without giving them the ability to sign tokens.
   - This is impossible with symmetric JWTs since sharing the secret would allow token generation.
5. **Supports More Advanced Use Cases**
   - Single sign-on (SSO) across multiple domains or services.
   - Multi-service authentication in microservices architectures.
   - Token verification in client-side apps or third-party services without exposing sensitive credentials.
### Wrapping Up
💻 Demo Code: [GitHub Repository](https://github.com/varsha766/jwk)






