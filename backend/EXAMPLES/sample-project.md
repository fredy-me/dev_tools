# Sample Backend Project

## Project: E-Commerce API

A complete example of a custom backend for an e-commerce platform.

### Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Runtime | Node.js | 20 LTS |
| Framework | Express.js | 4.18 |
| Language | TypeScript | 5.3 |
| Database | PostgreSQL | 15 |
| ORM | Prisma | 5.7 |
| Cache | Redis | 7 |
| Auth | JWT (RS256) | - |
| Testing | Jest + Supertest | 29.7 |
| Container | Docker | 24 |

### Project Structure

```
ecommerce-api/
├── src/
│   ├── config/
│   │   ├── app.ts
│   │   ├── database.ts
│   │   └── redis.ts
│   ├── modules/
│   │   ├── auth/
│   │   ├── products/
│   │   ├── orders/
│   │   ├── payments/
│   │   └── users/
│   ├── shared/
│   │   ├── errors/
│   │   ├── middleware/
│   │   └── utils/
│   ├── app.ts
│   └── server.ts
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── test/
│   ├── integration/
│   └── unit/
├── docker-compose.yml
├── Dockerfile
└── package.json
```

### Database Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  name          String
  passwordHash  String    @map("password_hash")
  role          Role      @default(CUSTOMER)
  isVerified    Boolean   @default(false) @map("is_verified")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")

  orders        Order[]
  addresses     Address[]
  reviews       Review[]

  @@map("users")
}

model Product {
  id          String   @id @default(uuid())
  name        String
  description String?
  price       Decimal  @db.Decimal(10, 2)
  sku         String   @unique
  stock       Int      @default(0)
  category    String
  images      String[]
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  orderItems  OrderItem[]
  reviews     Review[]

  @@index([category, isActive])
  @@index([sku])
  @@map("products")
}

model Order {
  id            String      @id @default(uuid())
  userId        String      @map("user_id")
  status        OrderStatus @default(PENDING)
  totalAmount   Decimal     @db.Decimal(10, 2) @map("total_amount")
  shippingAddress String    @map("shipping_address")
  createdAt     DateTime    @default(now()) @map("created_at")
  updatedAt     DateTime    @updatedAt @map("updated_at")

  user          User        @relation(fields: [userId], references: [id])
  items         OrderItem[]
  payment       Payment?

  @@index([userId, createdAt(sort: Desc)])
  @@map("orders")
}

model OrderItem {
  id        String  @id @default(uuid())
  orderId   String  @map("order_id")
  productId String  @map("product_id")
  quantity  Int
  unitPrice Decimal @db.Decimal(10, 2) @map("unit_price")

  order   Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product Product @relation(fields: [productId], references: [id])

  @@map("order_items")
}

model Payment {
  id          String   @id @default(uuid())
  orderId     String   @unique @map("order_id")
  providerId  String   @map("provider_id")
  status      PaymentStatus
  amount      Decimal  @db.Decimal(10, 2)
  currency    String   @default("USD")
  createdAt   DateTime @default(now()) @map("created_at")

  order Order @relation(fields: [orderId], references: [id])

  @@map("payments")
}

model Address {
  id        String  @id @default(uuid())
  userId    String  @map("user_id")
  label     String
  street    String
  city      String
  state     String
  zipCode   String  @map("zip_code")
  country   String
  isDefault Boolean @default(false) @map("is_default")

  user User @relation(fields: [userId], references: [id])

  @@map("addresses")
}

model Review {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  productId String   @map("product_id")
  rating    Int      @db.SmallInt
  comment   String?
  createdAt DateTime @default(now()) @map("created_at")

  user    User    @relation(fields: [userId], references: [id])
  product Product @relation(fields: [productId], references: [id])

  @@unique([userId, productId])
  @@map("reviews")
}

enum Role {
  CUSTOMER
  ADMIN
}

enum OrderStatus {
  PENDING
  CONFIRMED
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

enum PaymentStatus {
  PENDING
  COMPLETED
  FAILED
  REFUNDED
}
```

### API Endpoints

```yaml
# Authentication
POST   /api/v1/auth/register    # Register new user
POST   /api/v1/auth/login       # Login
POST   /api/v1/auth/refresh     # Refresh token
POST   /api/v1/auth/logout      # Logout

# Users
GET    /api/v1/users/me         # Get current user
PATCH  /api/v1/users/me         # Update profile
GET    /api/v1/users/me/addresses  # List addresses
POST   /api/v1/users/me/addresses  # Add address

# Products
GET    /api/v1/products         # List products (public)
GET    /api/v1/products/:id     # Get product (public)
POST   /api/v1/products         # Create product (admin)
PATCH  /api/v1/products/:id     # Update product (admin)
DELETE /api/v1/products/:id     # Delete product (admin)

# Orders
POST   /api/v1/orders           # Create order
GET    /api/v1/orders           # List user's orders
GET    /api/v1/orders/:id       # Get order details
POST   /api/v1/orders/:id/cancel  # Cancel order

# Admin
GET    /api/v1/admin/orders     # List all orders (admin)
PATCH  /api/v1/admin/orders/:id  # Update order status (admin)
GET    /api/v1/admin/users      # List users (admin)
GET    /api/v1/admin/analytics  # Get analytics (admin)
```

### Example Implementation

```typescript
// src/modules/orders/orders.service.ts
import { Decimal } from '@prisma/client/runtime/library';
import { prisma } from '@/config/database';
import { NotFoundError, ValidationError, ConflictError } from '@/shared/errors';
import { logger } from '@/shared/utils/logger';

interface CreateOrderInput {
  items: Array<{ productId: string; quantity: number }>;
  shippingAddressId: string;
}

export class OrderService {
  async createOrder(userId: string, input: CreateOrderInput): Promise<Order> {
    // Validate address belongs to user
    const address = await prisma.address.findFirst({
      where: { id: input.shippingAddressId, userId },
    });

    if (!address) {
      throw new NotFoundError('Address', input.shippingAddressId);
    }

    // Fetch products and validate stock
    const productIds = input.items.map((item) => item.productId);
    const products = await prisma.product.findMany({
      where: { id: { in: productIds }, isActive: true },
    });

    if (products.length !== productIds.length) {
      throw new ValidationError('Some products are not available', [
        { field: 'items', message: 'One or more products are unavailable', code: 'INVALID' },
      ]);
    }

    // Check stock and calculate total
    let totalAmount = new Decimal(0);
    const orderItems = input.items.map((item) => {
      const product = products.find((p) => p.id === item.productId)!;

      if (product.stock < item.quantity) {
        throw new ValidationError(`Insufficient stock for ${product.name}`, [
          { field: 'items', message: `Only ${product.stock} available`, code: 'INSUFFICIENT_STOCK' },
        ]);
      }

      const subtotal = product.price.mul(item.quantity);
      totalAmount = totalAmount.add(subtotal);

      return {
        productId: product.id,
        quantity: item.quantity,
        unitPrice: product.price,
      };
    });

    // Create order in transaction
    const order = await prisma.$transaction(async (tx) => {
      // Create order
      const order = await tx.order.create({
        data: {
          userId,
          totalAmount,
          shippingAddress: JSON.stringify(address),
          items: { create: orderItems },
        },
        include: { items: true },
      });

      // Decrement stock
      for (const item of orderItems) {
        await tx.product.update({
          where: { id: item.productId },
          data: { stock: { decrement: item.quantity } },
        });
      }

      return order;
    });

    logger.info({ orderId: order.id, userId, total: totalAmount }, 'Order created');

    return order;
  }

  async getOrder(userId: string, orderId: string): Promise<Order> {
    const order = await prisma.order.findFirst({
      where: { id: orderId, userId },
      include: {
        items: { include: { product: true } },
        payment: true,
      },
    });

    if (!order) {
      throw new NotFoundError('Order', orderId);
    }

    return order;
  }

  async cancelOrder(userId: string, orderId: string): Promise<Order> {
    const order = await prisma.order.findFirst({
      where: { id: orderId, userId, status: 'PENDING' },
    });

    if (!order) {
      throw new NotFoundError('Pending order', orderId);
    }

    return prisma.$transaction(async (tx) => {
      // Update status
      const updated = await tx.order.update({
        where: { id: orderId },
        data: { status: 'CANCELLED' },
      });

      // Restore stock
      const items = await tx.orderItem.findMany({
        where: { orderId },
      });

      for (const item of items) {
        await tx.product.update({
          where: { id: item.productId },
          data: { stock: { increment: item.quantity } },
        });
      }

      logger.info({ orderId, userId }, 'Order cancelled');

      return updated;
    });
  }
}
```

### Environment Configuration

```bash
# .env (development)
APP_NAME=ecommerce-api
APP_ENV=development
APP_PORT=3000
DATABASE_URL=postgresql://postgres:password@localhost:5432/ecommerce_dev
REDIS_URL=redis://localhost:6379
JWT_SECRET=dev-secret-key-change-in-production
JWT_ALGORITHM=RS256
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
LOG_LEVEL=debug
```

### Deployment

```yaml
# docker-compose.prod.yml
services:
  api:
    image: ecommerce-api:latest
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```
