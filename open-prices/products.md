```
class ProductViewSet(  
    mixins.ListModelMixin, mixins.RetrieveModelMixin, viewsets.GenericViewSet  
):  
    authentication_classes = []  # see get_authenticators  
    permission_classes = [IsAuthenticatedOrReadOnly]  
    queryset = Product.objects.all()  
    serializer_class = ProductFullSerializer  
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]  
    filterset_class = ProductFilter  
    ordering_fields = (  
        Product.OFF_SCORE_FIELDS + Product.COUNT_FIELDS + ["id", "created"]  
    )  
    ordering = ["id"]
```
# GET /api/products
```
def get_authenticators(self):  
    if self.request and self.request.method in ["GET"]:  
        return super().get_authenticators()  
    return [CustomAuthentication()]
```
# 2️⃣ Behavior for GET Requests

if self.request.method in ["GET"]:  
    return super().get_authenticators()

Meaning:

GET requests use the default authentication system

Example:

GET /locations

Authentication might be:

SessionAuthentication  
TokenAuthentication

So users can access endpoints normally.

---

# 3️⃣ Behavior for Non-GET Requests

For:

POST  
PUT  
PATCH  
DELETE

The code runs:

return [CustomAuthentication()]

Meaning:

Only CustomAuthentication is allowed

Example:

POST /locations

Authentication flow:

request  
   ↓  
CustomAuthentication  
   ↓  
user verified  
   ↓  
request allowed

---

# 4️⃣ Example API Behavior

|Request|Authentication|
|---|---|
|GET /locations|default DRF authentication|
|POST /locations|CustomAuthentication|
|DELETE /locations/10|CustomAuthentication|

---

# 5️⃣ Why This Design Is Used

Often APIs want:

Read → public  
Write → restricted

Example:

|Operation|Access|
|---|---|
|View locations|anyone|
|Add location|authenticated user|
|Delete location|authenticated user|

So the logic becomes:

GET → relaxed authentication  
POST/PUT/DELETE → strict authentication

---

# 6️⃣ Example Flow

Example request:

POST /locations

Processing:

LocationViewSet  
      ↓  
get_authenticators()  
      ↓  
CustomAuthentication  
      ↓  
check credentials  
      ↓  
allow or reject request

---

# 7️⃣ What `CustomAuthentication` Might Do

Typical examples:

API key authentication  
OAuth token  
internal service authentication  
signed request verification

Example header:

Authorization: ApiKey 123456
# GET /products/code/{code}
```
@action(detail=False, methods=["GET"], url_path=r"code/(?P<code>\d+)")  
def get_by_code(self, request: Request, code):  
    code = normalize_barcode(code)  
    product = get_object_or_drf_404(Product, code=code)  
    serializer = self.get_serializer(product)  
    return Response(serializer.data)
```
# 2️⃣ Function Definition

def get_by_code(self, request: Request, code):

The `code` argument comes directly from the URL.

Example:

GET /products/code/123456

Inside the function:

code = "123456"

---

# 3️⃣ Normalize Barcode

code = normalize_barcode(code)

This function probably cleans the barcode.

Typical fixes:

|Input|Output|
|---|---|
|`0012345678901`|`12345678901`|
|`123-456-789`|`123456789`|

Purpose:

make sure barcode format is consistent

---

# 4️⃣ Query the Database

product = get_object_or_drf_404(Product, code=code)

Equivalent SQL:

SELECT * FROM product  
WHERE code = '1234567890123';

Two outcomes:

|Case|Result|
|---|---|
|Product exists|return product|
|Product missing|return **404 error**|

Example product row:

|id|code|name|
|---|---|---|
|5|1234567890123|Coca Cola 1L|

---

# 5️⃣ Serialize Product

serializer = self.get_serializer(product)

The serializer converts the Django model to JSON.

Example serializer output:

{  
  "id": 5,  
  "code": "1234567890123",  
  "product_name": "Coca Cola 1L"  
}

---

# 6️⃣ Return Response

return Response(serializer.data)

HTTP response:

200 OK

with JSON body.

---

# 7️⃣ Full Request Flow

GET /products/code/1234567890123  
        ↓  
router  
        ↓  
ProductViewSet.get_by_code()  
        ↓  
normalize barcode  
        ↓  
query Product table  
        ↓  
serialize product  
        ↓  
return JSON response
# PATCH /products/code/{code}/off-update
```
@action(  
    detail=False,  
    methods=["PATCH"],  
    url_path=r"code/(?P<code>\d+)/off-update",  
)  
def create_or_update_in_off(self, request: Request, code):  
    result = create_or_update_product_in_off(  
        code,  
        flavor=request.data.get("flavor", Flavor.off),  
        country_code=request.data.get("product_language_code", "en"),  
        owner=self.request.user.user_id,  
        update_params=request.data.get("update_params", {}),  
    )  
    if result:  
        return Response(result, status=200)  
    return Response(status=400)
```
# 2️⃣ Function Purpose

def create_or_update_in_off(self, request: Request, code):

This endpoint:

Create or update a product in Open Food Facts (OFF)

OFF = **Open Food Facts**

So the API **syncs product data with OFF**.

---

# 3️⃣ Call Core Function

result = create_or_update_product_in_off(...)

This function performs the real work.

Parameters passed:

create_or_update_product_in_off(  
    code,  
    flavor=request.data.get("flavor", Flavor.off),  
    country_code=request.data.get("product_language_code", "en"),  
    owner=self.request.user.user_id,  
    update_params=request.data.get("update_params", {}),  
)

---

# 4️⃣ Parameters Explained

|Parameter|Source|Meaning|
|---|---|---|
|code|URL|product barcode|
|flavor|request body|OFF instance type|
|country_code|request body|product language|
|owner|logged-in user|user performing update|
|update_params|request body|product fields to update|

---

# 5️⃣ Example Request

Example PATCH request:

PATCH /products/code/1234567890123/off-update

Body:

{  
  "flavor": "off",  
  "product_language_code": "en",  
  "update_params": {  
    "product_name": "Coca Cola 1L",  
    "brands": "Coca-Cola",  
    "categories": "Beverages"  
  }  
}

---

# 6️⃣ What the Core Function Might Do

Typical flow:

1. check if product exists in OFF  
2. if not → create product  
3. if yes → update product fields  
4. return result

Example returned data:

{  
  "status": "updated",  
  "code": "1234567890123"  
}

---

# 7️⃣ Handle Result

If success:

if result:  
    return Response(result, status=200)

Example response:

{  
  "status": "updated",  
  "code": "1234567890123"  
}

Status:

200 OK

---

# 8️⃣ Handle Failure

If the update fails:

return Response(status=400)

HTTP response:

400 Bad Request

Meaning the operation failed.

---

# 9️⃣ Full Request Flow

PATCH /products/code/{barcode}/off-update  
        ↓  
ProductViewSet.create_or_update_in_off()  
        ↓  
extract request body  
        ↓  
call create_or_update_product_in_off()  
        ↓  
update product in Open Food Facts  
        ↓  
return result
# PATCH /products/code/{barcode}/off-upload-image
```
@action(  
    detail=False,  
    methods=["PATCH"],  
    url_path=r"code/(?P<code>\d+)/off-upload-image",  
)  
def upload_image_in_off(self, request: Request, code):  
    product_language_code = request.data.get("product_language_code", "en")  
    result = upload_product_image_in_off(  
        code,  
        flavor=request.data.get("flavor", Flavor.off),  
        country_code=product_language_code,  
        image_data_base64=request.data.get("image_data_base64"),  
        selected={"front": {product_language_code: {}}},  
    )  
    if result:  
        return Response(result, status=200)  
    return Response(status=400)
```

# 2️⃣ Function Purpose

def upload_image_in_off(self, request: Request, code):

This endpoint uploads an image to **Open Food Facts**.

Typical use case:

mobile app scans product  
↓  
user takes photo  
↓  
photo uploaded to OFF

---

# 3️⃣ Read Language Code

product_language_code = request.data.get("product_language_code", "en")

This determines which language the image belongs to.

Example request body:

{  
  "product_language_code": "en"  
}

If not provided:

default = "en"

---

# 4️⃣ Call Core Upload Function

result = upload_product_image_in_off(...)

This helper function handles the real upload to OFF.

Parameters passed:

upload_product_image_in_off(  
    code,  
    flavor=request.data.get("flavor", Flavor.off),  
    country_code=product_language_code,  
    image_data_base64=request.data.get("image_data_base64"),  
    selected={"front": {product_language_code: {}}},  
)

---

# 5️⃣ Parameters Explained

|Parameter|Meaning|
|---|---|
|code|product barcode|
|flavor|which OFF instance to use|
|country_code|language of the product|
|image_data_base64|image encoded as Base64|
|selected|tells OFF this is the **front image**|

---

# 6️⃣ Example Request

Example PATCH request:

PATCH /products/code/1234567890123/off-upload-image

Body:

{  
  "product_language_code": "en",  
  "image_data_base64": "iVBORw0KGgoAAAANSUhEUgAA..."  
}

`image_data_base64` is the image encoded as Base64.

Typical workflow:

image file  
↓  
convert to base64  
↓  
send to API

---

# 7️⃣ Selected Image Type

This part:

selected={"front": {product_language_code: {}}}

means the uploaded image will be used as the **front product image**.

Example OFF structure:

product images  
   ├─ front  
   ├─ ingredients  
   └─ nutrition

Here we select:

front image

---

# 8️⃣ Success Response

If upload succeeds:

return Response(result, status=200)

Example response:

{  
  "status": "image uploaded",  
  "code": "1234567890123"  
}

HTTP status:

200 OK

---

# 9️⃣ Failure Response

If upload fails:

return Response(status=400)

HTTP response:

400 Bad Request

Meaning something was wrong with the request or upload.

---

# 🔟 Full Request Flow

PATCH /products/code/{barcode}/off-upload-image  
        ↓  
ProductViewSet.upload_image_in_off()  
        ↓  
read request body  
        ↓  
call upload_product_image_in_off()  
        ↓  
upload image to Open Food Facts  
        ↓  
return success or error