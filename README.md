# Bluesky OAuth Client Implementation

A browser-based OAuth 2.0 client for Bluesky authentication with DPoP (Demonstrating Proof of Possession) support.

## Overview

This implementation demonstrates the Bluesky OAuth authentication flow in a pure browser environment. It handles the complex requirements of the AT Protocol's OAuth implementation, including:

- PKCE (Proof Key for Code Exchange)
- DPoP (Demonstrating Proof of Possession)
- JWT token handling and decoding

## How It Works

### 1. Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant Bluesky
    participant PDS as User's PDS

    User->>Browser: Enter Bluesky handle
    Browser->>Browser: Generate state & PKCE
    Browser->>Bluesky: Resolve handle to DID (200 OK)
    Browser->>PDS: Get PDS URL from DID (200 OK)
    Browser->>PDS: Get auth server info (200 OK)
    Browser->>Bluesky: Make PAR request (201 Created)
    Bluesky-->>Browser: Request URI + DPoP nonce
    Browser->>Bluesky: Redirect to auth endpoint (302 Found)
    Bluesky->>User: Show login & permission screen
    User->>Bluesky: Approve authorization
    Bluesky->>Browser: Redirect with auth code (302 Found)
    Browser->>Bluesky: Exchange code for tokens (400 Bad Request)
    Bluesky-->>Browser: Error: use_dpop_nonce
    Browser->>Browser: Create new DPoP proof with nonce
    Browser->>Bluesky: Retry with DPoP nonce (200 OK)
    Bluesky-->>Browser: Access token
    Browser->>Browser: Decode & display token
```

### 2. Step-by-Step Explanation

1. **Enter Bluesky Handle**
   - **User → Browser**: The user inputs their Bluesky handle (e.g., "username.bsky.social").

2. **Generate State & PKCE**
   - **Browser → Browser**: Generates a random state string to prevent CSRF attacks and PKCE code verifier/challenge pair for secure code exchange.

3. **Resolve Handle to DID (200 OK)**
   - **Browser → Bluesky**: Converts the user's handle to a Decentralized Identifier (DID) which is the permanent identifier in the AT Protocol.
   - **Expected Response**: 200 OK with DID in JSON response.

4. **Get PDS URL from DID (200 OK)**
   - **Browser → PDS**: Fetches the DID document to find the user's Personal Data Server URL.
   - **Expected Response**: 200 OK with DID document containing service endpoints.

5. **Get Auth Server Info (200 OK)**
   - **Browser → PDS**: Retrieves OAuth metadata from the PDS to find the authorization server.
   - **Expected Response**: 200 OK with OAuth protected resource metadata.

6. **Make PAR Request (201 Created)**
   - **Browser → Bluesky**: Makes a Pushed Authorization Request to pre-register the auth parameters.
   - **Expected Response**: 201 Created with request_uri and DPoP-Nonce header.

7. **Request URI + DPoP Nonce**
   - **Bluesky → Browser**: Returns a request URI and DPoP nonce for the next step.

8. **Redirect to Auth Endpoint (302 Found)**
   - **Browser → Bluesky**: Redirects user to Bluesky's authorization page with the request URI.
   - **Expected Response**: 302 Found redirect to authorization UI.

9. **Show Login & Permission Screen**
   - **Bluesky → User**: Displays the authentication UI to the user.

10. **Approve Authorization**
    - **User → Bluesky**: User logs in (if needed) and approves the authorization request.

11. **Redirect with Auth Code (302 Found)**
    - **Bluesky → Browser**: Redirects back to the app with an authorization code.
    - **Expected Response**: 302 Found redirect to the redirect_uri with code parameter.

12. **Exchange Code for Tokens (400 Bad Request)**
    - **Browser → Bluesky**: Attempts to exchange the auth code for tokens.
    - **Expected Response**: 400 Bad Request with use_dpop_nonce error.

13. **Error: use_dpop_nonce**
    - **Bluesky → Browser**: Server requires a DPoP proof with the current nonce.

14. **Create New DPoP Proof with Nonce**
    - **Browser → Browser**: Generates a cryptographically signed JWT with the required nonce.

15. **Retry with DPoP Nonce (200 OK)**
    - **Browser → Bluesky**: Retries the token request with proper DPoP proof.
    - **Expected Response**: 200 OK with access and refresh tokens.

16. **Access Token**
    - **Bluesky → Browser**: Server returns a JWT access token.

17. **Decode & Display Token**
    - **Browser → Browser**: Decodes and displays the JWT token information.

### 3. Key Components

#### Identity Resolution
- Resolves Bluesky handles to DIDs
- Fetches DID document to find user's PDS
- Gets auth server info from PDS

#### PKCE Implementation
- Generates random code verifier
- Creates S256 code challenge
- Verifies during token exchange

#### DPoP Security
- Generates unique keypair for each session
- Creates signed JWT with precise format
- Handles nonce requirements

#### Token Handling
- Securely exchanges auth code for tokens
- Decodes JWT token to display payload
- Stores tokens in localStorage

## Technical Details

### OAuth Client Metadata

The OAuth client is defined by a metadata file hosted at a public URL. This file includes:

```json
{
  "client_id": "https://your-domain.com/oauth/client-metadata.json",
  "application_type": "web",
  "grant_types": ["authorization_code", "refresh_token"],
  "scope": "atproto",
  "response_types": ["code"],
  "redirect_uris": ["https://your-domain.com/index.html"],
  "dpop_bound_access_tokens": true,
  "token_endpoint_auth_method": "none",
  "client_name": "Your Bluesky App Name",
  "client_uri": "https://your-domain.com"
}
```

### DPoP JWT Structure

The DPoP proof is a JWT with specific requirements:

**Header:**
```json
{
  "typ": "dpop+jwt",
  "alg": "ES256",
  "jwk": {
    "crv": "P-256",
    "kty": "EC",
    "x": "...",
    "y": "..."
  }
}
```

**Payload:**
```json
{
  "jti": "[random string]",
  "htm": "POST",
  "htu": "https://bsky.social/oauth/token",
  "iat": 1642526915,
  "nonce": "[server-provided nonce]"
}
```

### Example Access Token

A successful authentication results in an access token like:

```
eyJ0eXAiOiJhdCtqd3QiLCJhbGciOiJFUzI1NksifQ.eyJhdWQiOiJkaWQ6d2ViOmluay...
```

When decoded, it contains:

**Header:**
```json
{
  "typ": "at+jwt",
  "alg": "ES256K"
}
```

**Payload:**
```json
{
  "aud": "did:web:user-pds-url",
  "iat": 1742226916,
  "exp": 1742230516,
  "sub": "did:plc:user-did",
  "jti": "tok-token-id",
  "cnf": {
    "jkt": "key-thumbprint"
  },
  "client_id": "https://your-domain.com/oauth/client-metadata.json",
  "scope": "atproto",
  "iss": "https://bsky.social"
}
```

## Implementation Notes

### Browser-Only Limitations

This implementation demonstrates the complete OAuth flow up to obtaining an access token, but has some limitations:

- **CORS Restrictions**: Direct API calls from browsers to Bluesky's token endpoint may be blocked by CORS policies
- **Key Security**: In a production environment, better protection for cryptographic keys would be advisable
- **Production Recommendation**: For production use, implement a backend service to handle the token exchange

### Key Security Details

The implementation includes:

- No client secrets (public client)
- Fresh keypair generation for each session
- Web Crypto API for secure cryptographic operations
- Error handling for all authentication steps

## Usage

1. Host the client metadata JSON file at an HTTPS URL
2. Update the `CLIENT_ID` constant in the code to point to this URL
3. Host the HTML file at an HTTPS URL that matches a redirect URI in the metadata
4. Users enter their Bluesky handle to begin authentication

## Related Resources

- [AT Protocol OAuth Specification](https://atproto.com/specs/oauth)
- [DPoP Specification (RFC 9449)](https://datatracker.ietf.org/doc/html/rfc9449)
- [PKCE (RFC 7636)](https://datatracker.ietf.org/doc/html/rfc7636)
- [Bluesky API Documentation](https://github.com/bluesky-social/atproto)

## License

MIT License - Feel free to use and modify this implementation for your own projects.
