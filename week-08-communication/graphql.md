# GraphQL

## Learning Objectives
- Understand GraphQL query language
- Learn resolvers and schema design
- Master N+1 problem and DataLoader

## Notes

### The Problem GraphQL Solves

#### Over-Fetching & Under-Fetching with REST
**Over-fetching**: Get 20 fields, only need 5
- REST: `GET /users/789` returns all user data (phone, address, birthday, etc.)
- Only needed: name and email
- Wastes bandwidth, especially on mobile

**Under-fetching**: Need data from multiple endpoints
- REST: 3 separate calls for user profile page
  - `GET /users/789` (user info)
  - `GET /users/789/orders` (50 orders, only need 3)
  - `GET /users/789/favorites` (favorites)
- Multiple network round trips = slow

### GraphQL Solution

**One request, exact data:**
```graphql
query {
  user(id: 789) {
    name
    email
    orders(limit: 3) {
      restaurant
      total
    }
    favorites {
      name
      rating
    }
  }
}
```

**Benefits:**
- ✅ One network call (not 3)
- ✅ No over-fetching (only requested fields)
- ✅ No under-fetching (exactly 3 orders, not 50)

### How GraphQL Works

#### 1. Schema Definition (The Contract)
```graphql
type Restaurant {
  id: ID!              # ! means required
  name: String!
  rating: Float!
  menu: [MenuItem!]!
}

type Query {
  restaurant(id: ID!): Restaurant
  searchRestaurants(cuisine: String): [Restaurant!]!
}
```

Schema is like `.proto` file in gRPC - defines what's available

#### 2. Resolvers (The Logic)
```javascript
const resolvers = {
  Query: {
    restaurant: async (parent, { id }) => {
      return await db.query('SELECT * FROM restaurants WHERE id = ?', id);
    }
  },
  Restaurant: {
    menu: async (restaurant) => {
      return await db.query('SELECT * FROM menu_items WHERE restaurant_id = ?', restaurant.id);
    }
  }
};
```

Resolvers = Functions that fetch data for each field

#### 3. Execution
GraphQL only calls resolvers for fields you requested!

### The N+1 Problem (CRITICAL!)

**Most important GraphQL challenge**

**The Problem:**
```graphql
query {
  orders(limit: 10) {
    id
    restaurant {
      name
    }
  }
}
```

**Naive implementation:**
```javascript
// 1 query to get orders
const orders = await db.query('SELECT * FROM orders LIMIT 10');

// 10 queries for each restaurant! ❌
for (const order of orders) {
  order.restaurant = await db.query(
    'SELECT * FROM restaurants WHERE id = ?',
    order.restaurant_id
  );
}
// Total: 1 + 10 = 11 queries (should be 2!)
```

**At scale:** 100 orders = 101 queries! Database explodes!

### Solution: DataLoader (Batching & Caching)

```javascript
const DataLoader = require('dataloader');

const restaurantLoader = new DataLoader(async (restaurantIds) => {
  // Gets called ONCE with ALL IDs
  const restaurants = await db.query(
    'SELECT * FROM restaurants WHERE id IN (?)',
    restaurantIds // [1, 5, 3, 7, ...]
  );
  return restaurantIds.map(id =>
    restaurants.find(r => r.id === id)
  );
});

const resolvers = {
  Order: {
    restaurant: (order) => {
      return restaurantLoader.load(order.restaurant_id);
    }
  }
};
```

**What happens:**
- DataLoader queues all restaurant requests
- Batches them into ONE query
- Caches duplicate IDs
- **Result**: 1 + 1 = 2 queries ✅ (not 11!)

### Query Complexity Attacks

**Problem:** Clients can craft expensive queries
```graphql
query {
  users {                    # 10,000 users
    orders {                 # × 100 orders each
      items {                # × 20 items each
        restaurant {         # Can nest infinitely!
          reviews {
            user {
              orders { ... } # Circular reference!
            }
          }
        }
      }
    }
  }
}
```

Could fetch billions of records and crash your system!

**Solution: Complexity Limits**
```javascript
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const server = new ApolloServer({
  validationRules: [
    depthLimit(5),              // Max 5 levels deep
    createComplexityLimitRule(1000)  // Max complexity score
  ]
});
```

Rejects expensive queries before execution

### Type Safety (GraphQL Strength!)

**Schema enforces types:**
```graphql
type User {
  id: ID!              # Required
  name: String!        # Required string
  age: Int             # Optional int
  orders: [Order!]!    # Required array of required Orders
}
```

**Auto-validation:**
- Client sends `age: "twenty"` → Rejected before hitting your code!
- Field names misspelled → Rejected
- Type mismatch → Rejected

**Tools generate TypeScript types automatically** (`graphql-codegen`)

### Caching Strategies

#### Problem: Same Endpoint, Different Queries
```
POST /graphql
Body: query { user(id: 789) { name } }

POST /graphql
Body: query { user(id: 789) { name, email } }

// Same URL, different data - how to cache?
```

#### Solution: Normalized Caching
Cache by **TYPE + ID**, not query:

```javascript
// Apollo Client cache
{
  "User:789": {
    name: "John",
    email: "john@example.com"
  },
  "Restaurant:5": {
    name: "Pizza Place",
    rating: 4.5
  }
}
```

**Benefits:**
- ANY query for User:789's name gets cache hit
- Missing fields trigger partial fetch (only missing data)
- Cache hit rate: ~80% vs ~20% with query-based caching

**Server-side caching with Redis:**
```javascript
const resolvers = {
  Query: {
    user: async (parent, { id }) => {
      const cached = await redis.get(`User:${id}`);
      if (cached) return JSON.parse(cached);

      const user = await db.query('SELECT * FROM users WHERE id = ?', id);
      await redis.setex(`User:${id}`, 300, JSON.stringify(user));
      return user;
    }
  }
};
```

### When to Use GraphQL vs REST

#### Use GraphQL when:
✅ **Mobile apps with varying needs**
- Different screens need different data
- Bandwidth matters (3G/4G)

✅ **Complex nested data**
- Users → Orders → Items → Restaurants
- Many relationships

✅ **Rapid UI iteration**
- Frontend adds fields without backend changes
- No new endpoints needed

#### Use REST when:
✅ **Simple CRUD operations**
- Get user, create order, update profile

✅ **File uploads/downloads**
- REST multipart is simpler

✅ **Caching is critical**
- CDN caching by URL

✅ **Public APIs**
- Easier for third-party developers

## Practice Questions

1. **What is the N+1 problem and how does DataLoader solve it?**
   - N+1: Fetching N items requires N+1 queries (1 for items, N for related data)
   - DataLoader batches multiple requests into single query
   - Also caches duplicate requests within same request
   - Example: 10 orders → 2 queries instead of 11

2. **Why is normalized caching better than query-based caching?**
   - Query-based: Different queries = different cache keys = no reuse
   - Normalized: Cache by object ID, any query accessing same object gets cache hit
   - Partial fetches: Only missing fields are fetched
   - Much higher cache hit rate

3. **How do you prevent expensive/malicious queries?**
   - Depth limiting (max 5 levels deep)
   - Complexity scoring (reject queries over threshold)
   - Rate limiting
   - Query timeout limits

4. **When would you choose REST over GraphQL?**
   - Simple APIs without complex relationships
   - File uploads/downloads
   - Public APIs (easier for developers)
   - CDN caching important

## Real-World Examples

- **Facebook**: Created GraphQL, uses for all mobile APIs
- **GitHub**: Public GraphQL API (v4), REST API still available (v3)
- **Shopify**: GraphQL for store APIs, reduces API calls by 80%
- **Netflix**: GraphQL for some client APIs, gRPC for service-to-service

## Resources & References

- GraphQL Official Documentation
- Apollo GraphQL Platform
- DataLoader GitHub Repository
- How to GraphQL Tutorial
