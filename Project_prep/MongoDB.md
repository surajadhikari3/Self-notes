
---

In **MongoDB**, document relationships are **not managed like foreign keys in SQL**. Instead, MongoDB offers two main approaches to modeling related data:

## 🔗 1. **Embedded Documents (Denormalized / Nested Structure)**

- MongoDB **stores the nested structure directly inside the parent document**.
    
- Best when the child data is:
    
    - Not shared with other documents
        
    - Relatively small
        
    - Accessed together with the parent
        

### ✅ Example:

```json
{
  "_id": "user123",
  "name": "Alice",
  "address": {
    "street": "123 Park Ave",
    "city": "Toronto",
    "zip": "M9W 6W7"
  },
  "orders": [
    { "id": 1, "item": "Laptop", "price": 1000 },
    { "id": 2, "item": "Mouse", "price": 25 }
  ]
}
```

> ✅ Reads are fast and atomic – everything is in one place.

---

## 🔗 2. **Referenced Documents (Normalized / Linked by ID)**

- Stores only the **reference `_id` of another document**
    
- Used when:
    
    - Data is large or reused across multiple documents
        
    - Relationships are many-to-many
        
    - You want to **avoid duplication**
        

### ✅ Example:

```json
{
  "_id": "user123",
  "name": "Alice",
  "addressId": "addr567"
}

{
  "_id": "addr567",
  "street": "123 Park Ave",
  "city": "Toronto",
  "zip": "M9W 6W7"
}
```

In Java, it would look like:

### 🧩 `User.java`

```java
@Document
public class User {
    @Id
    private String id;
    private String name;

    private String addressId; // referencing another document
}
```

Then you use `MongoTemplate` or an aggregation `$lookup` to join if needed:

### 🔍 `$lookup` with `MongoTemplate`

```java
Aggregation aggregation = Aggregation.newAggregation(
    Aggregation.lookup("addresses", "addressId", "_id", "addressDetails")
);
List<Document> result = mongoTemplate.aggregate(aggregation, "users", Document.class).getMappedResults();
```

---

## 🧠 So, in summary:

|Type|How?|Pros|Cons|
|---|---|---|---|
|Embedded|Nested object in parent|Fast reads, atomic updates|Redundancy, doc size limit|
|Referenced|Stores child’s ID (foreign key)|Normalized, reusable data|Requires manual joins|

---

## 🧩 How Spring Boot handles it?

Spring Boot (with Spring Data MongoDB):

- ✅ Supports both via simple Java object mapping
    
- 🧬 Embedded: Just use `@Document` and nested Java objects
    
- 🔗 Reference: Store just the ID, then manually fetch or use aggregation
    
