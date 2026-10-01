# Day 04 - SQL JOIN, Order Modeling, Aggregation, and N+1 Query

## Problem

An Order Management API needs to return order information together with:

- Order ID
- User name
- User email
- Product name
- Quantity
- Product price at purchase time
- Order total
- Order status
- Order date

The PostgreSQL database contains four main tables:

```sql
users
-----
id
name
email
created_at
```

```sql
products
--------
id
name
price
stock
created_at
```

```sql
orders
------
id
user_id
status
created_at
```

```sql
order_items
-----------
id
order_id
product_id
quantity
price
```

Relationships:

```text
User
  │
  │ 1:N
  ▼
Order
  │
  │ 1:N
  ▼
OrderItem
  │
  │ N:1
  ▼
Product
```

Or:

```text
users
  ↓
orders
  ↓
order_items
  ↓
products
```

---

# Sample Data

## users

```text
1 | Mulia | mulia@mail.com
2 | Andi  | andi@mail.com
```

## products

```text
1 | Keyboard | 500000
2 | Mouse    | 200000
3 | Monitor  | 2500000
```

## orders

```text
101 | user_id=1 | PAID
102 | user_id=2 | PENDING
```

## order_items

```text
1 | order_id=101 | product_id=1 | quantity=2 | price=500000
2 | order_id=101 | product_id=2 | quantity=1 | price=200000
3 | order_id=102 | product_id=3 | quantity=1 | price=2500000
```

Order `101` therefore contains:

```text
Keyboard
2 × 500000

Mouse
1 × 200000
```

Total:

```text
1,000,000
+
200,000
=
1,200,000
```

---

# Interview Question

> How would you query the database to return orders together with the user and product information?

Follow-up:

> One order can contain multiple products. I also need the total value of each order.

---

# Interview Answer

I would first identify the relationships between the tables before writing the SQL.

The foreign key relationships are:

```text
orders.user_id
→ users.id

order_items.order_id
→ orders.id

order_items.product_id
→ products.id
```

The JOIN conditions must follow those relationships.

I would not simply join columns because they are all named `id`.

---

# Basic JOIN Query

To get:

```text
order_id
user_name
user_email
product_name
quantity
price
```

the query can be:

```sql
SELECT
  o.id AS order_id,
  u.name AS user_name,
  u.email AS user_email,
  p.name AS product_name,
  oi.quantity,
  oi.price
FROM orders AS o
INNER JOIN users AS u
  ON u.id = o.user_id
INNER JOIN order_items AS oi
  ON oi.order_id = o.id
INNER JOIN products AS p
  ON p.id = oi.product_id;
```

---

# Why These JOIN Conditions?

The first relationship:

```sql
INNER JOIN users AS u
  ON u.id = o.user_id
```

means:

```text
orders.user_id
references
users.id
```

Example:

```text
orders

id: 101
user_id: 1
```

matches:

```text
users

id: 1
name: Mulia
```

---

The second relationship:

```sql
INNER JOIN order_items AS oi
  ON oi.order_id = o.id
```

means:

```text
order_items.order_id
references
orders.id
```

Example:

```text
order_items

order_id: 101
```

belongs to:

```text
orders

id: 101
```

---

The third relationship:

```sql
INNER JOIN products AS p
  ON p.id = oi.product_id
```

means:

```text
order_items.product_id
references
products.id
```

This lets us retrieve current product information such as:

```text
product name
```

while still using the historical item price stored in:

```text
order_items.price
```

---

# Result Example

The query may return:

```text
order_id | user_name | product_name | quantity | price
------------------------------------------------------
101      | Mulia     | Keyboard     | 2        | 500000
101      | Mulia     | Mouse        | 1        | 200000
102      | Andi      | Monitor      | 1        | 2500000
```

Notice that order `101` appears twice.

This is expected because one order has multiple order items.

---

# INNER JOIN

`INNER JOIN` only returns rows where matching records exist on both sides.

Example:

```sql
SELECT
  o.id,
  oi.id
FROM orders AS o
INNER JOIN order_items AS oi
  ON oi.order_id = o.id;
```

Suppose we have:

```text
Order 101
→ has order items

Order 102
→ has order items

Order 103
→ no order items
```

The result contains:

```text
101
102
```

but not:

```text
103
```

because there is no matching row in `order_items`.

---

# LEFT JOIN

If the requirement says:

> Show orders even if they do not have any order items.

then I would use:

```sql
LEFT JOIN
```

Example:

```sql
SELECT
  o.id,
  oi.product_id
FROM orders AS o
LEFT JOIN order_items AS oi
  ON oi.order_id = o.id;
```

If order `103` has no items, it can still appear:

```text
order_id | product_id
---------------------
101      | 1
101      | 2
102      | 3
103      | NULL
```

---

# INNER JOIN vs LEFT JOIN

## INNER JOIN

```text
Only rows that have matches
```

Example:

```text
Order without items
→ excluded
```

## LEFT JOIN

```text
Keep every row from the left table
even when no matching row exists
```

Example:

```text
Order without items
→ still returned
→ order_item columns become NULL
```

---

# Which One Should Be Used?

It depends on the requirement.

For a normal completed e-commerce order, there usually should be at least one order item.

However, for:

```text
draft orders
shopping carts
partially created orders
failed workflows
```

an order might temporarily exist without items.

Therefore the correct JOIN depends on the business model.

---

# Calculating Item Total

Each line item total can be calculated using:

```sql
quantity * price
```

Example:

```sql
SELECT
  quantity,
  price,
  quantity * price AS item_total
FROM order_items;
```

For:

```text
quantity = 2
price = 500000
```

the result is:

```text
item_total = 1000000
```

---

# Calculating Order Total

To calculate the total of all items in one order:

```sql
SUM(
  oi.quantity * oi.price
)
```

Example:

```sql
SELECT
  oi.order_id,
  SUM(
    oi.quantity * oi.price
  ) AS total
FROM order_items AS oi
GROUP BY oi.order_id;
```

Result:

```text
order_id | total
----------------
101      | 1200000
102      | 2500000
```

---

# JOIN + Total Order

A complete query:

```sql
SELECT
  o.id AS order_id,
  u.name AS user_name,
  u.email AS user_email,
  o.status,
  o.created_at,
  SUM(
    oi.quantity * oi.price
  ) AS total
FROM orders AS o
INNER JOIN users AS u
  ON u.id = o.user_id
INNER JOIN order_items AS oi
  ON oi.order_id = o.id
GROUP BY
  o.id,
  u.name,
  u.email,
  o.status,
  o.created_at
ORDER BY o.created_at DESC;
```

---

# Why GROUP BY Is Required

Consider:

```sql
SELECT
  o.id,
  u.name,
  SUM(
    oi.quantity * oi.price
  ) AS total
FROM orders AS o
JOIN users AS u
  ON u.id = o.user_id
JOIN order_items AS oi
  ON oi.order_id = o.id;
```

This query is incomplete in PostgreSQL.

Why?

Because:

```sql
SUM(...)
```

is an aggregate function.

But:

```sql
o.id
u.name
```

are normal columns.

PostgreSQL needs to know how rows should be grouped before calculating the aggregate.

Therefore:

```sql
GROUP BY
  o.id,
  u.name
```

is needed.

Correct version:

```sql
SELECT
  o.id,
  u.name,
  SUM(
    oi.quantity * oi.price
  ) AS total
FROM orders AS o
JOIN users AS u
  ON u.id = o.user_id
JOIN order_items AS oi
  ON oi.order_id = o.id
GROUP BY
  o.id,
  u.name;
```

---

# Why Store Price Inside order_items?

This is one of the most important database design decisions in an order system.

We already have:

```text
products.price
```

but we also store:

```text
order_items.price
```

Why?

Because:

```text
products.price
```

represents the product's **current price**.

While:

```text
order_items.price
```

represents the **price paid at the time of the transaction**.

---

# Example

Today:

```text
Keyboard

current price:
500000
```

The customer places an order.

We save:

```text
order_items.price = 500000
```

One month later:

```text
Keyboard price
changes to:
650000
```

The old order must still show:

```text
500000
```

not:

```text
650000
```

because the customer purchased the product for Rp500,000.

---

# Why This Matters

If we calculate historical orders using:

```text
products.price
```

the total could change whenever the product price changes.

That would corrupt historical transaction information.

Order data should represent an immutable business event.

Conceptually:

```text
Product

Current state
```

while:

```text
OrderItem

Historical transaction snapshot
```

---

# Practical Example

At checkout:

```ts
const product =
  await getProduct(productId);
```

Suppose:

```ts
product.price = 500000;
```

When creating the order item:

```ts
await createOrderItem({
  orderId,
  productId,
  quantity: 2,

  price: product.price,
});
```

We store:

```text
500000
```

inside the order item.

Later, changing:

```text
products.price
```

does not affect old orders.

---

# Why order_items Exists

A bad design could be:

```text
orders

id
user_id
product_id
quantity
```

This means one order can naturally hold only one product.

Suppose a customer buys:

```text
Keyboard
Mouse
Monitor
```

You would need either:

```text
three orders
```

or strange columns such as:

```text
product_1
product_2
product_3
```

which is poor relational design.

---

# Better Design

Use:

```text
orders

id
user_id
status
created_at
```

and:

```text
order_items

id
order_id
product_id
quantity
price
```

Now:

```text
Order 101
```

can contain:

```text
order_items

order_id 101 → Keyboard
order_id 101 → Mouse
order_id 101 → Monitor
```

This models a:

```text
one-to-many
```

relationship.

One order:

```text
1
```

can have many order items:

```text
N
```

---

# Why Product Is Referenced Through order_items

The actual relationship is:

```text
Order
   │
   │ 1:N
   ▼
OrderItem
   │
   │ N:1
   ▼
Product
```

not directly:

```text
Order
→ Product
```

because the relationship itself contains additional information:

```text
quantity
price
discount
tax
```

These attributes belong to the relationship between an order and a product.

That is why `order_items` exists.

---

# N+1 Query Problem

Consider:

```ts
const orders =
  await prisma.order.findMany();

for (const order of orders) {
  const user =
    await prisma.user.findUnique({
      where: {
        id: order.userId,
      },
    });

  const items =
    await prisma.orderItem.findMany({
      where: {
        orderId: order.id,
      },
    });

  console.log({
    order,
    user,
    items,
  });
}
```

Suppose we have:

```text
100 orders
```

---

# Number of Queries

First:

```ts
prisma.order.findMany();
```

produces approximately:

```text
1 query
```

Then each order runs:

```text
1 query for user
+
1 query for items
```

For 100 orders:

```text
100 user queries
+
100 item queries
```

Total:

```text
1 + 100 + 100

= 201 queries
```

And this does not yet include fetching product details inside every item.

---

# What Is N+1?

The N+1 query problem happens when:

```text
1 query
```

retrieves a collection of `N` records, then additional queries are executed for each record.

Conceptually:

```text
Get orders
1 query

↓ for each order

Get related data
N queries
```

Example:

```text
1 query for 100 orders
+
100 queries for users

=
101 queries
```

If more relationships are fetched inside the loop, the number becomes even larger.

---

# Why N+1 Is Bad

Every database query has overhead:

```text
Network round trip
Database parsing
Planning
Connection usage
Query execution
Serialization
```

Running hundreds of small queries can be much slower than fetching relationships efficiently.

Possible symptoms:

```text
High API latency
High database connection usage
Database overload
Reduced throughput
```

---

# Improving the Prisma Query

Instead of querying related data manually:

```ts
for (const order of orders) {
  // query
  // query
}
```

we can request the relationships through Prisma:

```ts
const orders =
  await prisma.order.findMany({
    include: {
      user: true,

      items: {
        include: {
          product: true,
        },
      },
    },
  });
```

Conceptually:

```text
Order
├── User
└── OrderItems
      └── Product
```

This expresses the relationships in one ORM operation.

The exact SQL/query strategy generated by Prisma can depend on Prisma's relation loading behavior and version, so the important engineering point is to avoid manually issuing queries inside an N-sized loop.

---

# Prefer select When We Only Need Specific Fields

If the frontend only needs:

```text
order.id
user.name
product.name
quantity
price
```

we do not need every database column.

Example:

```ts
const orders =
  await prisma.order.findMany({
    select: {
      id: true,
      status: true,
      createdAt: true,

      user: {
        select: {
          name: true,
          email: true,
        },
      },

      items: {
        select: {
          quantity: true,
          price: true,

          product: {
            select: {
              id: true,
              name: true,
            },
          },
        },
      },
    },
  });
```

---

# include vs select in Prisma

## include

Used primarily when we want related models included.

Example:

```ts
include: {
  user: true,
  items: true,
}
```

Conceptually:

```text
Give me the order
plus these relationships.
```

---

## select

Used when we want explicit control over fields.

Example:

```ts
select: {
  id: true,
  status: true,
}
```

Conceptually:

```text
Only return these fields.
```

Nested relations can also be selected:

```ts
select: {
  id: true,

  user: {
    select: {
      name: true,
    },
  },
}
```

---

# Why Not Select Everything?

Returning unnecessary fields can increase:

```text
Database work
Memory usage
Serialization cost
Network payload
API response size
```

It can also accidentally expose fields the frontend does not need.

Therefore, if the contract only needs:

```text
id
name
quantity
price
```

I prefer returning those fields instead of blindly loading everything.

---

# Prisma Schema Example

```prisma
model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  createdAt DateTime @default(now())

  orders Order[]
}

model Product {
  id        Int      @id @default(autoincrement())
  name      String
  price     Decimal  @db.Decimal(12, 2)
  stock     Int
  createdAt DateTime @default(now())

  orderItems OrderItem[]
}

model Order {
  id        Int         @id @default(autoincrement())
  userId    Int
  status    OrderStatus
  createdAt DateTime    @default(now())

  user User @relation(
    fields: [userId],
    references: [id]
  )

  items OrderItem[]

  @@index([userId])
  @@index([createdAt])
}

model OrderItem {
  id        Int     @id @default(autoincrement())
  orderId   Int
  productId Int
  quantity  Int
  price     Decimal @db.Decimal(12, 2)

  order Order @relation(
    fields: [orderId],
    references: [id]
  )

  product Product @relation(
    fields: [productId],
    references: [id]
  )

  @@index([orderId])
  @@index([productId])
}

enum OrderStatus {
  PENDING
  PAID
  CANCELLED
}
```

---

# Database Constraints

Some important constraints should also exist.

For example:

```text
quantity > 0
price >= 0
```

Prisma alone may not express every PostgreSQL CHECK constraint directly depending on the workflow, so these can be added through SQL migration where needed.

Example:

```sql
ALTER TABLE order_items
ADD CONSTRAINT order_items_quantity_positive
CHECK (quantity > 0);
```

And:

```sql
ALTER TABLE order_items
ADD CONSTRAINT order_items_price_non_negative
CHECK (price >= 0);
```

---

# API Design

## Endpoint

```http
GET /orders/101
```

A good response should preserve the structure of the business domain.

Example:

```json
{
  "data": {
    "id": 101,
    "status": "PAID",
    "createdAt": "2026-10-01T08:00:00.000Z",
    "user": {
      "id": 1,
      "name": "Mulia",
      "email": "mulia@mail.com"
    },
    "items": [
      {
        "product": {
          "id": 1,
          "name": "Keyboard"
        },
        "quantity": 2,
        "price": 500000,
        "subtotal": 1000000
      },
      {
        "product": {
          "id": 2,
          "name": "Mouse"
        },
        "quantity": 1,
        "price": 200000,
        "subtotal": 200000
      }
    ],
    "total": 1200000
  }
}
```

This is preferable to returning a completely flat structure such as:

```json
[
  {
    "orderId": 101,
    "product": "Keyboard"
  },
  {
    "orderId": 101,
    "product": "Mouse"
  }
]
```

for an order-detail endpoint because the API response should reflect:

```text
one order
with many items
```

---

# Service Implementation

```ts
import { prisma } from "../lib/prisma";

export async function getOrderById(
  orderId: number,
) {
  const order =
    await prisma.order.findUnique({
      where: {
        id: orderId,
      },

      select: {
        id: true,
        status: true,
        createdAt: true,

        user: {
          select: {
            id: true,
            name: true,
            email: true,
          },
        },

        items: {
          select: {
            quantity: true,
            price: true,

            product: {
              select: {
                id: true,
                name: true,
              },
            },
          },
        },
      },
    });

  if (!order) {
    return null;
  }

  const items =
    order.items.map((item) => {
      const price =
        Number(item.price);

      const subtotal =
        price * item.quantity;

      return {
        product: item.product,
        quantity: item.quantity,
        price,
        subtotal,
      };
    });

  const total =
    items.reduce(
      (sum, item) =>
        sum + item.subtotal,
      0,
    );

  return {
    id: order.id,
    status: order.status,
    createdAt:
      order.createdAt,

    user: order.user,

    items,

    total,
  };
}
```

---

# Important Note About Money

For a production payment or accounting system, monetary values require special care.

Using JavaScript `number` for decimal financial calculations can introduce floating-point precision problems.

For example:

```ts
0.1 + 0.2;
```

does not produce an exact decimal representation.

Possible approaches include:

```text
Store money in the smallest integer unit

or

Use a decimal library/type
```

For Indonesian Rupiah without fractional rupiah, storing:

```text
500000
```

as an integer can simplify calculations.

For systems supporting decimal currencies, decimal-safe arithmetic should be considered.

---

# Controller

```ts
import {
  NextFunction,
  Request,
  Response,
} from "express";

import {
  getOrderById,
} from "../services/order.service";

export async function
getOrderByIdController(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  try {
    const orderId =
      Number(req.params.orderId);

    if (
      !Number.isInteger(orderId) ||
      orderId <= 0
    ) {
      return res.status(400).json({
        message:
          "Invalid order ID",
      });
    }

    const order =
      await getOrderById(
        orderId,
      );

    if (!order) {
      return res.status(404).json({
        message:
          "Order not found",
      });
    }

    return res.status(200).json({
      data: order,
    });
  } catch (error) {
    next(error);
  }
}
```

---

# Correct SQL for One Order Detail

```sql
SELECT
  o.id AS order_id,
  o.status,
  o.created_at,

  u.id AS user_id,
  u.name AS user_name,
  u.email AS user_email,

  p.id AS product_id,
  p.name AS product_name,

  oi.quantity,
  oi.price,

  oi.quantity * oi.price
    AS subtotal

FROM orders AS o

INNER JOIN users AS u
  ON u.id = o.user_id

INNER JOIN order_items AS oi
  ON oi.order_id = o.id

INNER JOIN products AS p
  ON p.id = oi.product_id

WHERE o.id = 101;
```

This returns one row per order item.

The backend can then transform the rows into a nested API structure if necessary.

---

# Correct Aggregate Query

If the goal is only to return an order summary:

```sql
SELECT
  o.id AS order_id,
  u.name AS user_name,
  o.status,

  SUM(
    oi.quantity * oi.price
  ) AS total

FROM orders AS o

INNER JOIN users AS u
  ON u.id = o.user_id

INNER JOIN order_items AS oi
  ON oi.order_id = o.id

GROUP BY
  o.id,
  u.name,
  o.status;
```

---

# Handling Order Without Items

If orders without items must still appear:

```sql
SELECT
  o.id AS order_id,
  u.name AS user_name,

  COALESCE(
    SUM(
      oi.quantity * oi.price
    ),
    0
  ) AS total

FROM orders AS o

INNER JOIN users AS u
  ON u.id = o.user_id

LEFT JOIN order_items AS oi
  ON oi.order_id = o.id

GROUP BY
  o.id,
  u.name;
```

Why `COALESCE`?

An order with no matching items may produce:

```text
SUM(...) = NULL
```

But the API may want:

```text
0
```

So:

```sql
COALESCE(
  SUM(...),
  0
)
```

converts `NULL` to zero.

---

# Code Review

Incorrect query:

```sql
SELECT *
FROM orders
JOIN users
  ON users.id = orders.id
JOIN order_items
  ON order_items.id = orders.id
JOIN products
  ON products.id = order_items.id;
```

There are multiple problems.

---

## Problem 1: Wrong User Relationship

Incorrect:

```sql
users.id = orders.id
```

This compares:

```text
user primary key
```

with:

```text
order primary key
```

Those values have no direct relational meaning.

Correct:

```sql
users.id =
orders.user_id
```

---

## Problem 2: Wrong OrderItem Relationship

Incorrect:

```sql
order_items.id =
orders.id
```

This compares two unrelated primary keys.

Correct:

```sql
order_items.order_id =
orders.id
```

---

## Problem 3: Wrong Product Relationship

Incorrect:

```sql
products.id =
order_items.id
```

Correct:

```sql
products.id =
order_items.product_id
```

---

## Problem 4: SELECT *

```sql
SELECT *
```

can return unnecessary columns.

This can:

```text
Increase response size
Create naming collisions
Make the API contract unclear
Expose unnecessary data
```

Prefer explicit columns:

```sql
SELECT
  o.id,
  u.name,
  p.name,
  oi.quantity,
  oi.price
```

---

# Corrected Query

```sql
SELECT
  o.id AS order_id,
  u.name AS user_name,
  u.email AS user_email,
  p.name AS product_name,
  oi.quantity,
  oi.price
FROM orders AS o

INNER JOIN users AS u
  ON u.id = o.user_id

INNER JOIN order_items AS oi
  ON oi.order_id = o.id

INNER JOIN products AS p
  ON p.id = oi.product_id;
```

---

# Testing

## 1. Order With One Product

Data:

```text
Monitor
quantity = 1
price = 2500000
```

Expected:

```text
total = 2500000
```

---

## 2. Order With Multiple Products

Data:

```text
Keyboard
2 × 500000

Mouse
1 × 200000
```

Expected:

```text
total = 1200000
```

---

## 3. Different Users

Create:

```text
Order 101 → Mulia
Order 102 → Andi
```

Verify each order contains the correct user.

This helps detect incorrect JOIN conditions.

---

## 4. Order Without Items

If the API supports empty orders:

```text
items = []
total = 0
```

If empty orders are invalid according to the business rule, test that creation prevents them instead.

---

## 5. Product Price Changes

Initial product:

```text
Keyboard
500000
```

Create order.

Saved:

```text
order_items.price
=
500000
```

Then change:

```text
products.price
=
650000
```

Expected old order:

```text
price = 500000
```

The historical total must remain unchanged.

---

## 6. Quantity Greater Than One

Data:

```text
quantity = 3
price = 200000
```

Expected:

```text
subtotal = 600000
```

---

## 7. Total Calculation

For:

```text
3 × 200000
+
2 × 500000
```

expected:

```text
1600000
```

---

## 8. Order Not Found

Request:

```http
GET /orders/999999
```

Expected:

```http
404 Not Found
```

Example response:

```json
{
  "message": "Order not found"
}
```

---

# Integration Test Example

```ts
import request from "supertest";

import {
  describe,
  expect,
  it,
} from "vitest";

import {
  app,
} from "../src/app";

describe(
  "GET /orders/:orderId",
  () => {
    it(
      "returns order with items and total",
      async () => {
        const response =
          await request(app)
            .get(
              "/orders/101",
            );

        expect(
          response.status,
        ).toBe(200);

        expect(
          response.body.data.id,
        ).toBe(101);

        expect(
          Array.isArray(
            response.body.data
              .items,
          ),
        ).toBe(true);

        expect(
          response.body.data
            .total,
        ).toBe(1200000);
      },
    );
  },
);
```

---

# Product Price History Test

```ts
it(
  "keeps historical order item price",
  async () => {
    const order =
      await createOrder({
        productId: 1,
        quantity: 1,
      });

    await prisma.product.update({
      where: {
        id: 1,
      },

      data: {
        price: 650000,
      },
    });

    const result =
      await getOrderById(
        order.id,
      );

    expect(
      result?.items[0].price,
    ).toBe(500000);
  },
);
```

---

# N+1 Test Strategy

There are several ways to detect N+1 problems.

## Code Review

Look for patterns such as:

```ts
for (const item of items) {
  await database.query(...);
}
```

This is a strong warning sign.

---

## Database Logging

Enable query logging during development.

Then compare:

```text
10 orders
100 orders
1000 orders
```

The query count should not grow linearly because of manual per-record relationship lookups.

---

## Performance Testing

Measure:

```text
Query count
API latency
Database connections
Database CPU
```

as the number of orders increases.

For example:

```text
10 orders
→ 21 queries

100 orders
→ 201 queries

1000 orders
→ 2001 queries
```

would clearly indicate an N+1 pattern.

---

# Performance Considerations

Indexes should exist on important foreign keys.

For example:

```sql
CREATE INDEX
idx_orders_user_id
ON orders(user_id);
```

```sql
CREATE INDEX
idx_order_items_order_id
ON order_items(order_id);
```

```sql
CREATE INDEX
idx_order_items_product_id
ON order_items(product_id);
```

These improve common JOIN and lookup operations.

---

# Why Foreign Key Indexes Matter

Suppose:

```sql
WHERE order_items.order_id = 101
```

Without an appropriate index, PostgreSQL may need to scan many rows.

With:

```sql
INDEX(order_id)
```

the database can locate matching items more efficiently.

However, indexes should still be chosen based on query patterns and verified with tools such as:

```sql
EXPLAIN ANALYZE
```

rather than added blindly.

---

# Project Structure

```text
fullstack-interview-lab/
└── day-04-sql-join-orders/
    ├── prisma/
    │   └── schema.prisma
    │
    ├── src/
    │   ├── controllers/
    │   │   └── order.controller.ts
    │   │
    │   ├── services/
    │   │   └── order.service.ts
    │   │
    │   ├── routes/
    │   │   └── order.route.ts
    │   │
    │   ├── lib/
    │   │   └── prisma.ts
    │   │
    │   ├── app.ts
    │   └── server.ts
    │
    ├── tests/
    │   └── order.test.ts
    │
    ├── sql/
    │   ├── joins.sql
    │   └── aggregate.sql
    │
    ├── package.json
    ├── tsconfig.json
    └── README.md
```

---

# Tech Stack

```text
Node.js
Express.js
TypeScript
PostgreSQL
Prisma
Vitest
Supertest
```

---

# How To Run

Install dependencies:

```bash
npm install
```

Configure:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/interview_lab"
PORT=4000
```

Generate Prisma client:

```bash
npx prisma generate
```

Run migration:

```bash
npx prisma migrate dev
```

Start development server:

```bash
npm run dev
```

Run tests:

```bash
npm test
```

---

# Better Interview Answer

> I would first identify the foreign key relationships before writing the JOINs. `orders.user_id` references `users.id`, `order_items.order_id` references `orders.id`, and `order_items.product_id` references `products.id`.
>
> To retrieve an order with its user and products, I would join those tables using those foreign keys. If I only want orders that contain items, `INNER JOIN` is appropriate. If orders without items must still appear, I would use `LEFT JOIN` from orders to order items.
>
> To calculate the order total, I would calculate `quantity * price` for each order item and aggregate the values using `SUM`, grouped by the order.
>
> I would use `order_items.price` rather than `products.price` for historical order calculations because product prices can change after the transaction. The order item should preserve the price that the customer actually paid.
>
> In the application layer, I would also avoid manually fetching related data inside a loop because that can cause an N+1 query problem. With Prisma, I would fetch the required relationships using nested `select` or `include`, while selecting only the fields required by the API.
>
> I would also make sure foreign key columns used frequently in joins are indexed and validate query performance using `EXPLAIN ANALYZE` when necessary.

---

# Follow-Up Interview Questions

## 1. Why not calculate order total using products.price?

Because `products.price` represents the current product price.

Historical orders must preserve the transaction price.

Use:

```text
order_items.price
```

for historical calculations.

---

## 2. Why not store product_id directly in orders?

Because one order can contain many products.

The `order_items` table represents the one-to-many relationship and stores relationship-specific data such as:

```text
quantity
price
discount
```

---

## 3. What is an N+1 query?

It occurs when one query retrieves N parent records and then additional queries are executed individually for those records.

Example:

```text
1 order query
+
100 user queries
=
101 queries
```

---

## 4. When should LEFT JOIN be used?

When rows from the main table must still be returned even if no matching related row exists.

Example:

```text
Show orders
even if they have zero items.
```

---

## 5. Why use select instead of selecting everything?

To reduce:

```text
Unnecessary data transfer
Memory usage
Serialization
Potential exposure of irrelevant fields
```

and make the API contract more explicit.

---

# What I Learned

The most important lesson is that SQL JOINs must follow actual database relationships.

Wrong approach:

```text
users.id = orders.id
```

simply because both columns are named `id`.

Correct approach:

```text
users.id =
orders.user_id
```

because:

```text
orders.user_id
```

is the foreign key.

The correct thinking process is:

```text
Understand entities
↓
Identify primary keys
↓
Identify foreign keys
↓
Understand cardinality
↓
Choose JOIN type
↓
Aggregate where necessary
↓
Avoid N+1
↓
Measure query performance
```

Other important concepts learned:

```text
INNER JOIN
LEFT JOIN
GROUP BY
SUM
Foreign key
One-to-many relationship
Historical transaction data
N+1 query
Prisma include
Prisma select
Database indexes
```

---

# Interview Key Takeaway

Do not memorize:

```sql
JOIN A
JOIN B
JOIN C
```

Instead ask:

```text
What relationship am I following?
```

For this case:

```text
User
  ↓
Order
  ↓
OrderItem
  ↓
Product
```

And always remember:

```text
Primary Key
≠
Foreign Key
```

Two columns named:

```text
id
```

do not automatically have a relationship.

---

# Git Workflow

Create branch:

```bash
git checkout -b feat/day-04-order-query
```

Suggested commits:

```bash
git commit -m "feat: add relational order data model"
```

```bash
git commit -m "feat: add order detail query with relations"
```

```bash
git commit -m "perf: add indexes for order relationships"
```

```bash
git commit -m "test: add order aggregation and pricing tests"
```

---

# Progress Summary

```text
Day:
04

Topic:
SQL JOIN, Aggregation, and Relational Data Modeling

Difficulty:
Fundamental

Main Skill:
SQL JOIN and Relational Database Design

Secondary Skills:
INNER JOIN
LEFT JOIN
GROUP BY
SUM
Foreign Keys
Prisma Relations
N+1 Query
API Design
Database Indexing

Project:
Order Management API

GitHub Folder:
day-04-sql-join-orders

Suggested Branch:
feat/day-04-order-query

Suggested Commits:
feat: add relational order data model
feat: add order detail query with relations
perf: add indexes for order relationships
test: add order aggregation and pricing tests
```