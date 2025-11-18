# User Authentication with Authorization Code Flow

## Overview

Extend Keycloak security from Lab 5.2 to support user-based authentication using OAuth2 Authorization Code Flow. This enables end-user login through front-end applications with JWT tokens.

**Time:** 60-90 minutes

---

## Prerequisites

- ✅ Completed Week 11-1 (Keycloak Client Credentials)
- ✅ Keycloak running with `spring-microservices-security-realm`
- ✅ API Gateway configured with OAuth2 Resource Server
- ✅ Postman installed

---

## Background

Lab 5.2 implemented **service-to-service** authentication using Client Credentials. Lab 5.3 adds **user-based** authentication for front-end applications (web, mobile, SPA) where actual users log in with credentials.

### Authorization Code Flow

This OAuth2 flow is designed for user authentication:

1. **User Login**: User redirected to Keycloak login page
2. **Authorization Code**: Keycloak returns authorization code after successful login
3. **Token Exchange**: Application exchanges code for JWT access token
4. **API Access**: Access token sent in `Authorization: Bearer <token>` header

**When to Use:**
- Front-end applications requiring user login
- Role-based access control (RBAC)
- Single Sign-On (SSO) scenarios

**Key Benefits:**
- Secure user session management
- Centralized user authentication
- Support for refresh tokens

---

## Step 1: Create Frontend OAuth2 Client

### 1.1 Access Keycloak Admin Console

Ensure Keycloak is running:

```bash
# Stop integrated docker-compose if running (to avoid port conflicts)
cd microservices-parent
docker-compose -p microservices-parent down

# Check if Keycloak is running
docker ps | grep keycloak

# If not running, start standalone Keycloak
cd docker/standalone/keycloak
docker-compose -p keycloak-standalone up -d
```

Access admin console:

1. Navigate to `http://localhost:8080`
2. Click **Administration Console**
3. Login with `admin` / `password`
4. Select **spring-microservices-security-realm** (dropdown, top-left)

### 1.2 Create Frontend Client

Create new OAuth2 client for user authentication:

1. Click **Clients** (left sidebar)
2. Click **Create Client** button

**Step 1: General Settings**

- Client Type: `OpenID Connect`
- Client ID: `frontend-client`
- Name: `Frontend Client`
- Click **Next**

**Step 2: Capability Config**

Configure client authentication settings:

- Client Authentication: **OFF** (public client for front-end)
- Authorization: **OFF**
- Authentication flow:
  - ☑ Standard Flow (Authorization Code Flow)
  - ☐ Direct Access Grants
  - ☐ Implicit Flow
  - ☐ Service Accounts Roles
  - ☐ OAuth 2.0 Device Authorization Grant
  - ☐ OIDC CIBA Grant

Click **Next**

**Step 3: Login Settings**

Configure redirect URIs:

- Root URL: (leave blank)
- Home URL: (leave blank)
- Valid Redirect URIs: `https://oauth.pstmn.io/v1/callback`
- Valid Post Logout Redirect URIs: (leave blank)
- Web Origins: `*`

Click **Save**

### 1.3 Understanding Redirect Configuration

**Valid Redirect URIs:**

Purpose: Specifies allowed URLs where Keycloak redirects after authentication. The redirect URI in authentication requests must exactly match this entry for security.

Lab 5.3 Requirement: Postman uses `https://oauth.pstmn.io/v1/callback` as its OAuth 2.0 callback URL. This must be explicitly allowed.

**Web Origins:**

Purpose: Controls Cross-Origin Resource Sharing (CORS) for JavaScript clients making API calls to Keycloak.

Lab 5.3 Requirement: Setting `*` allows CORS from any origin (simplified for testing). In production, restrict to specific domains (e.g., `https://yourapp.com`).

### 1.4 Verify Client Configuration

1. Click **Clients** → **frontend-client**
2. Verify **Settings** tab shows:
   - Client authentication: **OFF**
   - Standard flow enabled: **ON**
   - Valid redirect URIs: `https://oauth.pstmn.io/v1/callback`

---

## Step 2: Create Test User

### 2.1 Add User

Create user account for testing:

1. Click **Users** (left sidebar)
2. Click **Create new user** button

**User Configuration:**

- Email verified: **Yes** (toggle ON)
- Username: `testuser` (required field)
- Email: `testuser@example.com`
- First name: `Test`
- Last name: `User`
- Click **Create**

### 2.2 Set User Password

1. After creating user, click **Credentials** tab
2. Click **Set password** button

**Password Settings:**

- Password: `password`
- Password confirmation: `password`
- Temporary: **OFF** (toggle OFF to make password permanent)

Click **Save**

Confirm password reset in dialog.

### 2.3 Assign User Roles (Optional)

Grant user access to view resources:

1. Stay on testuser details page
2. Click **Role mappings** tab
3. Click **Assign role** button
4. Filter: Select **Filter by clients**
5. Search: `realm-management`
6. Select: `view-users` role
7. Click **Assign**

This allows the user to have basic realm viewing permissions.

---

## Step 3: Export Realm Configuration

### 3.1 Export Realm from Keycloak

Export realm to include frontend-client and testuser:

1. Stay in Keycloak Admin Console
2. Select **spring-microservices-security-realm**
3. Click **Realm settings** (left sidebar)
4. Click **Action** dropdown (top-right)
5. Select **Partial export**
6. Check **Export groups and roles**
7. Check **Export clients**
8. Click **Export**
9. Save as `realm-export.json`

### 3.2 Update Realm Files

Copy exported realm to both integrated and standalone directories:

```bash
cd microservices-parent

# Backup old realm export (if exists)
mv docker/integrated/keycloak/realms/realm-export.json docker/integrated/keycloak/realms/realm-export.json.bak 2>/dev/null || true
mv docker/standalone/keycloak/realms/realm-export.json docker/standalone/keycloak/realms/realm-export.json.bak 2>/dev/null || true

# Copy new export to both locations
cp ~/Downloads/realm-export.json docker/integrated/keycloak/realms/
cp ~/Downloads/realm-export.json docker/standalone/keycloak/realms/
```

### 3.3 Stop Standalone Keycloak

Stop standalone Keycloak before starting integrated stack:

```bash
cd docker/standalone/keycloak
docker-compose -p keycloak-standalone down
```

---

## Step 4: Verify API Gateway Configuration

### 4.1 Check application.properties

The API Gateway from Lab 5.2 already validates JWTs. User-based tokens from Authorization Code Flow use the same validation mechanism.

**Location:** `api-gateway/src/main/resources/application.properties`

Verify issuer URI is configured:

```properties
spring.application.name=api-gateway
server.port=9000

# Services
services.product-url=http://localhost:8084
services.order-url=http://localhost:8082

# Keycloak Security
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:8080/realms/spring-microservices-security-realm
```

**Location:** `api-gateway/src/main/resources/application-docker.properties`

For Docker environment:

```properties
spring.application.name=api-gateway
server.port=9000

# Services
services.product-url=http://product-service:8084
services.order-url=http://order-service:8082

# Keycloak Security (Docker)
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://keycloak:8080/realms/spring-microservices-security-realm
```

### 4.2 Verify SecurityConfig

No changes needed to SecurityConfig. It validates all JWTs (service and user tokens) the same way.

**Location:** `api-gateway/src/main/java/ca/gbc/apigateway/config/SecurityConfig.java`

Confirm it contains:

```java
@Slf4j
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {

        log.info("Initializing Security Filter Chain...");

        return httpSecurity
                .csrf(AbstractHttpConfigurer::disable)
                .authorizeHttpRequests(authorize -> authorize
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2
                        .jwt(Customizer.withDefaults()))
                .build();
    }
}
```

This configuration:
- Requires authentication for all requests
- Validates JWT tokens using Keycloak's public key
- Works for both Client Credentials and Authorization Code flows

---

## Step 5: Test User Authentication with Postman

### 5.1 Start All Services

```bash
cd microservices-parent
docker-compose -p microservices-parent -f docker-compose.yml up -d --build
```

Wait ~30 seconds for services to stabilize.

Verify services are running:

```bash
docker ps
```

Expected containers:
- keycloak
- postgres-keycloak
- api-gateway
- product-service
- order-service
- inventory-service
- mongodb
- redis

### 5.2 Configure Postman OAuth 2.0

Create new request or use existing:

1. Open Postman
2. Create request: `GET http://localhost:9000/api/product`
3. Click **Authorization** tab
4. Type: Select **OAuth 2.0**
5. Configure New Token: Scroll down and click **Configure New Token**

**Token Configuration:**

- Token Name: `Keycloak User Token`
- Grant Type: `Authorization Code`
- Callback URL: `https://oauth.pstmn.io/v1/callback`
- Auth URL: `http://localhost:8080/realms/spring-microservices-security-realm/protocol/openid-connect/auth`
- Access Token URL: `http://localhost:8080/realms/spring-microservices-security-realm/protocol/openid-connect/token`
- Client ID: `frontend-client`
- Client Secret: (leave blank - public client)
- Scope: `openid profile email`
- State: `randomstring`
- Client Authentication: `Send as Basic Auth header`

Click **Get New Access Token**

### 5.3 Authenticate as User

Postman will open browser window to Keycloak login:

1. Keycloak login page appears (`http://localhost:8080/...`)
2. Enter credentials:
   - Username or email: `testuser`
   - Password: `password`
3. Click **Sign In**

After successful login:
- Keycloak redirects to Postman callback URL
- Postman exchanges authorization code for access token
- Token appears in Postman dialog

Click **Use Token**

### 5.4 Test Secured Endpoint

Make API request:

1. Ensure request is `GET http://localhost:9000/api/product`
2. Authorization tab shows `OAuth 2.0` with token selected
3. Click **Send**

**Expected Result:** `200 OK` with product data

**Example Response:**

```json
[
  {
    "id": "507f1f77bcf86cd799439011",
    "name": "Sample Product",
    "description": "Product description",
    "price": 29.99
  }
]
```

### 5.5 Test Additional Endpoints

Test other microservices endpoints:

**Order Service:**

```
GET http://localhost:9000/api/order
```

**Product Service (POST):**

```
POST http://localhost:9000/api/product
Content-Type: application/json

{
  "name": "New Product",
  "description": "Created via user auth",
  "price": 49.99
}
```

**Note:** Remember to update token if expired (JWT tokens typically expire after 5 minutes). Click **Get New Access Token** if you receive `401 Unauthorized`.

---

## Step 6: Debugging

### 6.1 Login Fails in Browser

**Check:**

- Redirect URI in Keycloak: `https://oauth.pstmn.io/v1/callback`
- Keycloak accessibility: `http://localhost:8080`
- Correct Auth URL in Postman
- User credentials: `testuser` / `password`

**Verify User:**

1. Keycloak Admin → Users
2. Search for `testuser`
3. Check Email Verified: **Yes**
4. Credentials tab: Password is set and not temporary

### 6.2 API Call Returns 401 Unauthorized

**Check:**

- Token is selected in Postman Authorization tab
- API Gateway running: `http://localhost:9000`
- JWT issuer URI matches in `application.properties`

**Verify Token:**

1. Copy access token from Postman
2. Go to https://jwt.io
3. Paste token in Debugger
4. Verify claims:
   - `iss`: `http://localhost:8080/realms/spring-microservices-security-realm`
   - `sub`: `<user-id>`
   - `preferred_username`: `testuser`

### 6.3 Token Expired

Symptoms: `401 Unauthorized` with valid configuration

Solution:
1. In Postman, scroll to Authorization section
2. Click **Get New Access Token**
3. Login again
4. Click **Use Token**

### 6.4 Check Keycloak Logs

```bash
docker logs keycloak
```

Look for authentication events and errors.

### 6.5 Check API Gateway Logs

```bash
docker logs api-gateway
```

Look for JWT validation errors or security filter chain issues.

---

## Step 7: Understanding OAuth2 Flows Comparison

### Client Credentials Flow (Lab 5.2)

**Use Case:** Service-to-service authentication

**Flow:**
1. Service sends `client_id` + `client_secret` to Keycloak
2. Keycloak returns JWT access token
3. Service uses token for API requests

**When to Use:**
- Backend microservices communication
- No user involved
- Trusted systems

### Authorization Code Flow (Lab 5.3)

**Use Case:** User authentication

**Flow:**
1. User redirected to Keycloak login
2. User enters username/password
3. Keycloak returns authorization code
4. App exchanges code for access token
5. App uses token for API requests with user context

**When to Use:**
- Web applications with user login
- Mobile applications
- SPAs (Single Page Applications)
- Role-based access control

### Key Differences

| Aspect | Client Credentials | Authorization Code |
|--------|-------------------|-------------------|
| **Authentication** | Service | User |
| **Client Type** | Confidential | Public or Confidential |
| **User Interaction** | None | Login required |
| **Token Contains** | Service identity | User identity + roles |
| **Refresh Token** | No | Yes (optional) |
| **Use Case** | Backend APIs | Frontend apps |

---

## Troubleshooting

### Postman Cannot Open Browser

Error: Browser window doesn't open when clicking "Get New Access Token"

**Solution:**
1. Click **Authorize using browser** checkbox
2. Try again
3. If still fails, manually copy Auth URL and open in browser
4. Complete login
5. Copy authorization code from redirect URL
6. Paste in Postman

### CORS Error in Browser

Error: CORS policy blocks request

**Solution:**
1. Keycloak Admin → Clients → frontend-client
2. Set Web Origins to `*` (testing) or specific domain (production)
3. Save

### Invalid Redirect URI

Error: "Invalid redirect_uri"

**Solution:**
1. Verify Callback URL in Postman: `https://oauth.pstmn.io/v1/callback`
2. Verify Valid Redirect URIs in Keycloak matches exactly
3. No trailing slashes
4. HTTPS required for Postman callback

### User Account Disabled

Error: "Account is disabled"

**Solution:**
1. Keycloak Admin → Users → testuser
2. Check **Enabled** toggle is ON
3. Check **Email Verified** is ON
4. Save

---

## Summary

You now have:

- ✅ Frontend OAuth2 client for user authentication
- ✅ Test user account with credentials
- ✅ Authorization Code Flow configured
- ✅ Postman testing setup for user login
- ✅ Updated realm export with frontend-client and users

**Key Achievements:**
- Understand OAuth2 Authorization Code Flow
- Configure Keycloak for user authentication
- Test user-based API access via Postman
- Distinguish between service and user authentication flows

**Next Steps:**
- Implement role-based access control (RBAC)
- Add refresh token support
- Create front-end application for user login
- Implement PKCE for enhanced security
- Configure Single Sign-On (SSO) across multiple apps
