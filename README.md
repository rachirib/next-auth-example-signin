# NextAuth.js Authentication Bug Reproduction

## Overview

This repository is based on the [nextjs-auth-example](https://github.com/nextauthjs/next-auth-example). Please read their [readme](https://github.com/nextauthjs/next-auth-example?tab=readme-ov-file#getting-started) for initial setup instructions.

This sample reproduces a bug where hostnames containing the word `signin` break the authentication callback, causing incorrect URL redirection.

## Bug Description

When using a hostname that contains the word `signin`, NextAuth.js incorrectly redirects to a URL that replaces `signin` with `callback` during the authentication flow.

**Expected behavior:** Authentication should work regardless of hostname  
**Actual behavior:** Hostname `signin.vercel.add` redirects to `callback.vercel.add`

## Project Configuration

This sample includes the following modifications from the base example:

- **Next.js:** Version 14
- **NextAuth.js:** Version 5.0.0-beta.22
- **Authentication Provider:** Simple Credentials provider
  - Username: `jsmith`
  - Password: `verysecret122@`
- **Custom Login Page:** Located at `app/login`

## Prerequisites

- Node.js and pnpm installed
- `mkcert` for SSL certificate generation (macOS: `brew install mkcert`)

## Reproduction Steps

### 1. Initial Setup

Follow the [nextjs-auth-example setup instructions](https://github.com/nextauthjs/next-auth-example?tab=readme-ov-file#getting-started) to configure the project.

### 2. Verify Normal Functionality

Test that authentication works with localhost:

```bash
pnpm dev
```

Navigate to `http://localhost:3000` and `Sign in`. This should work correctly.

### 3. Reproduce the Bug

#### Step 3.1: Add Host Alias

Add a hostname containing "signin" to your `/etc/hosts` file:

```bash
# Add this line to /etc/hosts
127.0.0.1       signin.vercel.add
```

#### Step 3.2: Generate SSL Certificates

Since non-localhost domains require HTTPS for authentication:

```bash
# Install and configure mkcert (if not already done)
brew install mkcert
mkcert --install

# Generate certificates for the test domain
mkcert localhost 127.0.0.1 ::1 signin.vercel.add
```

This will create `signin.vercel.add+3.pem` and `signin.vercel.add+3-key.pem` files.

#### Step 3.3: Run with HTTPS

Start the development server with HTTPS and custom hostname:

```bash
pnpm dev --hostname signin.vercel.add --experimental-https --experimental-https-key ./signin.vercel.add+3-key.pem --experimental-https-cert ./signin.vercel.add+3.pem
```

#### Step 3.4: Trigger the Bug

1. Navigate to `https://signin.vercel.add:3000`
2. Click `Sign in` and enter credentials:
   - Username: `jsmith`
   - Password: `verysecret122@`
3. **Observe the bug:** The browser will incorrectly redirect to `https://callback.vercel.add:3000` instead of staying on `signin.vercel.add`

## Expected vs Actual Behavior

| Scenario | Expected Redirect | Actual Redirect | Status |
|----------|------------------|-----------------|---------|
| `localhost:3000` | `localhost:3000/api/auth/callback` | `localhost:3000/api/auth/callback` | ✅ Works |
| `signin.vercel.add:3000` | `signin.vercel.add:3000/api/auth/callback` | `callback.vercel.add:3000/api/auth/callback` | ❌ Bug |

## Technical Notes

- The issue only occurs with hostnames containing the substring `signin`
- The bug appears to be related to string replacement logic in NextAuth.js callback URL generation [lines](https://github.com/nextauthjs/next-auth/blob/39dd3b92de194c1a835f2d87631f4deb9d9fdf65/packages/next-auth/src/lib/actions.ts#L65-L67)
- SSL certificates are required for non-localhost testing due to authentication security requirements
