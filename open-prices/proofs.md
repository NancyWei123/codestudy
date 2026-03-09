
```
class ProofViewSet(  
    mixins.ListModelMixin,  
    mixins.RetrieveModelMixin,  
    mixins.UpdateModelMixin,  
    mixins.DestroyModelMixin,  
    viewsets.GenericViewSet,  
):  
    authentication_classes = []  # see get_authenticators  
    permission_classes = [  
        IsAuthenticatedOrReadOnly,  
        OnlyObjectOwnerOrModeratorIsAllowed,  # for edit & delete  
    ]  
    http_method_names = ["get", "post", "patch", "delete"]  # disable "put"  
    queryset = Proof.objects.all()  
    serializer_class = ProofFullSerializer  # see get_serializer_class  
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]  
    filterset_class = ProofFilter  
    ordering_fields = ["id", "date", "price_count", "created"]  
    ordering = ["id"]  
  
    def get_authenticators(self):  
        if self.request and self.request.method in ["GET"]:  
            # Note: we also allow anonymous upload ("POST")  
            return super().get_authenticators()  
        return [CustomAuthentication()]
```
# 1️⃣ What This Class Is

class ProofViewSet(  
    mixins.ListModelMixin,  
    mixins.RetrieveModelMixin,  
    mixins.UpdateModelMixin,  
    mixins.DestroyModelMixin,  
    viewsets.GenericViewSet,  
)

This defines a **ViewSet** in **Django REST Framework**.

The mixins determine which API operations are allowed.

|Mixin|Endpoint|Purpose|
|---|---|---|
|ListModelMixin|`GET /proofs`|list proofs|
|RetrieveModelMixin|`GET /proofs/{id}`|get one proof|
|UpdateModelMixin|`PATCH /proofs/{id}`|update proof|
|DestroyModelMixin|`DELETE /proofs/{id}`|delete proof|

Notice:

CreateModelMixin

is **not included**, so creation is probably implemented elsewhere.
# 2️⃣ Authentication

authentication_classes = []  # see get_authenticators

Authentication is handled dynamically by:

def get_authenticators(self):

Logic:

if request.method == "GET":  
    use default authentication  
else:  
    use CustomAuthentication

So:

|Request|Authentication|
|---|---|
|GET|default DRF authentication|
|POST/PATCH/DELETE|CustomAuthentication|

Purpose:

reading data is easier  
modifying data requires stronger authentication

---

# 3️⃣ Permissions

permission_classes = [  
    IsAuthenticatedOrReadOnly,  
    OnlyObjectOwnerOrModeratorIsAllowed,  
]

Meaning:

|User|Allowed|
|---|---|
|anonymous|read proofs|
|authenticated|modify proofs|
|owner|edit/delete own proof|
|moderator|edit/delete any proof|

---

# 4️⃣ Allowed HTTP Methods

http_method_names = ["get", "post", "patch", "delete"]

Allowed requests:

GET  
POST  
PATCH  
DELETE

Disabled:

PUT

PUT normally means **full object replacement**, which the project avoids.

---

# 5️⃣ Database Query

queryset = Proof.objects.all()

Equivalent SQL:

SELECT * FROM proofs;

This is the base query used by the API.

---

# 6️⃣ Serializer

serializer_class = ProofFullSerializer

The serializer converts:

Proof model  
↓  
JSON response

Example response:

{  
  "id": 25,  
  "image_url": "...",  
  "price_count": 5,  
  "created": "2024-06-01"  
}

---

# 7️⃣ Filtering

filter_backends = [DjangoFilterBackend, filters.OrderingFilter]  
filterset_class = ProofFilter

This allows filtering the API.

Example request:

GET /proofs?date=2024-06-01

Or sorting:

GET /proofs?ordering=-created

Backends used:

- **django-filter**
    
- DRF OrderingFilter
    

---

# 8️⃣ Sorting Fields

ordering_fields = ["id", "date", "price_count", "created"]  
ordering = ["id"]

Example:

GET /proofs?ordering=-price_count

Sort by number of prices.

Default sorting:

id ascending

---

# 9️⃣ Custom Authentication Logic

def get_authenticators(self):

Implementation:

if self.request.method in ["GET"]:  
    return super().get_authenticators()  
return [CustomAuthentication()]

Meaning:

|Request|Authentication|
|---|---|
|GET|default authentication|
|POST|CustomAuthentication|
|PATCH|CustomAuthentication|
|DELETE|CustomAuthentication|

So write operations require:

CustomAuthentication

---

# 🔟 Full Request Flow

Example request:

PATCH /proofs/25

Processing:

router  
↓  
ProofViewSet  
↓  
get_authenticators()  
↓  
CustomAuthentication  
↓  
permission check  
↓  
serializer validation  
↓  
update database  
↓  
return response

```
def get_queryset(self):  
    queryset = self.queryset  
    if self.request.method in ["GET"]:  
        queryset = queryset.select_related("location")  
        if self.action == "retrieve":  
            queryset = queryset.prefetch_related("predictions")  
    return queryset
```
# 1️⃣ The Function

def get_queryset(self):

This overrides the default method in **Django REST Framework**.

Normally DRF would simply use:

queryset = Proof.objects.all()

But here we **modify the query depending on the request**.

---

# 2️⃣ Start With Base Query

queryset = self.queryset

Earlier in the class:

queryset = Proof.objects.all()

So initially:

SELECT * FROM proof;

---

# 3️⃣ Optimize Only for GET Requests

if self.request.method in ["GET"]:

Meaning:

|Request|Optimization|
|---|---|
|GET|optimized query|
|POST/PATCH/DELETE|normal query|

Reason:

GET requests read lots of data  
writes don't need heavy optimization

---

# 4️⃣ Load Location in the Same Query

queryset = queryset.select_related("location")

`select_related` performs a **SQL JOIN**.

Without it:

Example loop:

for proof in proofs:  
    proof.location

would run **N+1 queries**.

Example problem:

1 query → load proofs  
100 queries → load locations

Total:

101 queries

With `select_related`:

SELECT proof.*, location.*  
FROM proof  
JOIN location  
ON proof.location_id = location.id

Now:

1 query only

---

# 5️⃣ Special Case: Retrieve Endpoint

if self.action == "retrieve":

This means the endpoint:

GET /proofs/{id}

Example:

GET /proofs/25

---

# 6️⃣ Load Predictions Efficiently

queryset = queryset.prefetch_related("predictions")

`prefetch_related` is used for **reverse relationships**.

Example relationship:

Proof  
  ↓  
Prediction

Without prefetch:

proof.predictions.all()

would trigger **extra queries**.

With `prefetch_related`:

Django runs 2 queries total:  
1 → proofs  
1 → predictions

Instead of many.

---

# 7️⃣ Return Optimized Query

return queryset

So the final behavior:

|Request|Query Optimization|
|---|---|
|GET /proofs|load location|
|GET /proofs/{id}|load location + predictions|
|POST/PATCH/DELETE|basic query|

---

# 8️⃣ Full Request Flow

Example:

GET /proofs/25

Flow:

ProofViewSet  
      ↓  
get_queryset()  
      ↓  
select_related(location)  
      ↓  
prefetch_related(predictions)  
      ↓  
optimized database queries  
      ↓  
serializer  
      ↓  
JSON response

```
def get_serializer_class(self):  
    if self.request.method == "PATCH":  
        return ProofUpdateSerializer  
    elif self.action == "list":  
        return ProofHalfFullSerializer  
    return self.serializer_class
```
# 1️⃣ The Function

def get_serializer_class(self):

Normally a ViewSet uses **one serializer**:

serializer_class = ProofFullSerializer

But this function overrides that behavior.

---

# 2️⃣ Case 1 — PATCH (Update)

if self.request.method == "PATCH":  
    return ProofUpdateSerializer

Used when updating a proof.

Example request:

PATCH /proofs/25

Serializer used:

ProofUpdateSerializer

Purpose:

only allow specific fields to be updated

Example request body:

{  
  "image_url": "new_receipt.jpg"  
}

---

# 3️⃣ Case 2 — List Endpoint

elif self.action == "list":  
    return ProofHalfFullSerializer

Used for:

GET /proofs

Serializer used:

ProofHalfFullSerializer

Why?

Because listing thousands of proofs with **full data would be heavy**.

So the project returns **lighter data**.

Example response:

[  
  {  
    "id": 1,  
    "date": "2024-06-01",  
    "price_count": 5  
  },  
  {  
    "id": 2,  
    "date": "2024-06-02",  
    "price_count": 3  
  }  
]

---

# 4️⃣ Default Case

return self.serializer_class

Earlier in the class:

serializer_class = ProofFullSerializer

So default:

ProofFullSerializer

Used for:

GET /proofs/{id}  
POST /proofs

This serializer returns **complete proof details**.

Example:

{  
  "id": 25,  
  "image_url": "...",  
  "date": "2024-06-01",  
  "location": {...},  
  "predictions": [...]  
}

---

# 5️⃣ Final Behavior

|Request|Serializer|
|---|---|
|GET /proofs|ProofHalfFullSerializer|
|GET /proofs/{id}|ProofFullSerializer|
|PATCH /proofs/{id}|ProofUpdateSerializer|

---

# 6️⃣ Why This Design Is Used

Benefits:

1. list endpoints return smaller responses  
2. update endpoints restrict editable fields  
3. retrieve endpoints return full object details

This improves:

performance  
security  
API clarity

---

# 7️⃣ Request Flow Example

Example request:

GET /proofs

Flow:

ProofViewSet  
      ↓  
get_serializer_class()  
      ↓  
ProofHalfFullSerializer  
      ↓  
serialize data  
      ↓  
return JSON
```
def destroy(self, request: Request, *args, **kwargs) -> Response:  
    proof = self.get_object()  
    if proof.prices.count():  
        return Response(  
            {"detail": "Cannot delete proof with associated prices"},  
            status=status.HTTP_403_FORBIDDEN,  
        )  
    return super().destroy(request, *args, **kwargs)
```
# 1️⃣ The Function

def destroy(self, request: Request, *args, **kwargs) -> Response:

This method handles:

DELETE /proofs/{id}

Example:

DELETE /proofs/25

In **Django REST Framework**, `destroy()` normally just deletes the object.

But here we add **custom logic before deletion**.

---

# 2️⃣ Get the Proof Object

proof = self.get_object()

This fetches the proof from the database.

Equivalent SQL:

SELECT * FROM proof WHERE id = 25;

Example object:

|id|image|price_count|
|---|---|---|
|25|receipt.jpg|3|

---

# 3️⃣ Check if Prices Exist

if proof.prices.count():

This checks if the proof has related prices.

Relationship example:

Proof  
   ↓  
Price

Meaning one proof can have many prices.

Example database:

### Proof table

|id|image|
|---|---|
|25|receipt.jpg|

### Price table

|id|proof_id|price|
|---|---|---|
|1|25|3.50|
|2|25|2.00|

Here:

proof.prices.count() = 2

So deletion is blocked.

---

# 4️⃣ Return Error Response

return Response(  
    {"detail": "Cannot delete proof with associated prices"},  
    status=status.HTTP_403_FORBIDDEN,  
)

API response:

{  
  "detail": "Cannot delete proof with associated prices"  
}

HTTP status:

403 Forbidden

Meaning:

You are not allowed to delete this proof  
because it is still used by prices.

---

# 5️⃣ If No Prices Exist

If:

proof.prices.count() == 0

Then the code runs:

return super().destroy(request, *args, **kwargs)

This calls the default DRF delete behavior.

Equivalent SQL:

DELETE FROM proof WHERE id=25;

Response:

204 No Content

Meaning the object was successfully deleted.

---

# 6️⃣ Full Request Flow

Example request:

DELETE /proofs/25

Processing:

ProofViewSet.destroy()  
        ↓  
get proof object  
        ↓  
check related prices  
        ↓  
if prices exist → return 403  
else → delete proof

---

# 7️⃣ Why This Rule Exists

Without this check:

delete proof  
↓  
prices still reference proof_id  
↓  
database inconsistency

So this protects **data integrity**.

---

# 8️⃣ Example Notes (Good Learning Style)

destroy()  
  
Endpoint  
DELETE /proofs/{id}  
  
Logic  
1. get proof  
2. check if proof has prices  
3. if yes → block deletion  
4. if no → delete proof
```
@extend_schema(request=ProofUploadSerializer, responses=ProofFullSerializer)  
@action(  
    detail=False,  
    methods=["POST"],  
    url_path="upload",  
    parser_classes=[MultiPartParser],  
    permission_classes=[],  # allow anonymous upload  
)  
def upload(self, request: Request) -> Response:  
    # build proof  
    file = request.data.get("file")  
    if not file:  
        return Response(  
            {"file": ["This field is required."]},  
            status=status.HTTP_400_BAD_REQUEST,  
        )  
    # NOTE: image will be stored even if the proof serializer fails...  
    file_path, mimetype, image_thumb_path = store_file(file)  
    image_md5_hash = compute_file_md5(file)  
    proof_create_data = {  
        "file_path": file_path,  
        "mimetype": mimetype,  
        "image_thumb_path": image_thumb_path,  
        **{  
            key: request.data.get(key)  
            for key in Proof.CREATE_FIELDS  
            if key in request.data  
        },  
    }  
    # validate  
    serializer = ProofCreateSerializer(data=proof_create_data)  
    serializer.is_valid(raise_exception=True)  
  
    # get owner (we allow anonymous upload, only if token is not present)  
    if self.request.user.is_authenticated:  
        owner = self.request.user.user_id  
    else:  
        if has_token_from_cookie_or_header(self.request):  
            return Response(  
                {  
                    "detail": "Authentication failed. Please pass a valid token, or remove it to upload the proof anonymously."  
                },  
                status=status.HTTP_400_BAD_REQUEST,  
            )  
        else:  
            owner = settings.ANONYMOUS_USER_ID  
  
    # get source  
    source = get_source_from_request(self.request)  
  
    # We check if a proof with the same MD5 hash already exists,  
    # uploaded by the same user, with the same type, location and date.    # If yes, we return it instead of creating a new one    duplicate_proof = Proof.objects.filter(  
        image_md5_hash=image_md5_hash,  
        owner=owner,  
        type=serializer.validated_data.get("type"),  
        # location OSM id/type can be null (for online stores)  
        location_osm_id=serializer.validated_data.get("location_osm_id"),  
        location_osm_type=serializer.validated_data.get("location_osm_type"),  
        date=serializer.validated_data.get("date"),  
    ).first()  
    if duplicate_proof:  
        # We remove the uploaded file as it's a duplicate  
        (settings.IMAGES_DIR / file_path).unlink(missing_ok=True)  
        response_status_code = status.HTTP_200_OK  
        # see note in common/openfoodfacts.py  
        if common_openfoodfacts.is_smoothie_app_version_leq_4_20(source):  
            response_status_code = status.HTTP_201_CREATED  
  
        return Response(  
            {**ProofFullSerializer(duplicate_proof).data, "detail": "duplicate"},  
            status=response_status_code,  
        )  
  
    save_kwargs = {  
        "owner": owner,  
        "image_md5_hash": image_md5_hash,  
        "source": source,  
    }  
    # see note in common/openfoodfacts.py  
    if common_openfoodfacts.is_smoothie_app_version_4_20(source):  
        save_kwargs["ready_for_price_tag_validation"] = False  
    # save  
    proof = serializer.save(**save_kwargs)  
    # return full proof  
    return Response(ProofFullSerializer(proof).data, status=status.HTTP_201_CREATED)
```
This method implements the **image upload endpoint for proofs (receipts / price tags)**.  
It’s one of the **core endpoints** of the project because it allows users (even anonymous ones) to **upload price evidence**.

Let’s break it down step by step.

---

# 1️⃣ API Endpoint

Decorators:

```python
@action(
    detail=False,
    methods=["POST"],
    url_path="upload",
    parser_classes=[MultiPartParser],
    permission_classes=[],
)
```

This creates the route:

```
POST /proofs/upload
```

Important settings:

|Setting|Meaning|
|---|---|
|methods=["POST"]|upload is a POST request|
|parser_classes=[MultiPartParser]|allows file uploads|
|permission_classes=[]|**anonymous users allowed**|

The parser comes from **Django REST Framework**.

---

# 2️⃣ API Documentation

```python
@extend_schema(request=ProofUploadSerializer, responses=ProofFullSerializer)
```

This describes the request/response structure for **drf-spectacular**.

Swagger will show:

```
POST /proofs/upload
Request: ProofUploadSerializer
Response: ProofFullSerializer
```

---

# 3️⃣ Get Uploaded File

```python
file = request.data.get("file")
```

Example request (multipart):

```
POST /proofs/upload
Content-Type: multipart/form-data
```

Body:

```
file: receipt.jpg
date: 2024-06-01
type: receipt
location_osm_id: 123
```

If no file is provided:

```python
return Response({"file": ["This field is required."]}, status=400)
```

Response:

```json
{
  "file": ["This field is required."]
}
```

---

# 4️⃣ Store File

```python
file_path, mimetype, image_thumb_path = store_file(file)
```

This function saves the uploaded file to disk.

Example result:

```
file_path = "images/abc123.jpg"
mimetype = "image/jpeg"
image_thumb_path = "images/thumb_abc123.jpg"
```

---

# 5️⃣ Compute File Hash

```python
image_md5_hash = compute_file_md5(file)
```

Example result:

```
9e107d9d372bb6826bd81d3542a419d6
```

Purpose:

```
detect duplicate uploads
```

---

# 6️⃣ Build Proof Data

```python
proof_create_data = {
    "file_path": file_path,
    "mimetype": mimetype,
    "image_thumb_path": image_thumb_path,
    ...
}
```

Additional fields come from:

```
Proof.CREATE_FIELDS
```

Example final data:

```json
{
  "file_path": "images/abc123.jpg",
  "mimetype": "image/jpeg",
  "date": "2024-06-01",
  "type": "receipt",
  "location_osm_id": 123
}
```

---

# 7️⃣ Validate Input

```python
serializer = ProofCreateSerializer(data=proof_create_data)
serializer.is_valid(raise_exception=True)
```

This ensures the data is valid before saving.

---

# 8️⃣ Determine Owner

Two cases.

### Authenticated user

```python
owner = self.request.user.user_id
```

### Anonymous upload

If no valid token:

```python
owner = settings.ANONYMOUS_USER_ID
```

But if an **invalid token is present**, the request fails:

```json
{
  "detail": "Authentication failed. Please pass a valid token..."
}
```

---

# 9️⃣ Get Source

```python
source = get_source_from_request(self.request)
```

Example:

```
web
mobile_app
api
```

Used for analytics and compatibility logic.

---

# 🔟 Detect Duplicate Proof

Query:

```python
duplicate_proof = Proof.objects.filter(...)
```

Conditions:

|Field|Must match|
|---|---|
|image_md5_hash|same image|
|owner|same user|
|type|same proof type|
|location_osm_id|same location|
|date|same date|

If found:

```
return existing proof instead of creating a new one
```

Also delete the uploaded duplicate file:

```python
(settings.IMAGES_DIR / file_path).unlink()
```

Response:

```json
{
  "id": 25,
  "detail": "duplicate"
}
```

---

# 11️⃣ Save New Proof

If no duplicate:

```python
proof = serializer.save(**save_kwargs)
```

Additional fields saved:

```
owner
image_md5_hash
source
```

---

# 12️⃣ Return Response

```python
return Response(
    ProofFullSerializer(proof).data,
    status=201
)
```

Example response:

```json
{
  "id": 30,
  "file_path": "images/abc123.jpg",
  "date": "2024-06-01",
  "owner": 42
}
```

HTTP status:

```
201 Created
```

---

# 13️⃣ Full Upload Flow

```
POST /proofs/upload
        ↓
get uploaded file
        ↓
store image
        ↓
compute MD5 hash
        ↓
validate proof data
        ↓
determine owner
        ↓
check duplicate
        ↓
create proof
        ↓
return JSON response
```

---

# Example Notes (Good Study Format)

```
upload()

Endpoint
POST /proofs/upload

Purpose
Upload receipt / price tag image.

Steps
1. receive multipart file
2. store image
3. compute MD5 hash
4. validate proof fields
5. check duplicate upload
6. create proof
7. return proof data
```

---

✅ **Simple summary**

This endpoint allows users to **upload a receipt or price-tag image as proof of a product price**, while preventing duplicate uploads.

---

If you're studying this project, the **next thing that will make everything clearer** is understanding the main data pipeline:

```
User
 ↓
Proof (image upload)
 ↓
Price extraction (OCR / ML)
 ↓
Product price database
```

That flow explains **why this upload endpoint is so important in the system.**
```
@extend_schema(request=ProofProcessWithGeminiSerializer)  
@action(  
    detail=False,  
    methods=["POST"],  
    url_path="process-with-gemini",  
    parser_classes=[MultiPartParser],  
)  
def process_with_gemini(self, request: Request) -> Response:  
    files = request.FILES.getlist("files")  
    sample_files = [PIL.Image.open(file.file) for file in files]  
    labels = [  
        extract_from_price_tag(sample_file).parsed.model_dump()  
        for sample_file in sample_files  
    ]  
    return Response({"labels": labels}, status=status.HTTP_200_OK)
```
# 3️⃣ Get Uploaded Files

files = request.FILES.getlist("files")

The client sends multiple images.

Example request:

POST /proofs/process-with-gemini  
Content-Type: multipart/form-data

Body:

files: price_tag1.jpg  
files: price_tag2.jpg  
files: receipt.jpg

Result:

files = [file1, file2, file3]

---

# 4️⃣ Load Images with PIL

sample_files = [PIL.Image.open(file.file) for file in files]

This opens each uploaded image using **Pillow**.

Example result:

sample_files = [  
  Image(price_tag1.jpg),  
  Image(price_tag2.jpg),  
  Image(receipt.jpg)  
]

Now the images can be processed by AI.

---

# 5️⃣ Run AI Price Tag Extraction

labels = [  
    extract_from_price_tag(sample_file).parsed.model_dump()  
    for sample_file in sample_files  
]

This is the key step.

Function:

extract_from_price_tag(image)

Most likely uses **Gemini** to analyze the image.

Typical tasks:

detect product name  
detect price  
detect currency  
detect product code

Example extraction result:

{  
  "product_name": "Coca Cola",  
  "price": 3.50,  
  "currency": "AUD"  
}

The method:

model_dump()

comes from Pydantic models and converts the result to a dictionary.

---

# 6️⃣ Example AI Output

If two images are uploaded:

price_tag1.jpg  
price_tag2.jpg

Example result:

{  
  "labels": [  
    {  
      "product_name": "Coca Cola 1L",  
      "price": 3.5,  
      "currency": "AUD"  
    },  
    {  
      "product_name": "Milk 2L",  
      "price": 4.2,  
      "currency": "AUD"  
    }  
  ]  
}

---

# 7️⃣ Return Response

return Response({"labels": labels}, status=status.HTTP_200_OK)

HTTP response:

200 OK

Returned JSON:

labels extracted from images

---

# 8️⃣ Full Request Flow

POST /proofs/process-with-gemini  
        ↓  
upload images  
        ↓  
load images with PIL  
        ↓  
AI analyzes price tags  
        ↓  
extract labels  
        ↓  
return extracted data
```
@extend_schema(responses=ProofHistorySerializer(many=True))  
@action(detail=True, methods=["GET"])  
def history(self, request: Request, pk=None) -> Response:  
    proof = self.get_object()  
    return Response(proof.get_history_list(), status=200)
```
# 3️⃣ Get the Proof Object

proof = self.get_object()

This fetches the proof from the database.

Equivalent SQL:

SELECT * FROM proof WHERE id = 25;

Example object:

|id|image|date|
|---|---|---|
|25|receipt.jpg|2024-06-01|

---

# 4️⃣ Get Proof History

proof.get_history_list()

This is a **model method** on the `Proof` model.

It likely returns a list of **changes made to the proof**.

Typical history entries might include:

creation  
update  
moderation changes  
price extraction updates

Example history list:

[  
  {  
    "date": "2024-06-01T10:00:00",  
    "action": "created",  
    "user": "user_123"  
  },  
  {  
    "date": "2024-06-01T10:10:00",  
    "action": "price_extracted",  
    "user": "system"  
  },  
  {  
    "date": "2024-06-02T09:30:00",  
    "action": "updated",  
    "user": "moderator_1"  
  }  
]

---

# 5️⃣ Return Response

return Response(proof.get_history_list(), status=200)

HTTP response:

200 OK

Returned JSON:

list of proof history records

---

# 6️⃣ Full Request Flow

Example request:

GET /proofs/25/history

Processing:

router  
 ↓  
ProofViewSet.history()  
 ↓  
get proof object  
 ↓  
get history list  
 ↓  
return JSON response
```
@extend_schema(  
    request=FlagCreateSerializer, responses=FlagSerializer, tags=["moderation"]  
)  
@action(detail=True, methods=["POST"])  
def flag(self, request: Request, pk=None) -> Response:  
    proof = self.get_object()  
    serializer = FlagCreateSerializer(data=request.data)  
    serializer.is_valid(raise_exception=True)  
    # get owner  
    owner = self.request.user.user_id  
    # get source  
    source = get_source_from_request(self.request)  
    # save  
    flag = serializer.save(content_object=proof, owner=owner, source=source)  
    return Response(FlagSerializer(flag).data, status=status.HTTP_201_CREATED)
```
# 3️⃣ Get the Proof

proof = self.get_object()

This loads the proof from the database.

Equivalent SQL:

SELECT * FROM proof WHERE id=25;

Example object:

|id|image|
|---|---|
|25|receipt.jpg|

---

# 4️⃣ Validate Request Data

serializer = FlagCreateSerializer(data=request.data)  
serializer.is_valid(raise_exception=True)

Example request body:

{  
  "reason": "incorrect price",  
  "comment": "Price seems wrong"  
}

Serializer checks:

required fields  
data types  
allowed values

If invalid → **400 error**.

---

# 5️⃣ Get the User (Owner)

owner = self.request.user.user_id

This is the **user who reported the proof**.

Example:

user_id = 42

So the flag belongs to that user.

---

# 6️⃣ Get the Source

source = get_source_from_request(self.request)

Example values:

web  
mobile  
api

This helps track **where the moderation report came from**.

---

# 7️⃣ Save the Flag

flag = serializer.save(  
    content_object=proof,  
    owner=owner,  
    source=source  
)

This creates a **Flag object**.

Example database row:

|id|proof_id|owner|reason|
|---|---|---|---|
|10|25|42|incorrect price|

Important:

content_object=proof

This means the flag is attached to that proof.

Many projects use **generic relations** for flags.

---

# 8️⃣ Return the Flag

return Response(  
    FlagSerializer(flag).data,  
    status=201  
)

Response:

{  
  "id": 10,  
  "reason": "incorrect price",  
  "owner": 42,  
  "created": "2024-06-01"  
}

HTTP status:

201 Created

---

# 9️⃣ Full Request Flow

POST /proofs/25/flag  
        ↓  
ProofViewSet.flag()  
        ↓  
retrieve proof  
        ↓  
validate request  
        ↓  
get user  
        ↓  
create flag  
        ↓  
return flag data

