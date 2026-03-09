router.register(r"v1/users", UserViewSet, basename="users")
GET /v1/users
GET /v1/users/{user_id}
```  
class UserViewSet(  
    mixins.ListModelMixin, mixins.RetrieveModelMixin, viewsets.GenericViewSet  
):  
    queryset = User.objects.all()  
    serializer_class = UserSerializer  
    lookup_field = "user_id"  
    filter_backends = [DjangoFilterBackend, filters.OrderingFilter]  
    filterset_class = UserFilter  
    ordering_fields = ["user_id"] + User.COUNT_FIELDS  
    ordering = ["user_id"]  
  
    def get_queryset(self):  
        # list: return only users with prices  
        if not self.kwargs.get("user_id", None):  
            return self.queryset.has_prices()  
        return self.queryset
```
Return all rows from the `users` table as a QuerySet of `User` objects.
Only users with prices are returned.
if request is list
    return users with prices
else
    return specific user

