
# Row-Level Permissions: Families & Users

This project demonstrates **row-level access control** for `Families` and `Users` 

---

## Logic 1 — Users can only access **their own family details**

### **Permission definition:**

```yaml
---
kind: ModelPermissions
version: v1
definition:
  modelName: Families
  permissions:
    - role: admin
      select:
        filter: null
        allowSubscriptions: true
    - role: family-admin
      select:
        filter: null
        allowSubscriptions: true
    - role: family-member
      select:
        filter: null
        allowSubscriptions: true
    - role: user
      select:
        filter:
          fieldComparison:
            field: primaryUserId
            operator: _eq
            value:
              sessionVariable: x-hasura-user-id
```

This ensures that a user can **only fetch their own family** where `primaryUserId` matches `x-hasura-user-id`.

### **header:**

```http
x-hasura-role: user
x-hasura-user-id: 1b31e87a-d2eb-4fd5-891f-334376fbcdd8
```

### **Query:**

```graphql
query MyQuery {
  families {
    createdAt
    id
    name
    primaryUserId
  }
}
```

### **Response:**

```json
{
  "data": {
    "families": [
      {
        "createdAt": "2024-02-18T19:43:42.304888",
        "id": "de41357e-a017-4b5e-9afc-9fdd998c324f",
        "name": "Nguyen Family",
        "primaryUserId": "1b31e87a-d2eb-4fd5-891f-334376fbcdd8"
      }
    ]
  }
}
```

### **Same result with an explicit `where` clause:**

```graphql
query MyQuery {
  families(where: {primaryUserId: {_eq: "1b31e87a-d2eb-4fd-891f-334376fbcdd8"}}) {
    id
    name
    primaryUserId
    createdAt
  }
}
```

---

## Logic 2 — Family admins can only access **users in their own family**

### **Permission definition:**

```yaml
---
kind: ModelPermissions
version: v1
definition:
  modelName: Users
  permissions:
    - role: admin
      select:
        filter: null
        allowSubscriptions: true
    - role: family-admin
      select:
        filter:
          relationship:
            name: familyMembers
            predicate:
              fieldComparison:
                field: familyId
                operator: _eq
                value:
                  sessionVariable: x-hasura-family-id
    - role: family-member
      select:
        filter: null
        allowSubscriptions: true
```

This restricts `family-admin` users to only select `Users` that belong to their `familyId`.

### **header:**

```http
x-hasura-role: family-admin
x-hasura-family-id: de41357e-a017-4b5e-9afc-9fdd998c324f
```

### **Query:**

```graphql
query MyQuery {
  users {
    createdAt
    dob
    email
    firstName
    id
    isPrimary
    lastName
    phone
    updatedAt
    familyMembers {
      familyId
      relationship
    }
  }
}
```

### **Example response:**

```json
{
  "data": {
    "users": [
      {
        "createdAt": "2022-11-12T07:10:58.551161",
        "dob": "2012-05-27",
        "email": "lcosta@example.net",
        "firstName": "John",
        "id": "5f0a6d5e-f6d7-435f-b134-1176fac75c06",
        "isPrimary": false,
        "lastName": "Williams",
        "phone": "2970543043",
        "updatedAt": "2024-06-04T23:03:09.791002",
        "familyMembers": [
          {
            "familyId": "de41357e-a017-4b5e-9afc-9fdd998c324f",
            "relationship": "spouse"
          }
        ]
      }
    ]
  }
}
```

### **Same with `where` clause for the family ID:**

```graphql
query MyQuery {
  users(
    where: {familyMembers: {familyId: {_eq: "de41357e-a017-4b5e-9afc-9fdd998c324f"}}}
  ) {
    lastName
    isPrimary
    id
    firstName
    phone
    updatedAt
    dob
    email
    familyMembers {
      familyId
      relationship
      userId
    }
  }
}
```

## To generate a JWT token

```
curl --request POST \
  --url https://dev-oqe0yjx7y55dn5so.us.auth0.com/oauth/token \
  --header 'content-type: application/json' \
  --data '{
    "client_id":"EyhPzDAdXrHbb86Af6SYrKKqJ1WMW0TM",
    "client_secret":"fBepmCzDM0yCyCFWHHXHYxPxyvHAEmQCd7AneBNKHUHxRfQQ4MVMqaVhNZuqTuuP",
    "audience":"https://dev-oqe0yjx7y55dn5so.us.auth0.com/api/v2/",
    "grant_type":"client_credentials"
  }'

```

