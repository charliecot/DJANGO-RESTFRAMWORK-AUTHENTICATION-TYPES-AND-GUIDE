# Django REST Authentication — Complete Guide

## 1. The Big Picture

Authentication answers:

> **"Who is making this API request?"**

Authorization/permissions answer:

> **"Is this authenticated user allowed to do this?"**

For example:

```text
React / React Native
        |
        | username + password
        v
      LOGIN
        |
        v
 Authentication system
        |
        v
 access credential
        |
        v
 Authorization header
        |
        v
 Django REST Framework
        |
        v
 authentication_classes
        |
        v
 request.user
        |
        v
 permission_classes
        |
        v
 IsAuthenticated
        |
        v
 API endpoint
```

DRF authentication runs before permission checks. Successful authentication normally populates:

```python
request.user
request.auth
```

Authentication itself does not decide whether access is permitted; permissions do that.

---

# 2. The Main Authentication Choices

| System                  | What it is                     |        Login |                   Refresh | Logout/revoke |               Social login |         Blacklist |
| ----------------------- | ------------------------------ | -----------: | ------------------------: | ------------: | -------------------------: | ----------------: |
| SessionAuthentication   | Django sessions/cookies        | Django login |             Session-based |           Yes |               With allauth |     Session-based |
| DRF TokenAuthentication | Simple database token          |          Yes |                        No |  Delete token |                         No | Not JWT blacklist |
| SimpleJWT               | JWT authentication             |          Yes |                       Yes | Via blacklist |                         No |               Yes |
| Djoser + Token          | REST user API + DRF Token      |          Yes |                        No |           Yes |                         No |    Token deletion |
| Djoser + JWT            | REST user API + SimpleJWT      |          Yes |                       Yes | JWT mechanism |                         No |               Yes |
| django-allauth          | Account/social authentication  |          Yes | Depends on token strategy |           Yes |                    **Yes** |           Depends |
| allauth Headless + JWT  | API-oriented allauth           |          Yes |                       Yes |           Yes |                    **Yes** |    Token strategy |
| Knox                    | Advanced token authentication  |          Yes |           Token lifecycle |           Yes |                         No |  Token revocation |
| OAuth Toolkit           | OAuth 2.0 authorization server |   OAuth flow |              OAuth tokens |    Revocation | Possible through providers |  OAuth revocation |

DRF itself describes its built-in `TokenAuthentication` as a fairly simple implementation and points to Knox when more advanced token behavior is needed.

---

# 3. OPTION 1 — SimpleJWT

## What is SimpleJWT?

SimpleJWT is a Django REST Framework package that implements:

```text
JSON Web Token authentication
```

It normally gives you:

```text
Access Token
+
Refresh Token
```

The standard endpoints are:

```text
POST /api/token/
POST /api/token/refresh/
```

You can optionally add:

```text
POST /api/token/verify/
POST /api/token/blacklist/
```

The official SimpleJWT documentation defines `TokenObtainPairView` as the endpoint that accepts credentials and returns an access/refresh pair.

---

# 4. Install SimpleJWT

```bash
pip install djangorestframework-simplejwt
```

---

# 5. Basic SimpleJWT Settings

```python
# settings.py

INSTALLED_APPS = [
    # Django
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Third-party
    'rest_framework',

    # JWT blacklist
    'rest_framework_simplejwt.token_blacklist',

    # Your apps
    'api',
]
```

Then:

```bash
python manage.py migrate
```

The blacklist app creates the tables used to track outstanding and blacklisted refresh/sliding tokens.

---

# 6. Configure DRF

```python
REST_FRAMEWORK = {

    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],

    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

Now protected endpoints expect:

```http
Authorization: Bearer <access_token>
```

---

# 7. SimpleJWT Settings

A practical configuration:

```python
from datetime import timedelta

SIMPLE_JWT = {

    # Access token
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),

    # Refresh token
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),

    # Refresh token rotation
    'ROTATE_REFRESH_TOKENS': True,

    # Blacklist old refresh token after rotation
    'BLACKLIST_AFTER_ROTATION': True,

    # Update last_login
    'UPDATE_LAST_LOGIN': False,

    # Authorization header
    'AUTH_HEADER_TYPES': ('Bearer',),

    # Header name
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',

    # Token type
    'TOKEN_TYPE_CLAIM': 'token_type',

    # Unique token identifier
    'JTI_CLAIM': 'jti',

    # User identification
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',

    # Authentication class
    'AUTH_TOKEN_CLASSES': (
        'rest_framework_simplejwt.tokens.AccessToken',
    ),
}
```

SimpleJWT supports configurable access/refresh lifetimes, rotation, blacklist behavior, token classes, token claims, and serializers.

---

# 8. SimpleJWT URLs

```python
# urls.py

from django.urls import path

from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
    TokenBlacklistView,
)

urlpatterns = [

    path(
        'api/token/',
        TokenObtainPairView.as_view(),
        name='token_obtain_pair'
    ),

    path(
        'api/token/refresh/',
        TokenRefreshView.as_view(),
        name='token_refresh'
    ),

    path(
        'api/token/verify/',
        TokenVerifyView.as_view(),
        name='token_verify'
    ),

    path(
        'api/token/blacklist/',
        TokenBlacklistView.as_view(),
        name='token_blacklist'
    ),
]
```

---

# 9. SimpleJWT Login

Request:

```http
POST http://127.0.0.1:8000/api/token/
Content-Type: application/json
```

```json
{
    "username": "nzegge",
    "password": "12345678"
}
```

Response:

```json
{
    "refresh": "eyJ...",
    "access": "eyJ..."
}
```

---

# 10. Access Protected API

```http
GET http://127.0.0.1:8000/api/orders/
Authorization: Bearer eyJ...
```

The backend authenticates the access token.

Then:

```python
request.user
```

represents the authenticated user.

---

# 11. Refresh Access Token

When the access token expires:

```http
POST http://127.0.0.1:8000/api/token/refresh/
Content-Type: application/json
```

```json
{
    "refresh": "eyJ..."
}
```

Response:

```json
{
    "access": "new_access_token"
}
```

---

# 12. JWT Blacklist

This answers one of your main questions:

## Which authentication supports token blacklist?

**SimpleJWT does.**

Install:

```python
'rest_framework_simplejwt.token_blacklist',
```

Then migrate:

```bash
python manage.py migrate
```

SimpleJWT tracks outstanding tokens and blacklisted tokens and provides `TokenBlacklistView`.

Logout:

```http
POST /api/token/blacklist/
Authorization: Bearer <access-token>
```

Depending on the blacklist endpoint, the request body contains the refresh token being revoked:

```json
{
    "refresh": "eyJ..."
}
```

---

# 13. Refresh Token Rotation

This is an important security feature:

```python
SIMPLE_JWT = {
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
}
```

Conceptually:

```text
Refresh Token A
       |
       v
Refresh
       |
       +------> Access Token B
       |
       +------> Refresh Token B
                   |
                   v
             Token A revoked
```

SimpleJWT's blacklist documentation specifically describes blacklisting rotated refresh tokens when rotation and blacklist support are enabled.

---

# 14. OPTION 2 — DRF TokenAuthentication

This is built directly into Django REST Framework.

Package:

```python
rest_framework.authtoken
```

It stores tokens in your database.

---

# 15. Install/configure DRF TokenAuthentication

No separate package is required because it comes with DRF.

Add:

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework.authtoken',
]
```

Then:

```bash
python manage.py migrate
```

Configure:

```python
REST_FRAMEWORK = {

    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],

    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

DRF's official documentation confirms that `rest_framework.authtoken` provides the database-backed token model and migrations.

---

# 16. Create Token Manually

```python
from rest_framework.authtoken.models import Token

token = Token.objects.create(user=user)

print(token.key)
```

Or:

```python
token, created = Token.objects.get_or_create(
    user=user
)
```

---

# 17. Token Login URL

DRF provides:

```python
from rest_framework.authtoken import views

urlpatterns = [
    path(
        'api-token-auth/',
        views.obtain_auth_token
    ),
]
```

Then:

```http
POST /api-token-auth/
```

```json
{
    "username": "nzegge",
    "password": "12345678"
}
```

Response:

```json
{
    "token": "9944b09199c..."
}
```

DRF officially provides this `obtain_auth_token` endpoint for obtaining a token from username/password credentials.

---

# 18. Use DRF Token

```http
GET /api/orders/
Authorization: Token 9944b09199c...
```

Notice:

```text
Token
```

not:

```text
Bearer
```

So:

```text
DRF TokenAuthentication
Authorization: Token abc123
```

while:

```text
SimpleJWT
Authorization: Bearer eyJ...
```

---

# 19. TokenAuthentication Logout

There is no JWT-style refresh/blacklist system.

You normally revoke the token by deleting it:

```python
token.delete()
```

Or:

```python
Token.objects.filter(user=user).delete()
```

Therefore:

```text
JWT
→ expiration + blacklist

DRF Token
→ database token remains valid until revoked/deleted
```

DRF itself describes the built-in implementation as fairly simple.

---

# 20. OPTION 3 — Djoser

This is where many beginners get confused.

## Djoser is NOT another token type.

Djoser is a:

```text
REST API authentication/user-management endpoint generator
```

It provides ready-made endpoints for things such as:

```text
registration
activation
login
logout
password reset
password change
user information
```

It can work with:

```text
DRF TokenAuthentication
```

or:

```text
SimpleJWT
```

Djoser's documentation explicitly supports both token-based and JWT authentication.

---

# 21. Djoser + DRF TokenAuthentication

Install:

```bash
pip install djoser djangorestframework
```

Settings:

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework.authtoken',
    'djoser',
]
```

DRF:

```python
REST_FRAMEWORK = {

    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],

    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

---

# 22. Djoser Token URLs

```python
from django.urls import include, path

urlpatterns = [

    path(
        'auth/',
        include('djoser.urls')
    ),

    path(
        'auth/',
        include('djoser.urls.authtoken')
    ),
]
```

Djoser documents this exact combination for token authentication.

---

# 23. Djoser Token URLs

You get:

```text
POST /auth/users/
POST /auth/token/login/
POST /auth/token/logout/
GET  /auth/users/me/
POST /auth/users/set_password/
POST /auth/users/reset_password/
...
```

The important token URLs are:

```text
POST /auth/token/login/
POST /auth/token/logout/
```

Djoser's token documentation identifies `/token/login/` as the default token-create endpoint and `/token/logout/` as the token-destroy endpoint.

---

# 24. Djoser Registration

```http
POST /auth/users/
Content-Type: application/json
```

Example:

```json
{
    "username": "nzegge",
    "password": "12345678",
    "re_password": "12345678",
    "email": "nzegge@example.com"
}
```

Djoser handles the user-management API for you.

---

# 25. Djoser Token Login

```http
POST /auth/token/login/
```

```json
{
    "username": "nzegge",
    "password": "12345678"
}
```

Response:

```json
{
    "auth_token": "abc123..."
}
```

---

# 26. Djoser Token Logout

```http
POST /auth/token/logout/
Authorization: Token abc123...
```

The token is destroyed.

---

# 27. Djoser + JWT

This is probably the "two login URLs" system you remembered.

Install:

```bash
pip install djoser djangorestframework-simplejwt
```

Settings:

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'djoser',
]
```

You don't need:

```python
'rest_framework.authtoken',
```

if you're using Djoser purely with JWT.

DRF:

```python
REST_FRAMEWORK = {

    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],

    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

Djoser:

```python
DJOSER = {
    'TOKEN_MODEL': None,
}
```

Djoser's JWT documentation notes that its settings do not configure the JWT resources themselves; SimpleJWT controls those settings.

---

# 28. Djoser JWT URLs

```python
from django.urls import include, path

urlpatterns = [

    path(
        'auth/',
        include('djoser.urls')
    ),

    path(
        'auth/',
        include('djoser.urls.jwt')
    ),
]
```

You now get:

```text
POST /auth/jwt/create/
POST /auth/jwt/refresh/
POST /auth/jwt/verify/
```

Djoser documents these as its JWT create, refresh, and verify endpoints.

---

# 29. Djoser JWT Login

```http
POST /auth/jwt/create/
```

```json
{
    "username": "nzegge",
    "password": "12345678"
}
```

Response:

```json
{
    "access": "eyJ...",
    "refresh": "eyJ..."
}
```

---

# 30. Djoser JWT Refresh

```http
POST /auth/jwt/refresh/
```

```json
{
    "refresh": "eyJ..."
}
```

Response:

```json
{
    "access": "eyJ..."
}
```

---

# 31. Djoser vs SimpleJWT

This distinction is extremely important.

```text
SimpleJWT
    =
JWT authentication engine
```

while:

```text
Djoser
    =
ready-made authentication/user-management API
```

Therefore:

```text
Djoser
   +
SimpleJWT
```

is completely valid.

Djoser gives you:

```text
registration
user management
password management
JWT URLs
```

while SimpleJWT gives you:

```text
JWT creation
JWT validation
JWT refresh
JWT blacklist
JWT configuration
```

---

# 32. OPTION 4 — django-allauth

`django-allauth` is different from Djoser.

Its major strengths are:

```text
Account management
+
Email verification
+
Password management
+
Social login
+
Google
+
GitHub
+
Facebook
+
X
+
many other providers
+
MFA
+
headless API
```

The current allauth documentation describes it as a Django authentication/account-management system with third-party/social authentication, and its headless functionality is designed for SPA/API clients.

---

# 33. Install allauth

For normal account + social functionality:

```bash
pip install "django-allauth[socialaccount]"
```

For headless/API functionality:

```bash
pip install "django-allauth[headless]"
```

You may use:

```bash
pip install "django-allauth[socialaccount,headless]"
```

---

# 34. Basic allauth Settings

```python
INSTALLED_APPS = [

    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.sites',

    'allauth',
    'allauth.account',
    'allauth.socialaccount',

    # Providers
    'allauth.socialaccount.providers.google',
    'allauth.socialaccount.providers.github',

]
```

Depending on the version/configuration you use, allauth also requires its authentication backend:

```python
AUTHENTICATION_BACKENDS = [

    'django.contrib.auth.backends.ModelBackend',

    'allauth.account.auth_backends.AuthenticationBackend',
]
```

---

# 35. allauth URLs

Traditional Django/allauth:

```python
from django.urls import include, path

urlpatterns = [

    path(
        'accounts/',
        include('allauth.urls')
    ),
]
```

This provides account/social endpoints.

The official quickstart uses:

```text
/accounts/
```

for allauth URLs.

---

# 36. Social Providers

For Google:

```python
INSTALLED_APPS = [
    ...
    'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
]
```

For GitHub:

```python
'allauth.socialaccount.providers.github',
```

You can configure provider credentials using `SocialApp` objects or settings. Allauth's current documentation describes both approaches.

---

# 37. Google Configuration Example

```python
SOCIALACCOUNT_PROVIDERS = {

    'google': {

        'SCOPE': [
            'profile',
            'email',
        ],

        'AUTH_PARAMS': {
            'access_type': 'online',
        },

    },
}
```

Google's allauth documentation also documents PKCE support and provider-specific configuration.

---

# 38. Why Allauth Is Special

Imagine your application has:

```text
Sign up
Login
Logout
Email verification
Password reset
Google login
GitHub login
Facebook login
MFA
Account connections
```

Building all of those manually is a lot of work.

Allauth provides infrastructure for them.

For example:

```text
User
 |
 +-- Local account
 |
 +-- Google account
 |
 +-- GitHub account
 |
 +-- Facebook account
```

One Django user can have connected external accounts.

---

# 39. allauth Headless

This is particularly relevant to your React/React Native projects.

Current allauth has a **Headless** API designed for applications where Django does not render the frontend.

Install:

```bash
pip install "django-allauth[headless]"
```

Settings include:

```python
INSTALLED_APPS = [
    ...
    'allauth',
    'allauth.account',
    'allauth.headless',
]
```

Then:

```python
urlpatterns = [

    path(
        'accounts/',
        include('allauth.urls')
    ),

    path(
        '_allauth/',
        include('allauth.headless.urls')
    ),
]
```

The official headless installation documentation shows this structure.

---

# 40. allauth Headless Token Strategies

Current allauth supports token strategies including:

```text
Session Tokens
JWT Tokens
```

Its JWT strategy issues:

```text
access token
+
refresh token
```

and supports JWT authentication for DRF.

---

# 41. allauth JWT + DRF

You can authenticate a DRF API with allauth's JWT authentication class:

```python
from allauth.headless.contrib.rest_framework.authentication import (
    JWTTokenAuthentication,
)
```

Then:

```python
class ProductView(APIView):

    authentication_classes = [
        JWTTokenAuthentication,
    ]

    permission_classes = [
        IsAuthenticated,
    ]
```

Allauth documents this DRF integration directly.

---

# 42. allauth vs Djoser

This is a useful mental model:

```text
Djoser
    =
REST user-management API

allauth
    =
account + social authentication ecosystem
```

Djoser is particularly convenient when you want:

```text
React
      ↓
REST API
      ↓
Djoser
      ↓
JWT
```

Allauth is particularly useful when you want:

```text
React
      ↓
Django
      ↓
allauth
      ├── Email/password
      ├── Google
      ├── GitHub
      ├── Facebook
      ├── MFA
      └── other providers
```

---

# 43. OPTION 5 — SessionAuthentication

This is Django's traditional authentication model.

The browser receives a session cookie.

Conceptually:

```text
POST login
     ↓
Django creates session
     ↓
Browser receives cookie
     ↓
Browser sends cookie automatically
     ↓
Django identifies user
```

DRF supports this through:

```python
'rest_framework.authentication.SessionAuthentication'
```

It is especially appropriate for AJAX clients sharing the same session context as a Django website. Unsafe methods require CSRF protection.

Configuration:

```python
REST_FRAMEWORK = {

    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

This is normally more natural for:

```text
Django templates
+
Django website
```

than for a completely separate React Native mobile application.

---

# 44. OPTION 6 — BasicAuthentication

DRF also has:

```python
'rest_framework.authentication.BasicAuthentication'
```

Request:

```http
Authorization: Basic base64(username:password)
```

It is mainly useful for testing and should be used over HTTPS. DRF explicitly notes that BasicAuthentication is generally appropriate for testing rather than normal production API authentication.

---

# 45. OPTION 7 — Knox

Knox is an alternative to basic DRF TokenAuthentication.

DRF's own documentation points to Knox as a more secure/extensible token-based solution that supports multiple tokens per user and token expiry.

Conceptually:

```text
User
 |
 +-- Phone token
 |
 +-- Laptop token
 |
 +-- Browser token
 |
 +-- Tablet token
```

You can revoke individual tokens.

This is useful when you want:

```text
multiple devices
+
token expiration
+
server-side token revocation
```

without using JWT.

---

# 46. OPTION 8 — OAuth Toolkit

Django OAuth Toolkit implements OAuth 2.0 functionality.

Install:

```bash
pip install django-oauth-toolkit
```

Settings:

```python
INSTALLED_APPS = [
    ...
    'oauth2_provider',
]
```

DRF:

```python
REST_FRAMEWORK = {

    'DEFAULT_AUTHENTICATION_CLASSES': [
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
    ],
}
```

DRF identifies Django OAuth Toolkit as its recommended third-party package for OAuth 2.0 support.

OAuth is especially relevant when your application needs an authorization server and delegated access rather than simply "my React app logs into my Django API."

---

# 47. The Most Important Difference

Do not think:

```text
JWT
Djoser
allauth
TokenAuthentication
```

are four competing versions of the exact same thing.

Think:

```text
                    AUTHENTICATION STACK

              ┌──────────────────────┐
              │     User Management  │
              │                      │
              │ Djoser / allauth     │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ Credential mechanism │
              │                      │
              │ JWT / Token / Session│
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ DRF Authentication   │
              │                      │
              │ JWTAuthentication    │
              │ TokenAuthentication  │
              │ SessionAuthentication│
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │     Permissions      │
              │                      │
              │ IsAuthenticated      │
              │ IsAdminUser          │
              │ custom permissions   │
              └──────────────────────┘
```

---

# 48. Complete Comparison

## SimpleJWT

```text
Package:
djangorestframework-simplejwt

Purpose:
JWT authentication

Best for:
REST APIs
React
React Native
Mobile applications

Provides:
Access token
Refresh token
Token verification
Token blacklist
Token rotation
```

Typical:

```text
POST /api/token/
POST /api/token/refresh/
POST /api/token/blacklist/
```

---

# 49. DRF TokenAuthentication

```text
Package:
Django REST Framework itself

Purpose:
Simple database token authentication

Best for:
Simple APIs
Small projects
Simple mobile clients
Learning
```

Provides:

```text
Token
```

Not:

```text
JWT
Refresh token
JWT blacklist
```

---

# 50. Djoser

```text
Package:
djoser

Purpose:
Ready-made REST authentication/user-management endpoints
```

Can use:

```text
DRF Token
OR
JWT
```

Provides endpoints for:

```text
registration
activation
login
logout
password change
password reset
user details
```

---

# 51. django-allauth

```text
Package:
django-allauth
```

Purpose:

```text
account management
+
social authentication
+
OAuth providers
+
email verification
+
MFA
+
headless authentication
```

Excellent when you need:

```text
Google
GitHub
Facebook
etc.
```

---

# 52. Knox

```text
Package:
django-rest-knox
```

Purpose:

```text
advanced token authentication
```

Useful for:

```text
multiple tokens per user
token expiration
token revocation
mobile/device tokens
```

---

# 53. SessionAuthentication

```text
Django sessions
```

Best for:

```text
Django website
Django templates
same-browser AJAX
```

---

# 54. BasicAuthentication

Best mainly for:

```text
testing
development
internal APIs
```

Use HTTPS if used beyond testing.

---

# 55. OAuth Toolkit

```text
OAuth 2.0 authorization server
```

Useful for:

```text
delegated authorization
third-party applications
OAuth2 clients
authorization-server scenarios
```

---

# 56. What Supports Blacklisting?

This is one of your key questions.

### SimpleJWT

```text
YES
```

Use:

```python
'rest_framework_simplejwt.token_blacklist'
```

and:

```python
'ROTATE_REFRESH_TOKENS': True,
'BLACKLIST_AFTER_ROTATION': True,
```

SimpleJWT provides `OutstandingToken` and `BlacklistedToken` models for this purpose.

### DRF TokenAuthentication

```text
NO JWT blacklist
```

Instead:

```python
token.delete()
```

### Djoser + Token

```text
Token deletion
```

rather than JWT blacklist.

### Djoser + JWT

```text
YES
```

because Djoser is using SimpleJWT underneath.

### allauth

Depends on its token strategy. Its current headless JWT strategy uses access/refresh tokens and includes mechanisms for invalidation, while its session-token strategy is stateful.

### Knox

```text
YES — server-side token revocation
```

---

# 57. The "Two URLs" You Remember

There are actually several similar situations.

## SimpleJWT

```text
/api/token/
```

Login/access + refresh pair.

```text
/api/token/refresh/
```

Refresh access token.

---

## Djoser + JWT

```text
/auth/jwt/create/
```

Login.

```text
/auth/jwt/refresh/
```

Refresh.

```text
/auth/jwt/verify/
```

Verify.

---

## Djoser + TokenAuthentication

```text
/auth/token/login/
```

Login.

```text
/auth/token/logout/
```

Logout.

So if you remembered:

```text
login
logout
```

that is likely:

```text
Djoser + TokenAuthentication
```

If you remembered:

```text
create
refresh
verify
```

that is:

```text
Djoser + JWT
```

---

# 58. Recommended Architecture for Your React/React Native API

For a project such as your e-commerce API:

```text
React / React Native
          |
          |
          v
       Django
          |
          +----------------+
          |                |
          v                v
      SimpleJWT        allauth
          |                |
          |                |
       normal          social login
       login           Google/GitHub
          |                |
          +-------+--------+
                  |
                  v
             Django User
```

A very clean architecture is:

```text
Normal login
      ↓
SimpleJWT

Google/GitHub/Facebook
      ↓
django-allauth
      ↓
same Django User
```

This means:

```text
SimpleJWT
```

handles your normal API JWT authentication while:

```text
allauth
```

handles external identity providers.

---

# 59. Alternative Architecture — Djoser + JWT

If you want less authentication code of your own:

```text
React
  |
  v
Djoser
  |
  +---- Registration
  |
  +---- Login
  |
  +---- Logout/user management
  |
  v
SimpleJWT
  |
  +---- Access token
  +---- Refresh token
  +---- Blacklist
```

This is a very practical REST API architecture.

---

# 60. Complete Djoser + SimpleJWT Example

## Install

```bash
pip install djangorestframework
pip install djangorestframework-simplejwt
pip install djoser
```

## settings.py

```python
INSTALLED_APPS = [

    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'rest_framework',

    'djoser',

    'rest_framework_simplejwt.token_blacklist',

    'api',
]
```

DRF:

```python
REST_FRAMEWORK = {

    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],

    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

JWT:

```python
from datetime import timedelta

SIMPLE_JWT = {

    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),

    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),

    'ROTATE_REFRESH_TOKENS': True,

    'BLACKLIST_AFTER_ROTATION': True,

    'AUTH_HEADER_TYPES': ('Bearer',),

    'UPDATE_LAST_LOGIN': False,
}
```

Djoser:

```python
DJOSER = {
    'TOKEN_MODEL': None,
}
```

---

# 61. urls.py

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [

    path(
        'admin/',
        admin.site.urls
    ),

    path(
        'auth/',
        include('djoser.urls')
    ),

    path(
        'auth/',
        include('djoser.urls.jwt')
    ),
]
```

Run:

```bash
python manage.py migrate
```

---

# 62. Your API Becomes

```text
POST /auth/users/
```

Register.

```text
POST /auth/jwt/create/
```

Login.

```text
POST /auth/jwt/refresh/
```

Refresh.

```text
POST /auth/jwt/verify/
```

Verify.

```text
GET /auth/users/me/
```

Current user.

```text
POST /auth/users/set_password/
```

Change password.

```text
POST /auth/users/reset_password/
```

Request password reset.

---

# 63. Add JWT Blacklist

Add:

```python
'rest_framework_simplejwt.token_blacklist',
```

Then:

```bash
python manage.py migrate
```

Add:

```python
from rest_framework_simplejwt.views import TokenBlacklistView
```

and:

```python
path(
    'auth/jwt/blacklist/',
    TokenBlacklistView.as_view(),
    name='jwt-blacklist'
),
```

Now:

```text
POST /auth/jwt/blacklist/
```

can revoke a refresh token.

---

# 64. React Frontend Flow

Login:

```text
React
  |
  | POST /auth/jwt/create/
  |
  | username/password
  v
Django
  |
  v
access + refresh
```

Store/use the credentials appropriately.

Then:

```http
Authorization: Bearer <access>
```

When access expires:

```text
React
  |
  | POST /auth/jwt/refresh/
  |
  | refresh
  v
new access token
```

Logout:

```text
React
  |
  | blacklist refresh token
  v
Django
  |
  v
refresh token revoked
```

---

# 65. A Very Important Security Concept

Never confuse:

```text
authentication
```

with:

```text
authorization
```

Example:

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],

    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

The first line says:

```text
"How do I identify the user?"
```

The second says:

```text
"Must the user be logged in?"
```

You can have:

```text
JWTAuthentication
+
AllowAny
```

for a public endpoint.

Or:

```text
JWTAuthentication
+
IsAuthenticated
```

for a protected endpoint.

---

# 66. Authentication Request Lifecycle

Suppose:

```http
GET /api/orders/
Authorization: Bearer eyJ...
```

DRF approximately does:

```text
HTTP request
     |
     v
Authentication
     |
     v
JWTAuthentication
     |
     v
Validate token
     |
     v
Find/identify user
     |
     v
request.user
     |
     v
Permission
     |
     v
IsAuthenticated
     |
     v
View
```

This is why:

```python
request.user
```

works inside your DRF views.

---

# 67. What I Would Learn in Order

For your learning path, understand these in this order:

```text
1. Django User
       ↓
2. SessionAuthentication
       ↓
3. DRF TokenAuthentication
       ↓
4. JWT
       ↓
5. SimpleJWT
       ↓
6. Djoser
       ↓
7. django-allauth
       ↓
8. OAuth
       ↓
9. MFA / advanced security
```

Do not try to learn all of them as if you must install all of them together.

---

# 68. What You Should NOT Install Together by Default

Don't blindly do this:

```python
'rest_framework.authtoken',
'djoser',
'allauth',
'rest_framework_simplejwt',
'oauth2_provider',
'knox',
```

just because they are authentication packages.

That creates unnecessary complexity.

Instead choose an architecture.

---

# 69. Architecture A — Simple JWT Only

```text
React
  ↓
SimpleJWT
  ↓
Django REST Framework
```

Use when:

```text
normal username/password
+
JWT
```

Packages:

```bash
pip install djangorestframework
pip install djangorestframework-simplejwt
```

---

# 70. Architecture B — Djoser + JWT

```text
React
  ↓
Djoser
  ↓
SimpleJWT
  ↓
DRF
```

Use when you want ready-made:

```text
registration
login
password reset
password change
user endpoints
JWT
```

Packages:

```bash
pip install djangorestframework
pip install djangorestframework-simplejwt
pip install djoser
```

---

# 71. Architecture C — JWT + Allauth

```text
                    ┌── Google
                    │
React/React Native ─┼── GitHub
                    │
                    └── Facebook
                         |
                         v
                      allauth
                         |
                         v
                      Django User
                         ^
                         |
                      SimpleJWT
                         ^
                         |
                    normal login
```

Use when:

```text
normal login
+
social login
```

are required.

---

# 72. Architecture D — Djoser + JWT + Allauth

You can combine them, but understand their responsibilities:

```text
Djoser
    ↓
REST user management

SimpleJWT
    ↓
JWT authentication

allauth
    ↓
social authentication
```

This can be powerful, but it is more complex.

---

# 73. My Practical Comparison

Rather than a "best" ranking, choose based on your requirements:

| Requirement                             | Appropriate choice            |
| --------------------------------------- | ----------------------------- |
| Simple REST token                       | DRF TokenAuthentication       |
| JWT API                                 | SimpleJWT                     |
| JWT + ready user endpoints              | Djoser + SimpleJWT            |
| Google/GitHub/etc.                      | django-allauth                |
| JWT + social login                      | SimpleJWT + allauth           |
| Ready REST user management              | Djoser                        |
| Multiple revocable tokens/device tokens | Knox                          |
| Django website                          | SessionAuthentication         |
| OAuth2 authorization server             | Django OAuth Toolkit          |
| React Native API                        | SimpleJWT                     |
| React SPA + API                         | SimpleJWT or allauth Headless |
| Token blacklist                         | SimpleJWT                     |
| Refresh tokens                          | SimpleJWT                     |
| Email verification/account workflows    | allauth or Djoser             |
| MFA                                     | allauth                       |

---

# 74. Final Mental Model

Remember this:

```text
                         DJANGO AUTH
                             |
             +---------------+---------------+
             |               |               |
             v               v               v
          SESSION          TOKEN            JWT
             |               |               |
             |               |               |
       SessionAuth      DRF TokenAuth    SimpleJWT
                                             |
                                             |
                                      +------+------+
                                      |             |
                                      v             v
                                   access        refresh
                                                   |
                                                   v
                                               blacklist
```

Then add the API-management libraries:

```text
                     USER/API MANAGEMENT
                              |
                    +---------+---------+
                    |                   |
                    v                   v
                  Djoser             allauth
                    |                   |
                    |                   +-- Google
                    |                   +-- GitHub
                    |                   +-- Facebook
                    |                   +-- MFA
                    |
                    +-- registration
                    +-- password reset
                    +-- user management
```

The key combinations are:

```text
SimpleJWT
```

for JWT itself.

```text
Djoser + SimpleJWT
```

for ready-made REST authentication/user endpoints using JWT.

```text
allauth + SimpleJWT
```

for normal JWT authentication plus social providers.

```text
Djoser + SimpleJWT + allauth
```

when you specifically need both Djoser's REST user-management endpoints and allauth's social/account ecosystem.

---

# 75. Official Documentation

DRF authentication:
https://www.django-rest-framework.org/api-guide/authentication/

SimpleJWT:
https://django-rest-framework-simplejwt.readthedocs.io/

Djoser:
https://djoser.readthedocs.io/

django-allauth:
https://docs.allauth.org/

Django OAuth Toolkit:
https://django-oauth-toolkit.readthedocs.io/

These are the primary documentation sources to keep alongside this reference.
