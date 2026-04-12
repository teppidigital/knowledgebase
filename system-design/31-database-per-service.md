# Database per Service

## Category
Architectural, Data Management, Microservices

## Context

The Database per Service pattern assigns each microservice its own dedicated database that can only be accessed by that service. Other services must request data through the service's public API, never by directly querying the database. This enforces strong encapsulation, enables polyglot persistence, and aligns with the microservices principle of loose coupling.

---

## Pros

- **Loose coupling**: A service can change its data model, technology, or schema without impacting other services.
- **Polyglot persistence**: Each service chooses the database best suited for its data (document, relational, graph, key-value).
- **Independent scaling**: Database instances can be sized and scaled for the specific service's load.
- **Fault isolation**: A database failure affects only its owning service.
- **Team autonomy**: Teams own their service's data model end-to-end.

---

## Cons

- **Data duplication**: Shared data must be copied or synchronized across services.
- **No cross-service joins**: Joining data from multiple services requires API composition or event-driven synchronization.
- **Distributed transactions**: Maintaining consistency across services requires Saga or Outbox patterns.
- **Increased infrastructure**: More database instances to provision, monitor, and pay for.
- **Eventual consistency**: Replicated data is not immediately consistent across all services.

---

## Design Diagram

```mermaid
graph TD
    UserSvc["User Service"]
    OrderSvc["Order Service"]
    ProductSvc["Product Service"]
    PaymentSvc["Payment Service"]

    UserDB[("User DB<br/>(PostgreSQL)")]
    OrderDB[("Order DB<br/>(PostgreSQL)")]
    ProductDB[("Product DB<br/>(MongoDB)")]
    PaymentDB[("Payment DB<br/>(MySQL)")]

    UserSvc --> UserDB
    OrderSvc --> OrderDB
    ProductSvc --> ProductDB
    PaymentSvc --> PaymentDB

    OrderSvc -->|"GET /users/{id}"| UserSvc
    OrderSvc -->|"GET /products/{id}"| ProductSvc
    OrderSvc -->|"POST /payments"| PaymentSvc
```

---

## Code Sample

### Service Enforcing Data Ownership (TypeScript)

```typescript
// order-service/src/services/order.service.ts
import { OrderRepository } from '../repositories/order.repository';
import { UserServiceClient } from '../clients/user.client';
import { ProductServiceClient } from '../clients/product.client';

// Order Service NEVER queries UserDB or ProductDB directly.
// It always calls APIs.
export class OrderService {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly userClient: UserServiceClient,
    private readonly productClient: ProductServiceClient
  ) {}

  async createOrder(userId: string, productIds: string[]): Promise<Order> {
    // Fetch user from User Service API (not User DB)
    const user = await this.userClient.getUser(userId);
    if (!user) throw new Error(`User ${userId} not found`);

    // Fetch products from Product Service API (not Product DB)
    const products = await this.productClient.getProducts(productIds);

    const total = products.reduce((sum, p) => sum + p.price, 0);

    // Persist to OWN database only
    return this.orderRepo.create({ userId, products, total, status: 'PENDING' });
  }
}
```

### HTTP Client for Inter-Service Communication

```typescript
// order-service/src/clients/user.client.ts
import axios from 'axios';

export class UserServiceClient {
  private readonly baseUrl = process.env.USER_SERVICE_URL!;

  async getUser(userId: string): Promise<User | null> {
    try {
      const { data } = await axios.get(`${this.baseUrl}/users/${userId}`);
      return data;
    } catch (err: any) {
      if (err.response?.status === 404) return null;
      throw new Error(`User Service unavailable: ${err.message}`);
    }
  }
}
```

### Data Synchronization via Events (avoiding coupling)

```typescript
// When User Service updates a user, publish an event
// Order Service listens and stores only the fields it needs

// user-service — publisher
await eventBus.publish({
  topic: 'user.updated',
  payload: { userId, name, email },
});

// order-service — subscriber (stores denormalized copy)
eventBus.subscribe('user.updated', async (event) => {
  await orderDb.query(
    'INSERT INTO user_cache (id, name, email) VALUES ($1, $2, $3) ON CONFLICT (id) DO UPDATE SET name=$2, email=$3',
    [event.userId, event.name, event.email]
  );
});
```

### Docker Compose — Separate DBs per Service

```yaml
version: '3.8'
services:
  user-service:
    image: user-service:1.0.0
    environment:
      - DATABASE_URL=postgres://postgres:secret@user-db:5432/users

  user-db:
    image: postgres:15
    environment:
      POSTGRES_DB: users
      POSTGRES_PASSWORD: secret

  order-service:
    image: order-service:1.0.0
    environment:
      - DATABASE_URL=postgres://postgres:secret@order-db:5432/orders

  order-db:
    image: postgres:15
    environment:
      POSTGRES_DB: orders
      POSTGRES_PASSWORD: secret

  product-service:
    image: product-service:1.0.0
    environment:
      - DATABASE_URL=mongodb://product-db:27017/products

  product-db:
    image: mongo:7
```

## Related Patterns

- [01 — Microservices](./01-microservices.md) — Database-per-service is a core tenet of the microservices pattern
- [06 — Saga Pattern](./06-saga-pattern.md) — Saga manages distributed transactions that span multiple service databases
- [28 — Change Data Capture](./28-change-data-capture.md) — CDC propagates state changes across services without shared databases
- [11 — Database Sharding](./11-database-sharding.md) — Shard within a service's own database for further horizontal scale
