# Day 02 - Async/Await, Promise.all, and Partial Failure Handling

## Problem

A dashboard endpoint needs to load three independent resources:

```ts
getUser(userId)
getOrders(userId)
getNotifications(userId)
```

Current implementation:

```ts
app.get("/dashboard/:userId", async (req, res, next) => {
  try {
    const user = await getUser(req.params.userId);
    const orders = await getOrders(req.params.userId);
    const notifications = await getNotifications(req.params.userId);

    return res.status(200).json({
      user,
      orders,
      notifications,
    });
  } catch (error) {
    next(error);
  }
});
```

Average latency:

```text
getUser()            ≈ 300 ms
getOrders()          ≈ 500 ms
getNotifications()   ≈ 400 ms
```

The endpoint takes approximately:

```text
300 + 500 + 400 = 1200 ms
```

or around:

```text
1.2 seconds
```

The goal is to improve response time while still handling failures correctly.

---

# Interview Question

> Why is this endpoint taking around 1.2 seconds, and how would you improve it?

Follow-up:

> Would you always use `Promise.all()` for this kind of problem?

---

# Interview Answer

The endpoint takes around 1.2 seconds because the asynchronous operations are executed sequentially.

The current code:

```ts
const user = await getUser(userId);
const orders = await getOrders(userId);
const notifications = await getNotifications(userId);
```

means that `getOrders()` does not start until `getUser()` finishes.

Similarly, `getNotifications()` does not start until `getOrders()` finishes.

The execution is approximately:

```text
getUser
300 ms
   ↓
getOrders
500 ms
   ↓
getNotifications
400 ms
```

Total:

```text
1200 ms
```

Because these operations are independent, they can be executed concurrently.

A basic optimization would be:

```ts
const [user, orders, notifications] = await Promise.all([
  getUser(userId),
  getOrders(userId),
  getNotifications(userId),
]);
```

Now the operations start at approximately the same time:

```text
getUser             300 ms
getOrders            500 ms
getNotifications     400 ms
```

The total execution time becomes approximately equal to the slowest operation:

```text
≈ 500 ms
```

instead of:

```text
≈ 1200 ms
```

However, I would not use `Promise.all()` blindly because the endpoint has different failure requirements.

---

# Sequential vs Concurrent Execution

## Sequential

```ts
const user = await getUser(userId);
const orders = await getOrders(userId);
const notifications = await getNotifications(userId);
```

Execution:

```text
getUser
   ↓
getOrders
   ↓
getNotifications
```

Total latency:

```text
300 + 500 + 400
= 1200 ms
```

---

## Concurrent

```ts
const [user, orders, notifications] = await Promise.all([
  getUser(userId),
  getOrders(userId),
  getNotifications(userId),
]);
```

Execution:

```text
getUser          ───────── 300 ms
getOrders        ───────────────── 500 ms
getNotifications ───────────── 400 ms
```

Total latency is approximately:

```text
max(300, 500, 400)

≈ 500 ms
```

The exact API latency can still be slightly higher because of:

- Network overhead
- Serialization
- Middleware
- Database connection waiting
- Runtime scheduling
- Logging

But conceptually, the execution changes from:

```text
O(sum of latency)
```

to approximately:

```text
O(max latency)
```

for independent I/O operations.

---

# Important Behavior of Promise.all

`Promise.all()` has fail-fast behavior.

Example:

```ts
await Promise.all([
  getUser(userId),
  getOrders(userId),
  getNotifications(userId),
]);
```

If `getNotifications()` rejects, the entire `Promise.all()` rejects.

Even if:

```text
getUser() succeeded
getOrders() succeeded
```

the result will not be returned normally.

Example:

```ts
Promise.all([
  Promise.resolve("user"),
  Promise.resolve("orders"),
  Promise.reject(new Error("Notification service unavailable")),
]);
```

will reject.

This is important because our business requirement says:

```text
User          required
Orders        required
Notifications optional
```

Therefore, using a single `Promise.all()` directly is not ideal.

---

# Requirements

The endpoint has these rules:

```text
User
→ required

Orders
→ required

Notifications
→ optional
```

If the user does not exist:

```http
404 Not Found
```

If orders fail:

```http
5xx Server Error
```

If notifications fail:

```text
Dashboard should still load.
```

Example valid response:

```json
{
  "user": {
    "id": "123",
    "name": "John"
  },
  "orders": [],
  "notifications": []
}
```

---

# Recommended Approach

I would keep the required operations together while handling the optional dependency separately.

One reasonable implementation is:

```ts
const userPromise = getUser(userId);
const ordersPromise = getOrders(userId);

const notificationsPromise = getNotifications(userId)
  .catch((error) => {
    console.error("Failed to fetch notifications", error);

    return [];
  });

const [user, orders, notifications] = await Promise.all([
  userPromise,
  ordersPromise,
  notificationsPromise,
]);
```

Now:

```text
getUser fails
→ Promise.all rejects

getOrders fails
→ Promise.all rejects

getNotifications fails
→ error handled locally
→ []
→ Promise.all still resolves
```

This matches the business requirement better.

---

# Better Implementation

A clearer implementation:

```ts
async function getDashboard(userId: string) {
  const userPromise = getUser(userId);

  const ordersPromise = getOrders(userId);

  const notificationsPromise = getNotifications(userId)
    .catch((error) => {
      console.error(
        `Failed to fetch notifications for user ${userId}`,
        error,
      );

      return [];
    });

  const [user, orders, notifications] = await Promise.all([
    userPromise,
    ordersPromise,
    notificationsPromise,
  ]);

  return {
    user,
    orders,
    notifications,
  };
}
```

---

# But There Is an Important Trade-off

There is another subtle issue.

Suppose:

```text
getUser()
→ user does not exist

getOrders()
→ expensive database query

getNotifications()
→ external API request
```

If all three operations are started simultaneously, the orders and notification requests still start even though the user might not exist.

Depending on the system, it might therefore be reasonable to perform user validation first.

For example:

```ts
const user = await getUser(userId);

const [orders, notifications] = await Promise.all([
  getOrders(userId),

  getNotifications(userId)
    .catch(() => []),
]);

return {
  user,
  orders,
  notifications,
};
```

Execution:

```text
getUser
300 ms
   ↓

getOrders            500 ms
getNotifications     400 ms
```

Total:

```text
300 + 500
≈ 800 ms
```

This is slower than executing everything concurrently:

```text
≈ 500 ms
```

but it avoids unnecessary downstream work when the user does not exist.

Therefore the decision depends on requirements.

---

# Option 1 - Maximum Performance

Run everything concurrently:

```ts
const [user, orders, notifications] = await Promise.all([
  getUser(userId),

  getOrders(userId),

  getNotifications(userId)
    .catch(() => []),
]);
```

Approximate latency:

```text
500 ms
```

Advantages:

- Lowest latency
- Simple
- Independent operations start immediately

Disadvantages:

- Orders may run for nonexistent users
- Notifications may be requested unnecessarily
- More downstream resource usage

---

# Option 2 - Validate User First

```ts
const user = await getUser(userId);

const [orders, notifications] = await Promise.all([
  getOrders(userId),

  getNotifications(userId)
    .catch(() => []),
]);

return {
  user,
  orders,
  notifications,
};
```

Approximate latency:

```text
300 + 500
≈ 800 ms
```

Advantages:

- Avoid unnecessary downstream calls for invalid users
- Clear business flow
- Easier 404 behavior

Disadvantages:

- Slightly higher latency for valid users

For many real applications, I would choose this approach if validating the user first is important and inexpensive.

---

# Should We Use Promise.allSettled?

Another possibility is:

```ts
const results = await Promise.allSettled([
  getUser(userId),
  getOrders(userId),
  getNotifications(userId),
]);
```

`Promise.allSettled()` does not reject when one promise fails.

Instead, it returns something like:

```ts
[
  {
    status: "fulfilled",
    value: user,
  },

  {
    status: "fulfilled",
    value: orders,
  },

  {
    status: "rejected",
    reason: Error,
  },
];
```

This is useful when each task can succeed or fail independently.

However, it creates more manual handling.

Example:

```ts
const [userResult, ordersResult, notificationResult] =
  await Promise.allSettled([
    getUser(userId),
    getOrders(userId),
    getNotifications(userId),
  ]);

if (userResult.status === "rejected") {
  throw userResult.reason;
}

if (ordersResult.status === "rejected") {
  throw ordersResult.reason;
}

const notifications =
  notificationResult.status === "fulfilled"
    ? notificationResult.value
    : [];
```

This works, but for this specific case I would probably prefer:

```ts
Promise.all([
  requiredPromise,
  requiredPromise,
  optionalPromise.catch(() => fallback),
]);
```

because it expresses the requirements more clearly.

---

# Promise.all vs Promise.allSettled

| Behavior | `Promise.all()` | `Promise.allSettled()` |
|---|---|---|
| Reject when one fails | Yes | No |
| Fail-fast | Yes | No |
| Easy for all-required tasks | Yes | Less convenient |
| Easy for partial failures | Requires handling | Yes |
| Result handling | Simple | More verbose |

Use:

```text
Promise.all
```

when:

```text
All operations are required
```

or failures can be converted into fallbacks individually.

Use:

```text
Promise.allSettled
```

when:

```text
You need the result of every operation regardless of failures.
```

---

# Production Implementation

## Types

```ts
interface User {
  id: string;
  name: string;
}

interface Order {
  id: string;
  total: number;
}

interface Notification {
  id: string;
  message: string;
}

interface Dashboard {
  user: User;
  orders: Order[];
  notifications: Notification[];
}
```

---

# Custom Errors

```ts
export class NotFoundError extends Error {
  constructor(message: string) {
    super(message);

    this.name = "NotFoundError";
  }
}
```

---

# Service

```ts
export async function getDashboard(
  userId: string,
): Promise<Dashboard> {
  const user = await getUser(userId);

  if (!user) {
    throw new NotFoundError("User not found");
  }

  const ordersPromise = getOrders(userId);

  const notificationsPromise = getNotifications(userId)
    .catch((error: unknown) => {
      console.error(
        "Notification service failed",
        {
          userId,
          error,
        },
      );

      return [];
    });

  const [orders, notifications] = await Promise.all([
    ordersPromise,
    notificationsPromise,
  ]);

  return {
    user,
    orders,
    notifications,
  };
}
```

---

# Controller

```ts
import {
  NextFunction,
  Request,
  Response,
} from "express";

import { getDashboard } from "../services/dashboard.service";

export async function getDashboardController(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  try {
    const dashboard = await getDashboard(
      req.params.userId,
    );

    return res.status(200).json({
      data: dashboard,
    });
  } catch (error) {
    next(error);
  }
}
```

---

# Error Middleware

```ts
import {
  NextFunction,
  Request,
  Response,
} from "express";

import { NotFoundError } from "../errors/not-found.error";

export function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction,
) {
  if (error instanceof NotFoundError) {
    return res.status(404).json({
      message: error.message,
    });
  }

  console.error(error);

  return res.status(500).json({
    message: "Internal server error",
  });
}
```

---

# API

## Endpoint

```http
GET /dashboard/:userId
```

Example:

```http
GET /dashboard/123
```

Successful response:

```json
{
  "data": {
    "user": {
      "id": "123",
      "name": "John"
    },
    "orders": [
      {
        "id": "ORD-001",
        "total": 250000
      }
    ],
    "notifications": [
      {
        "id": "NOTIF-001",
        "message": "Your order has been shipped"
      }
    ]
  }
}
```

---

# Notification Service Failure

If:

```ts
getNotifications()
```

fails, the endpoint can still return:

```http
200 OK
```

```json
{
  "data": {
    "user": {
      "id": "123",
      "name": "John"
    },
    "orders": [],
    "notifications": []
  }
}
```

The error should still be logged for observability.

---

# User Not Found

If:

```ts
getUser()
```

returns no user:

```http
404 Not Found
```

Response:

```json
{
  "message": "User not found"
}
```

---

# Orders Failure

Orders are required.

If:

```ts
getOrders()
```

fails unexpectedly:

```http
500 Internal Server Error
```

The dashboard should not pretend that an empty order list is a successful result.

This distinction is important.

Bad implementation:

```ts
getOrders(userId).catch(() => []);
```

This would hide a real system failure.

An empty array means:

```text
The user has no orders.
```

But a failed database query means:

```text
We do not know whether the user has orders.
```

Those are different states.

---

# Code Review

Original implementation:

```ts
async function getDashboard(userId: string) {
  const results = [];

  const user = await getUser(userId);
  results.push(user);

  const orders = await getOrders(userId);
  results.push(orders);

  const notifications = await getNotifications(userId);
  results.push(notifications);

  return results;
}
```

There are several problems.

## 1. Independent Tasks Are Sequential

This:

```ts
await getOrders(userId);
await getNotifications(userId);
```

creates unnecessary waiting.

Independent calls should execute concurrently when appropriate.

---

## 2. Weak Return Structure

Returning:

```ts
[
  user,
  orders,
  notifications,
]
```

forces consumers to know:

```text
index 0 = user
index 1 = orders
index 2 = notifications
```

A better response is:

```ts
{
  user,
  orders,
  notifications,
}
```

This is clearer and safer.

---

## 3. Weak Type Inference

This:

```ts
const results = [];
```

does not clearly communicate the intended result structure.

The function should have an explicit return type:

```ts
Promise<Dashboard>
```

---

## 4. No Partial Failure Strategy

If notifications fail, the entire function currently fails.

That violates the requirement that notifications are optional.

---

## 5. Missing Business Error Handling

The function does not clearly distinguish:

```text
User not found
```

from:

```text
Database failure
```

These should produce different behavior.

---

# Improved Code

```ts
async function getDashboard(
  userId: string,
): Promise<Dashboard> {
  const user = await getUser(userId);

  if (!user) {
    throw new NotFoundError("User not found");
  }

  const [orders, notifications] = await Promise.all([
    getOrders(userId),

    getNotifications(userId)
      .catch((error: unknown) => {
        console.error(
          "Notification service unavailable",
          {
            userId,
            error,
          },
        );

        return [];
      }),
  ]);

  return {
    user,
    orders,
    notifications,
  };
}
```

---

# Is Parallel Execution Always Better?

No.

Concurrency should only be used when operations are independent.

Suppose:

```ts
const user = await createUser(input);

const profile = await createProfile({
  userId: user.id,
});

const session = await createSession({
  userId: user.id,
});
```

`createProfile()` requires:

```ts
user.id
```

Therefore it cannot start before `createUser()` finishes.

This is a real dependency.

---

# Another Sequential Example

Consider payment processing:

```ts
const order = await createOrder();

const payment = await createPayment({
  orderId: order.id,
});

await savePaymentReference(
  order.id,
  payment.id,
);
```

Each operation depends on the previous result.

Trying to execute all of them using:

```ts
Promise.all()
```

would be logically incorrect.

---

# Another Reason Not to Parallelize Everything

Even independent operations should not always be executed with unlimited concurrency.

Example:

```ts
const users = await getOneMillionUsers();

await Promise.all(
  users.map((user) =>
    sendEmail(user),
  ),
);
```

This could attempt to create one million concurrent operations.

Possible consequences:

```text
Memory exhaustion
Database connection exhaustion
API rate limiting
Network saturation
Downstream service overload
```

For large batches, concurrency should be controlled.

For example:

```text
10 workers
50 concurrent operations
Queue
Batch processing
```

depending on the requirement.

---

# Testing

## Test Cases

### 1. Everything Succeeds

```text
getUser          success
getOrders        success
getNotifications success
```

Expected:

```http
200 OK
```

with all three resources.

---

### 2. User Not Found

```text
getUser
→ null
```

Expected:

```http
404 Not Found
```

Orders and notifications should not be started if the implementation validates the user first.

---

### 3. Orders Fail

```text
getUser
→ success

getOrders
→ reject
```

Expected:

```http
500 Internal Server Error
```

because orders are required.

---

### 4. Notifications Fail

```text
getNotifications
→ reject
```

Expected:

```http
200 OK
```

with:

```json
{
  "notifications": []
}
```

---

### 5. Notification Service Is Slow

The test should verify that orders and notifications execute concurrently.

For example:

```text
orders        500 ms
notifications 400 ms
```

Expected combined time should be around:

```text
500 ms
```

rather than:

```text
900 ms
```

after user validation.

---

# Unit Test Example

```ts
import {
  describe,
  expect,
  it,
  vi,
} from "vitest";

describe("getDashboard", () => {
  it("returns complete dashboard data", async () => {
    vi.mocked(getUser).mockResolvedValue({
      id: "123",
      name: "John",
    });

    vi.mocked(getOrders).mockResolvedValue([
      {
        id: "order-1",
        total: 100000,
      },
    ]);

    vi.mocked(getNotifications).mockResolvedValue([
      {
        id: "notification-1",
        message: "Hello",
      },
    ]);

    const result = await getDashboard("123");

    expect(result.user.id).toBe("123");
    expect(result.orders).toHaveLength(1);
    expect(result.notifications).toHaveLength(1);
  });
});
```

---

# Notification Failure Test

```ts
it(
  "returns empty notifications when notification service fails",
  async () => {
    vi.mocked(getUser).mockResolvedValue({
      id: "123",
      name: "John",
    });

    vi.mocked(getOrders).mockResolvedValue([]);

    vi.mocked(getNotifications)
      .mockRejectedValue(
        new Error("Notification service unavailable"),
      );

    const result = await getDashboard("123");

    expect(result.notifications).toEqual([]);
  },
);
```

---

# User Not Found Test

```ts
it(
  "throws NotFoundError when user does not exist",
  async () => {
    vi.mocked(getUser).mockResolvedValue(null);

    await expect(
      getDashboard("unknown-user"),
    ).rejects.toThrow("User not found");

    expect(getOrders).not.toHaveBeenCalled();

    expect(
      getNotifications,
    ).not.toHaveBeenCalled();
  },
);
```

---

# Orders Failure Test

```ts
it(
  "fails when orders service fails",
  async () => {
    vi.mocked(getUser).mockResolvedValue({
      id: "123",
      name: "John",
    });

    vi.mocked(getOrders)
      .mockRejectedValue(
        new Error("Database unavailable"),
      );

    vi.mocked(getNotifications)
      .mockResolvedValue([]);

    await expect(
      getDashboard("123"),
    ).rejects.toThrow(
      "Database unavailable",
    );
  },
);
```

---

# Testing Parallel Execution

Avoid relying only on real execution time because timing-based tests can become flaky.

A conceptual test can verify that both promises are started before either is resolved.

Example:

```ts
it(
  "starts orders and notifications concurrently",
  async () => {
    const executionOrder: string[] = [];

    vi.mocked(getUser).mockResolvedValue({
      id: "123",
      name: "John",
    });

    vi.mocked(getOrders).mockImplementation(
      async () => {
        executionOrder.push(
          "orders-start",
        );

        await new Promise(
          (resolve) =>
            setTimeout(resolve, 50),
        );

        executionOrder.push(
          "orders-end",
        );

        return [];
      },
    );

    vi.mocked(
      getNotifications,
    ).mockImplementation(
      async () => {
        executionOrder.push(
          "notifications-start",
        );

        await new Promise(
          (resolve) =>
            setTimeout(resolve, 20),
        );

        executionOrder.push(
          "notifications-end",
        );

        return [];
      },
    );

    await getDashboard("123");

    expect(
      executionOrder.slice(0, 2),
    ).toEqual([
      "orders-start",
      "notifications-start",
    ]);
  },
);
```

The important point is that both operations start before the first one finishes.

---

# Project Structure

```text
fullstack-interview-lab/
└── day-02-async-dashboard/
    ├── src/
    │   ├── controllers/
    │   │   └── dashboard.controller.ts
    │   │
    │   ├── services/
    │   │   ├── dashboard.service.ts
    │   │   ├── user.service.ts
    │   │   ├── order.service.ts
    │   │   └── notification.service.ts
    │   │
    │   ├── errors/
    │   │   └── not-found.error.ts
    │   │
    │   ├── middleware/
    │   │   └── error-handler.ts
    │   │
    │   ├── routes/
    │   │   └── dashboard.route.ts
    │   │
    │   ├── app.ts
    │   └── server.ts
    │
    ├── tests/
    │   └── dashboard.service.test.ts
    │
    ├── package.json
    ├── tsconfig.json
    └── README.md
```

---

# How To Run

Install dependencies:

```bash
npm install
```

Run development server:

```bash
npm run dev
```

Run tests:

```bash
npm test
```

---

# Performance Comparison

## Before

```ts
const user = await getUser();

const orders = await getOrders();

const notifications =
  await getNotifications();
```

Approximate:

```text
300 + 500 + 400

≈ 1200 ms
```

---

## Fully Concurrent

```ts
await Promise.all([
  getUser(),
  getOrders(),
  getNotifications(),
]);
```

Approximate:

```text
max(300, 500, 400)

≈ 500 ms
```

---

## User Validation + Concurrent Dependencies

```ts
const user = await getUser();

await Promise.all([
  getOrders(),
  getNotifications(),
]);
```

Approximate:

```text
300 + max(500, 400)

≈ 800 ms
```

This demonstrates that performance optimization always has trade-offs.

---

# Key Engineering Lessons

## 1. `await` Does Not Automatically Mean Parallel

This:

```ts
await A();
await B();
```

means:

```text
Wait for A
then start B
```

not:

```text
Run A and B together
```

---

## 2. Promise Creation Matters

These promises start before the `await`:

```ts
const aPromise = A();
const bPromise = B();

const [a, b] =
  await Promise.all([
    aPromise,
    bPromise,
  ]);
```

Both operations are already in progress.

---

## 3. Do Not Hide Required Failures

This can be dangerous:

```ts
const orders =
  await getOrders(userId)
    .catch(() => []);
```

because:

```text
[]
```

could mean either:

```text
User has zero orders
```

or:

```text
Orders database failed
```

Those states should not be confused.

---

## 4. Optional Dependencies Can Degrade Gracefully

For optional functionality such as notifications:

```ts
getNotifications(userId)
  .catch(() => []);
```

can be reasonable when the business accepts degraded functionality.

The error should still be logged and monitored.

---

## 5. Concurrency Is Not the Same as Parallel CPU Execution

`Promise.all()` allows asynchronous operations to make progress concurrently.

For I/O operations such as:

```text
Database queries
HTTP requests
File I/O
Redis
External APIs
```

this can significantly reduce waiting time.

It does not mean JavaScript magically executes arbitrary CPU-heavy JavaScript functions in parallel on the same event loop.

---

# Better Interview Answer

> The endpoint takes around 1.2 seconds because each independent asynchronous operation is awaited sequentially. `getOrders()` only starts after `getUser()` finishes, and `getNotifications()` only starts after `getOrders()` finishes.
>
> If the operations are independent, I can start them concurrently. With latencies of 300, 500, and 400 milliseconds, the total time can approach the slowest request, around 500 milliseconds, instead of the sum of all three.
>
> However, I would not blindly put everything inside `Promise.all()`. `Promise.all()` rejects if any promise rejects, while our business requirements say that user and orders are required but notifications are optional.
>
> I would either handle the notification failure locally and return an empty notification list, or use `Promise.allSettled()` if I need to inspect every result independently.
>
> I would also consider whether the user should be validated before starting the other requests. Running all three concurrently gives lower latency, but validating the user first avoids unnecessary downstream requests when the user does not exist.
>
> The key is not simply using `Promise.all()`. The key is identifying dependencies, understanding failure requirements, and then deciding which operations should run sequentially and which can safely run concurrently.

---

# Interview Follow-Up Questions

## 1. What happens if notification takes 30 seconds?

Even though notification is optional, the current code still waits for it.

A better production system should introduce a timeout.

Conceptually:

```text
Notification API
↓
Maximum allowed wait
↓
Timeout
↓
Fallback []
```

This prevents an optional dependency from making the entire dashboard slow.

---

## 2. What happens if all dashboard services call the same database?

Running queries concurrently may reduce application waiting time but could increase database pressure.

For example:

```text
1000 dashboard requests
×
3 database queries

=
3000 queries
```

Concurrency should therefore be evaluated together with:

```text
Connection pool
Database load
Query performance
Traffic
```

---

## 3. Would you use `Promise.allSettled()` everywhere?

No.

It is useful when independent tasks can fail individually.

But if all operations are required, `Promise.all()` provides simpler fail-fast behavior.

---

## 4. What if notifications are completely non-essential?

Another architecture could remove notification fetching from the dashboard request entirely.

The frontend could call:

```http
GET /dashboard/:userId
```

and separately:

```http
GET /notifications
```

This allows the core dashboard to load without waiting for notifications.

Whether this is appropriate depends on frontend UX and API architecture.

---

## 5. What if the operation is CPU-heavy?

`Promise.all()` does not solve CPU-bound JavaScript work.

For CPU-intensive operations, solutions could include:

```text
Worker Threads
Separate workers
Background jobs
Separate services
```

depending on the workload.

---

# What I Learned

This challenge demonstrates that async optimization requires understanding three things:

```text
Dependency
Failure Requirement
Concurrency
```

Before using `Promise.all()`, ask:

```text
Can these operations run independently?

Does every result have to succeed?

Can some features degrade gracefully?

Could concurrency overload another system?
```

The decision process should be:

```text
Identify operations
↓
Identify dependencies
↓
Classify required vs optional
↓
Choose sequential/concurrent execution
↓
Define failure behavior
↓
Test
↓
Measure
```

---

# Git Workflow

Create branch:

```bash
git checkout -b feat/day-02-async-dashboard
```

Suggested commits:

```bash
git commit -m "feat: add dashboard aggregation service"
```

```bash
git commit -m "perf: execute independent dashboard requests concurrently"
```

```bash
git commit -m "feat: handle optional notification service failure"
```

```bash
git commit -m "test: add dashboard concurrency and failure tests"
```

---

# Progress Summary

```text
Day:
02

Topic:
Async/Await, Promise.all, and Partial Failure Handling

Difficulty:
Fundamental

Main Skill:
Asynchronous JavaScript

Secondary Skills:
Promise.all
Promise.allSettled
Error Handling
Performance
Graceful Degradation
Testing

Project:
Dashboard Aggregation API

GitHub Folder:
day-02-async-dashboard

Suggested Branch:
feat/day-02-async-dashboard

Suggested Commits:
feat: add dashboard aggregation service
perf: execute independent dashboard requests concurrently
feat: handle optional notification service failure
test: add dashboard concurrency and failure tests
```