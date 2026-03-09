This file defines **`PriceTagViewSet`**, which manages **price tags extracted from proofs (usually receipt or shelf images)**. I’ll walk through it the way engineers analyze a DRF file quickly.

---

# 1️⃣ What this ViewSet controls

It manages **PriceTag objects**.

A **PriceTag** usually represents a **bounding box around a price in an image**.

Example:

```
Receipt image
 ┌─────────────┐
 │  Milk $2.50 │  ← PriceTag (bounding box)
 └─────────────┘
```

Typical flow in this system:

```
Proof (image)
     ↓
PriceTag (bounding box around price)
     ↓
Price (actual numeric price record)
```

---

# 2️⃣ Endpoints provided

Because of these mixins:

```python
mixins.ListModelMixin
mixins.RetrieveModelMixin
mixins.UpdateModelMixin
mixins.DestroyModelMixin
```

You get:

|Endpoint|Method|Description|
|---|---|---|
|`/price-tags`|GET|list price tags|
|`/price-tags/{id}`|GET|retrieve|
|`/price-tags`|POST|create|
|`/price-tags/{id}`|PATCH|update|
|`/price-tags/{id}`|DELETE|delete|

---

# 3️⃣ Query optimization

```python
queryset = (
    PriceTag.objects.select_related("proof", "proof__location")
    .prefetch_related("predictions")
    .all()
)
```

This prevents **N+1 database queries**.

### select_related

Used for **foreign keys**

```
PriceTag
   → proof
       → location
```

One SQL JOIN query.

---

### prefetch_related

Used for **many-to-many / reverse relations**

```
PriceTag
   → predictions
```

So predictions are fetched in one additional query.

---

# 4️⃣ Authentication logic

```python
authentication_classes = []
```

Actual logic:

```python
def get_authenticators(self):
```

Rules:

|Method|Auth|
|---|---|
|GET|public|
|POST|login required|
|PATCH|login required|
|DELETE|login required|

Because:

```
GET → default
others → CustomAuthentication
```

---

# 5️⃣ Permission

```python
permission_classes = [IsAuthenticatedOrReadOnly]
```

Meaning:

|user|permission|
|---|---|
|anonymous|read|
|logged user|write|

---

# 6️⃣ Serializer switching

```python
def get_serializer_class(self):
```

Different serializers depending on method.

|Method|Serializer|
|---|---|
|POST|PriceTagCreateSerializer|
|PATCH|PriceTagUpdateSerializer|
|GET|PriceTagFullSerializer|

This keeps validation clean.

---

# 7️⃣ Special queryset for create/update

```python
def get_queryset(self):
```

For create/update:

```python
PriceTag.objects.select_related("proof", "price")
```

Why?

Because validation needs:

```
proof_id
price_id
```

This avoids extra queries during serializer validation.

---

# 8️⃣ Delete logic (soft delete)

Destroy is overridden:

```python
def destroy(self):
```

First check:

```python
if price_tag.price_id is not None:
```

Meaning:

If the tag is already linked to a price:

❌ cannot delete.

Response:

```
403 Forbidden
```

Otherwise:

```
status = deleted
```

So it's **soft delete** instead of database delete.

---

# 9️⃣ Create logic

```python
def create(self):
```

Steps:

### 1 validate

```
serializer.is_valid()
```

---

### 2 save object

```python
price_tag = serializer.save(
    updated_by=user_id,
    created_by=user_id
)
```

Tracking who created the tag.

---

### 3 optional AI extraction

```python
use_ai = request.data.get("use_ai", False)
```

If enabled:

```python
async_task(
 "open_prices.proofs.ml.price_tags.run_and_save_price_tag_extraction_from_id"
)
```

This means:

```
AI reads image
→ detect price text
→ fill fields automatically
```

---

# 🔟 Update logic

```python
def update(self):
```

First store old bounding box:

```python
previous_bounding_box = previous_price_tag.bounding_box
```

Then update.

---

### AI re-run condition

AI runs **only if**:

```
1 not testing
2 bounding box changed
3 use_ai = true
```

Code:

```python
if previous_bounding_box != price_tag.bounding_box
```

Meaning:

User moved the price box.

Example:

```
Old bounding box → wrong position
User drags box → correct position
AI reruns extraction
```

---

# 1️⃣1️⃣ Async AI task

This function runs:

```
open_prices.proofs.ml.price_tags.update_price_tag_extraction
```

Probably:

```
1 crop image
2 OCR price
3 parse number
4 store prediction
```

And it's run in background via:

```
django-q
celery
rq
```

(using `async_task`)

---

# 1️⃣2️⃣ Data flow in this system

Full pipeline:

```
User uploads receipt image
        ↓
Proof
        ↓
PriceTag (bounding box)
        ↓
AI extraction
        ↓
Prediction
        ↓
Price object
```

---

# 1️⃣3️⃣ Why this architecture is good

Separating:

|object|responsibility|
|---|---|
|Proof|image|
|PriceTag|location of price|
|Prediction|AI result|
|Price|validated price|

This makes the system **human + AI collaborative**.

---

# 1️⃣4️⃣ Diagram of this file

```
PriceTagViewSet
     |
     +-- Create PriceTag
     |
     +-- Update bounding box
     |      |
     |      → run AI extraction
     |
     +-- Delete (soft delete)
     |
     +-- List / retrieve
```

---

# 1️⃣5️⃣ How senior engineers read this file (real trick)

They only scan:

```
1 class definition
2 queryset
3 serializers
4 create()
5 update()
6 destroy()
```

Total time:

```
~2 minutes
```

Understanding:

```
80–90%
```

---

✅ If you'd like, I can also show you something extremely useful for reading this project:

**How to understand the entire `open_prices` backend architecture in 10 minutes**  
(Proof → PriceTag → Price → Product → Location).

This is exactly how senior engineers approach unfamiliar codebases.