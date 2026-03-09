router.register(r"v1/locations", LocationViewSet, basename="locations")
```
class LocationViewSet(  
    mixins.ListModelMixin,  
    mixins.CreateModelMixin,  
    mixins.RetrieveModelMixin,  
    viewsets.GenericViewSet,  
):  
    queryset = Location.objects.all()  
    serializer_class = LocationSerializer  
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]  
    filterset_class = LocationFilter  
    ordering_fields = ["id", "created"] + Location.COUNT_FIELDS  
    ordering = ["id"]
```
# GET /locations → LocationSerializer
# POST /locations → LocationCreateSerializer
```
def get_serializer_class(self):  
    if self.request.method == "POST":  
        return LocationCreateSerializer  
    return self.serializer_class
```
# 2️⃣ Why This Is Needed

Because **reading data and creating data often require different fields**.
Example:
### LocationSerializer (used for GET)

{  
  "id": 10,  
  "osm_name": "Woolworths Sydney",  
  "lat": -33.86,  
  "lon": 151.20,  
  "price_count": 23,  
  "created": "2024-01-01"  
}
Contains **many computed fields**.

---

### LocationCreateSerializer (used for POST)

Client only needs to send:

{  
  "type": "osm",  
  "osm_id": 123456  
}

Much simpler.

---

# 3️⃣ What Happens Without This Function

If you remove it, the ViewSet uses:
serializer_class = LocationSerializer
for **every request**.
So:
POST /locations
would use:
LocationSerializer
Problems:
### ❌ Too many required fields

Client might be forced to send:
id  
created  
price_count  
proof_count
which should **not be provided by users**.

---

### ❌ Security risk

User could try to set fields that should be server-controlled:
{  
  "price_count": 9999  
}

---

### ❌ Validation errors

The API might reject valid requests because fields are missing.

# POST /locations
```
def create(self, request: Request, *args, **kwargs):  
    # validate  
    serializer = self.get_serializer(data=request.data)  
    serializer.is_valid(raise_exception=True)  
    # get source  
    source = get_source_from_request(self.request)  
    # save  
    try:  
        location = serializer.save(source=source)  
        response_data = self.serializer_class(location).data  
        response_status_code = status.HTTP_201_CREATED  
    # avoid duplicates: return existing location instead  
    except ValidationError as e:  
        if any(  
            constraint_name in e.messages[0]  
            for constraint_name in location_constants.UNIQUE_CONSTRAINT_NAME_LIST  
        ):  
            location = Location.objects.get(**serializer.validated_data)  
            response_data = {  
                **self.serializer_class(location).data,  
                "detail": "duplicate",  
            }  
            response_status_code = status.HTTP_200_OK  
    return Response(response_data, status=response_status_code)
```
# 1️⃣ Method Purpose

def create(self, request, *args, **kwargs):

Handles this API request:

POST /locations

Default DRF behavior:

validate → save → return 201

But this project **adds duplicate handling**.

---

# 2️⃣ Step 1 — Validate Input

serializer = self.get_serializer(data=request.data)  
serializer.is_valid(raise_exception=True)

Example request:

POST /locations  
  
{  
  "type": "osm",  
  "osm_id": 12345,  
  "osm_name": "Woolworths Sydney"  
}

Serializer checks:

✔ field types  
✔ required fields  
✔ format

If invalid → DRF automatically returns:

400 Bad Request

---

# 3️⃣ Step 2 — Determine Source

source = get_source_from_request(self.request)

This function probably determines **where the request came from**.

Example:

api  
web  
mobile  
script

Example result:

source = "api"

---

# 4️⃣ Step 3 — Save Location

location = serializer.save(source=source)

This creates a new database row.

Example table before:

|id|osm_id|name|
|---|---|---|
|1|11111|Coles|

After request:

|id|osm_id|name|
|---|---|---|
|1|11111|Coles|
|2|12345|Woolworths|

---

# 5️⃣ Step 4 — Successful Response

response_data = self.serializer_class(location).data

Example response:

{  
  "id": 2,  
  "osm_id": 12345,  
  "osm_name": "Woolworths Sydney"  
}

Status:

HTTP_201_CREATED

---

# 6️⃣ Duplicate Handling

If the location **already exists**, the database raises:

ValidationError

Example duplicate request:

POST /locations  
  
{  
  "type": "osm",  
  "osm_id": 12345  
}

But `osm_id=12345` already exists.

---

# 7️⃣ Detect Duplicate Error

except ValidationError as e:

It checks if the error comes from a **unique constraint**:

if any(  
    constraint_name in e.messages[0]  
    for constraint_name in UNIQUE_CONSTRAINT_NAME_LIST  
)

Meaning:

This error happened because the location already exists.

---

# 8️⃣ Return Existing Location

Instead of failing, it fetches the existing record:

location = Location.objects.get(**serializer.validated_data)

Example:

|id|osm_id|
|---|---|
|2|12345|

---

# 9️⃣ Duplicate Response

Response:

{  
  "id": 2,  
  "osm_id": 12345,  
  "detail": "duplicate"  
}

Status:

HTTP_200_OK

Meaning:

The location already exists, so we returned it instead of creating a new one.

---

# 🔟 Full Request Flow

POST /locations  
        ↓  
LocationViewSet.create()  
        ↓  
validate request  
        ↓  
try save location  
        ↓  
success → 201  
duplicate → return existing location (200)
# GET /locations/osm/{osm_type}/{osm_id}
```
@extend_schema(  
    parameters=[  
        OpenApiParameter(  
            name="osm_type",  
            type=OpenApiTypes.STR,  
            location=OpenApiParameter.PATH,  
            enum=location_constants.OSM_TYPE_LIST,  
            required=True,  
        ),  
        OpenApiParameter(  
            name="osm_id",  
            type=OpenApiTypes.INT,  
            location=OpenApiParameter.PATH,  
            required=True,  
        ),  
    ],  
)
@action(  
    detail=False, methods=["GET"], url_path=r"osm/(?P<osm_type>\w+)/(?P<osm_id>\d+)"  
)  
def get_by_osm(self, request, osm_type, osm_id):  
    location = get_object_or_drf_404(Location, osm_type=osm_type, osm_id=osm_id)  
    serializer = self.get_serializer(location)  
    return Response(serializer.data)
```
# 1️⃣ The Custom API Endpoint

The important part is this decorator:

@action(  
    detail=False,  
    methods=["GET"],  
    url_path=r"osm/(?P<osm_type>\w+)/(?P<osm_id>\d+)"  
)

This creates a new route.

Final URL:

GET /locations/osm/{osm_type}/{osm_id}

Example request:

GET /locations/osm/node/123456

Parameters extracted from the URL:

|Parameter|Value|
|---|---|
|osm_type|node|
|osm_id|123456|

---

# 2️⃣ What `@extend_schema` Does

@extend_schema(...)

This is used by **drf-spectacular** to generate API documentation.

It tells Swagger/OpenAPI:

Parameter: osm_type (string)  
Parameter: osm_id (integer)

So the documentation shows how to call the endpoint.

It **does not affect runtime logic**.

---

# 3️⃣ The View Function

def get_by_osm(self, request, osm_type, osm_id):

The parameters come directly from the URL.

Example:

GET /locations/osm/node/123456

becomes:

osm_type = "node"  
osm_id = 123456

---

# 4️⃣ Fetch Location From Database

location = get_object_or_drf_404(  
    Location,  
    osm_type=osm_type,  
    osm_id=osm_id  
)

This means:

SELECT * FROM locations  
WHERE osm_type='node'  
AND osm_id=123456

If found → return the object.  
If not found → return **404 error**.

Example database:

|id|osm_type|osm_id|name|
|---|---|---|---|
|5|node|123456|Woolworths|

Returned object:

Location(id=5)

---

# 5️⃣ Serialize the Object

serializer = self.get_serializer(location)

This converts the Django model to JSON.

Example JSON response:

{  
  "id": 5,  
  "osm_type": "node",  
  "osm_id": 123456,  
  "name": "Woolworths"  
}

---

# 6️⃣ Return API Response

return Response(serializer.data)

HTTP response:

200 OK

with JSON body.

---

# 7️⃣ Full Request Flow

GET /locations/osm/node/123456  
        ↓  
router  
        ↓  
LocationViewSet.get_by_osm()  
        ↓  
query database  
        ↓  
LocationSerializer  
        ↓  
JSON response

---

# 8️⃣ Why This Endpoint Exists

Normally you fetch a location like this:

GET /locations/5

But sometimes you only know the **OpenStreetMap ID**.

So this endpoint allows:

GET /locations/osm/node/123456

Very useful when integrating with OSM data.

---

# 9️⃣ Short Notes Example

You could write notes like this:

get_by_osm()  
  
Endpoint:  
GET /locations/osm/{osm_type}/{osm_id}  
  
Purpose:  
Retrieve location by OSM identifier.  
  
Flow:  
extract URL params → query DB → serialize → return JSON

# GET /locations/osm/countries
```
# TODO: disable pagination  
@extend_schema(responses=CountrySerializer(many=True), filters=False)  
@action(detail=False, methods=["GET"], url_path="osm/countries")  
def list_osm_countries(self, request):  
    cache_key = "osm_countries_list"  
    cache_timeout = 60 * 60 * 6  # 6 hours  
    countries_list = cache.get(cache_key)  
  
    if countries_list is None:  
        # get countries from JSON file  
        countries_list = utils.read_json(openstreetmap.COUNTRIES_JSON_PATH)  
        # enrich with existing stats  
        location_qs = (  
            Location.objects.filter(  
                type=location_constants.TYPE_OSM,  
                osm_address_country_code__isnull=False,  
            )  
            .values("osm_address_country_code")  
            .annotate(location_count=Count("id"), price_count=Sum("price_count"))  
        )  
        for i, country in enumerate(countries_list):  
            countries_list[i]["location_count"] = next(  
                (  
                    loc["location_count"]  
                    for loc in location_qs  
                    if loc["osm_address_country_code"] == country["country_code_2"]  
                ),  
                0,  
            )  
            countries_list[i]["price_count"] = next(  
                (  
                    loc["price_count"]  
                    for loc in location_qs  
                    if loc["osm_address_country_code"] == country["country_code_2"]  
                ),  
                0,  
            )  
        # cache results  
        cache.set(cache_key, countries_list, timeout=cache_timeout)  
  
    return Response(countries_list)
```
# 2️⃣ API Documentation

@extend_schema(responses=CountrySerializer(many=True), filters=False)

This is documentation metadata for **drf-spectacular**.

It tells Swagger/OpenAPI:

|Setting|Meaning|
|---|---|
|responses|returns many `CountrySerializer` objects|
|filters=False|no filtering parameters|

So Swagger will display:

GET /locations/osm/countries  
Response: List[CountrySerializer]

---

# 3️⃣ Cache Setup

cache_key = "osm_countries_list"  
cache_timeout = 60 * 60 * 6

Meaning:

Cache key = osm_countries_list  
Cache duration = 6 hours

Then:

countries_list = cache.get(cache_key)

If cached data exists → return it immediately.

This avoids recomputing the data every request.

---

# 4️⃣ If Cache Is Empty

if countries_list is None:

Then the function builds the list.

---

# 5️⃣ Load Country List

countries_list = utils.read_json(openstreetmap.COUNTRIES_JSON_PATH)

This reads a JSON file like:

Example JSON:

[  
  {"country_code_2": "AU", "name": "Australia"},  
  {"country_code_2": "FR", "name": "France"},  
  {"country_code_2": "US", "name": "United States"}  
]

So initially:

countries_list =  
[  
  {"country_code_2": "AU"},  
  {"country_code_2": "FR"},  
  {"country_code_2": "US"}  
]

---

# 6️⃣ Query Database Statistics

location_qs = (  
    Location.objects.filter(...)  
    .values("osm_address_country_code")  
    .annotate(  
        location_count=Count("id"),  
        price_count=Sum("price_count")  
    )  
)

This groups locations by country.

Equivalent SQL:

SELECT osm_address_country_code,  
       COUNT(id) AS location_count,  
       SUM(price_count) AS price_count  
FROM locations  
WHERE type='osm'  
GROUP BY osm_address_country_code;

Example result:

[  
 {osm_address_country_code: "AU", location_count: 5, price_count: 100},  
 {osm_address_country_code: "FR", location_count: 2, price_count: 20}  
]

---

# 7️⃣ Enrich Country List

Loop through all countries:

for i, country in enumerate(countries_list):

For each country, find matching stats.

Example:

Before:

{"country_code_2": "AU"}

After enrichment:

{  
 "country_code_2": "AU",  
 "location_count": 5,  
 "price_count": 100  
}

If a country has no locations:

{  
 "country_code_2": "JP",  
 "location_count": 0,  
 "price_count": 0  
}

---

# 8️⃣ Store Result in Cache

cache.set(cache_key, countries_list, timeout=cache_timeout)

Now the result is stored for **6 hours**.

Next request:

cache.get("osm_countries_list")

will return the data instantly.

---

# 9️⃣ Return Response

return Response(countries_list)

Example API response:

[  
  {  
    "country_code_2": "AU",  
    "location_count": 5,  
    "price_count": 100  
  },  
  {  
    "country_code_2": "FR",  
    "location_count": 2,  
    "price_count": 20  
  }  
]

---

# 🔟 Full Request Flow

GET /locations/osm/countries  
        ↓  
LocationViewSet.list_osm_countries()  
        ↓  
check cache  
        ↓  
(if empty)  
load countries JSON  
        ↓  
query location statistics  
        ↓  
merge stats with countries  
        ↓  
store in cache  
        ↓  
return JSON
# GET /locations/osm/countries/{country_code}/cities
```
# TODO: disable pagination  
@extend_schema(responses=CountryCitySerializer(many=True), filters=False)  
@action(  
    detail=False,  
    methods=["GET"],  
    url_path="osm/countries/(?P<country_code>\\w{2})/cities",  
)  
def list_osm_country_cities(self, request, country_code):  
    location_qs = (  
        Location.objects.filter(  
            type=location_constants.TYPE_OSM,  
            osm_address_country_code=country_code,  
            osm_address_city__isnull=False,  
        )  
        .values("osm_address_city")  
        .annotate(osm_name=F("osm_address_city"))  
        .annotate(country_code_2=F("osm_address_country_code"))  
        .annotate(location_count=Count("id"), price_count=Sum("price_count"))  
        .order_by("osm_name")  
    )  
    return Response(CountryCitySerializer(location_qs, many=True).data)
```
# 2️⃣ API Documentation

@extend_schema(responses=CountryCitySerializer(many=True), filters=False)

This tells **drf-spectacular** that the endpoint returns:

List[CountryCitySerializer]

Swagger/OpenAPI will show:

GET /locations/osm/countries/{country_code}/cities

---

# 3️⃣ Query the Database

location_qs = (  
    Location.objects.filter(  
        type=location_constants.TYPE_OSM,  
        osm_address_country_code=country_code,  
        osm_address_city__isnull=False,  
    )

Filters locations where:

|Condition|Meaning|
|---|---|
|type = OSM|location from OpenStreetMap|
|country_code = AU|only locations in that country|
|city not null|location has a city|

Example filtered rows:

|id|city|country|price_count|
|---|---|---|---|
|1|Melbourne|AU|10|
|2|Melbourne|AU|5|
|3|Sydney|AU|7|

---

# 4️⃣ Group by City

.values("osm_address_city")

This groups results by city.

Equivalent SQL:

SELECT osm_address_city  
FROM locations  
WHERE country='AU'  
GROUP BY osm_address_city

---

# 5️⃣ Rename Fields

.annotate(osm_name=F("osm_address_city"))  
.annotate(country_code_2=F("osm_address_country_code"))

This simply creates new fields:

|Field|Value|
|---|---|
|osm_name|city name|
|country_code_2|country code|

Example:

osm_name = "Melbourne"  
country_code_2 = "AU"

---

# 6️⃣ Aggregate Statistics

.annotate(  
    location_count=Count("id"),  
    price_count=Sum("price_count")  
)

For each city:

|Field|Meaning|
|---|---|
|location_count|number of locations|
|price_count|total number of prices|

Example result:

|city|location_count|price_count|
|---|---|---|
|Melbourne|2|15|
|Sydney|1|7|

---

# 7️⃣ Sort Cities

.order_by("osm_name")

Cities returned alphabetically:

Melbourne  
Sydney

---

# 8️⃣ Serialize Results

CountryCitySerializer(location_qs, many=True).data

Converts query results to JSON.

Example response:

[  
  {  
    "osm_name": "Melbourne",  
    "country_code_2": "AU",  
    "location_count": 2,  
    "price_count": 15  
  },  
  {  
    "osm_name": "Sydney",  
    "country_code_2": "AU",  
    "location_count": 1,  
    "price_count": 7  
  }  
]

---

# 9️⃣ Full Request Flow

GET /locations/osm/countries/AU/cities  
        ↓  
LocationViewSet.list_osm_country_cities()  
        ↓  
query Location table  
        ↓  
group by city  
        ↓  
calculate statistics  
        ↓  
serialize results  
        ↓  
return JSON
# GET /locations/compare
```
@extend_schema(  
    parameters=[  
        OpenApiParameter(  
            name="location_id_a",  
            type=OpenApiTypes.INT,  
            location=OpenApiParameter.QUERY,  
            required=True,  
        ),  
        OpenApiParameter(  
            name="location_id_b",  
            type=OpenApiTypes.INT,  
            location=OpenApiParameter.QUERY,  
            required=True,  
        ),  
        OpenApiParameter(  
            name="date__gte",  
            type=OpenApiTypes.DATE,  
            location=OpenApiParameter.QUERY,  
            required=False,  
            description="Filter prices with date greater than or equal to this date (YYYY-MM-DD)",  
        ),  
        OpenApiParameter(  
            name="date__lte",  
            type=OpenApiTypes.DATE,  
            location=OpenApiParameter.QUERY,  
            required=False,  
            description="Filter prices with date less than or equal to this date (YYYY-MM-DD)",  
        ),  
        OpenApiParameter(  
            name="price_is_discounted",  
            type=OpenApiTypes.BOOL,  
            location=OpenApiParameter.QUERY,  
            required=False,  
            description="Filter to keep only discounted or non-discounted prices",  
        ),  
    ],  
    responses=LocationCompareSerializer(many=False),  
    filters=False,  
)  
@action(detail=False, methods=["GET"])  
def compare(self, request: Request) -> Response:  
    """  
    Compare two locations by their IDs.    Returns shared product prices with the latest price per location,    the date of that price, and the total sum.    """    location_id_a = request.query_params.get("location_id_a")  
    location_id_b = request.query_params.get("location_id_b")  
    date__gte = request.query_params.get("date__gte")  
    date__lte = request.query_params.get("date__lte")  
    price_is_discounted = request.query_params.get("price_is_discounted")  
  
    # Validate parameters  
    if not location_id_a or not location_id_b:  
        return Response(  
            {  
                "detail": "location_id_a and location_id_b query parameters are required"  
            },  
            status=status.HTTP_400_BAD_REQUEST,  
        )  
  
    try:  
        location_id_a = int(location_id_a)  
        location_id_b = int(location_id_b)  
    except ValueError:  
        return Response(  
            {"detail": "location_id_a and location_id_b must be integers"},  
            status=status.HTTP_400_BAD_REQUEST,  
        )  
  
    # Verify both locations exist  
    location_a = get_object_or_drf_404(Location, id=location_id_a)  
    location_b = get_object_or_drf_404(Location, id=location_id_b)  
  
    if location_id_a == location_id_b:  
        return Response(  
            {"detail": "location_id_a and location_id_b must be different"},  
            status=status.HTTP_400_BAD_REQUEST,  
        )  
  
    # prepare response structure  
    results = {  
        "location_a": LocationSerializer(location_a).data,  
        "location_b": LocationSerializer(location_b).data,  
        "shared_products": [],  
        "total_sum_location_a": Decimal(0),  
        "total_sum_location_b": Decimal(0),  
    }  
  
    # Single query: fetch all prices from both locations  
    price_filters = Q(location_id=location_id_a) | Q(location_id=location_id_b)  
    price_filters &= Q(product_code__isnull=False)  
  
    if date__gte:  
        price_filters &= Q(date__gte=date__gte)  
    if date__lte:  
        price_filters &= Q(date__lte=date__lte)  
    if price_is_discounted is not None:  
        price_filters &= Q(  
            price_is_discounted=price_is_discounted.lower() == "true"  
        )  
  
    prices_qs = (  
        Price.objects.select_related("product")  
        .filter(price_filters)  
        .order_by("product_code", "-date")  
        .values(  
            "product_code",  
            "product__product_name",  
            "location_id",  
            "price",  
            "price_is_discounted",  
            "date",  
        )  
    )  
    price_list = list(prices_qs)  
  
    # Group latest prices by product_code and location_id  
    latest_prices = {}  
    for price in price_list:  
        product_code = price["product_code"]  
        location_id = price["location_id"]  
  
        if product_code not in latest_prices:  
            latest_prices[product_code] = {}  
  
        # Only store if we haven't seen this location yet  
        # first occurrence is latest price due to ordering        if location_id not in latest_prices[product_code]:  
            latest_prices[product_code][location_id] = price  
  
    # Find shared products and build response  
    for product_code, locations_dict in latest_prices.items():  
        if len(locations_dict.keys()) == 2:  
            product_result = {  
                "product_code": product_code,  
                "product_name": locations_dict[location_id_a][  
                    "product__product_name"  
                ],  
                "location_a": {  
                    key: locations_dict[location_id_a][key]  
                    for key in ("price", "price_is_discounted", "date")  
                },  
                "location_b": {  
                    key: locations_dict[location_id_b][key]  
                    for key in ("price", "price_is_discounted", "date")  
                },  
            }  
            results["shared_products"].append(product_result)  
            results["total_sum_location_a"] += product_result["location_a"]["price"]  
            results["total_sum_location_b"] += product_result["location_b"]["price"]  
  
    return Response(LocationCompareSerializer(results).data)
```
# 2️⃣ API Documentation

@extend_schema(...)

This documents the query parameters for **drf-spectacular**.

Swagger will show parameters:

|Parameter|Type|Required|
|---|---|---|
|location_id_a|integer|yes|
|location_id_b|integer|yes|
|date__gte|date|no|
|date__lte|date|no|
|price_is_discounted|boolean|no|

---

# 3️⃣ Read Query Parameters

location_id_a = request.query_params.get("location_id_a")  
location_id_b = request.query_params.get("location_id_b")

Example request:

GET /locations/compare?location_id_a=10&location_id_b=20

Result:

location_id_a = 10  
location_id_b = 20

---

# 4️⃣ Validate Inputs

Checks:

### Required parameters

if not location_id_a or not location_id_b

Error response:

{  
  "detail": "location_id_a and location_id_b query parameters are required"  
}

---

### Must be integers

location_id_a = int(location_id_a)

If invalid:

{  
  "detail": "location_id_a and location_id_b must be integers"  
}

---

### Locations must exist

location_a = get_object_or_drf_404(Location, id=location_id_a)

Equivalent SQL:

SELECT * FROM locations WHERE id=10

---

### Locations must be different

if location_id_a == location_id_b

Error:

{  
  "detail": "location_id_a and location_id_b must be different"  
}

---

# 5️⃣ Prepare Response Structure

results = {  
    "location_a": LocationSerializer(location_a).data,  
    "location_b": LocationSerializer(location_b).data,  
    "shared_products": [],  
    "total_sum_location_a": 0,  
    "total_sum_location_b": 0,  
}

Example:

{  
  "location_a": { "id": 10 },  
  "location_b": { "id": 20 },  
  "shared_products": [],  
  "total_sum_location_a": 0,  
  "total_sum_location_b": 0  
}

---

# 6️⃣ Fetch Prices From Both Locations

Query:

Price.objects.filter(  
    location_id=location_id_a OR location_id=location_id_b  
)

Equivalent SQL:

SELECT product_code, location_id, price, date  
FROM prices  
WHERE location_id IN (10,20)

Also filters:

product_code not null  
date filters  
discount filters

---

# 7️⃣ Get Latest Price Per Product

Prices are ordered:

product_code ASC  
date DESC

Meaning the **first row per product is the latest price**.

Example raw data:

|product|location|price|date|
|---|---|---|---|
|milk|10|3.5|2024-06-01|
|milk|10|3.0|2024-05-01|
|milk|20|4.0|2024-06-01|
|bread|10|2.5|2024-06-01|

Latest stored:

milk → location 10 → 3.5  
milk → location 20 → 4.0  
bread → location 10 → 2.5

---

# 8️⃣ Find Shared Products

This part:

if len(locations_dict.keys()) == 2

Means:

product exists in both locations

Example:

|product|location A|location B|
|---|---|---|
|milk|3.5|4.0|

Bread only exists in location A → ignored.

---

# 9️⃣ Build Response

Example product result:

{  
  "product_code": "milk",  
  "product_name": "Milk 1L",  
  "location_a": {  
    "price": 3.5,  
    "price_is_discounted": false,  
    "date": "2024-06-01"  
  },  
  "location_b": {  
    "price": 4.0,  
    "price_is_discounted": true,  
    "date": "2024-06-01"  
  }  
}

---

# 🔟 Calculate Total Prices

results["total_sum_location_a"] += price

Example totals:

location A total = 25.50  
location B total = 28.00

---

# 11️⃣ Final Response

Returned as:

Response(LocationCompareSerializer(results).data)

Example API output:

{  
  "location_a": { "id": 10 },  
  "location_b": { "id": 20 },  
  "shared_products": [  
    {  
      "product_code": "milk",  
      "product_name": "Milk 1L",  
      "location_a": { "price": 3.5, "date": "2024-06-01" },  
      "location_b": { "price": 4.0, "date": "2024-06-01" }  
    }  
  ],  
  "total_sum_location_a": 25.5,  
  "total_sum_location_b": 28.0  
}

---

# 12️⃣ Full Request Flow

GET /locations/compare  
        ↓  
validate parameters  
        ↓  
fetch both locations  
        ↓  
query prices  
        ↓  
group latest price per product  
        ↓  
keep products in both locations  
        ↓  
calculate totals  
        ↓  
return comparison