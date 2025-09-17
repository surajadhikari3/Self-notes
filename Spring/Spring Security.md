
| Question                                                                | Sample Answer                                                                                                                                                                                            |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Q1:** How does Spring Security work under the hood?                   | It uses a **filter chain** that intercepts HTTP requests, performs **authentication**, then checks **authorization** before allowing access. All security data is stored in the `SecurityContextHolder`. |
| **Q2:** How do you secure REST APIs?                                    | Use **JWT tokens** with stateless sessions. Disable CSRF for APIs and enable `HttpBasic` or `Bearer` token authentication.                                                                               |
| **Q3:** What is the role of `UserDetailsService`?                       | It is used to fetch user credentials from a custom source like a database during authentication.                                                                                                         |
| **Q4:** What’s the difference between `hasRole()` and `hasAuthority()`? | `hasRole("ADMIN")` checks for `ROLE_ADMIN`. `hasAuthority("ROLE_ADMIN")` checks the authority directly.                                                                                                  |
| **Q5:** How does Spring Security prevent CSRF?                          | It adds a token to each request and validates it on the server. This prevents external malicious sites from submitting forms on behalf of the user.                                                      |


JWT (Json web token) Based security) 

## Typical Interview Questions on JWT

| Question                                     | Sample Answer                                                                                                                             |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Q:** Why use JWT instead of sessions?      | JWT enables **stateless authentication**, making it ideal for distributed or microservice architectures. No server-side storage required. |
| **Q:** How do you validate JWT?              | Using the secret key, verify signature and expiration. Parse the token to extract the subject (username).                                 |
| **Q:** Where do you store JWT in the client? | Typically in **localStorage** or **Authorization header** (preferable for security).                                                      |
| **Q:** What are JWT vulnerabilities?         | XSS (if stored in localStorage), token leakage, no logout capability unless token is invalidated or blacklisted.                          |


> In stateless JWT authentication, CSRF protection is **not required** because tokens are manually sent in the `Authorization` header — **not automatically attached** by the browser like cookies. Since the server doesn’t rely on sessions, the typical CSRF attack vector doesn't exist.

“CSRF happens because browsers automatically attach cookies, like session IDs, to cross-site requests. That’s why in session-based authentication we need CSRF tokens. With JWT-based authentication, tokens are usually sent in the `Authorization` header or manually attached by the client, so a malicious site cannot force the browser to include the token. This removes the automatic CSRF risk. However, JWTs are still vulnerable to XSS if stored in localStorage, and if you store them in cookies, you still need CSRF protection.”

---

## 🧠 How to Remember?

**Stateless + Token Header → No Cookie → No CSRF**

# Cookie vs JWT — Direct Comparison Chart

| Aspect                      | Cookie-based (Session)             | JWT-based (Token)                              |
| --------------------------- | ---------------------------------- | ---------------------------------------------- |
| Server Storage              | YES (Session in server memory)     | NO (Stateless)                                 |
| Browser sends automatically | YES (cookies auto-attached)        | NO (manual Authorization header)               |
| Scalability                 | Poor (session replication needed)  | Excellent (no session tracking)                |
| CSRF risk                   | YES (cookies auto-send)            | NO (tokens sent manually)                      |
| Performance                 | Session lookup needed              | Only token decoding needed                     |
| Logout/Invalidate           | Easy (delete session)              | Hard (token must expire or blacklist manually) |
| Typical Usage               | Traditional web apps (login forms) | APIs, Mobile apps, Microservices               |


Monilithic application --> State(Sessions is maintained)

User login --> Server validates And generates the token with jsession-id---> Sends the cookie to the client. --> Browser stores the cookie and sends cookie with every request --> Server validates. the cookie with the jsession id and serves the resource


Here since the browser stored the jsession id hacker can use the jsession id to make the malicious call then the csrf token is needed-->

Server fetch the csrf token in the hidden field in the html and based on the csrf token and the jsession id the server authorize the users..

For interview answers..

Cookie-based auth stores user sessions on the server, which breaks down in distributed systems due to the need for sticky sessions or a shared session store. This adds latency, complexity, and violates microservice statelessness.  

JWT solves this by being stateless and self-contained — any microservice can validate the token without extra network calls or storage, making it ideal for scalable, cloud-native architectures.

![[Screenshot 2025-04-29 at 4.29.03 PM.png]]




User Sends HTTP Request
        ↓
SpringSecurityFilterChain
    - SecurityContextPersistenceFilter
    - Authentication Filters (e.g., UsernamePasswordAuthenticationFilter or JWT Filter)
    - Authorization Filters (FilterSecurityInterceptor)
        ↓ (if authorized)
DispatcherServlet
    ↓
HandlerMapping
    ↓
Controller
    ↓
Service Layer
    ↓
DAO Layer (Database)
    ↓
Response goes back up through DispatcherServlet


SecurityContextHolder (ThreadLocal)
    └── SecurityContext
            └── Authentication
                    ├── Principal (UserDetails)
                    ├── Credentials (password, token)
                    └── Authorities (roles/permissions)

In the stateless authentication with spring security using JWT need to set the SecurityContextHolder on every request


### What happens on each request:

1. The JWT is sent in the HTTP request (usually as `Authorization: Bearer <token>`).
    
2. A **custom filter** (often extending `OncePerRequestFilter`) intercepts the request.
    
3. The filter:
    
    - Extracts the JWT
        
    - Validates it
        
    - Parses the claims (username, roles, etc.)
        
    - Builds an `Authentication` object
        
    - **Sets it into `SecurityContextHolder`**
        
4. After that, the request continues with an authenticated context.

### Summary Comparison Table:

| Aspect                     | **Stateful**                  | **Stateless (JWT)**                |
| -------------------------- | ----------------------------- | ---------------------------------- |
| Storage                    | Server-side session           | Client-side token                  |
| `SecurityContextHolder`    | Auto-restored from session    | Must be set manually every request |
| `UserDetailsService` usage | Only at login                 | Every request (if needed)          |
| Scalability                | Limited (session sync needed) | Highly scalable                    |
| Logout                     | Invalidate session            | Token expiration/blacklist         |
| Best for                   | Web apps                      | APIs, SPAs, microservices          |

Authentication Decision Flow


          What are you building?
               |
     +---------+----------+
     |                    |
   UI App              REST API
     |                    |
Form Login         Use Token (JWT)
     |                    |
 Need SSO?            Need SSO?  (Single sign on)
     |                    |
 Use OAuth2           Use OAuth2 or API Gateway
     |
Enterprise login?
     |
 Use LDAP or SAML


| Method         | Stateless? | Best For                      | Notes                      | How Credentials Are Passed / User Validated                                                                                                  |
| -------------- | ---------- | ----------------------------- | -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Form Login** | ❌          | UI apps with server rendering | Uses session               | User submits form (POST `/login`), credentials extracted by `UsernamePasswordAuthenticationFilter` and validated using `UserDetailsService`  |
| **Basic Auth** | ❌          | Internal APIs/testing         | Not secure without HTTPS   | `Authorization: Basic base64(user:pass)` header; decoded and validated by `BasicAuthenticationFilter` and `UserDetailsService`               |
| **JWT**        | ✅          | REST APIs, mobile, SPA        | Scalable, stateless        | `Authorization: Bearer <JWT>` header; token validated by a custom JWT filter, user extracted from claims, no session                         |
| **OAuth2**     | ✅          | Social login, SSO             | Complex but powerful       | External OAuth provider (e.g., Google) authenticates, Spring exchanges auth code for access token, user info fetched via `OAuth2UserService` |
| **LDAP**       | ❌          | Enterprise apps               | External directory service | Credentials submitted like form login; validated using `LdapAuthenticationProvider` via bind or search on LDAP server                        |
| **Custom**     | Depends    | Special login rules           | Plug in any logic          | Credentials can be passed via any method (e.g., headers, body, token); validation handled via custom `AuthenticationProvider`                |

for sending the credentials in the basic auth we send in the request headers

curl -u john:1234 https://active-life-canada/api/data

This automatically sets:
```
`Authorization: Basic am9objoxMjM0`
```
since the credentials is base64 encoded..


![[Pasted image 20250505131954.png]]