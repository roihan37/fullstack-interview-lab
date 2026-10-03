# Day 06 - Safe Partial Update, Validation, and Mass Assignment Protection

## Problem

An application provides an endpoint for updating a user profile:

```http
PATCH /users/:userId
```

The client should only be allowed to update:

```text
name
phone
bio
```

The database model also contains protected fields:

```text
id
email
role
createdAt
updatedAt
```

The original implementation is:

```ts
app.patch("/users/:userId", async (req, res) => {
  const user = await prisma.user.update({
    where: {
      id: Number(req.params.userId),
    },
    data: req.body,
  });

  return res.status(200).json({
    data: user,
  });
});
```

This code appears simple, but it creates several security and design problems.

For example:

```http
PATCH /users/10
```

```json
{
  "name": "Mulia",
  "role": "ADMIN"
}
```

can potentially allow a regular user to modify a protected field.

Other problems include:

- Missing runtime validation
- Invalid route parameter handling
- Mass assignment
- Weak TypeScript safety
- Incorrect not-found handling
- Possible information leakage
- Returning unnecessary fields
- No clear API contract

---

# Interview Question

> What is wrong with passing `req.body` directly into the ORM update operation?

Follow-up:

> How would you design a safe partial-update endpoint?

---

# Interview Answer

I would never pass an untrusted request body directly into a database update.

This:

```ts
data: req.body
```

gives the client too much control over which database fields are modified.

Even if the frontend only sends:

```json
{
  "name": "Mulia"
}
```

the backend must not assume that every client will behave correctly.

A malicious client can send:

```json
{
  "role": "ADMIN"
}
```

or:

```json
{
  "email": "attacker@example.com"
}
```

The backend should define an explicit allowlist of fields that are permitted for this use case.

For a profile update endpoint:

```text
Allowed:
name
phone
bio

Forbidden:
id
email
role
createdAt
updatedAt
```

I would validate the route parameter, validate the request body, reject unknown fields, construct a safe update object, handle not-found errors separately from infrastructure failures, and return only fields that belong in the API response.

---

# Mass Assignment

Passing an entire request body directly into a model update is commonly described as a **mass assignment** problem.

Example:

```ts
await prisma.user.update({
  where: {
    id,
  },
  data: req.body,
});
```

The backend is effectively saying:

```text
Whatever fields the client sends
may be passed to the database.
```

This violates an important principle:

```text
The client supplies values.

The server decides
which fields may be modified.
```

---

# Dangerous Example

Suppose the User model is:

```prisma
model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  phone     String?
  bio       String?
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

The legitimate UI sends:

```json
{
  "name": "Mulia Rahman"
}
```

But an attacker manually sends:

```json
{
  "name": "Mulia Rahman",
  "role": "ADMIN"
}
```

If the backend blindly uses:

```ts
data: req.body
```

the client may modify fields that the profile update feature was never supposed to expose.

---

# PATCH vs PUT

For this endpoint, `PATCH` is appropriate because the client updates only part of the resource.

Current user:

```json
{
  "name": "Mulia",
  "phone": "0812345",
  "bio": "Software Developer"
}
```

Client wants to update only:

```json
{
  "bio": "Full Stack Developer"
}
```

Using:

```http
PATCH /users/10
```

communicates:

```text
Update part of this resource.
```

The fields that are not sent remain unchanged.

---

# PUT

Conceptually, `PUT` is usually associated with replacing the complete representation of a resource.

For example:

```http
PUT /users/10
```

might conceptually expect:

```json
{
  "name": "Mulia",
  "phone": "0812345",
  "bio": "Full Stack Developer"
}
```

rather than only one field.

Actual API conventions can vary, but for a partial profile update, `PATCH` clearly matches the intended semantics.

---

# Validation Schema

A safe endpoint should validate:

- Allowed fields
- Data types
- Value constraints
- Unknown fields
- Empty request body

Using Zod:

```ts
import { z } from "zod";

export const updateProfileSchema = z
  .object({
    name: z
      .string()
      .trim()
      .min(
        1,
        "Name cannot be empty",
      )
      .max(
        100,
        "Name is too long",
      )
      .optional(),

    phone: z
      .string()
      .trim()
      .min(
        6,
        "Phone number is too short",
      )
      .max(
        20,
        "Phone number is too long",
      )
      .nullable()
      .optional(),

    bio: z
      .string()
      .trim()
      .max(
        500,
        "Bio is too long",
      )
      .nullable()
      .optional(),
  })
  .strict()
  .refine(
    (data) =>
      Object.keys(data).length > 0,
    {
      message:
        "At least one field must be provided",
    },
  );

export type UpdateProfileInput =
  z.infer<
    typeof updateProfileSchema
  >;
```

---

# Why `.strict()` Matters

Without strict object validation, unknown fields may be ignored depending on schema behavior.

For a sensitive update endpoint, I prefer explicitly rejecting unexpected fields.

Request:

```json
{
  "name": "Mulia",
  "role": "ADMIN"
}
```

should fail validation instead of silently accepting part of it.

This helps expose incorrect or malicious client behavior.

---

# Why Reject Unknown Fields?

There are two possible strategies:

```text
1. Ignore unknown fields

2. Reject the entire request
```

For a security-sensitive update API, rejecting the request is generally clearer.

Example:

```json
{
  "name": "Mulia",
  "role": "ADMIN"
}
```

Response:

```http
400 Bad Request
```

This makes it explicit that `role` does not belong to this API contract.

Silently ignoring `role` may hide client-side bugs.

---

# Empty Body Validation

Request:

```json
{}
```

should not silently perform an empty update.

A useful response is:

```http
400 Bad Request
```

```json
{
  "message": "Invalid request data",
  "errors": {
    "form": [
      "At least one field must be provided"
    ]
  }
}
```

---

# `undefined` vs `null`

These have different meanings in a partial-update API.

Suppose current data is:

```json
{
  "phone": "0812345",
  "bio": "Software Developer"
}
```

Request:

```json
{
  "bio": "Full Stack Developer"
}
```

does not contain `phone`.

Conceptually:

```text
phone = undefined
```

means:

```text
Do not modify phone.
```

The value remains:

```text
0812345
```

---

But this request:

```json
{
  "phone": null
}
```

can mean:

```text
Explicitly clear phone.
```

After update:

```json
{
  "phone": null
}
```

Therefore:

```text
undefined
→ field was not provided
→ do not modify

null
→ field was explicitly cleared
```

This distinction is important for partial updates.

---

# Route Parameter Validation

This is unsafe:

```ts
const id =
  Number(req.params.userId);
```

because:

```ts
Number("abc");
```

returns:

```ts
NaN
```

`NaN` is still a JavaScript `number` type, but it is not a valid user ID.

The route parameter should be validated before it reaches Prisma.

---

# Route Param Schema

```ts
import { z } from "zod";

export const userIdParamSchema =
  z.object({
    userId: z.coerce
      .number()
      .int()
      .positive(),
  });
```

Then:

```ts
const parsedParams =
  userIdParamSchema.safeParse(
    req.params,
  );

if (!parsedParams.success) {
  return res.status(400).json({
    message: "Invalid user ID",
  });
}
```

---

# User Not Found vs Database Failure

These represent different problems.

## User does not exist

Request:

```http
PATCH /users/999999
```

If user `999999` does not exist:

```http
404 Not Found
```

is appropriate.

The requested resource does not exist.

---

## Database unavailable

If PostgreSQL is unavailable:

```text
Database connection failed
```

the problem is not that the user does not exist.

It is an infrastructure failure.

Return:

```http
500 Internal Server Error
```

or another appropriate 5xx response depending on the architecture.

The important point is:

```text
Not found
≠
Database failure
```

---

# Prisma Not-Found Handling

Prisma may throw a known request error when an update cannot find the required record.

For example, an application can map a known not-found condition to:

```http
404 Not Found
```

rather than converting every database exception to:

```http
500 Internal Server Error
```

A production implementation should inspect known Prisma errors separately from unknown errors.

---

# Custom Errors

```ts
export class NotFoundError
  extends Error {
  statusCode = 404;

  constructor(
    message: string,
  ) {
    super(message);

    this.name =
      "NotFoundError";
  }
}
```

---

# Service Implementation

```ts
import {
  Prisma,
} from "@prisma/client";

import {
  prisma,
} from "../lib/prisma";

import {
  UpdateProfileInput,
} from "../schemas/user.schema";

import {
  NotFoundError,
} from "../errors/not-found.error";

export async function updateUserProfile(
  userId: number,
  input: UpdateProfileInput,
) {
  const updateData: Prisma.UserUpdateInput =
    {};

  if (
    input.name !== undefined
  ) {
    updateData.name =
      input.name;
  }

  if (
    input.phone !== undefined
  ) {
    updateData.phone =
      input.phone;
  }

  if (
    input.bio !== undefined
  ) {
    updateData.bio =
      input.bio;
  }

  try {
    return await prisma.user.update({
      where: {
        id: userId,
      },

      data: updateData,

      select: {
        id: true,
        name: true,
        phone: true,
        bio: true,
        updatedAt: true,
      },
    });
  } catch (error: unknown) {
    if (
      error instanceof
        Prisma.PrismaClientKnownRequestError &&
      error.code === "P2025"
    ) {
      throw new NotFoundError(
        "User not found",
      );
    }

    throw error;
  }
}
```

---

# Why Build `updateData` Explicitly?

Instead of:

```ts
data: req.body
```

we explicitly decide which fields may enter the database layer.

```ts
if (
  input.name !== undefined
) {
  updateData.name =
    input.name;
}
```

This protects the service from accidentally accepting future request fields.

Imagine someone later changes the validation schema and adds:

```ts
role?: Role;
```

With:

```ts
data: validatedBody
```

that field could automatically reach the update operation.

An explicit transformation provides another boundary:

```text
HTTP input
↓
Validated input
↓
Explicit database update object
```

This is especially useful for sensitive mutations.

---

# Is `data: validatedBody` Always Wrong?

No.

If the runtime validation schema:

```text
strictly controls fields
```

and the schema exactly matches the database update contract, then:

```ts
data: validatedBody
```

can be acceptable.

For example:

```ts
const input =
  updateProfileSchema.parse(
    req.body,
  );

await prisma.user.update({
  where: {
    id,
  },

  data: input,
});
```

is much safer than:

```ts
data: req.body
```

because the input has already passed an allowlist.

However, explicit mapping can provide stronger separation between:

```text
API contract
and
database model
```

I would choose based on complexity and sensitivity.

---

# Controller

```ts
import {
  NextFunction,
  Request,
  Response,
} from "express";

import {
  updateProfileSchema,
  userIdParamSchema,
} from "../schemas/user.schema";

import {
  updateUserProfile,
} from "../services/user.service";

export async function
updateUserProfileController(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  try {
    const parsedParams =
      userIdParamSchema.safeParse(
        req.params,
      );

    if (!parsedParams.success) {
      return res
        .status(400)
        .json({
          message:
            "Invalid user ID",
        });
    }

    const parsedBody =
      updateProfileSchema.safeParse(
        req.body,
      );

    if (!parsedBody.success) {
      return res
        .status(400)
        .json({
          message:
            "Invalid request data",

          errors:
            parsedBody.error
              .flatten(),
        });
    }

    const user =
      await updateUserProfile(
        parsedParams.data
          .userId,

        parsedBody.data,
      );

    return res
      .status(200)
      .json({
        data: user,
      });
  } catch (error) {
    next(error);
  }
}
```

---

# Central Error Handler

```ts
import {
  NextFunction,
  Request,
  Response,
} from "express";

import {
  NotFoundError,
} from "../errors/not-found.error";

export function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction,
) {
  if (
    error instanceof
    NotFoundError
  ) {
    return res
      .status(404)
      .json({
        message:
          error.message,
      });
  }

  console.error(error);

  return res
    .status(500)
    .json({
      message:
        "Internal server error",
    });
}
```

---

# Why Not Return Raw Errors?

Bad:

```ts
catch (error) {
  return res
    .status(500)
    .json({
      error,
    });
}
```

A raw ORM error can contain internal information such as:

```text
Database schema details
Column names
Internal query information
Stack traces
File paths
ORM internals
```

The client should normally receive:

```json
{
  "message": "Internal server error"
}
```

while detailed information is stored in internal logs.

---

# Response Design

Request:

```http
PATCH /users/10
```

```json
{
  "name": "Mulia Rahman",
  "bio": "Full Stack Developer"
}
```

Response:

```http
200 OK
```

```json
{
  "data": {
    "id": 10,
    "name": "Mulia Rahman",
    "phone": "0812345",
    "bio": "Full Stack Developer",
    "updatedAt": "2026-10-03T01:00:00.000Z"
  }
}
```

The endpoint does not need to expose:

```text
role
email
createdAt
```

unless they are part of the API contract.

---

# Why Use Prisma `select`?

Instead of:

```ts
const user =
  await prisma.user.update(...);

return user;
```

use:

```ts
select: {
  id: true,
  name: true,
  phone: true,
  bio: true,
  updatedAt: true,
}
```

Benefits:

- Clear response contract
- Lower risk of leaking sensitive fields
- Less unnecessary data
- Easier maintenance
- API is less coupled to database structure

---

# TypeScript: Why `any` Is Dangerous

Bad:

```ts
async function updateUser(
  userId: string,
  body: any,
) {
```

With:

```ts
body: any
```

TypeScript effectively stops helping.

This becomes valid to TypeScript:

```ts
body.role;
body.password;
body.randomField;
body.foo.bar.baz;
```

even if those properties do not belong to the use case.

---

# Better Type

```ts
type UpdateProfileInput = {
  name?: string;
  phone?: string | null;
  bio?: string | null;
};
```

Now TypeScript helps ensure the application layer only works with expected fields.

---

# But TypeScript Is Not Runtime Validation

This is very important.

TypeScript exists during development and compilation.

An HTTP request comes from outside the application at runtime.

An attacker can still send:

```json
{
  "role": "ADMIN"
}
```

even if TypeScript declares:

```ts
type UpdateProfileInput = {
  name?: string;
};
```

Therefore:

```text
TypeScript
→ compile-time safety

Zod
→ runtime validation
```

Both solve different problems.

---

# Code Review

Original:

```ts
async function updateUser(
  userId: string,
  body: any,
) {
  const id =
    Number(userId);

  const user =
    await prisma.user.update({
      where: {
        id,
      },

      data: {
        ...body,
      },
    });

  return user;
}
```

There are several issues.

---

## Problem 1: `any`

```ts
body: any
```

removes meaningful TypeScript protection.

---

## Problem 2: Invalid ID Conversion

```ts
Number("abc")
```

becomes:

```ts
NaN
```

The route parameter must be explicitly validated.

---

## Problem 3: Mass Assignment

```ts
data: {
  ...body,
}
```

allows client-controlled fields to flow directly into the ORM.

---

## Problem 4: Missing Runtime Validation

There is no check for:

```text
Empty name
Invalid phone
Unknown fields
Empty body
```

---

## Problem 5: Protected Fields

The code does not prevent:

```json
{
  "role": "ADMIN"
}
```

or other protected fields.

---

## Problem 6: Missing Not-Found Handling

A missing user can become an ORM exception rather than a clean:

```http
404 Not Found
```

---

## Problem 7: Returns Entire User

```ts
return user;
```

couples the API response to the database representation.

---

## Problem 8: Missing Error Handling

Unexpected database failures are not categorized or handled centrally.

---

# Protected Fields

This profile endpoint must never update:

```text
id
email
role
createdAt
updatedAt
```

For example:

```json
{
  "role": "ADMIN"
}
```

should fail.

A role modification belongs to a different business capability.

---

# Separate Role Management

If administrators genuinely need to change user roles, I would prefer a dedicated use case such as:

```http
PATCH /admin/users/:userId/role
```

Request:

```json
{
  "role": "ADMIN"
}
```

This allows the backend to apply separate:

```text
Authentication
Authorization
Validation
Audit logging
Business rules
```

instead of mixing profile updates with administrative privilege changes.

---

# Why Separate Endpoints?

These operations have different meanings:

```text
Update own profile
```

and:

```text
Change another user's authorization role
```

The second action is security-sensitive.

Separating them makes:

- Authorization clearer
- Auditing easier
- Validation simpler
- API intent explicit

---

# Concurrent Partial Updates

Suppose current data is:

```json
{
  "name": "Mulia",
  "bio": "Developer"
}
```

Two requests arrive:

### Request A

```json
{
  "name": "Mulia Rahman"
}
```

### Request B

```json
{
  "bio": "Full Stack Developer"
}
```

Because both are partial updates, each updates only its own field.

Final result can be:

```json
{
  "name": "Mulia Rahman",
  "bio": "Full Stack Developer"
}
```

This is safer than replacing the entire resource with stale copies.

---

# Lost Update Risk

However, concurrent updates can still create problems if both modify the same field.

Request A:

```json
{
  "bio": "Frontend Developer"
}
```

Request B:

```json
{
  "bio": "Backend Developer"
}
```

Whichever update executes last may overwrite the previous value.

This is known as a lost-update style problem.

More advanced solutions may include:

```text
Optimistic locking
Version fields
updatedAt checks
Transactions
Pessimistic locking
```

depending on the business requirement.

That is not necessary for every profile endpoint, but it is important to understand that partial update does not eliminate all concurrency problems.

---

# Does Redis Solve This?

No.

Redis does not solve:

```text
Mass assignment
Invalid user input
Authorization problems
Protected fields
Incorrect status codes
Bad TypeScript
```

Those are application design and security problems.

Adding:

```text
Redis
```

would not make:

```ts
data: req.body
```

safe.

---

# Status Codes

| Scenario | Status |
|---|---:|
| Successful update | `200 OK` |
| Invalid route parameter | `400 Bad Request` |
| Invalid request body | `400 Bad Request` |
| Empty update body | `400 Bad Request` |
| Forbidden/unknown field in this contract | `400 Bad Request` |
| User not found | `404 Not Found` |
| Unexpected database failure | `500 Internal Server Error` |

If authorization is later introduced, attempting to update a resource the authenticated user is not allowed to modify may use:

```http
403 Forbidden
```

depending on the API's security policy.

---

# API Example

## Update Name

```http
PATCH /users/10
Content-Type: application/json
```

```json
{
  "name": "Mulia Rahman"
}
```

Response:

```http
200 OK
```

---

# Clear Phone

```http
PATCH /users/10
```

```json
{
  "phone": null
}
```

Result:

```json
{
  "phone": null
}
```

---

# Do Not Modify Phone

```http
PATCH /users/10
```

```json
{
  "bio": "Software Engineer"
}
```

The absence of `phone` means:

```text
Leave phone unchanged.
```

---

# Invalid Protected Field

```http
PATCH /users/10
```

```json
{
  "role": "ADMIN"
}
```

Expected:

```http
400 Bad Request
```

---

# Invalid ID

```http
PATCH /users/abc
```

Expected:

```http
400 Bad Request
```

---

# Missing User

```http
PATCH /users/999999
```

Expected:

```http
404 Not Found
```

---

# Testing

Important test cases:

```text
Successful single-field update
Multiple-field update
Empty body
Protected field
Unknown field
Invalid name
Invalid user ID
User not found
Nullable fields
Omitted fields remain unchanged
Unexpected database failure
```

---

# Happy Path Test

```ts
it(
  "updates user name",
  async () => {
    const response =
      await request(app)
        .patch("/users/10")
        .send({
          name:
            "Mulia Rahman",
        });

    expect(
      response.status,
    ).toBe(200);

    expect(
      response.body.data
        .name,
    ).toBe(
      "Mulia Rahman",
    );
  },
);
```

---

# Multiple Fields Test

```ts
it(
  "updates multiple allowed fields",
  async () => {
    const response =
      await request(app)
        .patch("/users/10")
        .send({
          name:
            "Mulia Rahman",
          bio:
            "Full Stack Developer",
        });

    expect(
      response.status,
    ).toBe(200);

    expect(
      response.body.data
        .name,
    ).toBe(
      "Mulia Rahman",
    );

    expect(
      response.body.data
        .bio,
    ).toBe(
      "Full Stack Developer",
    );
  },
);
```

---

# Protected Field Test

```ts
it(
  "rejects role modification",
  async () => {
    const response =
      await request(app)
        .patch("/users/10")
        .send({
          role: "ADMIN",
        });

    expect(
      response.status,
    ).toBe(400);
  },
);
```

This test is important because it verifies the mass-assignment protection.

---

# Mixed Allowed and Forbidden Fields

Request:

```json
{
  "name": "Mulia",
  "role": "ADMIN"
}
```

Expected:

```http
400 Bad Request
```

The backend should not update `name` while silently ignoring `role`.

Rejecting the request creates a clearer contract.

---

# Empty Body Test

```ts
it(
  "rejects empty update body",
  async () => {
    const response =
      await request(app)
        .patch("/users/10")
        .send({});

    expect(
      response.status,
    ).toBe(400);
  },
);
```

---

# Invalid ID Test

```ts
it(
  "rejects invalid user ID",
  async () => {
    const response =
      await request(app)
        .patch(
          "/users/abc",
        )
        .send({
          name: "Mulia",
        });

    expect(
      response.status,
    ).toBe(400);
  },
);
```

---

# User Not Found Test

```ts
it(
  "returns 404 when user does not exist",
  async () => {
    const response =
      await request(app)
        .patch(
          "/users/999999",
        )
        .send({
          name: "Mulia",
        });

    expect(
      response.status,
    ).toBe(404);
  },
);
```

---

# Nullable Field Test

Suppose current phone is:

```text
0812345
```

Request:

```json
{
  "phone": null
}
```

Test:

```ts
expect(
  response.body.data.phone,
).toBeNull();
```

This verifies the API can intentionally clear optional information.

---

# Omitted Field Test

Suppose:

```text
phone = 0812345
```

Request:

```json
{
  "bio": "Developer"
}
```

Expected:

```text
bio changes
phone remains 0812345
```

Example:

```ts
it(
  "does not modify omitted fields",
  async () => {
    const response =
      await request(app)
        .patch("/users/10")
        .send({
          bio:
            "Developer",
        });

    expect(
      response.body.data
        .phone,
    ).toBe(
      "0812345",
    );
  },
);
```

---

# Database Failure Test

If the database layer throws an unexpected error:

```text
Database unavailable
```

expected external response:

```http
500 Internal Server Error
```

```json
{
  "message": "Internal server error"
}
```

Internal details should be logged but not exposed.

---

# Project Structure

```text
fullstack-interview-lab/
└── day-06-safe-partial-update/
    ├── prisma/
    │   └── schema.prisma
    │
    ├── src/
    │   ├── controllers/
    │   │   └── user.controller.ts
    │   │
    │   ├── services/
    │   │   └── user.service.ts
    │   │
    │   ├── schemas/
    │   │   └── user.schema.ts
    │   │
    │   ├── routes/
    │   │   └── user.route.ts
    │   │
    │   ├── errors/
    │   │   └── not-found.error.ts
    │   │
    │   ├── middleware/
    │   │   └── error-handler.ts
    │   │
    │   ├── lib/
    │   │   └── prisma.ts
    │   │
    │   ├── app.ts
    │   └── server.ts
    │
    ├── tests/
    │   └── user-update.test.ts
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
Zod
Vitest
Supertest
```

---

# How To Run

Install dependencies:

```bash
npm install
```

Configure environment:

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

> I would not pass `req.body` directly into Prisma because HTTP request data is untrusted and the client should not control which model fields can be changed.
>
> The current implementation is vulnerable to mass assignment. For example, a user could send `role: "ADMIN"` even though the profile endpoint should only update `name`, `phone`, and `bio`.
>
> I would first validate the route parameter as a positive integer. Then I would validate the body using an explicit schema that allows only the permitted fields and rejects unknown properties. Because this is a partial update, the fields should be optional, but I would still require at least one field to be provided.
>
> I would distinguish between an omitted field and a field explicitly set to `null`. An omitted field means it should remain unchanged, while `null` can represent intentionally clearing an optional value.
>
> After validation, I would create a safe update payload rather than forwarding the raw request body. I would map a missing user to `404 Not Found`, while unexpected database failures should remain `500 Internal Server Error`.
>
> Finally, I would return only the fields required by the API contract using Prisma `select`, rather than returning the entire database model.
>
> TypeScript helps with compile-time correctness, but it does not validate untrusted HTTP data at runtime, so a runtime validator such as Zod is still necessary.

---

# Interview Follow-Up Questions

## 1. Is `data: validatedBody` safe?

It can be safe if the validation schema strictly defines the complete allowed update contract.

However, explicit mapping can provide additional separation between the HTTP request model and the database model.

---

## 2. Why distinguish `null` and `undefined`?

Because they can represent different business intentions.

```text
undefined
→ do not modify

null
→ explicitly remove value
```

---

## 3. Should role updates use the same endpoint?

Usually not if role changes have different authorization requirements.

A dedicated administrative operation makes the security boundary clearer.

---

## 4. What happens with concurrent updates?

Updates to different fields may coexist safely.

Updates to the same field can result in the last write winning.

More sensitive workflows may require optimistic locking or another concurrency strategy.

---

## 5. Why doesn't Redis solve mass assignment?

Because mass assignment is caused by unsafe application input handling.

Caching does not determine which fields the user is allowed to update.

---

# What I Learned

The most important lesson from this challenge is:

```text
Request body
is untrusted input.
```

Never design an update flow as:

```text
Client JSON
↓
Database
```

Prefer:

```text
Client
↓
Route parameter validation
↓
Body validation
↓
Allowlist
↓
Business rules
↓
Safe update object
↓
Database
↓
Safe response
```

Important concepts:

```text
PATCH
Partial update
Mass assignment
Allowlist
Protected fields
Runtime validation
TypeScript
null vs undefined
Prisma errors
HTTP status codes
Safe API responses
```

---

# Interview Key Takeaway

A weak implementation is:

```ts
await prisma.user.update({
  where: {
    id: Number(
      req.params.userId,
    ),
  },
  data: req.body,
});
```

A stronger engineering approach is:

```text
Validate
↓
Authorize
↓
Whitelist
↓
Transform
↓
Update
↓
Handle errors
↓
Return safe fields
```

The backend owns the update contract.

The client only supplies values for fields the backend explicitly permits.

---

# Git Workflow

Create branch:

```bash
git checkout -b feat/day-06-safe-partial-update
```

Suggested commits:

```bash
git commit -m "feat: add user profile update validation"
```

```bash
git commit -m "fix: prevent mass assignment in profile updates"
```

```bash
git commit -m "feat: handle user not found errors"
```

```bash
git commit -m "refactor: restrict user update response fields"
```

```bash
git commit -m "test: add secure profile update integration tests"
```

---

# Progress Summary

```text
Day:
06

Topic:
Safe Partial Update and Mass Assignment Protection

Difficulty:
Fundamental

Main Skill:
Secure REST API Update Design

Secondary Skills:
PATCH
Zod
TypeScript
Prisma
Mass Assignment
Input Validation
Error Handling
API Security

Project:
User Profile Update API

GitHub Folder:
day-06-safe-partial-update

Suggested Branch:
feat/day-06-safe-partial-update

Suggested Commits:
feat: add user profile update validation
fix: prevent mass assignment in profile updates
feat: handle user not found errors
refactor: restrict user update response fields
test: add secure profile update integration tests
```