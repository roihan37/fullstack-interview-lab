# Day 01 - REST API Pagination, Search, Filter, and Sorting

## Problem

An e-commerce application has a `GET /products` endpoint.

Initially, the application only had around 100 products, so the backend returned all products at once.

```ts
app.get("/products", async (req, res) => {
  const products = await prisma.product.findMany();

  res.json({
    data: products,
  });
});
```

As the application grew, the database reached approximately **50,000 products**.

Returning all products caused several problems:

* Large API responses
* Higher database load
* Increased memory usage
* Slower frontend rendering
* Higher network usage
* Poor user experience

The endpoint needs to support:

* Pagination
* Search
* Category filtering
* Sorting
* Query parameter validation

---

# Interview Question

> How would you redesign the `GET /products` endpoint so that it can efficiently handle tens of thousands of products?

---

# Interview Answer

I would first avoid returning all products at once and introduce pagination.

For this case, I would use offset-based pagination because the dataset is still relatively manageable and the frontend needs traditional page navigation.

The endpoint could look like this:

```http
GET /products?page=1&limit=20&search=iphone&category=3&sortBy=price&sortOrder=asc
```

I would support several query parameters:

```text
page
limit
search
category
sortBy
sortOrder
```

For pagination, I would calculate the offset using:

```text
offset = (page - 1) * limit
```

For example:

```text
page = 3
limit = 20

offset = (3 - 1) * 20
offset = 40
```

The database would skip the first 40 rows and return the next 20.

I would also validate all query parameters before sending them to the database.

For example:

* `page` must be at least 1
* `limit` must have a maximum value, for example 100
* `sortBy` must only contain allowed fields
* `sortOrder` must only be `asc` or `desc`

I would never pass a user-provided sorting field directly into the query.

For example:

```http
GET /products?sortBy=password
```

should return a validation error because `password` is not part of the allowed sorting fields.

For search, I would initially use case-insensitive name search.

For category filtering, I would add `categoryId` to the database condition.

I would also return pagination metadata so that the frontend knows the total number of pages.

For approximately 50,000 products, I would not immediately introduce Redis.

First, I would optimize:

1. API pagination
2. Database query
3. Database indexes
4. Selected response fields
5. Query performance

Redis introduces additional complexity such as cache invalidation and stale data.

I would only introduce caching if monitoring shows that database queries are still becoming a bottleneck.

---

# API Design

## Endpoint

```http
GET /products
```

## Query Parameters

| Parameter   | Type   |     Default | Description                 |
| ----------- | ------ | ----------: | --------------------------- |
| `page`      | number |           1 | Current page                |
| `limit`     | number |          10 | Number of products per page |
| `search`    | string |           - | Search product name         |
| `category`  | number |           - | Filter category             |
| `sortBy`    | string | `createdAt` | Sorting field               |
| `sortOrder` | string |      `desc` | `asc` or `desc`             |

Allowed `sortBy` values:

```text
price
name
createdAt
```

Maximum limit:

```text
100
```

---

# Request Examples

Basic pagination:

```http
GET /products?page=1&limit=10
```

Search:

```http
GET /products?page=1&limit=10&search=iphone
```

Category filtering:

```http
GET /products?page=1&limit=10&category=3
```

Sorting:

```http
GET /products?page=1&limit=20&sortBy=price&sortOrder=asc
```

Combined:

```http
GET /products?page=2&limit=20&search=iphone&category=3&sortBy=price&sortOrder=asc
```

---

# Response

```json
{
  "data": [
    {
      "id": 21,
      "name": "iPhone 17",
      "price": 14990000,
      "stock": 20,
      "categoryId": 3,
      "createdAt": "2026-09-20T10:00:00.000Z"
    }
  ],
  "meta": {
    "page": 2,
    "limit": 20,
    "total": 153,
    "totalPages": 8,
    "hasNextPage": true,
    "hasPreviousPage": true
  }
}
```

---

# Invalid Request

Request:

```http
GET /products?page=-2&limit=100000
```

Response:

```http
400 Bad Request
```

Example:

```json
{
  "message": "Invalid query parameters",
  "errors": {
    "page": "Page must be greater than or equal to 1",
    "limit": "Limit must be less than or equal to 100"
  }
}
```

---

# Why Limit Must Be Restricted

Without a maximum limit, a client could request:

```http
GET /products?limit=1000000
```

Even though pagination exists, the request would effectively behave like returning the entire database.

Therefore:

```text
1 <= limit <= 100
```

is enforced.

---

# Project Structure

```text
fullstack-interview-lab/
└── day-01-rest-api-pagination/
    ├── prisma/
    │   └── schema.prisma
    │
    ├── src/
    │   ├── controllers/
    │   │   └── product.controller.ts
    │   │
    │   ├── repositories/
    │   │   └── product.repository.ts
    │   │
    │   ├── routes/
    │   │   └── product.route.ts
    │   │
    │   ├── schemas/
    │   │   └── product.schema.ts
    │   │
    │   ├── services/
    │   │   └── product.service.ts
    │   │
    │   ├── lib/
    │   │   └── prisma.ts
    │   │
    │   ├── app.ts
    │   └── server.ts
    │
    ├── tests/
    │   └── product.test.ts
    │
    ├── .env.example
    ├── package.json
    ├── tsconfig.json
    └── README.md
```

The structure separates HTTP handling, business logic, database queries, and validation without introducing unnecessary abstraction.

---

# Tech Stack

```text
Node.js
Express.js
TypeScript
PostgreSQL
Prisma
Zod
Vitest
Supertest
```

---

# Prisma Schema

```prisma
model Product {
  id          Int      @id @default(autoincrement())
  name        String
  description String?
  price       Decimal  @db.Decimal(12, 2)
  stock       Int      @default(0)
  categoryId  Int
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([categoryId])
  @@index([price])
  @@index([createdAt])
}
```

Indexes are added to fields frequently used for filtering and sorting.

---

# Query Validation

`src/schemas/product.schema.ts`

```ts
import { z } from "zod";

export const productQuerySchema = z.object({
  page: z.coerce
    .number()
    .int()
    .min(1)
    .default(1),

  limit: z.coerce
    .number()
    .int()
    .min(1)
    .max(100)
    .default(10),

  search: z
    .string()
    .trim()
    .min(1)
    .optional(),

  category: z.coerce
    .number()
    .int()
    .positive()
    .optional(),

  sortBy: z
    .enum(["name", "price", "createdAt"])
    .default("createdAt"),

  sortOrder: z
    .enum(["asc", "desc"])
    .default("desc"),
});

export type ProductQuery = z.infer<typeof productQuerySchema>;
```

Using an enum prevents users from sorting using arbitrary database fields.

For example:

```http
GET /products?sortBy=password
```

will fail validation.

---

# Prisma Client

`src/lib/prisma.ts`

```ts
import { PrismaClient } from "@prisma/client";

export const prisma = new PrismaClient();
```

---

# Repository

`src/repositories/product.repository.ts`

```ts
import { Prisma } from "@prisma/client";
import { prisma } from "../lib/prisma";

interface FindProductsParams {
  skip: number;
  take: number;
  search?: string;
  category?: number;
  sortBy: "name" | "price" | "createdAt";
  sortOrder: "asc" | "desc";
}

export async function findProducts({
  skip,
  take,
  search,
  category,
  sortBy,
  sortOrder,
}: FindProductsParams) {
  const where: Prisma.ProductWhereInput = {};

  if (search) {
    where.name = {
      contains: search,
      mode: "insensitive",
    };
  }

  if (category) {
    where.categoryId = category;
  }

  return prisma.$transaction([
    prisma.product.findMany({
      where,
      skip,
      take,

      orderBy: {
        [sortBy]: sortOrder,
      },

      select: {
        id: true,
        name: true,
        price: true,
        stock: true,
        categoryId: true,
        createdAt: true,
      },
    }),

    prisma.product.count({
      where,
    }),
  ]);
}
```

The API does not return unnecessary fields such as long product descriptions when the frontend only needs product list information.

This reduces payload size.

---

# Service

`src/services/product.service.ts`

```ts
import { ProductQuery } from "../schemas/product.schema";
import { findProducts } from "../repositories/product.repository";

export async function getProducts(query: ProductQuery) {
  const {
    page,
    limit,
    search,
    category,
    sortBy,
    sortOrder,
  } = query;

  const skip = (page - 1) * limit;

  const [products, total] = await findProducts({
    skip,
    take: limit,
    search,
    category,
    sortBy,
    sortOrder,
  });

  const totalPages = Math.ceil(total / limit);

  return {
    data: products,

    meta: {
      page,
      limit,
      total,
      totalPages,

      hasNextPage: page < totalPages,

      hasPreviousPage: page > 1,
    },
  };
}
```

---

# Controller

`src/controllers/product.controller.ts`

```ts
import { Request, Response, NextFunction } from "express";

import { productQuerySchema } from "../schemas/product.schema";
import { getProducts } from "../services/product.service";

export async function getProductsController(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  try {
    const parsedQuery = productQuerySchema.safeParse(req.query);

    if (!parsedQuery.success) {
      return res.status(400).json({
        message: "Invalid query parameters",
        errors: parsedQuery.error.flatten().fieldErrors,
      });
    }

    const result = await getProducts(parsedQuery.data);

    return res.status(200).json(result);
  } catch (error) {
    next(error);
  }
}
```

---

# Route

`src/routes/product.route.ts`

```ts
import { Router } from "express";

import { getProductsController } from "../controllers/product.controller";

const router = Router();

router.get("/", getProductsController);

export default router;
```

---

# Express App

`src/app.ts`

```ts
import express, {
  NextFunction,
  Request,
  Response,
} from "express";

import productRoutes from "./routes/product.route";

export const app = express();

app.use(express.json());

app.use("/products", productRoutes);

app.use(
  (
    error: Error,
    req: Request,
    res: Response,
    next: NextFunction,
  ) => {
    console.error(error);

    return res.status(500).json({
      message: "Internal server error",
    });
  },
);
```

---

# Server

`src/server.ts`

```ts
import { app } from "./app";

const PORT = Number(process.env.PORT) || 4000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

# Search Query

The initial implementation uses:

```ts
where.name = {
  contains: search,
  mode: "insensitive",
};
```

Conceptually this is similar to:

```sql
WHERE name ILIKE '%iphone%'
```

This is acceptable as an initial implementation.

However, as the number of products becomes much larger, `%keyword%` searching can become expensive.

For larger datasets I would investigate PostgreSQL search optimization such as:

```text
pg_trgm
GIN index
Full-text search
```

before introducing another search technology.

---

# Database Index Consideration

For common queries such as:

```sql
WHERE category_id = 3
ORDER BY created_at DESC
```

a composite index could eventually be considered:

```sql
CREATE INDEX idx_products_category_created_at
ON products(category_id, created_at DESC);
```

However, indexes should not be added blindly.

They improve read performance but also:

* Consume disk space
* Increase INSERT cost
* Increase UPDATE cost
* Require maintenance

The correct indexes should be determined from actual query patterns and PostgreSQL execution plans.

---

# Why Not Redis Yet?

I would not introduce Redis immediately.

The current problem is primarily caused by returning too many database rows.

The first optimization should be:

```text
Pagination
↓
Validation
↓
Database indexing
↓
Query optimization
↓
Measure performance
```

Only after measuring the system would I decide whether caching is necessary.

Redis could help when:

* The same product queries are requested frequently
* Database traffic becomes high
* Product data does not change frequently
* API latency remains high after query optimization

But caching also introduces problems:

```text
Cache invalidation
Stale data
Redis availability
Additional infrastructure
More application complexity
```

Therefore Redis should solve a measured problem, not be added just because the dataset is large.

---

# Security Consideration

Never use raw user input directly for sorting.

Bad implementation:

```ts
const sortBy = req.query.sortBy;

orderBy: {
  [sortBy]: "asc",
}
```

Instead, use an allowlist:

```ts
const allowedSortFields = [
  "name",
  "price",
  "createdAt",
] as const;
```

In this implementation Zod already performs the allowlist validation:

```ts
z.enum([
  "name",
  "price",
  "createdAt",
]);
```

Therefore:

```http
GET /products?sortBy=password
```

returns:

```http
400 Bad Request
```

---

# Testing

Important scenarios:

## 1. Default Pagination

```http
GET /products
```

Expected:

```text
200 OK
page = 1
limit = 10
```

---

## 2. Custom Pagination

```http
GET /products?page=2&limit=20
```

Expected:

```text
200 OK
page = 2
limit = 20
```

---

## 3. Search

```http
GET /products?search=iphone
```

Expected:

```text
Only matching products are returned
```

---

## 4. Category Filter

```http
GET /products?category=3
```

Expected:

```text
Only products with categoryId = 3
```

---

## 5. Sorting

```http
GET /products?sortBy=price&sortOrder=asc
```

Expected:

```text
Products sorted from lowest price
```

---

## 6. Invalid Page

```http
GET /products?page=-1
```

Expected:

```http
400 Bad Request
```

---

## 7. Invalid Limit

```http
GET /products?limit=100000
```

Expected:

```http
400 Bad Request
```

---

## 8. Invalid Sort Field

```http
GET /products?sortBy=password
```

Expected:

```http
400 Bad Request
```

---

# Example Integration Test

`tests/product.test.ts`

```ts
import request from "supertest";
import { describe, expect, it } from "vitest";

import { app } from "../src/app";

describe("GET /products", () => {
  it("should reject page below 1", async () => {
    const response = await request(app)
      .get("/products?page=-1");

    expect(response.status).toBe(400);
  });

  it("should reject limit above 100", async () => {
    const response = await request(app)
      .get("/products?limit=100000");

    expect(response.status).toBe(400);
  });

  it("should reject invalid sort field", async () => {
    const response = await request(app)
      .get("/products?sortBy=password");

    expect(response.status).toBe(400);
  });

  it("should reject invalid sort order", async () => {
    const response = await request(app)
      .get("/products?sortOrder=random");

    expect(response.status).toBe(400);
  });
});
```

For a complete integration test suite, I would also prepare test product records in a dedicated test database and verify:

```text
Pagination results
Search results
Category filtering
Sorting
Pagination metadata
Empty results
Last page behavior
```

---

# Edge Cases

Several edge cases should be considered.

### Page exceeds available pages

Request:

```http
GET /products?page=9999
```

Response:

```json
{
  "data": [],
  "meta": {
    "page": 9999,
    "limit": 10,
    "total": 50,
    "totalPages": 5,
    "hasNextPage": false,
    "hasPreviousPage": true
  }
}
```

Returning an empty array is acceptable because the request itself is valid.

---

### Search returns no products

```http
GET /products?search=unknownproduct
```

Response:

```json
{
  "data": [],
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 0,
    "totalPages": 0,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

---

# Performance Considerations

For 50,000 products, offset pagination is still reasonable for normal page navigation.

However, very deep pagination can eventually become expensive.

Example:

```sql
OFFSET 900000
LIMIT 20
```

PostgreSQL still has to process many rows before reaching the requested offset.

If the application eventually grows to millions of records or supports infinite scrolling, I would consider:

```text
Cursor-based pagination
```

For example:

```http
GET /products?cursor=12345&limit=20
```

This is something I would introduce based on scale and product requirements rather than prematurely.

---

# Trade-offs

## Offset Pagination

Advantages:

```text
Simple
Easy page navigation
Easy frontend implementation
Supports "page 1, 2, 3..."
```

Disadvantages:

```text
Deep pagination becomes slower
Can produce inconsistent pages when records are inserted frequently
```

---

## Cursor Pagination

Advantages:

```text
Efficient for large datasets
Good for infinite scrolling
More stable when new records are inserted
```

Disadvantages:

```text
More complex
Difficult to jump directly to arbitrary pages
```

For this requirement, I would start with offset pagination.

---

# How To Run

Install dependencies:

```bash
npm install
```

Create `.env`:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/interview_lab"
PORT=4000
```

Run Prisma migration:

```bash
npx prisma migrate dev
```

Generate Prisma client:

```bash
npx prisma generate
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

# Follow-up Interview Questions

## Question 1

### What happens when the products table grows to 10 million rows?

Offset pagination can become inefficient for deep pages.

I would investigate cursor pagination and analyze queries using:

```sql
EXPLAIN ANALYZE
```

before making architecture changes.

---

## Question 2

### Should we use Redis when we reach 10 million products?

Not automatically.

The number of rows alone does not determine whether Redis is necessary.

I would check:

```text
Database CPU
Query latency
Cache hit opportunities
Request frequency
Repeated query patterns
Database connections
```

Then decide whether caching provides meaningful value.

---

## Question 3

### How would you improve product search?

For simple cases:

```text
PostgreSQL ILIKE
```

For larger search workloads:

```text
PostgreSQL pg_trgm
GIN indexes
PostgreSQL Full Text Search
```

Only when search requirements become significantly more complex would I evaluate dedicated search systems.

---

## Question 4

### What happens if two products have the same price when sorting?

Sorting should have a deterministic secondary field.

For example:

```ts
orderBy: [
  {
    price: "asc",
  },
  {
    id: "asc",
  },
]
```

This prevents unstable ordering when multiple rows contain the same price.

---

## Question 5

### How would you know whether your optimization worked?

I would measure:

```text
API latency
Database query duration
Response payload size
Rows scanned
Database CPU
Database connections
Throughput
p95 / p99 latency
```

and compare the results before and after optimization.

---

# What I Learned

From this challenge I learned that improving API performance does not immediately mean adding complex infrastructure.

The correct approach is:

```text
Understand the problem
↓
Reduce unnecessary data
↓
Validate requests
↓
Optimize database queries
↓
Add appropriate indexes
↓
Measure performance
↓
Introduce additional technology only if necessary
```

I also learned the difference between offset and cursor pagination, why query parameters must be validated, and why sorting fields should use an allowlist.

---

# Interview Key Takeaway

A weak answer would be:

> "I will add pagination and Redis."

A stronger engineering answer explains why:

> "The primary problem is that the API retrieves all rows. I would first implement pagination, restrict the maximum page size, validate filtering and sorting parameters, select only required columns, and inspect database indexes. I would measure query performance before considering Redis because adding a cache would introduce invalidation and consistency complexity without necessarily addressing the root cause."

The important principle is:

```text
Problem
→ Requirement
→ Bottleneck
→ Solution
→ Trade-off
→ Technology
```

not:

```text
Problem
→ Redis
```

---

# Git Workflow

Create branch:

```bash
git checkout -b feat/day-01-product-pagination
```

Recommended commits:

```bash
git add .
git commit -m "feat: add product pagination and filtering"
```

Then:

```bash
git commit -m "feat: add product query validation"
```

Then:

```bash
git commit -m "test: add product query validation tests"
```

If adding indexes separately:

```bash
git commit -m "perf: add indexes for product queries"
```

---

# Progress Summary

```text
Day:
01

Topic:
REST API Pagination, Search, Filter, and Sorting

Difficulty:
Fundamental

Main Skill:
REST API Design

Secondary Skills:
PostgreSQL
Prisma
Validation
Query Optimization
API Testing

Project:
Product Listing API

GitHub Folder:
day-01-rest-api-pagination

Suggested Branch:
feat/day-01-product-pagination

Suggested Commits:
feat: add product pagination and filtering
feat: add product query validation
test: add product query validation tests
perf: add indexes for product queries
```
