# DJANGO AUTHENTICATION MASTER REFERENCE

## JWT • SimpleJWT • DRF TokenAuthentication • Djoser • django-allauth • SessionAuthentication • BasicAuthentication • Knox • OAuth2

---

# PART 1 — THE FUNDAMENTAL IDEA

Before learning the libraries, understand this:

```text
AUTHENTICATION
    ↓
Who are you?

AUTHORIZATION / PERMISSIONS
    ↓
What are you allowed to do?
```

Example:

```http
GET /api/orders/
Authorization: Bearer eyJ...
```

Django receives the request.

```text
HTTP request
      ↓
Authentication
      ↓
Who is this user?
      ↓
request.user
      ↓
Permissions
      ↓
Is this user allowed?
      ↓
View
```

DRF runs authentication before permission and throttling checks. A successful authentication normally populates:

```python
request.user
request.auth
```

Authentication identifies the requester; permissions determine whether that requester may access the resource.

---

# PART 2 — THE MOST IMPORTANT DRF SETTINGS

Almost every DRF authentication system eventually connects to:

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        ...
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        ...
    ],
}
```

For example:

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

The difference:

```text
DEFAULT_AUTHENTICATION_CLASSES
        ↓
How do I identify the user?

DEFAULT_PERMISSION_CLASSES
        ↓
Who is allowed into the endpoint?
```

---

# PART 3 — IMPORTANT DRF IMPORTS

You will see these imports repeatedly.

## Authentication classes

```python
from rest_framework.authentication import (
    BasicAuthentication,
    SessionAuthentication,
    TokenAuthentication,
    RemoteUserAuthentication,
)
```

## Permissions

```python
from rest_framework.permissions import (
    AllowAny,
    IsAuthenticated,
    IsAdminUser,
    IsAuthenticatedOrReadOnly,
)
```

## APIView

```python
from rest_framework.views import APIView
```

## Function-based API

```python
from rest_framework.decorators import (
    api_view,
    authentication_classes,
    permission_classes,
)
```

## Response

```python
from rest_framework.response import Response
```

## Status

```python
from rest_framework import status
```

---

# PART 4 — SESSION AUTHENTICATION

# 4.1 What is SessionAuthentication?

This is Django's traditional authentication system.

Instead of sending a token every time:

```text
Browser
   ↓
login
   ↓
Django session
   ↓
session cookie
   ↓
browser automatically sends cookie
```

DRF's `SessionAuthentication` uses Django's session framework and is particularly suited to clients operating in the same session context as the Django website. POST/PUT/PATCH/DELETE requests require CSRF protection when authenticated through sessions.

---

# 4.2 Import

```python
from rest_framework.authentication import SessionAuthentication
```

---

# 4.3 Settings

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

---

# 4.4 APIView Example

```python
from rest_framework.authentication import SessionAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView


class ProfileView(APIView):

    authentication_classes = [
        SessionAuthentication
    ]

    permission_classes = [
        IsAuthenticated
    ]

    def get(self, request):

        return Response({
            "username": request.user.username,
            "email": request.user.email,
        })
```

---

# 4.5 Important properties

After authentication:

```python
request.user
```

returns the Django user.

And:

```python
request.auth
```

is normally:

```python
None
```

for SessionAuthentication.

---

# 4.6 When should you use it?

Good for:

```text
Django templates
Django website
same-browser AJAX
admin-like interfaces
```

Less convenient for:

```text
React Native
mobile applications
separate frontend/backend architectures
```

because cookies and CSRF become important.

---

# PART 5 — BASIC AUTHENTICATION

# 5.1 What is BasicAuthentication?

The client sends:

```http
Authorization: Basic <base64(username:password)>
```

It essentially sends username/password credentials with every request.

DRF considers BasicAuthentication mainly appropriate for testing and notes that HTTPS is required if used in production.

---

# 5.2 Import

```python
from rest_framework.authentication import BasicAuthentication
```

---

# 5.3 Settings

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.BasicAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

---

# 5.4 APIView

```python
from rest_framework.authentication import BasicAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.views import APIView
from rest_framework.response import Response


class ProfileView(APIView):

    authentication_classes = [
        BasicAuthentication
    ]

    permission_classes = [
        IsAuthenticated
    ]

    def get(self, request):

        return Response({
            "user": request.user.username,
        })
```

---

# 5.5 Request

```http
GET /api/profile/
Authorization: Basic <encoded-credentials>
```

---

# 5.6 When to use

Mostly:

```text
testing
development
internal/simple situations
```

Not normally the authentication mechanism you would choose for your React Native e-commerce application.

---

# PART 6 — DRF TokenAuthentication

This is the authentication system you were asking about earlier:

```python
rest_framework.authtoken
```

It is built into Django REST Framework.

It is a **database-backed token system**.

---

# 6.1 Concept

```text
User
 ↓
Token
 ↓
Database

User:
nzegge

Token:
abc123xyz...
```

The client sends:

```http
Authorization: Token abc123xyz
```

DRF looks up that token and identifies the user.

DRF describes this as a relatively simple token implementation; unlike JWT, the token is represented by a database record.

---

# 6.2 Imports

```python
from rest_framework.authentication import TokenAuthentication
```

Token model:

```python
from rest_framework.authtoken.models import Token
```

Built-in login endpoint:

```python
from rest_framework.authtoken import views
```

---

# 6.3 Installation

You do not install a separate package for this.

You already have:

```bash
pip install djangorestframework
```

Add:

```python
INSTALLED_APPS = [
    ...
    "rest_framework",
    "rest_framework.authtoken",
]
```

Then:

```bash
python manage.py migrate
```

The `rest_framework.authtoken` app supplies the database migrations and Token model.

---

# 6.4 Settings

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

---

# 6.5 Create a token manually

```python
from rest_framework.authtoken.models import Token

token = Token.objects.create(
    user=user
)

print(token.key)
```

But normally you want:

```python
token, created = Token.objects.get_or_create(
    user=user
)

print(token.key)
```

Why `get_or_create()`?

Because you usually don't want to create another token every time.

---

# 6.6 Django shell example

```bash
python manage.py shell
```

```python
from api.models import User
from rest_framework.authtoken.models import Token

user = User.objects.get(username="nzegge")

token, created = Token.objects.get_or_create(
    user=user
)

print(token.key)
```

---

# 6.7 Built-in login URL

Import:

```python
from rest_framework.authtoken import views
```

Then:

```python
from django.urls import path
from rest_framework.authtoken import views


urlpatterns = [
    path(
        "api-token-auth/",
        views.obtain_auth_token,
        name="api-token-auth",
    ),
]
```

The URL name itself is arbitrary.

You could use:

```python
path(
    "login/",
    views.obtain_auth_token,
)
```

The official DRF view is called:

```python
obtain_auth_token
```

and accepts username/password and returns a token.

---

# 6.8 Login request

```http
POST http://127.0.0.1:8000/api-token-auth/
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
    "token": "abc123..."
}
```

---

# 6.9 Use the token

```http
GET http://127.0.0.1:8000/api/orders/
Authorization: Token abc123...
```

Important:

```text
Token
```

not:

```text
Bearer
```

---

# 6.10 APIView

```python
from rest_framework.authentication import TokenAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView


class OrderView(APIView):

    authentication_classes = [
        TokenAuthentication
    ]

    permission_classes = [
        IsAuthenticated
    ]

    def get(self, request):

        return Response({
            "user": request.user.username,
            "auth": str(request.auth),
        })
```

Here:

```python
request.user
```

is the authenticated User.

And:

```python
request.auth
```

is the Token instance.

---

# 6.11 Delete/revoke a token

There isn't JWT-style blacklisting.

You can delete the token:

```python
from rest_framework.authtoken.models import Token

Token.objects.filter(
    user=user
).delete()
```

Or:

```python
token.delete()
```

After deletion:

```text
old token
   ↓
invalid
```

---

# 6.12 Regenerate a token

DRF provides a management command:

```bash
python manage.py drf_create_token username
```

Regenerate:

```bash
python manage.py drf_create_token -r username
```

This is useful if the token has leaked.

---

# PART 7 — SIMPLEJWT

Now we reach the authentication system you are currently using.

# 7.1 What is SimpleJWT?

SimpleJWT is a JWT authentication package for DRF.

Install:

```bash
pip install djangorestframework-simplejwt
```

It provides:

```text
Access Token
Refresh Token
Verification
Blacklist
Token rotation
Custom claims
Sliding tokens
```

SimpleJWT provides the JWT authentication backend and token views/serializers.

---

# 7.2 Most important imports

Authentication:

```python
from rest_framework_simplejwt.authentication import JWTAuthentication
```

Views:

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
    TokenBlacklistView,
)
```

Serializers:

```python
from rest_framework_simplejwt.serializers import (
    TokenObtainPairSerializer,
    TokenRefreshSerializer,
    TokenVerifySerializer,
    TokenBlacklistSerializer,
)
```

Tokens:

```python
from rest_framework_simplejwt.tokens import (
    AccessToken,
    RefreshToken,
    SlidingToken,
)
```

Exceptions:

```python
from rest_framework_simplejwt.exceptions import (
    AuthenticationFailed,
    InvalidToken,
    TokenError,
)
```

---

# 7.3 Basic settings

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

The official SimpleJWT setup requires adding `JWTAuthentication` to DRF's authentication classes.

---

# 7.4 URLs

```python
from django.urls import path

from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)


urlpatterns = [

    path(
        "api/token/",
        TokenObtainPairView.as_view(),
        name="token_obtain_pair",
    ),

    path(
        "api/token/refresh/",
        TokenRefreshView.as_view(),
        name="token_refresh",
    ),

    path(
        "api/token/verify/",
        TokenVerifyView.as_view(),
        name="token_verify",
    ),
]
```

These are the standard SimpleJWT views.

---

# 7.5 Login

```http
POST /api/token/
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

# 7.6 What is the access token?

The access token is used to access protected endpoints.

```http
GET /api/orders/
Authorization: Bearer eyJ...
```

Conceptually:

```text
Access token
    ↓
short lifetime
    ↓
used frequently
```

---

# 7.7 What is the refresh token?

The refresh token is used to obtain another access token.

```http
POST /api/token/refresh/
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
    "access": "new-access-token"
}
```

The `TokenRefreshView` validates a refresh token and returns a new access token.

---

# 7.8 Verify token

URL:

```text
POST /api/token/verify/
```

Request:

```json
{
    "token": "eyJ..."
}
```

Response:

```text
HTTP 200
```

or:

```text
HTTP 401
```

Important:

`TokenVerifyView` verifies validity; it does not determine whether the token is appropriate for a particular business action.

---

# 7.9 JWT blacklist

Add:

```python
INSTALLED_APPS = [
    ...
    "rest_framework_simplejwt.token_blacklist",
]
```

Then:

```bash
python manage.py migrate
```

SimpleJWT creates:

```text
OutstandingToken
BlacklistedToken
```

and checks refresh/sliding tokens against the blacklist.

---

# 7.10 Blacklist URL

```python
from rest_framework_simplejwt.views import TokenBlacklistView


urlpatterns = [
    path(
        "api/token/blacklist/",
        TokenBlacklistView.as_view(),
        name="token_blacklist",
    ),
]
```

Request:

```http
POST /api/token/blacklist/
Content-Type: application/json
```

```json
{
    "refresh": "eyJ..."
}
```

---

# 7.11 Programmatically blacklist

```python
from rest_framework_simplejwt.tokens import RefreshToken

refresh = RefreshToken(refresh_token)

refresh.blacklist()
```

The `blacklist()` method adds the token to the blacklist when blacklist support is enabled.

---

# 7.12 AccessToken import

```python
from rest_framework_simplejwt.tokens import AccessToken
```

Example:

```python
token = AccessToken(access_token_string)

print(token["user_id"])
print(token["exp"])
print(token["token_type"])
```

---

# 7.13 RefreshToken import

```python
from rest_framework_simplejwt.tokens import RefreshToken
```

Example:

```python
refresh = RefreshToken.for_user(user)

print(refresh)
print(refresh.access_token)
```

`RefreshToken.for_user(user)` creates a refresh token for a user, and `refresh.access_token` generates the corresponding access token.

---

# 7.14 Generate tokens manually

```python
from rest_framework_simplejwt.tokens import RefreshToken


def get_tokens_for_user(user):

    refresh = RefreshToken.for_user(user)

    return {
        "refresh": str(refresh),
        "access": str(refresh.access_token),
    }
```

Usage:

```python
tokens = get_tokens_for_user(user)

print(tokens)
```

---

# 7.15 Custom JWT claims

Suppose you want:

```json
{
    "user_id": 5,
    "username": "nzegge",
    "email": "nzegge@example.com"
}
```

Create:

```python
from rest_framework_simplejwt.serializers import (
    TokenObtainPairSerializer,
)


class MyTokenObtainPairSerializer(
    TokenObtainPairSerializer
):

    @classmethod
    def get_token(cls, user):

        token = super().get_token(user)

        token["username"] = user.username
        token["email"] = user.email

        return token
```

Then:

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
)


class MyTokenObtainPairView(
    TokenObtainPairView
):

    serializer_class = MyTokenObtainPairSerializer
```

URL:

```python
path(
    "api/token/",
    MyTokenObtainPairView.as_view(),
)
```

SimpleJWT officially supports custom claims through a custom `TokenObtainPairSerializer`.

---

# 7.16 SimpleJWT settings

Common settings:

```python
from datetime import timedelta


SIMPLE_JWT = {

    "ACCESS_TOKEN_LIFETIME": timedelta(
        minutes=15
    ),

    "REFRESH_TOKEN_LIFETIME": timedelta(
        days=7
    ),

    "ROTATE_REFRESH_TOKENS": True,

    "BLACKLIST_AFTER_ROTATION": True,

    "AUTH_HEADER_TYPES": (
        "Bearer",
    ),

    "AUTH_HEADER_NAME": "HTTP_AUTHORIZATION",

    "USER_ID_FIELD": "id",

    "USER_ID_CLAIM": "user_id",

    "TOKEN_TYPE_CLAIM": "token_type",

    "JTI_CLAIM": "jti",

    "UPDATE_LAST_LOGIN": False,
}
```

Important:

```python
"AUTH_HEADER_TYPES": ("Bearer",)
```

means:

```http
Authorization: Bearer <token>
```

SimpleJWT documents `AUTH_HEADER_TYPES`, `AUTH_HEADER_NAME`, token serializers, user claims and related settings.

---

# 7.17 Refresh rotation

```python
"ROTATE_REFRESH_TOKENS": True,
"BLACKLIST_AFTER_ROTATION": True,
```

Concept:

```text
Refresh A
    ↓
refresh
    ↓
Access B
Refresh B
    ↓
Refresh A blacklisted
```

This is useful for controlling refresh-token reuse.

---

# 7.18 Delete expired blacklist records

SimpleJWT provides:

```bash
python manage.py flushexpiredtokens
```

This removes expired tokens from the outstanding/blacklist tables.

You can schedule this periodically in production.

---

# PART 8 — DJOSER

# 8.1 What is Djoser?

Djoser is **not itself a token format**.

Think:

```text
Djoser
=
ready-made REST authentication/user-management endpoints
```

It supports:

```text
TokenAuthentication
+
SimpleJWT
```

Djoser currently documents both backends.

---

# 8.2 Install

```bash
pip install djoser
```

For JWT:

```bash
pip install djangorestframework-simplejwt
```

---

# 8.3 Djoser + TokenAuthentication

Installed apps:

```python
INSTALLED_APPS = [

    ...

    "rest_framework",

    "rest_framework.authtoken",

    "djoser",
]
```

Authentication:

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

Djoser documents this configuration directly.

---

# 8.4 URLs

```python
from django.urls import include, path


urlpatterns = [

    path(
        "auth/",
        include("djoser.urls"),
    ),

    path(
        "auth/",
        include("djoser.urls.authtoken"),
    ),
]
```

---

# 8.5 Important Djoser endpoints

You get:

```text
/auth/users/
/auth/users/me/
/auth/users/set_password/
/auth/users/reset_password/
/auth/users/reset_password_confirm/

/auth/token/login/
/auth/token/logout/
```

Djoser documents these endpoints as part of its token authentication backend.

---

# 8.6 Register

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

---

# 8.7 Login

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

# 8.8 Logout

```http
POST /auth/token/logout/
Authorization: Token abc123...
```

Response:

```text
204 No Content
```

Djoser's token logout endpoint destroys the authentication token.

---

# 8.9 Current user

```http
GET /auth/users/me/
Authorization: Token abc123...
```

This gives information about the authenticated user.

---

# 8.10 Password change

```http
POST /auth/users/set_password/
Authorization: Token abc123...
```

Example:

```json
{
    "current_password": "12345678",
    "new_password": "newpassword123"
}
```

---

# 8.11 Djoser + JWT

Install:

```bash
pip install djoser
pip install djangorestframework-simplejwt
```

Settings:

```python
INSTALLED_APPS = [
    ...
    "rest_framework",
    "djoser",
]
```

JWT blacklist can also be added:

```python
INSTALLED_APPS = [
    ...
    "rest_framework_simplejwt.token_blacklist",
]
```

---

# 8.12 DRF authentication

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

Djoser specifically documents using `JWTAuthentication` for its JWT backend.

---

# 8.13 Djoser JWT URLs

```python
from django.urls import include, path


urlpatterns = [

    path(
        "auth/",
        include("djoser.urls"),
    ),

    path(
        "auth/",
        include("djoser.urls.jwt"),
    ),
]
```

---

# 8.14 JWT URLs

You get:

```text
POST /auth/jwt/create/
POST /auth/jwt/refresh/
POST /auth/jwt/verify/
```

Djoser documents exactly these three JWT endpoints.

---

# 8.15 Login

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

# 8.16 Refresh

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

# 8.17 Verify

```http
POST /auth/jwt/verify/
```

```json
{
    "token": "eyJ..."
}
```

---

# 8.18 Djoser settings

Common examples:

```python
DJOSER = {

    "LOGIN_FIELD": "username",

    "USER_ID_FIELD": "id",

    "SEND_ACTIVATION_EMAIL": False,

    "SEND_CONFIRMATION_EMAIL": False,

    "PASSWORD_CHANGED_EMAIL_CONFIRMATION": False,

    "PASSWORD_RESET_CONFIRM_URL":
        "password/reset/confirm/{uid}/{token}",

    "SET_PASSWORD_RETYPE": True,

    "PASSWORD_RESET_SHOW_EMAIL_NOT_FOUND": False,

    "TOKEN_MODEL": None,
}
```

Important:

```python
"TOKEN_MODEL": None
```

when using JWT rather than Djoser's DRF token backend.

Djoser's JWT configuration delegates JWT behavior to SimpleJWT rather than trying to duplicate SimpleJWT's settings.

---

# PART 9 — DJANGO-ALLAUTH

# 9.1 What is allauth?

allauth handles:

```text
local accounts
registration
login
email verification
password management
social login
OAuth providers
MFA
headless authentication
```

The current project documentation describes it as an integrated Django authentication, registration, account-management and third-party account system.

---

# 9.2 Install basic allauth

```bash
pip install django-allauth
```

For social providers:

```bash
pip install "django-allauth[socialaccount]"
```

For headless:

```bash
pip install "django-allauth[headless]"
```

The current quickstart documents these extras.

---

# 9.3 Basic INSTALLED_APPS

```python
INSTALLED_APPS = [

    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django.contrib.sites",

    "allauth",
    "allauth.account",
]
```

Social login:

```python
"allauth.socialaccount",
```

---

# 9.4 Authentication backends

```python
AUTHENTICATION_BACKENDS = [

    "django.contrib.auth.backends.ModelBackend",

    "allauth.account.auth_backends.AuthenticationBackend",
]
```

---

# 9.5 Sites framework

Typically:

```python
SITE_ID = 1
```

Then:

```bash
python manage.py migrate
```

---

# 9.6 Context processor

For traditional allauth views/templates:

```python
TEMPLATES = [
    {
        ...
        "OPTIONS": {
            "context_processors": [
                ...
                "django.template.context_processors.request",
            ],
        },
    },
]
```

The current allauth quickstart calls out the request context processor as required by allauth.

---

# 9.7 URLs

```python
from django.urls import include, path


urlpatterns = [

    path(
        "accounts/",
        include("allauth.urls"),
    ),
]
```

The official quickstart uses `accounts/` as the allauth URL prefix.

---

# 9.8 Traditional allauth URLs

Depending on enabled functionality, allauth provides account URLs such as:

```text
/accounts/login/
/accounts/logout/
/accounts/signup/
/accounts/password/change/
/accounts/password/reset/
```

and social-provider URLs.

---

# PART 10 — SOCIAL LOGIN WITH ALLAUTH

Suppose:

```text
React
   ↓
Google
   ↓
Google authenticates user
   ↓
allauth
   ↓
Django User
```

Providers include:

```text
Google
GitHub
Facebook
```

and many others.

---

# 10.1 Google provider

Install:

```bash
pip install "django-allauth[socialaccount]"
```

Add:

```python
INSTALLED_APPS = [

    ...

    "allauth.socialaccount",

    "allauth.socialaccount.providers.google",
]
```

---

# 10.2 GitHub provider

```python
"allauth.socialaccount.providers.github",
```

---

# 10.3 Facebook provider

```python
"allauth.socialaccount.providers.facebook",
```

---

# 10.4 Provider settings

Example Google:

```python
SOCIALACCOUNT_PROVIDERS = {

    "google": {

        "SCOPE": [
            "profile",
            "email",
        ],

        "AUTH_PARAMS": {
            "access_type": "online",
        },
    },
}
```

Current allauth documentation supports provider configuration through settings or `SocialApp` objects.

---

# PART 11 — ALLAUTH HEADLESS

This is particularly important for:

```text
React
React Native
Vue
Angular
mobile apps
SPA applications
```

because your frontend is separate from Django templates.

allauth's headless functionality provides API-oriented authentication flows and supports session-token and JWT token strategies.

---

# 11.1 Install

```bash
pip install "django-allauth[headless]"
```

With social login:

```bash
pip install "django-allauth[socialaccount,headless]"
```

---

# 11.2 Apps

```python
INSTALLED_APPS = [

    ...

    "allauth",
    "allauth.account",
    "allauth.headless",

    # Optional
    "allauth.socialaccount",
]
```

The official headless installation requires `allauth`, `allauth.account`, and `allauth.headless`.

---

# 11.3 URLs

```python
from django.urls import include, path


urlpatterns = [

    path(
        "accounts/",
        include("allauth.urls"),
    ),

    path(
        "_allauth/",
        include("allauth.headless.urls"),
    ),
]
```

The `accounts/` URLs remain useful for provider OAuth callbacks, while `_allauth/` exposes the headless API.

---

# PART 12 — ALLAUTH SESSION TOKENS

For headless/mobile authentication, allauth can use:

```text
X-Session-Token
```

instead of relying on browser cookies.

During authentication:

```http
X-Session-Token: abc123
```

allauth uses this to keep track of authentication state.

The official documentation explains that headless authentication is stateful during the authentication process and uses `X-Session-Token` for non-browser clients.

---

# 12.1 DRF authentication class

Import:

```python
from allauth.headless.contrib.rest_framework.authentication import (
    XSessionTokenAuthentication,
)
```

Use:

```python
from rest_framework.permissions import IsAuthenticated
from rest_framework.views import APIView
from rest_framework.response import Response


class ProfileView(APIView):

    authentication_classes = [
        XSessionTokenAuthentication,
    ]

    permission_classes = [
        IsAuthenticated,
    ]

    def get(self, request):

        return Response({
            "username": request.user.username,
        })
```

This authentication class is documented by allauth for DRF.

---

# PART 13 — ALLAUTH JWT STRATEGY

allauth headless can also issue:

```text
Access token
+
Refresh token
```

The access token is a JWT.

The current allauth documentation describes its JWT strategy as an access/refresh pair and provides a DRF `JWTTokenAuthentication` class.

---

# 13.1 JWT authentication import

```python
from allauth.headless.contrib.rest_framework.authentication import (
    JWTTokenAuthentication,
)
```

---

# 13.2 APIView

```python
from rest_framework.permissions import IsAuthenticated
from rest_framework.views import APIView
from rest_framework.response import Response

from allauth.headless.contrib.rest_framework.authentication import (
    JWTTokenAuthentication,
)


class ProfileView(APIView):

    authentication_classes = [
        JWTTokenAuthentication,
    ]

    permission_classes = [
        IsAuthenticated,
    ]

    def get(self, request):

        return Response({
            "username": request.user.username,
        })
```

This is directly supported by current allauth documentation.

---

# 13.3 allauth JWT settings

Current allauth provides settings such as:

```python
HEADLESS_JWT_ALGORITHM = "RS256"

HEADLESS_JWT_ACCESS_TOKEN_EXPIRES_IN = 300

HEADLESS_JWT_REFRESH_TOKEN_EXPIRES_IN = 86400

HEADLESS_JWT_AUTHORIZATION_HEADER_SCHEME = "Bearer"

HEADLESS_JWT_ROTATE_REFRESH_TOKEN = True
```

The documented defaults include RS256, 300-second access-token lifetime, 86400-second refresh-token lifetime, Bearer authorization, and refresh-token rotation.

---

# 13.4 Stateful JWT validation

allauth also provides:

```python
HEADLESS_JWT_STATEFUL_VALIDATION_ENABLED = True
```

When enabled, allauth checks that the session in which the token was issued remains valid.

This means logout/password changes can invalidate access tokens rather than leaving already-issued access tokens usable until expiry.

---

# PART 14 — KNOX

Knox is a third-party token authentication system.

It is designed to improve on the simplicity of DRF's built-in token system.

DRF's documentation describes Knox as supporting per-client tokens, token expiry, server-enforced logout and logging out all clients.

---

# 14.1 Install

```bash
pip install django-rest-knox
```

---

# 14.2 Add app

```python
INSTALLED_APPS = [
    ...
    "knox",
]
```

---

# 14.3 Migrate

```bash
python manage.py migrate
```

---

# 14.4 Authentication class

```python
from knox.auth import TokenAuthentication
```

Then:

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "knox.auth.TokenAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

---

# 14.5 Concept

DRF TokenAuthentication:

```text
One user
   ↓
One token
```

Knox can support:

```text
User
 ├── laptop token
 ├── phone token
 ├── tablet token
 └── browser token
```

This makes individual-device/session revocation practical.

---

# PART 15 — OAUTH2 / DJANGO OAUTH TOOLKIT

OAuth2 is a different concept.

JWT says:

```text
Here is a signed token proving identity.
```

OAuth2 says:

```text
This application has authorization to access resources.
```

OAuth2 is especially useful for delegated access and authorization-server scenarios.

---

# 15.1 Install

```bash
pip install django-oauth-toolkit
```

---

# 15.2 App

```python
INSTALLED_APPS = [
    ...
    "oauth2_provider",
]
```

---

# 15.3 DRF authentication

```python
from oauth2_provider.contrib.rest_framework import (
    OAuth2Authentication,
)
```

Settings:

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "oauth2_provider.contrib.rest_framework.OAuth2Authentication",
    ],
}
```

DRF documents Django OAuth Toolkit as a third-party OAuth2 implementation.

---

# PART 16 — FUNCTION-BASED DRF AUTHENTICATION

Everything above can also be used with:

```python
@api_view
```

Example JWT:

```python
from rest_framework.decorators import (
    api_view,
    authentication_classes,
    permission_classes,
)

from rest_framework.permissions import IsAuthenticated
from rest_framework_simplejwt.authentication import JWTAuthentication
from rest_framework.response import Response


@api_view(["GET"])
@authentication_classes([
    JWTAuthentication
])
@permission_classes([
    IsAuthenticated
])
def profile(request):

    return Response({
        "username": request.user.username,
    })
```

DRF officially supports configuring authentication and permissions using decorators on function-based views.

---

# PART 17 — VIEWSET AUTHENTICATION

Example:

```python
from rest_framework.viewsets import ModelViewSet
from rest_framework.permissions import IsAuthenticated
from rest_framework_simplejwt.authentication import JWTAuthentication


class OrderViewSet(ModelViewSet):

    authentication_classes = [
        JWTAuthentication
    ]

    permission_classes = [
        IsAuthenticated
    ]

    ...
```

---

# PART 18 — DIFFERENT AUTHENTICATION FOR DIFFERENT ACTIONS

This is very useful.

Suppose:

```text
GET products
```

is public.

But:

```text
POST products
DELETE products
```

require authentication.

You can implement:

```python
from rest_framework.permissions import (
    AllowAny,
    IsAuthenticated,
)


class ProductViewSet(ModelViewSet):

    def get_permissions(self):

        if self.action == "list":
            return [AllowAny()]

        return [IsAuthenticated()]
```

Authentication can still be globally configured while permissions determine access.

---

# PART 19 — MULTIPLE AUTHENTICATION CLASSES

You can have:

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}
```

DRF attempts authentication classes in order and uses the first one that successfully authenticates the request.

So:

```text
Request
   ↓
SessionAuthentication
   ↓
success?
 ┌─┴─┐
yes  no
 ↓    ↓
user JWTAuthentication
```

---

# PART 20 — AUTHENTICATION VS PERMISSION

This is extremely important.

Authentication:

```python
authentication_classes = [
    JWTAuthentication
]
```

means:

```text
"How do I identify this request?"
```

Permission:

```python
permission_classes = [
    IsAuthenticated
]
```

means:

```text
"Must the requester be authenticated?"
```

---

# PART 21 — COMMON PERMISSIONS

## AllowAny

```python
from rest_framework.permissions import AllowAny
```

Everybody can access.

Example:

```python
permission_classes = [
    AllowAny
]
```

Good for:

```text
registration
login
public products
```

---

## IsAuthenticated

```python
from rest_framework.permissions import IsAuthenticated
```

Requires authenticated user.

---

## IsAdminUser

```python
from rest_framework.permissions import IsAdminUser
```

Requires:

```python
request.user.is_staff == True
```

---

## IsAuthenticatedOrReadOnly

```python
from rest_framework.permissions import (
    IsAuthenticatedOrReadOnly,
)
```

Allows:

```text
GET
HEAD
OPTIONS
```

but requires authentication for modifying requests.

---

# PART 22 — THE `request` OBJECT

After authentication:

```python
request.user
```

gives you the user.

For example:

```python
print(request.user)
```

or:

```python
user = request.user
```

Then:

```python
user.username
user.email
user.id
```

---

# PART 23 — `request.auth`

This depends on authentication system.

### SessionAuthentication

```python
request.auth
```

usually:

```text
None
```

### TokenAuthentication

```python
request.auth
```

is:

```text
Token object
```

### JWT

```python
request.auth
```

contains the authenticated token object.

This distinction is useful when debugging authentication.

---

# PART 24 — 401 VS 403

This causes confusion for many DRF developers.

A request can fail because:

```text
Authentication failed
```

or:

```text
Permission denied
```

For example:

```text
401 Unauthorized
```

often means authentication credentials are missing/invalid.

`403 Forbidden` can occur when permission is denied or depending on the authentication class/configuration and whether an authentication challenge is available.

DRF's authentication documentation specifically notes that authentication classes influence the `WWW-Authenticate` header and whether an unauthenticated denial results in 401 or 403.

---

# PART 25 — COMPLETE SIMPLEJWT PROJECT

If you want the clean JWT architecture:

## Install

```bash
pip install djangorestframework
pip install djangorestframework-simplejwt
```

---

## settings.py

```python
INSTALLED_APPS = [

    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",

    "rest_framework",

    "rest_framework_simplejwt.token_blacklist",

    "api",
]
```

---

## DRF

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

---

## JWT

```python
from datetime import timedelta


SIMPLE_JWT = {

    "ACCESS_TOKEN_LIFETIME": timedelta(
        minutes=15
    ),

    "REFRESH_TOKEN_LIFETIME": timedelta(
        days=7
    ),

    "ROTATE_REFRESH_TOKENS": True,

    "BLACKLIST_AFTER_ROTATION": True,

    "AUTH_HEADER_TYPES": (
        "Bearer",
    ),

    "AUTH_HEADER_NAME":
        "HTTP_AUTHORIZATION",

    "USER_ID_FIELD":
        "id",

    "USER_ID_CLAIM":
        "user_id",

    "TOKEN_TYPE_CLAIM":
        "token_type",

    "JTI_CLAIM":
        "jti",

    "UPDATE_LAST_LOGIN":
        False,
}
```

---

## urls.py

```python
from django.urls import path

from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
    TokenBlacklistView,
)


urlpatterns = [

    path(
        "api/token/",
        TokenObtainPairView.as_view(),
        name="token_obtain_pair",
    ),

    path(
        "api/token/refresh/",
        TokenRefreshView.as_view(),
        name="token_refresh",
    ),

    path(
        "api/token/verify/",
        TokenVerifyView.as_view(),
        name="token_verify",
    ),

    path(
        "api/token/blacklist/",
        TokenBlacklistView.as_view(),
        name="token_blacklist",
    ),
]
```

---

# PART 26 — COMPLETE DJOSER + JWT PROJECT

## Install

```bash
pip install djangorestframework
pip install djangorestframework-simplejwt
pip install djoser
```

---

## settings.py

```python
INSTALLED_APPS = [

    ...

    "rest_framework",

    "djoser",

    "rest_framework_simplejwt.token_blacklist",

    "api",
]
```

---

## DRF

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

---

## JWT

```python
from datetime import timedelta


SIMPLE_JWT = {

    "ACCESS_TOKEN_LIFETIME":
        timedelta(minutes=15),

    "REFRESH_TOKEN_LIFETIME":
        timedelta(days=7),

    "ROTATE_REFRESH_TOKENS":
        True,

    "BLACKLIST_AFTER_ROTATION":
        True,

    "AUTH_HEADER_TYPES":
        ("Bearer",),
}
```

---

## Djoser

```python
DJOSER = {

    "TOKEN_MODEL": None,

    "USER_ID_FIELD": "id",

    "LOGIN_FIELD": "username",

    "SEND_ACTIVATION_EMAIL": False,

    "SEND_CONFIRMATION_EMAIL": False,

    "SET_PASSWORD_RETYPE": True,
}
```

---

## URLs

```python
from django.urls import include, path


urlpatterns = [

    path(
        "auth/",
        include("djoser.urls"),
    ),

    path(
        "auth/",
        include("djoser.urls.jwt"),
    ),
]
```

---

# PART 27 — DJOSER + TOKEN PROJECT

If you instead want traditional DRF tokens:

```bash
pip install djoser
```

Settings:

```python
INSTALLED_APPS = [

    ...

    "rest_framework",

    "rest_framework.authtoken",

    "djoser",
]
```

DRF:

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

URLs:

```python
urlpatterns = [

    path(
        "auth/",
        include("djoser.urls"),
    ),

    path(
        "auth/",
        include("djoser.urls.authtoken"),
    ),
]
```

Endpoints:

```text
POST /auth/users/
POST /auth/token/login/
POST /auth/token/logout/
GET  /auth/users/me/
```

---

# PART 28 — ALLAUTH + SOCIAL LOGIN

Install:

```bash
pip install "django-allauth[socialaccount]"
```

Apps:

```python
INSTALLED_APPS = [

    ...

    "django.contrib.sites",

    "allauth",
    "allauth.account",
    "allauth.socialaccount",

    "allauth.socialaccount.providers.google",
    "allauth.socialaccount.providers.github",
]
```

Backend:

```python
AUTHENTICATION_BACKENDS = [

    "django.contrib.auth.backends.ModelBackend",

    "allauth.account.auth_backends.AuthenticationBackend",
]
```

Site:

```python
SITE_ID = 1
```

URLs:

```python
urlpatterns = [

    path(
        "accounts/",
        include("allauth.urls"),
    ),
]
```

Run:

```bash
python manage.py migrate
```

---

# PART 29 — ALLAUTH + HEADLESS + SOCIAL

Install:

```bash
pip install "django-allauth[socialaccount,headless]"
```

Apps:

```python
INSTALLED_APPS = [

    ...

    "allauth",
    "allauth.account",
    "allauth.headless",

    "allauth.socialaccount",

    "allauth.socialaccount.providers.google",
    "allauth.socialaccount.providers.github",
]
```

URLs:

```python
urlpatterns = [

    path(
        "accounts/",
        include("allauth.urls"),
    ),

    path(
        "_allauth/",
        include("allauth.headless.urls"),
    ),
]
```

This follows the current headless installation structure.

---

# PART 30 — WHICH ONE SHOULD YOU USE?

Don't think:

```text
"Which library is universally best?"
```

Instead ask:

```text
"What authentication requirements does my application have?"
```

---

## Requirement: Simple API token

Use:

```text
DRF TokenAuthentication
```

---

## Requirement: JWT API

Use:

```text
SimpleJWT
```

---

## Requirement: JWT + ready user-management endpoints

Use:

```text
Djoser + SimpleJWT
```

---

## Requirement: Social login

Use:

```text
django-allauth
```

---

## Requirement: React/React Native + social login + JWT

Possible architecture:

```text
allauth
   ↓
Google/GitHub/etc.
   ↓
Django User

SimpleJWT
   ↓
API access
```

---

## Requirement: Headless allauth

Use:

```text
django-allauth[headless]
```

and choose:

```text
Session Token
```

or:

```text
JWT Token Strategy
```

allauth officially provides both headless strategies.

---

# PART 31 — QUICK IMPORT CHEAT SHEET

## DRF

```python
from rest_framework.authentication import (
    BasicAuthentication,
    SessionAuthentication,
    TokenAuthentication,
    RemoteUserAuthentication,
)

from rest_framework.permissions import (
    AllowAny,
    IsAuthenticated,
    IsAdminUser,
    IsAuthenticatedOrReadOnly,
)

from rest_framework.decorators import (
    api_view,
    authentication_classes,
    permission_classes,
)

from rest_framework.views import APIView

from rest_framework.response import Response
```

---

# SimpleJWT

```python
from rest_framework_simplejwt.authentication import (
    JWTAuthentication,
)

from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
    TokenBlacklistView,
)

from rest_framework_simplejwt.serializers import (
    TokenObtainPairSerializer,
    TokenRefreshSerializer,
    TokenVerifySerializer,
    TokenBlacklistSerializer,
)

from rest_framework_simplejwt.tokens import (
    AccessToken,
    RefreshToken,
    SlidingToken,
)
```

---

# DRF Token

```python
from rest_framework.authtoken.models import Token

from rest_framework.authtoken import views

from rest_framework.authentication import (
    TokenAuthentication,
)
```

---

# allauth

```python
from allauth.headless.contrib.rest_framework.authentication import (
    XSessionTokenAuthentication,
)

from allauth.headless.contrib.rest_framework.authentication import (
    JWTTokenAuthentication,
)
```

---

# Knox

```python
from knox.auth import TokenAuthentication
```

---

# OAuth Toolkit

```python
from oauth2_provider.contrib.rest_framework import (
    OAuth2Authentication,
)
```

---

# PART 32 — THE AUTHENTICATION FLOW COMPARISON

## Session

```text
username/password
       ↓
Django login
       ↓
session
       ↓
cookie
       ↓
request
```

---

## DRF Token

```text
username/password
       ↓
Token
       ↓
database
       ↓
Authorization: Token xxx
```

---

## SimpleJWT

```text
username/password
       ↓
SimpleJWT
       ↓
access + refresh
       ↓
Authorization: Bearer xxx
```

---

## Djoser + Token

```text
username/password
       ↓
Djoser
       ↓
DRF TokenAuthentication
       ↓
auth_token
```

---

## Djoser + JWT

```text
username/password
       ↓
Djoser
       ↓
SimpleJWT
       ↓
access + refresh
```

---

## allauth Social

```text
React
  ↓
Google/GitHub/etc.
  ↓
Provider
  ↓
allauth
  ↓
Django User
```

---

## allauth Headless JWT

```text
React / React Native
        ↓
allauth headless
        ↓
authentication flow
        ↓
JWT access + refresh
        ↓
API
```

---

# PART 33 — TOKEN STORAGE CONCEPT

Your frontend receives:

```text
access token
refresh token
```

The access token is sent to your API:

```http
Authorization: Bearer <access>
```

When it expires:

```text
refresh token
       ↓
new access token
```

Do not put passwords into every API request.

---

# PART 34 — COMPLETE E-COMMERCE EXAMPLE

Your application could have:

```text
PUBLIC
GET /api/products/

AUTHENTICATED
GET /api/orders/
POST /api/orders/

ADMIN
POST /api/products/
PUT /api/products/1/
DELETE /api/products/1/
```

Settings:

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],

    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

Product:

```python
from rest_framework.permissions import AllowAny


class ProductViewSet(ModelViewSet):

    def get_permissions(self):

        if self.action == "list":
            return [AllowAny()]

        return [IsAuthenticated()]
```

Then:

```text
GET /products/
       ↓
public

POST /orders/
       ↓
JWT required

DELETE /products/5/
       ↓
authentication + custom admin permission
```

---

# PART 35 — A CUSTOM ADMIN PERMISSION

```python
from rest_framework.permissions import BasePermission


class IsStaffUser(BasePermission):

    def has_permission(self, request, view):

        return (
            request.user
            and request.user.is_authenticated
            and request.user.is_staff
        )
```

Use:

```python
permission_classes = [
    IsStaffUser
]
```

---

# PART 36 — THE MOST IMPORTANT CLASSES TO MEMORIZE

You don't need to memorize every class immediately.

Start with these:

```text
DRF
──────────────────────────────

SessionAuthentication
TokenAuthentication
BasicAuthentication

AllowAny
IsAuthenticated
IsAdminUser
```

Then:

```text
SimpleJWT
──────────────────────────────

JWTAuthentication

TokenObtainPairView
TokenRefreshView
TokenVerifyView
TokenBlacklistView

RefreshToken
AccessToken

TokenObtainPairSerializer
```

Then:

```text
Djoser
──────────────────────────────

djoser.urls
djoser.urls.authtoken
djoser.urls.jwt
```

Then:

```text
allauth
──────────────────────────────

allauth.account
allauth.socialaccount
allauth.headless

JWTTokenAuthentication
XSessionTokenAuthentication
```

---

# PART 37 — YOUR CURRENT PROJECT

For your current Django REST project, you are already using:

```text
Django
+
DRF
+
custom api.User
+
SimpleJWT
```

So your most important imports are:

```python
from rest_framework.permissions import (
    IsAuthenticated,
)

from rest_framework_simplejwt.authentication import (
    JWTAuthentication,
)

from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)
```

And your frontend sends:

```http
Authorization: Bearer <access_token>
```

Your API then gets:

```python
request.user
```

which is your authenticated:

```python
api.User
```

---

# PART 38 — THE THREE LEVELS YOU SHOULD REMEMBER

This is the single most useful mental model.

## Level 1 — Authentication mechanism

```text
Session
Token
JWT
OAuth
```

## Level 2 — Authentication package

```text
SimpleJWT
Knox
OAuth Toolkit
allauth
```

## Level 3 — API/user-management layer

```text
Djoser
allauth
```

For example:

```text
Djoser
   +
SimpleJWT
```

means:

```text
Djoser
= user-management/API endpoints

SimpleJWT
= JWT authentication
```

---

# PART 39 — FINAL DECISION TABLE

| Requirement                 | Main technology               |
| --------------------------- | ----------------------------- |
| Django website              | SessionAuthentication         |
| Simple API token            | DRF TokenAuthentication       |
| JWT                         | SimpleJWT                     |
| JWT refresh                 | SimpleJWT                     |
| JWT blacklist               | SimpleJWT                     |
| JWT rotation                | SimpleJWT                     |
| User registration endpoints | Djoser                        |
| Password-reset API          | Djoser                        |
| Token login/logout API      | Djoser + DRF Token            |
| JWT login/refresh API       | Djoser + SimpleJWT            |
| Google login                | allauth                       |
| GitHub login                | allauth                       |
| Facebook login              | allauth                       |
| MFA                         | allauth                       |
| Headless authentication     | allauth Headless              |
| Headless JWT                | allauth Headless JWT strategy |
| Multiple device tokens      | Knox                          |
| OAuth2 authorization server | Django OAuth Toolkit          |

---

# PART 40 — THE BIG PICTURE

Remember this diagram:

```text
                         DJANGO USER
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
          SESSION           TOKEN              JWT
             │                │                │
             ▼                ▼                ▼
       SessionAuth       DRF TokenAuth       SimpleJWT
                                                │
                                                │
                                  ┌─────────────┼─────────────┐
                                  │             │             │
                                  ▼             ▼             ▼
                                Access       Refresh       Blacklist
```

Then:

```text
                         USER MANAGEMENT
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
                  Djoser             allauth
                    │                   │
                    │                   ├── Google
                    │                   ├── GitHub
                    │                   ├── Facebook
                    │                   ├── MFA
                    │                   └── Headless
                    │
                    ├── registration
                    ├── password reset
                    ├── password change
                    └── user endpoints
```

And finally:

```text
                 YOUR REACT / REACT NATIVE APP
                              │
                              ▼
                    Django REST Framework
                              │
                              ▼
                       Authentication
                              │
              ┌───────────────┼───────────────┐
              │               │               │
           Session          Token             JWT
              │               │               │
              │               │               │
              └───────────────┼───────────────┘
                              ▼
                         request.user
                              │
                              ▼
                          Permissions
                              │
                              ▼
                            View
                              │
                              ▼
                          Database
```

The most important combinations to understand deeply are:

```text
1. DRF + TokenAuthentication

2. DRF + SimpleJWT

3. Djoser + TokenAuthentication

4. Djoser + SimpleJWT

5. allauth + Social Login

6. allauth Headless + JWT

7. SimpleJWT + allauth
```

Do **not** install all seven together. Each combination solves a different problem.
