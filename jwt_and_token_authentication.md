# Django REST Framework Authentication

# Complete DRF TokenAuthentication + SimpleJWT Guide

This document covers two major authentication systems:

```text
1. DRF TokenAuthentication
2. SimpleJWT
```

It explains both:

```text
NORMAL / DEFAULT USAGE
        +
CUSTOMIZATION
        +
CUSTOM SERIALIZERS
        +
CUSTOM VIEWS
        +
CUSTOM TOKENS
        +
LOGIN
        +
LOGOUT
        +
REFRESH
        +
BLACKLIST
        +
USER DATA
        +
FILES / IMAGES
        +
MULTIPLE DEVICES
        +
SECURITY
        +
COMMON MISTAKES
```

---

# PART 1 — Authentication Fundamentals

## 1. What is Authentication?

Authentication answers:

> "Who are you?"

Example:

```text
username = nzegge
password = ********
```

Django checks those credentials.

If correct:

```text
User authenticated
```

If incorrect:

```text
Authentication failed
```

---

# 2. Authentication vs Authorization

These are different.

## Authentication

```text
WHO ARE YOU?
```

Example:

```python
request.user
```

## Authorization

```text
WHAT ARE YOU ALLOWED TO DO?
```

Example:

```python
request.user.is_staff
```

or:

```python
IsAuthenticated
```

or:

```python
IsAdminUser
```

---

# 3. Basic Authentication Flow

```text
Username + Password
        │
        ▼
    authenticate()
        │
        ▼
      User
        │
        ▼
   Authentication
        │
        ▼
 Token / JWT
        │
        ▼
 Future API requests
```

---

# PART 2 — DRF TokenAuthentication

# 4. What Is DRF TokenAuthentication?

DRF provides a simple token system through:

```python
rest_framework.authtoken
```

The token is stored in the database.

Conceptually:

```text
User
 │
 ▼
Token
 │
 ▼
Database
```

---

# 5. Install / Enable Token Authentication

Add:

```python
INSTALLED_APPS = [

    # Django apps
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",

    # DRF
    "rest_framework",

    # Token authentication
    "rest_framework.authtoken",

    # Your apps
    "api",
]
```

Then:

```bash
python manage.py migrate
```

This creates the token-related database table.

---

# 6. Configure TokenAuthentication

In `settings.py`:

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

Now DRF knows that API requests can authenticate using tokens.

---

# 7. Import the Token Model

You can directly import:

```python
from rest_framework.authtoken.models import Token
```

Then:

```python
token = Token.objects.create(
    user=user
)
```

The token belongs to that user.

---

# 8. Get or Create a Token

A safer common pattern is:

```python
token, created = Token.objects.get_or_create(
    user=user
)
```

Why?

Because if the user already has a token:

```text
existing token
    ↓
return it
```

If not:

```text
no token
    ↓
create token
```

---

# 9. Default DRF Token Login

DRF gives you:

```python
from rest_framework.authtoken.views import (
    obtain_auth_token
)
```

URL:

```python
from django.urls import path
from rest_framework.authtoken.views import (
    obtain_auth_token
)

urlpatterns = [

    path(
        "login/",
        obtain_auth_token,
        name="login"
    ),

]
```

---

# 10. Login Request

Send:

```http
POST /login/
Content-Type: application/json
```

Body:

```json
{
    "username": "nzegge",
    "password": "12345678"
}
```

---

# 11. Default Token Response

DRF returns:

```json
{
    "token": "abc123..."
}
```

The token is now the user's authentication credential.

---

# 12. Using the Token

Future requests:

```http
GET /api/products/
Authorization: Token abc123...
```

The important format is:

```text
Authorization: Token <token>
```

Not:

```text
Bearer <token>
```

`Bearer` is normally used with JWT.

---

# 13. What Happens Internally?

Client sends:

```text
Authorization:
Token abc123
```

DRF:

```text
TokenAuthentication
       │
       ▼
find Token
       │
       ▼
find User
       │
       ▼
request.user
```

Therefore in your view:

```python
request.user
```

is the authenticated user.

---

# 14. Get Current User

Example:

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated


class ProfileView(APIView):

    permission_classes = [
        IsAuthenticated
    ]

    def get(self, request):

        user = request.user

        return Response({
            "id": user.id,
            "username": user.username,
            "email": user.email,
        })
```

---

# 15. `request.user` vs `request.auth`

This distinction is important.

```python
request.user
```

means:

> The authenticated Django user.

While:

```python
request.auth
```

means:

> The authentication credential used for this request.

With DRF TokenAuthentication:

```python
request.auth
```

is normally the `Token` object.

Therefore:

```python
request.auth.key
```

can give the token key.

---

# 16. Custom Token Login

Instead of:

```python
obtain_auth_token
```

you can create your own view.

```python
from django.contrib.auth import authenticate

from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

from rest_framework.authtoken.models import Token


class LoginView(APIView):

    def post(self, request):

        username = request.data.get(
            "username"
        )

        password = request.data.get(
            "password"
        )

        user = authenticate(
            username=username,
            password=password
        )

        if user is None:

            return Response(
                {
                    "detail":
                    "Invalid credentials."
                },
                status=status.HTTP_401_UNAUTHORIZED
            )

        token, created = Token.objects.get_or_create(
            user=user
        )

        return Response({

            "token": token.key,

            "user": {
                "id": user.id,
                "username": user.username,
                "email": user.email,
            }

        })
```

Now you completely control the login response.

---

# 17. Why Use `authenticate()`?

Don't do:

```python
if user.password == password:
```

Wrong.

Django passwords are hashed.

Use:

```python
user = authenticate(
    username=username,
    password=password
)
```

Django handles the password verification.

---

# 18. Custom Token Login With Serializer

You can separate validation from the view.

Create:

```python
from django.contrib.auth import authenticate

from rest_framework import serializers


class LoginSerializer(serializers.Serializer):

    username = serializers.CharField()

    password = serializers.CharField(
        write_only=True
    )

    def validate(self, attrs):

        username = attrs.get("username")
        password = attrs.get("password")

        user = authenticate(
            username=username,
            password=password
        )

        if user is None:

            raise serializers.ValidationError(
                "Invalid username or password."
            )

        attrs["user"] = user

        return attrs
```

Then:

```python
class LoginView(APIView):

    def post(self, request):

        serializer = LoginSerializer(
            data=request.data
        )

        serializer.is_valid(
            raise_exception=True
        )

        user = serializer.validated_data[
            "user"
        ]

        token, created = Token.objects.get_or_create(
            user=user
        )

        return Response({
            "token": token.key,
            "user": {
                "id": user.id,
                "username": user.username,
                "email": user.email,
            }
        })
```

This gives you a cleaner separation:

```text
Serializer
   ↓
Validate credentials

View
   ↓
Create token + response
```

---

# 19. Custom Token Login Response

You can return anything appropriate.

Example:

```python
return Response({

    "message": "Login successful",

    "token": token.key,

    "user": {
        "id": user.id,
        "username": user.username,
        "email": user.email,
    }

})
```

Response:

```json
{
    "message": "Login successful",
    "token": "abc123",
    "user": {
        "id": 5,
        "username": "nzegge",
        "email": "test@example.com"
    }
}
```

---

# 20. Include a File URL in Token Login Response

Suppose:

```python
class User(AbstractUser):

    cv = models.FileField(
        upload_to="cvs/",
        blank=True,
        null=True
    )
```

You can return:

```python
return Response({

    "token": token.key,

    "user": {

        "id": user.id,

        "username": user.username,

        "email": user.email,

        "cv": (
            user.cv.url
            if user.cv
            else None
        ),

    }

})
```

Important:

The file is **not inside the token**.

Only the file URL is in the login response.

---

# 21. Token Logout

Because the DRF token is stored in the database, logout is simple.

```python
class LogoutView(APIView):

    permission_classes = [
        IsAuthenticated
    ]

    def post(self, request):

        request.auth.delete()

        return Response({
            "detail":
            "Logged out successfully."
        })
```

Request:

```http
POST /logout/
Authorization: Token abc123
```

The token is deleted.

---

# 22. Logout Using `request.user.auth_token`

You can also do:

```python
request.user.auth_token.delete()
```

Example:

```python
class LogoutView(APIView):

    permission_classes = [
        IsAuthenticated
    ]

    def post(self, request):

        request.user.auth_token.delete()

        return Response({
            "detail":
            "Logout successful."
        })
```

---

# 23. Force a New Token on Every Login

You can delete the old token:

```python
Token.objects.filter(
    user=user
).delete()
```

Then create a new one:

```python
token = Token.objects.create(
    user=user
)
```

Complete:

```python
Token.objects.filter(
    user=user
).delete()

token = Token.objects.create(
    user=user
)
```

Now:

```text
Old token
    ↓
invalid

New login
    ↓
new token
```

---

# 24. TokenAuthentication and Multiple Devices

Default DRF TokenAuthentication gives a user one token.

Conceptually:

```text
User
 │
 └── Token
```

If you want:

```text
User
 ├── Laptop token
 ├── Phone token
 └── Tablet token
```

you need a custom token/device-token design or a different authentication architecture.

JWT naturally fits multiple sessions better because each login can issue different refresh/access tokens.

---

# PART 3 — SimpleJWT

# 25. What Is JWT?

JWT means:

```text
JSON Web Token
```

SimpleJWT provides JWT authentication for DRF.

Install:

```bash
pip install djangorestframework-simplejwt
```

---

# 26. Configure SimpleJWT

In:

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [

        "rest_framework_simplejwt.authentication.JWTAuthentication",

    ],

}
```

---

# 27. Default JWT Login URLs

Import:

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)
```

URLs:

```python
urlpatterns = [

    path(
        "login/",
        TokenObtainPairView.as_view(),
        name="login"
    ),

    path(
        "token/refresh/",
        TokenRefreshView.as_view(),
        name="token_refresh"
    ),

]
```

---

# 28. JWT Login

Request:

```http
POST /login/
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

# 29. Access Token

The access token is used for API requests.

```http
Authorization: Bearer eyJ...
```

It is normally short-lived.

---

# 30. Refresh Token

The refresh token is used to get another access token.

```http
POST /token/refresh/
```

Body:

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

---

# 31. JWT Structure

A JWT has three parts:

```text
HEADER.PAYLOAD.SIGNATURE
```

Example conceptually:

```text
eyJhbGciOiJIUzI1NiJ9
.
eyJ1c2VyX2lkIjo1fQ
.
signature
```

---

# 32. JWT Header

Contains information about the token.

Example:

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

---

# 33. JWT Payload

Contains claims.

Example:

```json
{
    "token_type": "access",
    "exp": 1790000000,
    "iat": 1789990000,
    "jti": "...",
    "user_id": 5
}
```

---

# 34. JWT Signature

The signature allows the server to verify that the token has not been modified.

Conceptually:

```text
Header
+
Payload
+
Secret/private key
        ↓
Signature
```

---

# 35. JWT Is Not a Place for Secrets

Don't put:

```text
password
credit card
API secret
private key
```

inside a JWT.

JWT payloads are generally encoded, not encrypted.

---

# PART 4 — Customizing SimpleJWT Serializers

# 36. The Main JWT Serializer

SimpleJWT provides:

```python
TokenObtainPairSerializer
```

Import:

```python
from rest_framework_simplejwt.serializers import (
    TokenObtainPairSerializer
)
```

You can inherit from it.

---

# 37. Why Inherit?

Instead of rewriting the entire authentication system:

```python
class CustomTokenObtainPairSerializer(
    TokenObtainPairSerializer
):
```

you reuse SimpleJWT's existing login logic.

Then customize only what you need.

This is one of the most important customization patterns.

---

# 38. Customize JWT Claims

Use:

```python
get_token()
```

Example:

```python
class CustomTokenObtainPairSerializer(
    TokenObtainPairSerializer
):

    @classmethod
    def get_token(cls, user):

        token = super().get_token(user)

        token["username"] = user.username

        return token
```

---

# 39. Why `super().get_token(user)`?

It gets the normal SimpleJWT token first.

Then:

```python
token["username"] = user.username
```

adds your custom claim.

Think:

```text
SimpleJWT default token
        +
your claims
        =
custom JWT
```

---

# 40. Add User ID

```python
token["user_id"] = user.id
```

Example:

```python
@classmethod
def get_token(cls, user):

    token = super().get_token(user)

    token["user_id"] = user.id

    return token
```

Note that SimpleJWT already has a user ID claim in its normal token, depending on configuration/version, so don't duplicate claims unnecessarily.

---

# 41. Add Username

```python
token["username"] = user.username
```

---

# 42. Add Email

```python
token["email"] = user.email
```

---

# 43. Add Role

Suppose your User model has:

```python
role = models.CharField(
    max_length=30,
    default="customer"
)
```

Then:

```python
token["role"] = user.role
```

---

# 44. Add Staff Status

Technically:

```python
token["is_staff"] = user.is_staff
```

But remember:

> Authorization decisions should generally use the current server-side user state rather than relying on an old JWT claim.

For example:

```python
if request.user.is_staff:
    ...
```

is safer for current permissions.

---

# 45. Add Profile Image URL

Suppose:

```python
user.profile_image
```

exists.

You could do:

```python
token["profile_image"] = (
    user.profile_image.url
    if user.profile_image
    else None
)
```

But again, consider whether this belongs in the JWT.

Often better:

```text
JWT
 ↓
authentication

Profile endpoint
 ↓
profile image
```

---

# 46. Add CV URL

You can technically:

```python
token["cv"] = (
    user.cv.url
    if user.cv
    else None
)
```

But this means every token contains that information.

Usually better:

```python
data["user"]["cv"] = ...
```

in the login response, or expose it from:

```text
GET /profile/
```

---

# 47. Custom Login Response

This is different from custom claims.

Override:

```python
validate()
```

Example:

```python
def validate(self, attrs):

    data = super().validate(attrs)

    data["user"] = {
        "id": self.user.id,
        "username": self.user.username,
        "email": self.user.email,
    }

    return data
```

---

# 48. `get_token()` vs `validate()`

This distinction is extremely important.

### `get_token()`

Controls information **inside the JWT**.

```python
token["username"] = user.username
```

### `validate()`

Controls the **login response**.

```python
data["user"] = {
    ...
}
```

Think:

```text
get_token()
    ↓
JWT contents

validate()
    ↓
HTTP login response
```

---

# 49. Complete Custom JWT Serializer

```python
from rest_framework_simplejwt.serializers import (
    TokenObtainPairSerializer
)


class CustomTokenObtainPairSerializer(
    TokenObtainPairSerializer
):

    @classmethod
    def get_token(cls, user):

        # Get the normal JWT
        token = super().get_token(user)

        # Add custom JWT claims
        token["username"] = user.username
        token["email"] = user.email

        return token


    def validate(self, attrs):

        # Run normal SimpleJWT authentication
        data = super().validate(attrs)

        # Customize login response
        data["user"] = {

            "id": self.user.id,

            "username": self.user.username,

            "email": self.user.email,

            "cv": (
                self.user.cv.url
                if self.user.cv
                else None
            )

        }

        return data
```

---

# 50. Custom JWT View

Import:

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView
)
```

Then:

```python
class CustomTokenObtainPairView(
    TokenObtainPairView
):

    serializer_class = (
        CustomTokenObtainPairSerializer
    )
```

---

# 51. URL

```python
path(
    "login/",
    CustomTokenObtainPairView.as_view(),
    name="login"
)
```

Now:

```text
POST /login/
```

uses your custom serializer.

---

# PART 5 — More SimpleJWT Serializers

SimpleJWT provides more than one serializer.

The important ones include:

```text
TokenObtainPairSerializer
TokenObtainSlidingSerializer
TokenRefreshSerializer
TokenVerifySerializer
TokenBlacklistSerializer
```

The exact available classes can depend on the installed SimpleJWT version, but these are the main concepts you will encounter.

---

# 52. `TokenObtainPairSerializer`

Used for:

```text
username/password
       ↓
access + refresh
```

This is the most common serializer for normal JWT login.

---

# 53. `TokenObtainSlidingSerializer`

Sliding tokens are another JWT strategy.

Instead of the classic:

```text
access
refresh
```

you can use a sliding token whose lifetime can be extended according to SimpleJWT's configuration.

This is less common than access/refresh pairs.

---

# 54. `TokenRefreshSerializer`

Used when the client sends:

```json
{
    "refresh": "..."
}
```

and wants:

```json
{
    "access": "..."
}
```

You can subclass it if you need custom refresh behavior.

Example:

```python
from rest_framework_simplejwt.serializers import (
    TokenRefreshSerializer
)


class CustomRefreshSerializer(
    TokenRefreshSerializer
):

    def validate(self, attrs):

        data = super().validate(attrs)

        # Add custom response data if needed
        data["message"] = "Token refreshed."

        return data
```

---

# 55. `TokenVerifySerializer`

Used to verify whether a JWT is valid.

Conceptually:

```text
Token
 ↓
Verify
 ↓
Valid / invalid
```

You normally don't need to call this for every normal API request because `JWTAuthentication` already authenticates requests.

---

# 56. `TokenBlacklistSerializer`

Used for blacklisting refresh tokens when blacklist support is enabled.

You can customize it if your logout endpoint needs additional behavior.

---

# PART 6 — Customizing JWT Views

# 57. Why Customize a View?

A serializer controls data/validation.

A view controls:

```text
HTTP request
permissions
serializer
HTTP response
```

Example:

```python
class CustomLoginView(
    TokenObtainPairView
):

    serializer_class = (
        CustomTokenObtainPairSerializer
    )
```

This is usually enough.

---

# 58. Override `post()`

You technically can:

```python
class CustomLoginView(
    TokenObtainPairView
):

    serializer_class = (
        CustomTokenObtainPairSerializer
    )

    def post(self, request, *args, **kwargs):

        response = super().post(
            request,
            *args,
            **kwargs
        )

        response.data["message"] = (
            "Login successful"
        )

        return response
```

Now the response gets:

```json
{
    "refresh": "...",
    "access": "...",
    "user": {},
    "message": "Login successful"
}
```

---

# 59. When Should You Override `post()`?

Use it when you need to modify the final HTTP response.

For example:

```text
cookies
headers
response message
additional metadata
```

Don't override it just because you want to add a user field.

For that, `validate()` is generally cleaner.

---

# PART 7 — JWT Manual Token Generation

# 60. Import `RefreshToken`

```python
from rest_framework_simplejwt.tokens import (
    RefreshToken
)
```

---

# 61. Create Tokens Manually

```python
refresh = RefreshToken.for_user(user)

access = refresh.access_token
```

Then:

```python
print(str(refresh))
print(str(access))
```

---

# 62. Manual Custom Claims

```python
refresh = RefreshToken.for_user(user)

refresh["username"] = user.username
refresh["email"] = user.email
refresh["role"] = user.role

access = refresh.access_token
```

Then:

```python
return Response({

    "refresh": str(refresh),

    "access": str(access)

})
```

---

# 63. Complete Custom JWT Login

```python
from django.contrib.auth import authenticate

from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

from rest_framework_simplejwt.tokens import (
    RefreshToken
)


class LoginView(APIView):

    def post(self, request):

        username = request.data.get(
            "username"
        )

        password = request.data.get(
            "password"
        )

        # Authenticate user
        user = authenticate(
            username=username,
            password=password
        )

        # Authentication failed
        if user is None:

            return Response(
                {
                    "detail":
                    "Invalid credentials."
                },
                status=status.HTTP_401_UNAUTHORIZED
            )

        # Create refresh token
        refresh = RefreshToken.for_user(user)

        # Custom claims
        refresh["username"] = user.username
        refresh["email"] = user.email

        # Create access token
        access = refresh.access_token

        return Response({

            "refresh": str(refresh),

            "access": str(access),

            "user": {

                "id": user.id,

                "username":
                user.username,

                "email":
                user.email,

            }

        })
```

---

# PART 8 — JWT Logout

# 64. Why JWT Logout Is Different

DRF Token:

```text
delete token from database
```

JWT:

```text
Access token is normally self-contained.
```

Therefore JWT logout needs a different strategy.

---

# 65. Enable Blacklisting

Add:

```python
INSTALLED_APPS = [

    "rest_framework_simplejwt.token_blacklist",

]
```

Then:

```bash
python manage.py migrate
```

---

# 66. Default Blacklist Logout

Import:

```python
from rest_framework_simplejwt.views import (
    TokenBlacklistView
)
```

URL:

```python
path(
    "logout/",
    TokenBlacklistView.as_view(),
    name="logout"
)
```

Request:

```http
POST /logout/
```

Body:

```json
{
    "refresh": "your-refresh-token"
}
```

---

# 67. Custom JWT Logout

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

from rest_framework_simplejwt.tokens import (
    RefreshToken
)


class LogoutView(APIView):

    def post(self, request):

        refresh_token = request.data.get(
            "refresh"
        )

        if not refresh_token:

            return Response(
                {
                    "detail":
                    "Refresh token is required."
                },
                status=status.HTTP_400_BAD_REQUEST
            )

        try:

            token = RefreshToken(
                refresh_token
            )

            token.blacklist()

            return Response({
                "detail":
                "Logout successful."
            })

        except Exception:

            return Response(
                {
                    "detail":
                    "Invalid refresh token."
                },
                status=status.HTTP_400_BAD_REQUEST
            )
```

---

# 68. JWT Logout Flow

```text
LOGIN
 │
 ├── Access
 │
 └── Refresh
       │
       ▼
     Client
       │
       ▼
API requests
       │
       ▼
Access token
       │
       ▼
Logout
       │
       ▼
Refresh token blacklisted
```

---

# PART 9 — JWT Settings

# 69. Configure Token Lifetimes

Example:

```python
from datetime import timedelta


SIMPLE_JWT = {

    "ACCESS_TOKEN_LIFETIME":
        timedelta(minutes=15),

    "REFRESH_TOKEN_LIFETIME":
        timedelta(days=7),

}
```

This means:

```text
Access:
15 minutes

Refresh:
7 days
```

---

# 70. Why Short Access Tokens?

If an access token is stolen:

```text
short lifetime
```

limits the period during which it can be used.

A common pattern is:

```text
Access:
short-lived

Refresh:
longer-lived
```

---

# 71. Refresh Rotation

SimpleJWT can rotate refresh tokens.

Example configuration:

```python
SIMPLE_JWT = {

    "ROTATE_REFRESH_TOKENS": True,

}
```

This means refreshing can issue a new refresh token.

You can combine it with:

```python
"BLACKLIST_AFTER_ROTATION": True,
```

when blacklist support is enabled.

---

# 72. JWT Signing Algorithm

Example:

```python
SIMPLE_JWT = {

    "ALGORITHM": "HS256",

}
```

You can also use asymmetric algorithms such as:

```text
RS256
```

with appropriate signing/verifying key configuration.

For a normal single Django backend, symmetric signing is often simpler.

---

# PART 10 — Custom User Models

# 73. Always Consider `get_user_model()`

If your project has:

```python
class User(AbstractUser):
    ...
```

use:

```python
from django.contrib.auth import get_user_model

User = get_user_model()
```

instead of assuming:

```python
from django.contrib.auth.models import User
```

Why?

Because your project may have replaced Django's default user model.

---

# 74. Token Login and Custom User Model

SimpleJWT can authenticate your configured Django user model.

Your custom user might have:

```python
class User(AbstractUser):

    cv = models.FileField(
        upload_to="cvs/",
        blank=True,
        null=True
    )

    profile_image = models.ImageField(
        upload_to="profiles/",
        blank=True,
        null=True
    )
```

Then your custom serializer can access:

```python
user.cv
```

and:

```python
user.profile_image
```

---

# PART 11 — User Serializer + JWT

# 75. Create a User Serializer

```python
from rest_framework import serializers

from django.contrib.auth import get_user_model


User = get_user_model()


class UserSerializer(
    serializers.ModelSerializer
):

    class Meta:

        model = User

        fields = [
            "id",
            "username",
            "email",
            "cv",
            "profile_image",
        ]

        read_only_fields = [
            "id"
        ]
```

---

# 76. Use It in JWT Login

```python
class CustomTokenObtainPairSerializer(
    TokenObtainPairSerializer
):

    def validate(self, attrs):

        data = super().validate(attrs)

        data["user"] = UserSerializer(
            self.user,
            context=self.context
        ).data

        return data
```

This is a very clean pattern.

---

# 77. Absolute File URL

If you want:

```text
http://127.0.0.1:8000/media/cvs/file.pdf
```

instead of:

```text
/media/cvs/file.pdf
```

use:

```python
class UserSerializer(
    serializers.ModelSerializer
):

    cv = serializers.SerializerMethodField()

    class Meta:

        model = User

        fields = [
            "id",
            "username",
            "email",
            "cv",
        ]

    def get_cv(self, obj):

        if not obj.cv:
            return None

        request = self.context.get(
            "request"
        )

        if request:

            return request.build_absolute_uri(
                obj.cv.url
            )

        return obj.cv.url
```

---

# PART 12 — Custom Registration + JWT

# 78. JWT Does Not Usually Register Users

Login and registration are separate concepts.

Registration:

```text
POST /register/
```

Login:

```text
POST /login/
```

Registration creates:

```text
User
```

Login authenticates:

```text
User
```

---

# 79. Registration Serializer

```python
class RegisterSerializer(
    serializers.ModelSerializer
):

    password = serializers.CharField(
        write_only=True
    )

    class Meta:

        model = User

        fields = [
            "username",
            "email",
            "password",
        ]

    def create(self, validated_data):

        user = User.objects.create_user(
            **validated_data
        )

        return user
```

Important:

Use:

```python
create_user()
```

rather than:

```python
User.objects.create(
    password=...
)
```

because `create_user()` hashes the password.

---

# PART 13 — Permissions

# 80. `AllowAny`

Anyone can access:

```python
permission_classes = [
    AllowAny
]
```

Common for:

```text
registration
login
password reset
```

---

# 81. `IsAuthenticated`

Requires login:

```python
permission_classes = [
    IsAuthenticated
]
```

---

# 82. `IsAdminUser`

Requires staff:

```python
permission_classes = [
    IsAdminUser
]
```

---

# 83. Object-Level Authorization

Authentication tells you:

```text
This is user 5.
```

But your application still needs to decide:

```text
Can user 5 edit this order?
```

For example:

```python
def get_queryset(self):

    return Order.objects.filter(
        user=self.request.user
    )
```

This is very important for APIs.

---

# PART 14 — Authentication Classes

# 84. JWT Authentication

```python
from rest_framework_simplejwt.authentication import (
    JWTAuthentication
)
```

Settings:

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [

        "rest_framework_simplejwt.authentication.JWTAuthentication",

    ],

}
```

---

# 85. Token Authentication

```python
from rest_framework.authentication import (
    TokenAuthentication
)
```

Settings:

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [

        "rest_framework.authentication.TokenAuthentication",

    ],
}
```

---

# 86. Session Authentication

DRF also provides:

```python
from rest_framework.authentication import (
    SessionAuthentication
)
```

This is commonly relevant when using Django's session login/admin/browser-based API access.

It can require CSRF protection for unsafe methods such as:

```text
POST
PUT
PATCH
DELETE
```

This is why you may encounter:

```text
CSRF Failed
```

when using SessionAuthentication.

---

# PART 15 — Using Multiple Authentication Classes

You technically can:

```python
REST_FRAMEWORK = {

    "DEFAULT_AUTHENTICATION_CLASSES": [

        "rest_framework_simplejwt.authentication.JWTAuthentication",

        "rest_framework.authentication.TokenAuthentication",

    ]

}
```

Then:

```text
Bearer <JWT>
```

can authenticate through JWT.

And:

```text
Token <token>
```

can authenticate through DRF TokenAuthentication.

Use this only when your architecture actually needs both.

---

# PART 16 — Authentication Order

If you configure:

```python
DEFAULT_AUTHENTICATION_CLASSES = [
    JWTAuthentication,
    TokenAuthentication,
]
```

DRF attempts authentication according to the configured classes.

The first successful authentication determines:

```python
request.user
request.auth
```

---

# PART 17 — Custom Authentication Class

You can even create your own authentication class.

Conceptually:

```python
from rest_framework.authentication import (
    BaseAuthentication
)
```

Then:

```python
class MyAuthentication(
    BaseAuthentication
):

    def authenticate(self, request):

        # Your authentication logic

        return (
            user,
            authentication
        )
```

The returned tuple is:

```text
(user, auth)
```

For example:

```python
return user, token
```

Then DRF sets:

```python
request.user = user
request.auth = token
```

---

# PART 18 — Authentication Errors

# 88. 401 Unauthorized

Usually means:

```text
Authentication failed
```

Example:

```json
{
    "detail": "Authentication credentials were not provided."
}
```

or:

```json
{
    "detail": "Given token not valid for any token type"
}
```

---

# 89. 403 Forbidden

Usually means:

```text
The request reached authentication/authorization
but access is forbidden.
```

Examples:

```text
permission denied
CSRF failure
```

A common mistake is assuming every `403` means the password/token is wrong.

---

# PART 19 — JWT Claims and User Information

# 90. What Should Go in a JWT?

Reasonable small claims can include:

```text
user_id
username
role
token type
issued time
expiration
JWT ID
```

Avoid:

```text
password
large files
images
CV binary data
large profile objects
sensitive secrets
```

---

# 91. File in JWT — Correct Thinking

Wrong concept:

```text
JWT
 └── actual image binary
```

Better:

```text
JWT
 └── maybe profile_image URL
```

Even better in many applications:

```text
JWT
 └── user identity

Profile API
 └── profile image URL
```

---

# PART 20 — JWT Token Introspection

# 92. Decode Token for Debugging

A JWT contains:

```text
header
payload
signature
```

You can decode the header/payload to inspect claims.

But remember:

> Decoding a JWT is not the same as verifying its signature.

Never treat an unverified decoded payload as trustworthy.

---

# 93. Verify Token

SimpleJWT provides verification functionality through its verification serializer/view.

Conceptually:

```text
JWT
 ↓
signature verification
 ↓
expiration verification
 ↓
valid / invalid
```

Normally `JWTAuthentication` handles this automatically during API requests.

---

# PART 21 — Refresh Token Tricks

# 94. Refresh Automatically in Frontend

Typical React logic:

```text
API request
    ↓
401
    ↓
try refresh token
    ↓
new access token
    ↓
retry original request
```

This is commonly implemented with an Axios interceptor.

---

# 95. Example Axios Concept

```javascript
axios.interceptors.response.use(
    response => response,

    async error => {

        if (error.response?.status === 401) {

            // Refresh access token

            // Save new access token

            // Retry original request
        }

        return Promise.reject(error);
    }
);
```

For a production application, carefully handle refresh races and logout when refresh fails.

---

# PART 22 — Storing Tokens in the Frontend

# 96. Common Choices

Tokens can be stored in:

```text
memory
localStorage
sessionStorage
secure cookies
```

Each has security/tradeoff implications.

Do not blindly assume:

```text
localStorage = safest
```

or:

```text
cookies = automatically safe
```

The correct design depends on your frontend architecture and threat model.

For browser applications, HttpOnly secure cookies can reduce JavaScript access to refresh credentials, but introduce CSRF considerations that must be handled correctly.

---

# PART 23 — Password Security

# 97. Never Put Passwords in JWT

Never:

```python
token["password"] = user.password
```

Never return:

```json
{
    "password": "..."
}
```

Never log passwords.

---

# 98. Password Serializer Field

Use:

```python
password = serializers.CharField(
    write_only=True
)
```

This means:

```text
accepted when writing
NOT returned when reading
```

---

# PART 24 — Custom Claims With Conditions

# 99. Conditional Claims

You can do:

```python
if user.is_staff:

    token["role"] = "admin"

else:

    token["role"] = "customer"
```

Example:

```python
@classmethod
def get_token(cls, user):

    token = super().get_token(user)

    if user.is_staff:

        token["role"] = "admin"

    else:

        token["role"] = "customer"

    return token
```

Again, use server-side permission checks for sensitive authorization decisions.

---

# PART 25 — Custom Login Validation

# 100. Check Active User

You can customize validation:

```python
def validate(self, attrs):

    data = super().validate(attrs)

    if not self.user.is_active:

        raise serializers.ValidationError(
            "Account is inactive."
        )

    return data
```

Django authentication already considers user state in its normal authentication flow, so this is mainly useful when adding application-specific checks.

---

# 101. Check Email Verification

If your application has:

```python
email_verified
```

you can check:

```python
if not self.user.email_verified:

    raise serializers.ValidationError(
        "Please verify your email first."
    )
```

Then only verified users receive the normal token response.

---

# 102. Custom Login Restrictions

You can add business rules such as:

```text
account active?
email verified?
subscription active?
account locked?
organization enabled?
```

Example:

```python
def validate(self, attrs):

    data = super().validate(attrs)

    if not self.user.email_verified:

        raise serializers.ValidationError(
            "Email verification required."
        )

    return data
```

---

# PART 26 — Custom Token Response

# 103. Return Permissions

You can return:

```python
data["permissions"] = [
    "orders.read",
    "orders.create"
]
```

But don't put huge permission lists into the JWT unless necessary.

---

# 104. Return User Profile

Better:

```python
data["user"] = UserSerializer(
    self.user,
    context=self.context
).data
```

This gives:

```json
{
    "access": "...",
    "refresh": "...",
    "user": {
        "id": 5,
        "username": "nzegge",
        "email": "test@example.com",
        "cv": "/media/cvs/file.pdf"
    }
}
```

---

# PART 27 — Complete Recommended JWT Structure

For your Django + React application, a clean login response can look like:

```json
{
    "access": "eyJ...",
    "refresh": "eyJ...",
    "user": {
        "id": 5,
        "username": "nzegge",
        "email": "example@gmail.com",
        "profile_image": "http://127.0.0.1:8000/media/profiles/me.png",
        "cv": "http://127.0.0.1:8000/media/cvs/me.pdf"
    }
}
```

Then:

```text
access
 ↓
API authentication

refresh
 ↓
new access

user
 ↓
frontend display/profile information
```

---

# PART 28 — Custom Serializer Hierarchy

Understand this structure:

```text
TokenObtainPairView
        │
        ▼
TokenObtainPairSerializer
        │
        ├── validate()
        │
        └── get_token()
                │
                ▼
              Token
```

You customize:

```python
class CustomTokenObtainPairSerializer(
    TokenObtainPairSerializer
):
```

Then:

```python
get_token()
```

for JWT claims.

And:

```python
validate()
```

for login response/validation.

---

# PART 29 — Custom Refresh Hierarchy

Conceptually:

```text
TokenRefreshView
        │
        ▼
TokenRefreshSerializer
        │
        ▼
refresh token
        │
        ▼
new access token
```

You can subclass the refresh serializer when you need custom refresh behavior.

---

# PART 30 — Custom Blacklist Hierarchy

Conceptually:

```text
TokenBlacklistView
        │
        ▼
TokenBlacklistSerializer
        │
        ▼
RefreshToken
        │
        ▼
Blacklist
```

---

# PART 31 — DRF Token vs JWT Customization

## DRF Token

You customize:

```text
Login View
Login Serializer
Token creation
Token deletion
Response
User serializer
Authentication class
```

But the token itself is fundamentally a database-backed key.

---

## JWT

You can customize:

```text
Login Serializer
Login View
JWT claims
Login response
Refresh serializer
Refresh view
Blacklist serializer
Blacklist view
Token lifetime
Signing algorithm
Token generation
Authentication behavior
```

JWT therefore gives you much more control over token contents.

---

# PART 32 — Complete DRF Token Project Structure

A possible structure:

```text
api/
│
├── models.py
│
├── serializers.py
│
├── views.py
│
├── authentication.py
│
└── urls.py
```

`serializers.py`:

```python
class LoginSerializer(...):
    ...
```

`views.py`:

```python
class LoginView(...):
    ...

class LogoutView(...):
    ...
```

`urls.py`:

```python
path(
    "login/",
    LoginView.as_view()
)

path(
    "logout/",
    LogoutView.as_view()
)
```

---

# PART 33 — Complete JWT Project Structure

You could organize:

```text
api/
│
├── models.py
│
├── serializers.py
│
├── views.py
│
├── authentication.py
│
├── permissions.py
│
└── urls.py
```

`serializers.py`:

```python
CustomTokenObtainPairSerializer
UserSerializer
RegisterSerializer
```

`views.py`:

```python
CustomTokenObtainPairView
LogoutView
RegisterView
ProfileView
```

---

# PART 34 — Common Mistakes

## Mistake 1

Using:

```http
Authorization: Bearer ...
```

with DRF TokenAuthentication.

Correct:

```http
Authorization: Token ...
```

---

## Mistake 2

Using:

```http
Authorization: Token ...
```

with JWTAuthentication.

Correct:

```http
Authorization: Bearer ...
```

---

## Mistake 3

Putting passwords in JWT.

Never do this.

---

## Mistake 4

Putting the actual image/file into JWT.

Don't.

Store the file using Django storage.

---

## Mistake 5

Manually decoding JWT and trusting the result.

Decode ≠ verify.

---

## Mistake 6

Using frontend user ID for authorization.

Instead:

```python
request.user
```

---

## Mistake 7

Deleting a JWT from the frontend and assuming the server has invalidated it.

With JWT, client-side deletion only removes the client's copy.

Use short access-token lifetimes and refresh-token blacklisting/rotation as appropriate.

---

## Mistake 8

Forgetting:

```python
"rest_framework_simplejwt.token_blacklist"
```

when using SimpleJWT blacklist functionality.

---

## Mistake 9

Forgetting:

```bash
python manage.py migrate
```

after enabling token/blacklist apps.

---

## Mistake 10

Returning passwords from serializers.

Use:

```python
write_only=True
```

---

# PART 35 — The Most Important Imports

## DRF Token

```python
from rest_framework.authtoken.models import Token
```

```python
from rest_framework.authtoken.views import (
    obtain_auth_token
)
```

```python
from rest_framework.authentication import (
    TokenAuthentication
)
```

---

## JWT

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
    TokenBlacklistView,
)
```

```python
from rest_framework_simplejwt.serializers import (
    TokenObtainPairSerializer,
    TokenRefreshSerializer,
    TokenVerifySerializer,
    TokenBlacklistSerializer,
)
```

```python
from rest_framework_simplejwt.tokens import (
    RefreshToken,
    AccessToken,
)
```

```python
from rest_framework_simplejwt.authentication import (
    JWTAuthentication
)
```

---

# PART 36 — Most Important Methods

## DRF Token

```python
Token.objects.create(...)
```

```python
Token.objects.get_or_create(...)
```

```python
token.delete()
```

```python
request.auth
```

```python
request.user
```

---

## JWT

```python
TokenObtainPairSerializer.get_token()
```

```python
super().get_token(user)
```

```python
super().validate(attrs)
```

```python
RefreshToken.for_user(user)
```

```python
refresh.access_token
```

```python
refresh.blacklist()
```

---

# PART 37 — The Four Most Important Customization Points

If you remember nothing else, remember these:

## 1. Customize JWT claims

```python
@classmethod
def get_token(cls, user):

    token = super().get_token(user)

    token["username"] = user.username

    return token
```

---

## 2. Customize login response

```python
def validate(self, attrs):

    data = super().validate(attrs)

    data["user"] = UserSerializer(
        self.user
    ).data

    return data
```

---

## 3. Customize login endpoint

```python
class CustomLoginView(
    TokenObtainPairView
):

    serializer_class = (
        CustomTokenObtainPairSerializer
    )
```

---

## 4. Customize logout

```python
refresh = RefreshToken(
    refresh_token
)

refresh.blacklist()
```

---

# PART 38 — Normal JWT vs Custom JWT

## Normal

```text
POST /login/
        ↓
TokenObtainPairView
        ↓
TokenObtainPairSerializer
        ↓
access + refresh
```

## Custom

```text
POST /login/
        ↓
CustomTokenObtainPairView
        ↓
CustomTokenObtainPairSerializer
        │
        ├── validate()
        │       ↓
        │   custom response
        │
        └── get_token()
                ↓
          custom claims
                ↓
          access + refresh
```

---

# PART 39 — Recommended Architecture

For a modern Django REST + React application, a clean architecture is:

```text
                   AUTHENTICATION
                         │
                         ▼
                       JWT
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
          ACCESS                  REFRESH
             │                       │
             ▼                       ▼
        API requests             refresh
             │
             ▼
        request.user
             │
      ┌──────┼─────────┐
      │      │         │
      ▼      ▼         ▼
    Orders  Profile   Files
                       │
                       ▼
                 Django Media
```

---

# PART 40 — Final Cheat Sheet

## DRF Token

```python
from rest_framework.authtoken.models import Token
```

Create:

```python
token = Token.objects.create(
    user=user
)
```

or:

```python
token, created = Token.objects.get_or_create(
    user=user
)
```

Header:

```http
Authorization: Token <token>
```

Current user:

```python
request.user
```

Current token:

```python
request.auth
```

Logout:

```python
request.auth.delete()
```

---

# JWT

Login:

```python
TokenObtainPairView
```

Refresh:

```python
TokenRefreshView
```

Verify:

```python
TokenVerifyView
```

Blacklist:

```python
TokenBlacklistView
```

Header:

```http
Authorization: Bearer <access>
```

Custom serializer:

```python
class CustomTokenObtainPairSerializer(
    TokenObtainPairSerializer
):
```

Custom claims:

```python
@classmethod
def get_token(cls, user):

    token = super().get_token(user)

    token["username"] = user.username

    return token
```

Custom login response:

```python
def validate(self, attrs):

    data = super().validate(attrs)

    data["user"] = UserSerializer(
        self.user
    ).data

    return data
```

Manual token:

```python
refresh = RefreshToken.for_user(user)

access = refresh.access_token
```

Blacklist:

```python
refresh.blacklist()
```

---

# PART 41 — Final Mental Model

The most important picture to remember is:

```text
                 LOGIN
                   │
         username + password
                   │
                   ▼
             authenticate()
                   │
                   ▼
                  User
                   │
          ┌────────┴────────┐
          │                 │
          ▼                 ▼
       DRF Token           JWT
          │                 │
          ▼                 ▼
       DB Token       Access + Refresh
          │                 │
          ▼                 ▼
 Authorization:       Authorization:
 Token xxx            Bearer xxx
          │                 │
          ▼                 ▼
     request.user      request.user
          │                 │
          ▼                 ▼
       LOGOUT             LOGOUT
          │                 │
          ▼                 ▼
     delete token      blacklist refresh
```

And for customization:

```text
                  JWT LOGIN
                      │
                      ▼
       TokenObtainPairSerializer
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
        get_token()          validate()
             │                 │
             ▼                 ▼
       JWT CONTENT       LOGIN RESPONSE
             │                 │
             │                 ├── access
             │                 ├── refresh
             │                 └── user
             │
             ├── username
             ├── email
             ├── role
             └── small claims
```

And remember the file distinction:

```text
              USER FILE
                  │
                  ▼
           Django FileField
                  │
                  ▼
             MEDIA_ROOT
                  │
                  ▼
          /media/cvs/file.pdf


JWT
 │
 └── authentication information


LOGIN RESPONSE
 │
 └── user information
       │
       └── cv URL
```

**The actual file should remain in file storage, not inside the JWT.**

---

# FINAL RULES TO MEMORIZE

```text
1. Authentication = WHO are you?

2. Authorization = WHAT can you do?

3. request.user = authenticated Django user.

4. request.auth = authentication credential.

5. DRF Token = database-backed token.

6. JWT = signed token containing claims.

7. DRF Token header:
   Authorization: Token <token>

8. JWT header:
   Authorization: Bearer <access>

9. JWT get_token() = customize JWT claims.

10. JWT validate() = customize login response/validation.

11. TokenObtainPairView = JWT login endpoint.

12. TokenRefreshView = obtain a new access token.

13. TokenBlacklistView = blacklist refresh token.

14. RefreshToken.for_user(user)
    = manually create JWT pair.

15. refresh.blacklist()
    = blacklist refresh token.

16. Never put passwords in JWT.

17. Never put actual binary files in JWT.

18. Put file URLs in API responses when appropriate.

19. Use request.user for current-user authorization.

20. Frontend deleting a JWT does NOT necessarily
    invalidate the JWT on the server.

21. Short access-token lifetime + refresh tokens
    is a common JWT architecture.

22. Use blacklist/rotation when your logout/session
    requirements need server-side refresh-token
    invalidation.

23. Extend framework serializers instead of rewriting
    authentication when the framework already does
    the part you need.

24. Use get_token() for JWT contents.

25. Use validate() for login response and additional
    login validation.

26. Use a custom view when you need to customize
    endpoint behavior.

27. Use UserSerializer for reusable user representation.

28. Use get_user_model() with custom user models.

29. Use create_user() when creating users with passwords.

30. Keep authentication data and profile/file data
    conceptually separate.
```

This gives you the complete mental model for moving from **DRF's ready-made login/logout endpoints** to **fully customized authentication logic**, without throwing away the authentication code that DRF/SimpleJWT already provides.
