```
class ReceiptItemViewSet(  
    mixins.ListModelMixin,  
    mixins.RetrieveModelMixin,  
    mixins.UpdateModelMixin,  
    mixins.DestroyModelMixin,  
    viewsets.GenericViewSet,  
):  
    queryset = ReceiptItem.objects.all()  
    serializer_class = ReceiptItemFullSerializer  
    permission_classes = [IsAuthenticatedOrReadOnly]  
    http_method_names = ["get", "post", "patch", "delete"]  
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]  
    filterset_class = ReceiptItemFilter  
    ordering_fields = ["id", "proof_id", "order", "status", "created"]  
    ordering = ["order"]  
  
    def get_authenticators(self):  
        if self.request and self.request.method in ["GET"]:  
            return super().get_authenticators()  
        return [CustomAuthentication()]  
  
    def get_queryset(self):  
        if self.action in ["create", "update"]:  
            # We need to prefetch the price object if it exists to validate the  
            # price_id field, and the proof object to validate the proof_id            # field            return (  
                ReceiptItem.objects.select_related("proof")  
                .select_related("price")  
                .all()  
            )  
        return super().get_queryset()  
  
    def create(self, request: Request, *args, **kwargs):  
        # validate  
        serializer = self.get_serializer(data=request.data)  
        serializer.is_valid(raise_exception=True)  
        # save  
  
        receipt_item = serializer.save()  
  
        # return full price tag  
        return Response(  
            self.serializer_class(receipt_item).data, status=status.HTTP_201_CREATED  
        )
```
This `ReceiptItemViewSet` is **very similar** to the previous two (`PriceViewSet`, `PriceTagViewSet`). Once you understand the pattern, you can read this file in **about 1 minute**. I'll break it down like a senior engineer would.

---

# 1️⃣ What `ReceiptItem` represents

A **ReceiptItem** is an **individual line item on a receipt**.

Example receipt:

```
Milk        $2.50
Bread       $3.00
Apple       $1.20
```

Each row becomes a **ReceiptItem**.

Data model idea:

```
Proof (receipt image)
     ↓
ReceiptItem (one row on receipt)
     ↓
Price
```

So the relationship is roughly:

```
Proof
 ├── ReceiptItem
 │     ├── price
 │     └── order
```

---

# 2️⃣ Endpoints provided

Because of the mixins:

```python
mixins.ListModelMixin
mixins.RetrieveModelMixin
mixins.UpdateModelMixin
mixins.DestroyModelMixin
```

Available APIs:

|Endpoint|Method|Description|
|---|---|---|
|`/receipt-items`|GET|list items|
|`/receipt-items/{id}`|GET|retrieve|
|`/receipt-items`|POST|create|
|`/receipt-items/{id}`|PATCH|update|
|`/receipt-items/{id}`|DELETE|delete|

---

# 3️⃣ Basic configuration

### queryset

```python
queryset = ReceiptItem.objects.all()
```

Default database query.

---

### serializer

```python
serializer_class = ReceiptItemFullSerializer
```

Used to convert model ↔ JSON.

---

### permissions

```python
permission_classes = [IsAuthenticatedOrReadOnly]
```

Meaning:

|user|permission|
|---|---|
|anonymous|read|
|logged user|write|

---

# 4️⃣ HTTP methods allowed

```python
http_method_names = ["get", "post", "patch", "delete"]
```

Disabled:

```
PUT
```

Preferred update style:

```
PATCH
```

This is common in REST APIs.

---

# 5️⃣ Filtering & ordering

```python
filter_backends = [DjangoFilterBackend, filters.OrderingFilter]
filterset_class = ReceiptItemFilter
```

Example queries:

```
/receipt-items?proof_id=10
/receipt-items?status=confirmed
```

Ordering:

```
/receipt-items?ordering=order
/receipt-items?ordering=-created
```

Allowed fields:

```python
ordering_fields = ["id", "proof_id", "order", "status", "created"]
```

Default ordering:

```python
ordering = ["order"]
```

Meaning items appear in **receipt order**.

Example:

```
1 Milk
2 Bread
3 Apple
```

---

# 6️⃣ Authentication logic

Same pattern as previous viewsets.

```python
def get_authenticators(self):
```

Rules:

|Method|Authentication|
|---|---|
|GET|public|
|POST|login required|
|PATCH|login required|
|DELETE|login required|

Because:

```
GET → default auth
others → CustomAuthentication
```

---

# 7️⃣ Special queryset for create/update

```python
def get_queryset(self):
```

If action is:

```
create
update
```

Use optimized query:

```python
ReceiptItem.objects.select_related("proof")
                   .select_related("price")
```

Why?

Serializer validation might check:

```
proof_id
price_id
```

So this avoids extra database queries.

---

# 8️⃣ Create logic

Create is very simple.

```python
def create(self, request):
```

### Step 1 — validate

```python
serializer = self.get_serializer(data=request.data)
serializer.is_valid(raise_exception=True)
```

Example request:

```json
{
 "proof_id": 101,
 "name": "Milk",
 "price": 2.50,
 "order": 1
}
```

---

### Step 2 — save

```python
receipt_item = serializer.save()
```

Django ORM creates database row.

---

### Step 3 — return full object

```python
return Response(
    self.serializer_class(receipt_item).data,
    status=201
)
```

Client receives full serialized item.

---

# 9️⃣ Why this ViewSet is simpler

Compared to **PriceTagViewSet**:

|Feature|PriceTag|ReceiptItem|
|---|---|---|
|AI extraction|yes|no|
|bounding boxes|yes|no|
|async tasks|yes|no|
|image OCR|yes|no|

So this file is **pure CRUD API**.

---

# 🔟 Where this fits in system architecture

Full data flow in this project probably looks like:

```
User uploads receipt
        ↓
Proof
        ↓
ReceiptItem (each row)
        ↓
Price
        ↓
Product
```

Example:

```
Proof: Woolworths receipt

ReceiptItem:
1 Milk
2 Bread
3 Apple
```

---

# 1️⃣1️⃣ How a senior engineer reads this file

They scan:

```
1 class name
2 queryset
3 serializer
4 filters
5 create()
```

And conclude:

```
Standard CRUD ViewSet
for receipt line items
```

Time to understand:

```
~30 seconds
```

---

# 1️⃣2️⃣ Big picture of these three ViewSets

You already looked at:

```
PriceViewSet
PriceTagViewSet
ReceiptItemViewSet
```

Architecture:

```
Proof (image)
   |
   +-- PriceTag  ← bounding boxes
   |
   +-- ReceiptItem ← receipt rows
           |
           +-- Price
```

---

✅ If you'd like, I can also show you something extremely useful for your learning:

**How to read a 50k-line Django project like this in 15 minutes**  
(with a visual map of the entire `open_prices` backend).