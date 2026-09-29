# DJANGO REST FRAMEWORK VIEWS

## Complete Guide: APIView, GenericAPIView, Generic Views, ViewSets, Mixins, Dynamic Behavior, Permissions, Serializers, Querysets and Actions

---

# 1. DRF VIEW HIERARCHY — THE BIG PICTURE

Django REST Framework provides several ways to build API endpoints.

The main hierarchy can be understood as:

```text
APIView
│
├── GenericAPIView
│   │
│   ├── ListAPIView
│   ├── CreateAPIView
│   ├── RetrieveAPIView
│   ├── UpdateAPIView
│   ├── DestroyAPIView
│   │
│   └── Combined Generic Views
│       ├── ListCreateAPIView
│       ├── RetrieveUpdateAPIView
│       ├── RetrieveDestroyAPIView
│       └── RetrieveUpdateDestroyAPIView
│
└── ViewSet
    │
    ├── ViewSet
    │
    ├── GenericViewSet
    │
    └── ModelViewSet
```

There is another important relationship:

```text
GenericViewSet
      +
Mixins
      ↓
Custom ViewSet
```

For example:

```python
class ProductViewSet(
    ListModelMixin,
    CreateModelMixin,
    GenericViewSet
):
    ...
```

---

# 2. THE BIG IDEA

Think about DRF views in layers:

```text
APIView
    ↓
HTTP methods

GenericAPIView
    ↓
queryset
serializer
object lookup
filtering
pagination

Generic Views
    ↓
standard CRUD behavior

ViewSet
    ↓
actions

ModelViewSet
    ↓
complete standard CRUD
```

---

# 3. APIView

Import:

```python
from rest_framework.views import APIView
```

Example:

```python
from rest_framework.views import APIView
from rest_framework.response import Response


class ProductView(APIView):

    def get(self, request):
        return Response({
            "message": "GET request"
        })

    def post(self, request):
        return Response({
            "message": "POST request"
        })

    def put(self, request):
        return Response({
            "message": "PUT request"
        })

    def patch(self, request):
        return Response({
            "message": "PATCH request"
        })

    def delete(self, request):
        return Response({
            "message": "DELETE request"
        })
```

HTTP methods map directly to methods:

```text
GET       → get()
POST      → post()
PUT       → put()
PATCH     → patch()
DELETE    → delete()
```

---

# 4. APIView COMMON CONFIGURATION

You can configure:

```python
class ProductView(APIView):

    authentication_classes = [
        ...
    ]

    permission_classes = [
        ...
    ]

    parser_classes = [
        ...
    ]

    renderer_classes = [
        ...
    ]

    throttle_classes = [
        ...
    ]
```

For example:

```python
from rest_framework.permissions import IsAuthenticated
from rest_framework.parsers import MultiPartParser, FormParser


class UploadCVView(APIView):

    permission_classes = [
        IsAuthenticated
    ]

    parser_classes = [
        MultiPartParser,
        FormParser
    ]

    def post(self, request):
        ...
```

---

# 5. APIView AND QUERYSET

You can write:

```python
class ProductView(APIView):

    queryset = Product.objects.all()
```

but `APIView` does not provide the generic queryset machinery automatically.

You normally have to access it yourself:

```python
products = Product.objects.all()
```

or:

```python
products = self.queryset
```

For automatic queryset handling, use:

```python
GenericAPIView
```

---

# 6. WHEN TO USE APIView

Use `APIView` when the endpoint is highly customized.

Examples:

```text
POST /api/login/
POST /api/upload-cv/
POST /api/send-email/
POST /api/payment/
POST /api/verify-payment/
```

These operations may not fit normal CRUD behavior.

---

# 7. GenericAPIView

Import:

```python
from rest_framework.generics import GenericAPIView
```

`GenericAPIView` builds on `APIView`.

Conceptually:

```text
APIView
   ↓
GenericAPIView
```

Example:

```python
class ProductView(GenericAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

GenericAPIView provides useful methods such as:

```python
self.get_queryset()
self.get_serializer()
self.get_serializer_class()
self.get_object()
self.filter_queryset()
self.paginate_queryset()
self.get_paginated_response()
```

---

# 8. queryset

You can define:

```python
class ProductView(GenericAPIView):

    queryset = Product.objects.all()
```

Then:

```python
products = self.get_queryset()
```

Instead of:

```python
products = Product.objects.all()
```

This becomes especially powerful when overriding:

```python
def get_queryset(self):
    ...
```

---

# 9. STATIC queryset

If your queryset is always the same:

```python
class ProductListView(ListAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

This is enough.

---

# 10. DYNAMIC get_queryset()

Use `get_queryset()` when the queryset depends on something.

Example:

```python
class OrderViewSet(ModelViewSet):

    queryset = Order.objects.all()
    serializer_class = OrderSerializer

    def get_queryset(self):

        if self.request.user.is_staff:
            return Order.objects.all()

        return Order.objects.filter(
            user=self.request.user
        )
```

Now:

```text
STAFF
    ↓
all orders

NORMAL USER
    ↓
only their orders
```

---

# 11. get_queryset() AND query parameters

You can also use query parameters:

```python
def get_queryset(self):

    queryset = Product.objects.all()

    category = self.request.query_params.get(
        "category"
    )

    if category:
        queryset = queryset.filter(
            category=category
        )

    return queryset
```

Request:

```http
GET /api/products/?category=phones
```

Flow:

```text
request
   ↓
query_params
   ↓
category
   ↓
filter queryset
```

---

# 12. serializer_class

You can define:

```python
class ProductView(GenericAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

Then:

```python
serializer = self.get_serializer(
    data=request.data
)
```

DRF gets the serializer from:

```python
serializer_class
```

---

# 13. get_serializer()

Instead of manually doing:

```python
serializer = ProductSerializer(
    data=request.data
)
```

you can use:

```python
serializer = self.get_serializer(
    data=request.data
)
```

This is better when using dynamic serializers because `get_serializer()` uses:

```python
get_serializer_class()
```

---

# 14. get_serializer_class()

You can dynamically choose the serializer.

Example:

```python
class ProductViewSet(ModelViewSet):

    queryset = Product.objects.all()

    def get_serializer_class(self):

        if self.action == "list":
            return ProductListSerializer

        if self.action == "retrieve":
            return ProductDetailSerializer

        if self.action == "create":
            return ProductCreateSerializer

        return ProductSerializer
```

Now:

```text
GET /products/
        ↓
list
        ↓
ProductListSerializer
```

```text
GET /products/5/
        ↓
retrieve
        ↓
ProductDetailSerializer
```

```text
POST /products/
        ↓
create
        ↓
ProductCreateSerializer
```

---

# 15. get_object()

GenericAPIView provides:

```python
self.get_object()
```

Example:

```python
class ProductDetailView(RetrieveAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer

    def get(self, request, *args, **kwargs):

        product = self.get_object()

        serializer = self.get_serializer(product)

        return Response(serializer.data)
```

`get_object()` uses the queryset and lookup configuration to find the requested object.

---

# 16. lookup_field

By default, DRF commonly looks up using:

```python
pk
```

You can change it:

```python
class ProductDetailView(RetrieveAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer

    lookup_field = "slug"
```

Now:

```text
/products/laptop/
```

can look up:

```python
Product.objects.get(
    slug="laptop"
)
```

---

# 17. lookup_url_kwarg

You can also control the URL keyword argument:

```python
lookup_field = "slug"
lookup_url_kwarg = "product_slug"
```

For example:

```text
/products/<product_slug>/
```

DRF can use:

```python
self.kwargs["product_slug"]
```

---

# 18. GENERIC VIEWS

DRF provides specialized generic views.

Main ones:

```text
ListAPIView
CreateAPIView
RetrieveAPIView
UpdateAPIView
DestroyAPIView

ListCreateAPIView

RetrieveUpdateAPIView
RetrieveDestroyAPIView
RetrieveUpdateDestroyAPIView
```

---

# 19. ListAPIView

Used for listing objects.

```python
from rest_framework.generics import ListAPIView


class ProductListView(ListAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

Provides:

```http
GET /products/
```

---

# 20. CreateAPIView

Used for creating.

```python
from rest_framework.generics import CreateAPIView


class ProductCreateView(CreateAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

Provides:

```http
POST /products/
```

---

# 21. RetrieveAPIView

Used to retrieve one object.

```python
class ProductDetailView(RetrieveAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

Provides:

```http
GET /products/5/
```

---

# 22. UpdateAPIView

Used for:

```text
PUT
PATCH
```

Example:

```python
class ProductUpdateView(UpdateAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

---

# 23. DestroyAPIView

Used for:

```text
DELETE
```

Example:

```python
class ProductDeleteView(DestroyAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

---

# 24. ListCreateAPIView

Provides:

```text
GET
POST
```

Example:

```python
class ProductListCreateView(
    ListCreateAPIView
):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

---

# 25. RetrieveUpdateAPIView

Provides:

```text
GET
PUT
PATCH
```

Example:

```python
class ProductDetailView(
    RetrieveUpdateAPIView
):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

---

# 26. RetrieveDestroyAPIView

Provides:

```text
GET
DELETE
```

Example:

```python
class ProductDetailView(
    RetrieveDestroyAPIView
):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

---

# 27. RetrieveUpdateDestroyAPIView

Provides:

```text
GET
PUT
PATCH
DELETE
```

Example:

```python
class ProductDetailView(
    RetrieveUpdateDestroyAPIView
):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

---

# 28. ViewSet

Import:

```python
from rest_framework.viewsets import ViewSet
```

ViewSets use actions instead of directly writing HTTP methods.

Example:

```python
class ProductViewSet(ViewSet):

    def list(self, request):
        ...

    def retrieve(self, request, pk=None):
        ...

    def create(self, request):
        ...
```

Common actions:

```text
list()
retrieve()
create()
update()
partial_update()
destroy()
```

---

# 29. ViewSet HTTP MAPPING

```text
GET /products/
        ↓
list()

POST /products/
        ↓
create()

GET /products/5/
        ↓
retrieve()

PUT /products/5/
        ↓
update()

PATCH /products/5/
        ↓
partial_update()

DELETE /products/5/
        ↓
destroy()
```

---

# 30. GenericViewSet

`GenericViewSet` provides the generic machinery:

```text
queryset
serializer_class
get_queryset()
get_serializer()
get_serializer_class()
get_object()
filtering
pagination
```

But it does not automatically provide all CRUD actions.

Example:

```python
from rest_framework.viewsets import GenericViewSet


class ProductViewSet(GenericViewSet):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

You normally combine it with mixins.

---

# 31. MIXINS

DRF provides:

```python
ListModelMixin
CreateModelMixin
RetrieveModelMixin
UpdateModelMixin
DestroyModelMixin
```

These provide standard actions.

---

# 32. List + Create ViewSet

```python
from rest_framework.mixins import (
    ListModelMixin,
    CreateModelMixin
)

from rest_framework.viewsets import GenericViewSet


class ProductViewSet(
    ListModelMixin,
    CreateModelMixin,
    GenericViewSet
):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

Provides:

```text
list()
create()
```

Therefore:

```text
GET /products/
POST /products/
```

---

# 33. List + Retrieve

```python
class ProductViewSet(
    ListModelMixin,
    RetrieveModelMixin,
    GenericViewSet
):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

Provides:

```text
list()
retrieve()
```

---

# 34. Why mixins?

Mixins allow you to choose exactly which operations your ViewSet supports.

For example:

```python
class ProductViewSet(
    ListModelMixin,
    RetrieveModelMixin,
    GenericViewSet
):
    ...
```

This gives:

```text
GET /products/
GET /products/5/
```

but not:

```text
POST
PUT
PATCH
DELETE
```

---

# 35. ModelViewSet

`ModelViewSet` gives you the standard CRUD actions.

```python
from rest_framework.viewsets import ModelViewSet


class ProductViewSet(ModelViewSet):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

It provides:

```text
list()
create()
retrieve()
update()
partial_update()
destroy()
```

---

# 36. ModelViewSet CRUD MAPPING

```text
GET /products/
        ↓
list()

POST /products/
        ↓
create()

GET /products/5/
        ↓
retrieve()

PUT /products/5/
        ↓
update()

PATCH /products/5/
        ↓
partial_update()

DELETE /products/5/
        ↓
destroy()
```

---

# 37. ROUTERS

ViewSets commonly use routers.

```python
from rest_framework.routers import DefaultRouter

router = DefaultRouter()

router.register(
    "products",
    ProductViewSet,
    basename="product"
)
```

Then:

```python
urlpatterns = router.urls
```

The router creates the standard URLs.

---

# 38. APIView VS Generic Views VS ViewSets

```text
APIView
    ↓
You control HTTP methods directly

GenericAPIView
    ↓
You get reusable queryset/serializer/object functionality

Generic Views
    ↓
You get predefined CRUD operations

ViewSet
    ↓
You work with actions

ModelViewSet
    ↓
You get complete CRUD actions + routers
```

---

# 39. SERIALIZER DIFFERENCE

You may see:

```python
serializer_class = ProductSerializer
```

You can also dynamically select one:

```python
def get_serializer_class(self):

    if self.action == "list":
        return ProductListSerializer

    if self.action == "retrieve":
        return ProductDetailSerializer

    return ProductSerializer
```

---

# 40. `self.action`

`self.action` is particularly important with ViewSets.

Standard actions:

```text
list
retrieve
create
update
partial_update
destroy
```

Example:

```python
def get_serializer_class(self):

    if self.action == "list":
        return ProductListSerializer

    if self.action == "retrieve":
        return ProductDetailSerializer

    if self.action == "create":
        return ProductCreateSerializer

    return ProductSerializer
```

---

# 41. Custom Actions

Import:

```python
from rest_framework.decorators import action
```

Example:

```python
class OrderViewSet(ModelViewSet):

    queryset = Order.objects.all()
    serializer_class = OrderSerializer

    @action(
        detail=False,
        methods=["get"]
    )
    def my_orders(self, request):

        orders = Order.objects.filter(
            user=request.user
        )

        serializer = self.get_serializer(
            orders,
            many=True
        )

        return Response(serializer.data)
```

URL:

```text
GET /orders/my_orders/
```

---

# 42. `detail=False`

Means the action operates on the collection.

```python
@action(
    detail=False,
    methods=["get"]
)
def my_orders(self, request):
    ...
```

Conceptually:

```text
/orders/my_orders/
```

No individual primary key is required.

---

# 43. `detail=True`

Means the action operates on one object.

```python
@action(
    detail=True,
    methods=["post"]
)
def cancel(self, request, pk=None):

    order = self.get_object()

    order.status = "Cancelled"
    order.save()

    return Response({
        "message": "Order cancelled"
    })
```

URL:

```text
POST /orders/5/cancel/
```

Flow:

```text
/orders/5/cancel/
        ↓
cancel()
        ↓
self.get_object()
        ↓
Order #5
```

---

# 44. Custom action + custom serializer

You can use a different serializer for a custom action:

```python
def get_serializer_class(self):

    if self.action == "cancel":
        return CancelOrderSerializer

    if self.action == "list":
        return OrderListSerializer

    return OrderSerializer
```

---

# 45. DYNAMIC PERMISSIONS

Static permission:

```python
permission_classes = [
    IsAuthenticated
]
```

Dynamic permissions:

```python
def get_permissions(self):

    if self.action == "list":
        permission_classes = [
            AllowAny
        ]

    elif self.action == "create":
        permission_classes = [
            IsAuthenticated
        ]

    elif self.action in [
        "update",
        "partial_update",
        "destroy"
    ]:
        permission_classes = [
            IsAdminUser
        ]

    else:
        permission_classes = [
            IsAuthenticated
        ]

    return [
        permission()
        for permission in permission_classes
    ]
```

---

# 46. Why `permission()`?

This:

```python
IsAuthenticated
```

is a class.

This:

```python
IsAuthenticated()
```

is an instance.

DRF expects permission instances.

Therefore:

```python
return [
    permission()
    for permission in permission_classes
]
```

---

# 47. APIView DYNAMIC PERMISSIONS

APIView does not normally have:

```python
self.action
```

like a ViewSet.

You can inspect:

```python
self.request.method
```

Example:

```python
def get_permissions(self):

    if self.request.method == "GET":
        permission_classes = [
            AllowAny
        ]

    else:
        permission_classes = [
            IsAuthenticated
        ]

    return [
        permission()
        for permission in permission_classes
    ]
```

---

# 48. DYNAMIC QUERYSET + SERIALIZER + PERMISSION

This is a very common professional pattern.

```python
class OrderViewSet(ModelViewSet):

    queryset = Order.objects.all()

    def get_queryset(self):

        if self.request.user.is_staff:
            return Order.objects.all()

        return Order.objects.filter(
            user=self.request.user
        )

    def get_serializer_class(self):

        if self.action == "list":
            return OrderListSerializer

        if self.action == "retrieve":
            return OrderDetailSerializer

        if self.action == "create":
            return OrderCreateSerializer

        return OrderSerializer

    def get_permissions(self):

        if self.action == "create":
            permission_classes = [
                IsAuthenticated
            ]

        elif self.action in [
            "update",
            "partial_update",
            "destroy"
        ]:
            permission_classes = [
                IsAdminUser
            ]

        else:
            permission_classes = [
                IsAuthenticated
            ]

        return [
            permission()
            for permission in permission_classes
        ]
```

---

# 49. Parser Classes

Parsers determine how DRF reads request data.

Common parsers:

```python
JSONParser
FormParser
MultiPartParser
```

Import:

```python
from rest_framework.parsers import (
    JSONParser,
    FormParser,
    MultiPartParser
)
```

---

# 50. JSONParser

Handles:

```text
application/json
```

Example:

```json
{
    "name": "Laptop",
    "price": 500
}
```

---

# 51. FormParser

Handles:

```text
application/x-www-form-urlencoded
```

---

# 52. MultiPartParser

Handles:

```text
multipart/form-data
```

It is particularly important for file uploads.

Example:

```python
class UploadCVView(APIView):

    parser_classes = [
        MultiPartParser,
        FormParser
    ]

    def post(self, request):
        ...
```

---

# 53. CV UPLOAD EXAMPLE

Serializer:

```python
from rest_framework import serializers


class CVUploadSerializer(serializers.ModelSerializer):

    class Meta:
        model = User
        fields = ["cv"]
```

View:

```python
class UploadCVView(APIView):

    serializer_class = CVUploadSerializer

    parser_classes = [
        MultiPartParser,
        FormParser
    ]

    def patch(self, request):

        user = request.user

        serializer = self.serializer_class(
            instance=user,
            data=request.data,
            partial=True
        )

        serializer.is_valid(
            raise_exception=True
        )

        serializer.save()

        return Response(serializer.data)
```

---

# 54. Why does `serializer.save()` call update()?

If you do:

```python
serializer = CVUploadSerializer(
    instance=user,
    data=request.data,
    partial=True
)
```

you supplied an existing instance.

Therefore:

```python
serializer.save()
```

means approximately:

```text
instance exists
      ↓
update()
```

If you do:

```python
serializer = CVUploadSerializer(
    data=request.data
)
```

there is no instance.

Therefore:

```text
no instance
     ↓
create()
```

---

# 55. ModelSerializer VS Serializer

With:

```python
class ProductSerializer(
    serializers.ModelSerializer
):
    ...
```

DRF provides standard model-aware implementations of:

```text
create()
update()
```

With:

```python
class ProductSerializer(
    serializers.Serializer
):
    ...
```

you generally implement database-saving behavior yourself when needed.

Example:

```python
class ProductSerializer(serializers.Serializer):

    name = serializers.CharField()
    price = serializers.DecimalField(
        max_digits=10,
        decimal_places=2
    )

    def create(self, validated_data):

        return Product.objects.create(
            **validated_data
        )

    def update(self, instance, validated_data):

        instance.name = validated_data.get(
            "name",
            instance.name
        )

        instance.price = validated_data.get(
            "price",
            instance.price
        )

        instance.save()

        return instance
```

---

# 56. Serializer `create()` VS ViewSet `create()`

These are NOT the same.

ViewSet:

```python
def create(
    self,
    request,
    *args,
    **kwargs
):
    ...
```

Serializer:

```python
def create(
    self,
    validated_data
):
    ...
```

ViewSet `create()` handles the HTTP/API operation.

Serializer `create()` handles creating the object/data.

---

# 57. Serializer `update()` VS ViewSet `update()`

Again, they are different.

ViewSet:

```python
def update(
    self,
    request,
    *args,
    **kwargs
):
    ...
```

Serializer:

```python
def update(
    self,
    instance,
    validated_data
):
    ...
```

ViewSet:

```text
HTTP request level
```

Serializer:

```text
object/data level
```

---

# 58. PATCH FLOW

For:

```http
PATCH /products/5/
```

the conceptual flow is:

```text
PATCH request
      ↓
ViewSet.partial_update()
      ↓
get_object()
      ↓
get_serializer(
    instance=product,
    data=request.data,
    partial=True
)
      ↓
serializer.is_valid()
      ↓
serializer.save()
      ↓
Serializer.update()
      ↓
product.save()
```

---

# 59. PUT VS PATCH

PUT generally represents a full update.

PATCH represents a partial update.

Example:

```text
Product:

name
price
stock
description
```

PATCH:

```json
{
    "price": 500
}
```

means:

```text
change price
leave other fields alone
```

This is why:

```python
partial=True
```

is associated with partial updates.

---

# 60. `perform_create()`

Generic views/ViewSets provide useful save hooks.

Example:

```python
def perform_create(self, serializer):

    serializer.save(
        user=self.request.user
    )
```

This is useful when the serializer shouldn't require the user from the client.

Client sends:

```json
{
    "product": 5,
    "quantity": 2
}
```

The server adds:

```python
user=request.user
```

---

# 61. `perform_update()`

Example:

```python
def perform_update(self, serializer):

    serializer.save(
        updated_by=self.request.user
    )
```

---

# 62. `perform_destroy()`

Example:

```python
def perform_destroy(self, instance):

    # Custom cleanup
    instance.delete()
```

You can also perform logging, cleanup, related operations, etc.

---

# 63. When to use `create()` vs `perform_create()`

If you only need to customize saving:

```python
def perform_create(self, serializer):
    serializer.save(
        user=self.request.user
    )
```

is usually simpler.

If you need to completely customize the HTTP response or workflow:

```python
def create(self, request, *args, **kwargs):
    ...
```

may be appropriate.

---

# 64. Filtering

Example:

```python
from django_filters.rest_framework import DjangoFilterBackend


class ProductListView(ListAPIView):

    queryset = Product.objects.all()
    serializer_class = ProductSerializer

    filter_backends = [
        DjangoFilterBackend
    ]

    filterset_fields = [
        "category",
        "in_stock"
    ]
```

Request:

```http
GET /products/?category=phones
```

---

# 65. Pagination

You can configure:

```python
pagination_class = ProductPagination
```

or disable pagination:

```python
pagination_class = None
```

For example:

```python
class OrderViewSet(ModelViewSet):

    queryset = Order.objects.all()
    serializer_class = OrderSerializer

    pagination_class = None
```

---

# 66. Authentication

Static:

```python
authentication_classes = [
    JWTAuthentication
]
```

For example:

```python
from rest_framework_simplejwt.authentication import (
    JWTAuthentication
)


class ProductViewSet(ModelViewSet):

    authentication_classes = [
        JWTAuthentication
    ]
```

Authentication answers:

```text
WHO ARE YOU?
```

---

# 67. Permission

Permission answers:

```text
ARE YOU ALLOWED TO DO THIS?
```

Example:

```python
permission_classes = [
    IsAuthenticated
]
```

Authentication and permission are different.

```text
Authentication
       ↓
identify user

Permission
       ↓
decide access
```

---

# 68. Renderer classes

Renderers determine how DRF produces responses.

For example:

```python
renderer_classes = [
    JSONRenderer
]
```

Common renderer:

```python
from rest_framework.renderers import JSONRenderer
```

Most API projects use JSON responses.

---

# 69. Throttling

You can configure:

```python
throttle_classes = [
    ...
]
```

This controls request rates.

For example:

```python
class ProductView(APIView):

    throttle_classes = [
        AnonRateThrottle
    ]
```

---

# 70. COMMON CONFIGURATION

Many DRF views can have configuration such as:

```python
queryset
serializer_class
permission_classes
authentication_classes
parser_classes
renderer_classes
throttle_classes
filter_backends
pagination_class
```

But their exact built-in behavior depends on the base class.

---

# 71. IMPORTANT DIFFERENCE

Do not assume every class gives every feature automatically.

For example:

```text
APIView
```

is more manual.

```text
GenericAPIView
```

provides generic queryset/serializer/object functionality.

```text
ListAPIView
```

provides list behavior.

```text
ModelViewSet
```

provides standard CRUD actions.

---

# 72. COMPARISON TABLE

```text
+----------------------+-----------+----------------+----------------+
| Feature              | APIView   | Generic View  | ModelViewSet   |
+----------------------+-----------+----------------+----------------+
| get()                | Yes       | Usually auto   | No             |
| post()               | Yes       | Usually auto   | No             |
| put()                | Yes       | Usually auto   | No             |
| patch()              | Yes       | Usually auto   | No             |
| delete()             | Yes       | Usually auto   | No             |
| queryset             | Manual    | Built-in       | Built-in       |
| get_queryset()       | Manual    | Built-in       | Built-in       |
| serializer_class     | Manual    | Built-in       | Built-in       |
| get_serializer()     | Manual    | Built-in       | Built-in       |
| get_object()         | Manual    | Built-in       | Built-in       |
| list()               | No        | Depending      | Yes            |
| retrieve()           | No        | Depending      | Yes            |
| create()             | No        | Depending      | Yes            |
| update()             | No        | Depending      | Yes            |
| partial_update()     | No        | Depending      | Yes            |
| destroy()            | No        | Depending      | Yes            |
| Router support       | No        | No             | Yes            |
| @action              | No        | No             | Yes            |
+----------------------+-----------+----------------+----------------+
```

---

# 73. MOST IMPORTANT METHODS BY CATEGORY

## Queryset

```python
queryset = Product.objects.all()
```

```python
def get_queryset(self):
    ...
```

---

## Serializer

```python
serializer_class = ProductSerializer
```

```python
def get_serializer_class(self):
    ...
```

```python
def get_serializer(self, *args, **kwargs):
    ...
```

---

## Object lookup

```python
def get_object(self):
    ...
```

---

## Permissions

```python
permission_classes = [...]
```

```python
def get_permissions(self):
    ...
```

---

## Authentication

```python
authentication_classes = [...]
```

```python
def get_authenticators(self):
    ...
```

---

## Filtering

```python
filter_backends = [...]
```

```python
filterset_class = ProductFilter
```

```python
filterset_fields = [...]
```

---

## Pagination

```python
pagination_class = ProductPagination
```

---

## Parsers

```python
parser_classes = [
    MultiPartParser,
    FormParser
]
```

---

## ViewSet actions

```python
list()
retrieve()
create()
update()
partial_update()
destroy()
```

---

## Save hooks

```python
perform_create()
perform_update()
perform_destroy()
```

---

## Custom actions

```python
@action(...)
```

---

# 74. APIView DYNAMIC BEHAVIOR

APIView doesn't give you `self.action`.

You can inspect:

```python
self.request.method
```

Example:

```python
class ProductView(APIView):

    def get_permissions(self):

        if self.request.method == "GET":
            return [AllowAny()]

        return [IsAuthenticated()]
```

---

# 75. Generic VIEW DYNAMIC BEHAVIOR

Generic views have methods such as:

```python
def get_queryset(self):
    ...

def get_serializer_class(self):
    ...

def get_permissions(self):
    ...
```

Example:

```python
class ProductListCreateView(
    ListCreateAPIView
):

    queryset = Product.objects.all()

    def get_serializer_class(self):

        if self.request.method == "GET":
            return ProductListSerializer

        return ProductCreateSerializer
```

---

# 76. ViewSet DYNAMIC BEHAVIOR

ViewSets have:

```python
self.action
```

Example:

```python
def get_serializer_class(self):

    if self.action == "list":
        return ProductListSerializer

    if self.action == "retrieve":
        return ProductDetailSerializer

    if self.action == "create":
        return ProductCreateSerializer

    return ProductSerializer
```

This is one of the major advantages of ViewSets.

---

# 77. FULL PROFESSIONAL EXAMPLE

Here is a more complete example:

```python
from rest_framework.viewsets import ModelViewSet
from rest_framework.permissions import (
    AllowAny,
    IsAuthenticated,
    IsAdminUser
)
from django_filters.rest_framework import (
    DjangoFilterBackend
)

from .models import Product

from .serializers import (
    ProductListSerializer,
    ProductDetailSerializer,
    ProductCreateSerializer,
    ProductSerializer
)


class ProductViewSet(ModelViewSet):

    queryset = Product.objects.all()

    filter_backends = [
        DjangoFilterBackend
    ]

    filterset_fields = [
        "category",
        "in_stock"
    ]

    def get_queryset(self):

        queryset = Product.objects.all()

        search = self.request.query_params.get(
            "search"
        )

        if search:
            queryset = queryset.filter(
                name__icontains=search
            )

        return queryset

    def get_serializer_class(self):

        if self.action == "list":
            return ProductListSerializer

        if self.action == "retrieve":
            return ProductDetailSerializer

        if self.action == "create":
            return ProductCreateSerializer

        return ProductSerializer

    def get_permissions(self):

        if self.action in [
            "list",
            "retrieve"
        ]:
            permission_classes = [
                AllowAny
            ]

        elif self.action == "create":
            permission_classes = [
                IsAuthenticated
            ]

        else:
            permission_classes = [
                IsAdminUser
            ]

        return [
            permission()
            for permission in permission_classes
        ]

    def perform_create(self, serializer):

        serializer.save(
            created_by=self.request.user
        )
```

This demonstrates:

```text
queryset
get_queryset()
serializer_class
get_serializer_class()
get_permissions()
filter_backends
filterset_fields
perform_create()
self.action
```

---

# 78. COMPLETE REQUEST FLOW

For:

```http
GET /products/
```

with a `ModelViewSet`:

```text
HTTP REQUEST
     ↓
Router
     ↓
ProductViewSet
     ↓
action = "list"
     ↓
get_permissions()
     ↓
get_queryset()
     ↓
filter_queryset()
     ↓
paginate_queryset()
     ↓
get_serializer_class()
     ↓
get_serializer()
     ↓
serializer.data
     ↓
HTTP RESPONSE
```

---

# 79. CREATE FLOW

For:

```http
POST /products/
```

approximately:

```text
POST
 ↓
Router
 ↓
create()
 ↓
get_serializer()
 ↓
is_valid()
 ↓
perform_create()
 ↓
serializer.save()
 ↓
Serializer.create()
 ↓
database
 ↓
Response
```

---

# 80. UPDATE FLOW

For:

```http
PUT /products/5/
```

approximately:

```text
PUT
 ↓
update()
 ↓
get_object()
 ↓
get_serializer(
    instance=product
)
 ↓
is_valid()
 ↓
perform_update()
 ↓
serializer.save()
 ↓
Serializer.update()
 ↓
database
```

---

# 81. PATCH FLOW

For:

```http
PATCH /products/5/
```

approximately:

```text
PATCH
 ↓
partial_update()
 ↓
get_object()
 ↓
get_serializer(
    instance=product,
    partial=True
)
 ↓
is_valid()
 ↓
perform_update()
 ↓
serializer.save()
 ↓
Serializer.update()
 ↓
database
```

---

# 82. DELETE FLOW

For:

```http
DELETE /products/5/
```

approximately:

```text
DELETE
 ↓
destroy()
 ↓
get_object()
 ↓
perform_destroy()
 ↓
instance.delete()
 ↓
Response
```

---

# 83. VERY IMPORTANT: WHERE TO PUT YOUR LOGIC

A useful rule:

```text
VIEW
 ↓
HTTP/API behavior
```

```text
SERIALIZER
 ↓
validation + transformation + object creation/update
```

```text
MODEL
 ↓
database/data rules
```

For example:

### View

```python
def get_permissions(self):
    ...
```

Access control.

### Serializer

```python
def validate(self, attrs):
    ...
```

Data validation.

### Serializer

```python
def create(self, validated_data):
    ...
```

Custom object creation.

### Serializer

```python
def update(self, instance, validated_data):
    ...
```

Custom object update.

### View

```python
def get_queryset(self):
    ...
```

Which objects the endpoint can work with.

---

# 84. COMMON MISTAKES

## Mistake 1

Trying:

```python
self.action
```

inside a normal `APIView`.

`APIView` doesn't work like a ViewSet.

Use:

```python
self.request.method
```

when appropriate.

---

## Mistake 2

Confusing:

```python
get_queryset()
```

with:

```python
queryset
```

`queryset` is the base configuration.

`get_queryset()` lets you dynamically determine the queryset.

---

## Mistake 3

Confusing:

```python
get_serializer_class()
```

with:

```python
get_serializer()
```

`get_serializer_class()` returns the serializer **class**.

Example:

```python
return ProductSerializer
```

`get_serializer()` creates a serializer **instance**.

Example:

```python
serializer = self.get_serializer(
    data=request.data
)
```

---

## Mistake 4

Confusing ViewSet `update()` with Serializer `update()`.

View:

```python
def update(
    self,
    request,
    *args,
    **kwargs
):
```

Serializer:

```python
def update(
    self,
    instance,
    validated_data
):
```

They have different jobs.

---

# 85. FINAL CHEAT SHEET

```text
APIView
│
├── get()
├── post()
├── put()
├── patch()
└── delete()
```

```text
GenericAPIView
│
├── queryset
├── serializer_class
├── get_queryset()
├── get_serializer()
├── get_serializer_class()
├── get_object()
├── filter_queryset()
├── paginate_queryset()
└── get_paginated_response()
```

```text
Generic Views
│
├── ListAPIView
├── CreateAPIView
├── RetrieveAPIView
├── UpdateAPIView
├── DestroyAPIView
│
├── ListCreateAPIView
├── RetrieveUpdateAPIView
├── RetrieveDestroyAPIView
└── RetrieveUpdateDestroyAPIView
```

```text
ViewSet
│
├── list()
├── retrieve()
├── create()
├── update()
├── partial_update()
└── destroy()
```

```text
ModelViewSet
│
├── list()
├── retrieve()
├── create()
├── update()
├── partial_update()
└── destroy()
```

```text
Dynamic configuration
│
├── get_queryset()
├── get_serializer_class()
├── get_permissions()
├── get_authenticators()
└── other get_* hooks where appropriate
```

```text
Saving hooks
│
├── perform_create()
├── perform_update()
└── perform_destroy()
```

```text
Custom ViewSet endpoints
│
└── @action()
```

---

# 86. THE MOST IMPORTANT CONCEPT TO REMEMBER

When choosing a DRF view:

```text
Do I need complete custom HTTP behavior?
        │
        └── YES → APIView


Do I want standard CRUD but separate explicit endpoints?
        │
        └── YES → Generic Views


Do I have a resource with CRUD operations
and want router-generated URLs?
        │
        └── YES → ModelViewSet
```

Then customize:

```text
Which records?
    → get_queryset()

Which serializer?
    → get_serializer_class()

Which object?
    → get_object()

Who can access?
    → get_permissions()

How is request data parsed?
    → parser_classes

How is it filtered?
    → filter_backends

How is it paginated?
    → pagination_class

How is creation customized?
    → perform_create()

How is updating customized?
    → perform_update()

How is deletion customized?
    → perform_destroy()

Need a special endpoint?
    → @action()
```

The central DRF architecture to remember is:

```text
                     APIView
                        │
             ┌──────────┴──────────┐
             │                     │
      GenericAPIView             ViewSet
             │                     │
       Generic Views        GenericViewSet
             │                     │
             │                   Mixins
             │                     │
             │               ModelViewSet
             │                     │
             └──────────┬──────────┘
                        │
              queryset / serializer
              permissions / parsing
              filtering / pagination
                        │
                     API
```

And the most important request flow is:

```text
REQUEST
   ↓
VIEW
   ↓
permission/authentication
   ↓
get_queryset()
   ↓
get_object()       ← when one object is needed
   ↓
get_serializer_class()
   ↓
get_serializer()
   ↓
serializer.is_valid()
   ↓
perform_create() / perform_update()
   ↓
serializer.save()
   ↓
serializer.create() / serializer.update()
   ↓
DATABASE
   ↓
RESPONSE
```

This distinction between **view-level logic** and **serializer-level logic** is the key to understanding DRF well.
