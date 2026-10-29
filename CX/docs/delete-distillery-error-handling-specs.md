# Specification Update: Error Handling for `DELETE /api/distilleries/{id}`

This document defines the update requirements for the deletion endpoint `DELETE /api/distilleries/{id}` to handle foreign key / active relationship constraints gracefully and return clear, actionable error responses.

---

## 1. Issue & Objective

* **Current Behavior**:
  When an administrator attempts to delete a distillery that is linked to active casks (`caskCount > 0`), the backend raises an unhandled database foreign key constraint error resulting in an **HTTP 500 Internal Server Error** with payload `{"message": "Internal server error"}`. The UI displays an unhelpful generic toast: *"Internal server error"*.
  
* **Required Updated Behavior**:
  The backend must perform a pre-deletion check for associated casks. If any exist (`caskCount > 0`), it must return an **HTTP 400 Bad Request** (or **HTTP 409 Conflict**) with an explicit error message: **`"Distillery has related casks and cannot be deleted"`**.

---

## 2. API Specification

### Endpoint: `DELETE /api/distilleries/{id}`

* **HTTP Method**: `DELETE`
* **Authentication**: Required (🔒 Admin / Authorized Role)
* **Path Parameters**:
  * `id` *(string / UUID)*: Unique identifier of the distillery.

---

### Response Scenarios

#### Scenario 1: Deletion Blocked (Distillery has related casks)

When `caskCount > 0` for the target distillery:

* **HTTP Status Code**: `400 Bad Request` *(or `409 Conflict`)*
* **Response Body**:

```json
{
  "statusCode": 400,
  "message": "Distillery has related casks and cannot be deleted",
  "error": "Bad Request"
}
```

---

#### Scenario 2: Successful Deletion (No related casks)

When `caskCount == 0` for the target distillery:

* **HTTP Status Code**: `200 OK` *(or `204 No Content`)*
* **Response Body (`200 OK`)**:

```json
{
  "success": true,
  "message": "Distillery deleted successfully"
}
```

---

#### Scenario 3: Distillery Not Found

When `{id}` does not match any existing distillery:

* **HTTP Status Code**: `404 Not Found`
* **Response Body**:

```json
{
  "statusCode": 404,
  "message": "Distillery not found",
  "error": "Not Found"
}
```

---

## 3. Backend Implementation Requirements

1. **Pre-deletion Relationship Check**:
   Before executing `DELETE FROM distilleries WHERE id = {id}`, query the associated master casks / variant casks:
   ```sql
   SELECT COUNT(*) FROM master_casks WHERE distillery_id = {id};
   ```
   *(Or check `distillery.caskCount > 0` / equivalent relation relation in ORM).*

2. **Validation Exception Interception**:
   If count > 0, throw a `BadRequestException` (or custom `DomainException`) with the message:
   `"Distillery has related casks and cannot be deleted"`

3. **Exception Filter Guarantee**:
   Ensure database foreign key violation exceptions (`23503` in PostgreSQL / constraint violations) are caught globally or inside the service method and wrapped into an HTTP 400/409 response rather than bubbling up as an unhandled 500 exception.

---

## 4. Frontend Toast Integration

When calling `DELETE /api/distilleries/{id}`:

```typescript
try {
  await api.delete(`/api/distilleries/${distilleryId}`);
  toast.success("Distillery deleted successfully");
} catch (error) {
  const errorMessage = error.response?.data?.message || "Failed to delete distillery";
  toast.error(errorMessage);
  // Displays: "Distillery has related casks and cannot be deleted"
}
```
