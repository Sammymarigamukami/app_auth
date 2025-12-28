# Project: Auth App with Email & Oauth (Google/GitHub) + JWT

## Description
This is a small authentication system supporting:
- Email & password login
- Google Oauth login
- GitHub Oauth login
- JWT-based access tokens
- Refresh tokens

All Oauth accounts links to single 'users' table for centralized identity.

## Tech Stack
- **Backend**: Node.js, Express.js
- **Database**: MySQL
- **Authentication**: JWT + Oauth

## Database Schema
### **Users Table**
```sql
CREATE TABLE users (
    id CHAR(36) NOT NULL PRIMARY KEY,        -- UUID
    email VARCHAR(255) NOT NULL UNIQUE,      -- user email
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
### **User Auth Providers table**
```sql
CREATE TABLE user_auth_providers (
    id CHAR(36) NOT NULL PRIMARY KEY,        -- UUID
    user_id CHAR(36) NOT NULL,               -- FK to users
    provider ENUM('local','google','github') NOT NULL,
    provider_user_id VARCHAR(255) DEFAULT NULL, -- OAuth provider ID (Google/Github)
    password_hash VARCHAR(255) DEFAULT NULL,    -- only for local login
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(provider, provider_user_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```
### **Refresh Tokens table**
```sql
CREATE TABLE refresh_tokens (
    id CHAR(36) NOT NULL PRIMARY KEY,        -- UUID
    user_id CHAR(36) NOT NULL,               -- FK to users
    token_hash CHAR(64) NOT NULL UNIQUE,     -- SHA-256 hash of the refresh token
    expires_at DATETIME NOT NULL,
    revoked TINYINT(1) DEFAULT 0,            -- 0 = active, 1 = revoked
    replaced_by_token CHAR(64) DEFAULT NULL, -- hash of new token if rotated
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```
## Key Design Notes
1. `users` is the single source of truth.
2. `user_auth_providers` allows linking multiple OAuth accounts to one user.
   - `local` provider uses `password_hash`.
   - `provider_user_id` must be unique per provider.
3. `refresh_tokens` supports:
   - Rotation( `replaced_by_token` )
   - Revocation( `revoked` )
   - Expiry ( `expires_at` )

## How Authentication Works
### 1. Email & Password
1. User signs up with email and password
2. Password is hashed with `bcrypt` and stored in `user_auth_providers` (provider=`local`)
3. Backend issues:
   - **JWT access token** -- short-lived, stored in HTTP_only cookie
   - **Refresh token** -- long-lived, stored hashed in DB, sent as HTTP-only cookie
### Flow Diagram
```pgsql
[ Browser ]
     |
     |  POST /auth/login (email, password)
     v
[ Backend ]
     |
     |  -> verify password (bcrypt)
     |  -> issue JWT (15m)
     |  -> issue Refresh Token (7d)
     |  -> store refresh token hash in DB
     |
     v
[ Set-Cookie: access_token, refresh_token ]
     |
     v
[ Browser ] ---> Redirect ---> /dashboard

```

### 2. GitHub Oauth Flow
1. Frontend redirects user to Github OAuth URL with:
   - `client_id`
   - `redirect_uri`
   - `scope` ( `email, username, id, avatar_url, name, bio` )
   - `state`
2. User authenticates on GitHub
3. Github redirects back to backend with `code` and `state`
4. Backend:
   - Verifies `state`
   - Exchanges `code` for GitHub access token
   - Uses token to fetch GitHub user ID
   - Finds or create user in `users` + `user_auth_providers`
   - Issues **JWT + refresh token** as HTTP-only cookies
5. Backend redirect user to frontend
### Flow Diagram
```pgsql
[ Browser ]
     |
     |  GET /auth/github
     v
[ Backend ]
     |
     |  → redirect to GitHub OAuth
     |    (client_id, redirect_uri, scope, state)
     v
[ GitHub ]
     |
     |  User authenticates
     v
[ GitHub ]
     |
     |  redirect_uri?code=XXX&state=YYY
     v
[ Backend ]
     |
     |  -> verify state
     |  -> exchange code for access token
     |  -> fetch GitHub user ID
     |  -> find/create user
     |  -> issue JWT + refresh token
     |
     v
[ Set-Cookie: access_token, refresh_token ]
     |
     v
[ Browser ] ---> Redirect ---> /dashboard
```

### 3. Google Oauth Flow
1. Frontend redirect user to Google Oauth URL with:
   - `client_id`
   - `redirect_uri`
   - `response_type`
   - `state` (for CSRF protection)
   - `scope` ( `email, profile, username, id, avatar_url, name, bio` )
2. User authenticates on Google
3. Google redirects back to backend with `code` and `state`
4. Backend:
   - Verifies `state`
   - Exchange `code` for Google tokens
   - Extracts user info from `id_token`
   - Finds or create user in `users` + `user_auth_providers`
   - Issues **JWT + refresh token** as HTTP-only cookies
5. Backend redirects user to frontend
### Flow Diagram
```pgsql
[ Browser ]
     |
     |  GET /auth/google
     v
[ Backend ]
     |
     |  → redirect to Google OAuth
     |    (client_id, redirect_uri, response_type, scope, state)
     v
[ Google ]
     |
     |  User authenticates
     v
[ Google ]
     |
     |  redirect_uri?code=XXX&state=YYY
     v
[ Backend ]
     |
     |  -> verify state (CSRF protection)
     |  -> exchange code for tokens
     |  -> extract Google user ID + email
     |  -> find/create user
     |  -> issue JWT + refresh token
     |
     v
[ Set-Cookie: access_token, refresh_token ]
     |
     v
[ Browser ] ---> Redirect ---> /dashboard

```

### 4. JWT + Refresh Token Usage
- **JWT (access token):**
   - Short-lived (15 min)
   - Sent in HTTP-only cookie ( `access_token` )
   - Used by backend to authorize requests
- **Refresh token:**
   - Long-lived (7 days)
   - Stored hashed in DB
   - Sent in HTTP-only cookie ( `refresh_token` )
   - Used to get a new access token when JWT expires

## Installation & Setup
### 1. Clone the repository
```bash
git clone https://github.com/yourusername/auth-app.git
cd auth-app
```
### 2. Install dependencies
```bash
npm install
```
### 3. Create `.env` file
```env
# mysql database configuration
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=your_db_name

# Application environment
NODE_ENV=development

# JWT configuration
JWT_SECRET=your_jwt_secret
JWT_EXPIRES_IN=900         # 15 minutes
REFRESH_TOKEN_EXPIRES_IN=604800

# GitHub OAuth configuration
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret
GITHUB_CALLBACK_URL=http://localhost:4000/auth/github/callback

# Google OAuth configuration
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_CALLBACK_URL=http://localhost:4000/auth/google/callback

# session secret
# Run the following command to generate a secure random secret:
#    node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
SESSION_SECRET=your_session_secret
```
### 4. Setup MySQL database
```sql
CREATE DATABASE auth_app;
USE auth_app;

-- Paste schema here
-- (users, user_auth_providers, refresh_tokens)
```
### 5. Run Backend
```bash
npm run dev
```
## Login & Oauth Entry Points
### 1. Email & Password Register
**Endpoint**
```bash
POST /api/user/register
```
**Body**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```
**What happens**
- Credentials are verified
- JWT + refresh token cookies are issued
- User is redirected or response is returned(depending on your implementation)

### 2. Email & Password Register
**Endpoint**
```bash
POST /api/user/login
```
**Body**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```
**What happens**
- Credentials are verified
- JWT + refresh token cookies are issued
- User is redirected or response is returned(depending on your implementation)

### 3. GitHub OAuth Login
**Visit in browser**
```bash
http://localhost:4000/auth/github
```
**Flow**
1. Backend redirects user to GitHub OAuth
2. User signs in with GitHub
3. Google redirects back to:
```bash
http://localhost:4000/auth/github/callback
```
4. Backend
   - verifies `state`
   - create or finds user
   - sets JWT + refresh token cookies
   - redirects user to:
```bash
http://localhost:3000/dashboard
```
**You never visit the callback URL manually**

### 4. Google OAuth Login
**Visit in browser**
```bash
http://localhost:4000/auth/google
```
**Flow**
1. Backend redirects user to Google OAuth
2. User signs in with Google
3. Google redirects back to:
```bash
http://localhost:4000/auth/google/callback
```
4. Backend
   - verifies `state`
   - create or finds user
   - sets JWT + refresh token cookies
   - redirects user to:
```bash
http://localhost:3000/dashboard
```
**Again, callback routes are not user-facing**

### 6. Security Notes
- Tokens stored as HTTP-only cookies
- Password hashed ( `bcrypt` )
- Short-lived access tokens
- Refresh token rotation & revocation
- CSRF protection via Oauth `state`
