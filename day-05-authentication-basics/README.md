# Day 05 - Authentication Basics, Password Hashing, Validation, and Error Handling

## Problem

An internal application needs basic authentication using:

```text
Node.js
Express.js
TypeScript
PostgreSQL
Prisma
bcrypt
Zod
```

The original registration implementation stores passwords directly:

```ts
app.post("/register", async (req, res) => {
  const user = await prisma.user.create({
    data: {
      email: req.body.email,
      password: req.body.password,
      name: req.body.name,
    },
  });

  return res.status(201).json(user);
});
```

The original login implementation also compares passwords directly:

```ts
app.post("/login", async (req, res) => {
  const user = await prisma.user.findUnique({
    where: {
      email: req.body.email,
    },
  });

  if (
    !user ||
    user.password !== req.body.password
  ) {
    return res.status(401).json({
      message: "Invalid credentials",
    });
  }

  return res.status(200).json({
    user,
  });
});
```

This implementation has several security and design problems:

- Passwords are stored as plaintext.
- Passwords are returned in API responses.
- Request validation is missing.
- Duplicate emails are not handled properly.
- Login compares plaintext passwords.
- Internal errors may not be handled safely.
- Authentication logic is mixed directly into the HTTP route.

---

# Interview Question

> What security and design problems do you see in this authentication implementation, and how would you redesign it?

Follow-up:

> Explain the complete flow from registration until a user successfully logs in.

---

# Main Problems

## 1. Plaintext Password Storage

This is dangerous:

```ts
password: req.body.password
```

If the database contains:

```text
email               password
---------------------------------
mulia@example.com   Secret123
andi@example.com    password123
```

and the database is leaked, attackers immediately obtain valid passwords.

This becomes even more dangerous because users frequently reuse passwords across multiple services.

Passwords should never be stored in recoverable plaintext form.

---

# Hashing vs Encryption

Password storage should use **hashing**, not reversible encryption.

## Encryption

Encryption is designed so data can later be decrypted using a key.

Conceptually:

```text
Plaintext
↓
Encryption + Key
↓
Ciphertext

Ciphertext
↓
Decryption + Key
↓
Plaintext
```

If an attacker obtains the encryption key, the original passwords can potentially be recovered.

---

## Hashing

Password hashing is designed to be one-way.

Conceptually:

```text
Password
↓
Hashing algorithm
↓
Hash
```

We do not need to recover the user's original password.

During login, we only need to verify:

```text
Does this entered password match the stored hash?
```

Therefore passwords should be hashed.

---

# bcrypt.hash()

During registration:

```ts
const passwordHash =
  await bcrypt.hash(password, 12);
```

`bcrypt.hash()` converts the password into a hash.

Example:

```text
Secret123
```

may become something similar to:

```text
$2b$12$...
```

The database stores the hash instead of:

```text
Secret123
```

---

# bcrypt.compare()

During login, we do not write:

```ts
user.password === password
```

Instead:

```ts
const passwordMatches =
  await bcrypt.compare(
    password,
    user.password,
  );
```

`bcrypt.compare()` receives:

```text
Plain password entered by user
+
Stored bcrypt hash
```

and returns:

```ts
true
```

or:

```ts
false
```

---

# Registration Flow

A good registration flow is:

```text
POST /auth/register
        ↓
Validate request
        ↓
Normalize input
        ↓
Check business rules
        ↓
Hash password
        ↓
Create user
        ↓
Database unique constraint
        ↓
Remove password from response
        ↓
201 Created
```

More concretely:

```text
Client
↓
Controller
↓
Validation
↓
Service
↓
bcrypt.hash()
↓
Prisma
↓
PostgreSQL
↓
Safe response
```

---

# Login Flow

A basic login flow is:

```text
POST /auth/login
        ↓
Validate request
        ↓
Normalize email
        ↓
Find user by email
        ↓
User exists?
        ↓
bcrypt.compare()
        ↓
Password valid?
        ↓
Authenticated
        ↓
Return safe response
```

At this stage, this challenge focuses on credential verification.

JWT and refresh-token architecture can be introduced later.

---

# API Design

## Register

```http
POST /auth/register
```

Request:

```json
{
  "name": "Mulia",
  "email": "mulia@example.com",
  "password": "Secret123"
}
```

Successful response:

```http
201 Created
```

```json
{
  "data": {
    "id": 1,
    "name": "Mulia",
    "email": "mulia@example.com",
    "createdAt": "2026-10-02T10:00:00.000Z"
  }
}
```

The password hash must not be returned.

---

# Login

```http
POST /auth/login
```

Request:

```json
{
  "email": "mulia@example.com",
  "password": "Secret123"
}
```

Successful response:

```http
200 OK
```

```json
{
  "data": {
    "id": 1,
    "name": "Mulia",
    "email": "mulia@example.com"
  }
}
```

---

# Invalid Credentials

For both:

```text
Email does not exist
```

and:

```text
Password is incorrect
```

return the same external response:

```http
401 Unauthorized
```

```json
{
  "message": "Invalid email or password"
}
```

---

# Why Not Return Different Login Errors?

Avoid responses such as:

```json
{
  "message": "Email is not registered"
}
```

and:

```json
{
  "message": "Wrong password"
}
```

because this can allow an attacker to discover which email addresses are registered.

For example:

```text
attacker@example.com
→ "Email is not registered"

real-user@example.com
→ "Wrong password"
```

The attacker now knows:

```text
real-user@example.com
```

is a valid account.

This technique is often called user enumeration.

Externally, return:

```text
Invalid email or password
```

while internally logging errors appropriately.

---

# Validation

Register requirements:

```text
name
→ required
→ trimmed
→ cannot be empty

email
→ required
→ valid email format
→ normalized
→ unique

password
→ required
→ minimum 8 characters
```

---

# Zod Validation

```ts
import { z } from "zod";

export const registerSchema =
  z.object({
    name: z
      .string()
      .trim()
      .min(
        1,
        "Name is required",
      ),

    email: z
      .string()
      .trim()
      .email(
        "Invalid email format",
      )
      .transform(
        (value) =>
          value.toLowerCase(),
      ),

    password: z
      .string()
      .min(
        8,
        "Password must contain at least 8 characters",
      ),
  });

export const loginSchema =
  z.object({
    email: z
      .string()
      .trim()
      .email()
      .transform(
        (value) =>
          value.toLowerCase(),
      ),

    password: z
      .string()
      .min(1),
  });

export type RegisterInput =
  z.infer<
    typeof registerSchema
  >;

export type LoginInput =
  z.infer<
    typeof loginSchema
  >;
```

---

# Why Normalize Email?

Consider:

```text
MULIA@example.com
```

and:

```text
mulia@example.com
```

For application login purposes, we generally want these to identify the same account.

Therefore:

```ts
email
  .trim()
  .toLowerCase();
```

is useful before storing and querying the email.

Normalization should be applied consistently during registration and login.

---

# Prisma Schema

```prisma
model User {
  id           Int      @id @default(autoincrement())
  name         String
  email        String   @unique
  passwordHash String
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}
```

Using:

```text
passwordHash
```

instead of:

```text
password
```

makes the database model more explicit.

It communicates that the application never intends to store the original password.

---

# Register Service

```ts
import bcrypt from "bcrypt";
import {
  Prisma,
} from "@prisma/client";

import {
  prisma,
} from "../lib/prisma";

import {
  RegisterInput,
} from "../schemas/auth.schema";

import {
  ConflictError,
} from "../errors/conflict.error";

export async function registerUser(
  input: RegisterInput,
) {
  const passwordHash =
    await bcrypt.hash(
      input.password,
      12,
    );

  try {
    const user =
      await prisma.user.create({
        data: {
          name: input.name,
          email: input.email,
          passwordHash,
        },

        select: {
          id: true,
          name: true,
          email: true,
          createdAt: true,
        },
      });

    return user;
  } catch (error: unknown) {
    if (
      error instanceof
        Prisma.PrismaClientKnownRequestError &&
      error.code === "P2002"
    ) {
      throw new ConflictError(
        "Email is already registered",
      );
    }

    throw error;
  }
}
```

---

# Why Rely on the Database Unique Constraint?

It is possible to do this first:

```ts
const existingUser =
  await prisma.user.findUnique({
    where: {
      email,
    },
  });
```

and then:

```ts
if (existingUser) {
  throw new ConflictError(
    "Email already exists",
  );
}
```

This may improve readability or avoid unnecessary hashing.

However, it cannot be the only protection.

Imagine two registration requests arrive at almost exactly the same time:

```text
Request A
→ check email
→ not found

Request B
→ check email
→ not found
```

Then:

```text
Request A → create
Request B → create
```

Without:

```prisma
email String @unique
```

both could theoretically succeed.

Therefore the database unique constraint is the final source of truth.

A production implementation may use both:

```text
Application check
+
Database unique constraint
```

but the database constraint is mandatory.

---

# Optional Pre-check Version

```ts
const existingUser =
  await prisma.user.findUnique({
    where: {
      email: input.email,
    },

    select: {
      id: true,
    },
  });

if (existingUser) {
  throw new ConflictError(
    "Email is already registered",
  );
}
```

This can be useful before performing the relatively expensive password hash.

Still keep:

```prisma
@unique
```

and still handle Prisma `P2002`.

---

# Login Service

```ts
import bcrypt from "bcrypt";

import {
  prisma,
} from "../lib/prisma";

import {
  LoginInput,
} from "../schemas/auth.schema";

import {
  UnauthorizedError,
} from "../errors/unauthorized.error";

export async function loginUser(
  input: LoginInput,
) {
  const user =
    await prisma.user.findUnique({
      where: {
        email: input.email,
      },

      select: {
        id: true,
        name: true,
        email: true,
        passwordHash: true,
      },
    });

  if (!user) {
    throw new UnauthorizedError(
      "Invalid email or password",
    );
  }

  const passwordMatches =
    await bcrypt.compare(
      input.password,
      user.passwordHash,
    );

  if (!passwordMatches) {
    throw new UnauthorizedError(
      "Invalid email or password",
    );
  }

  return {
    id: user.id,
    name: user.name,
    email: user.email,
  };
}
```

---

# Why `===` Does Not Work With bcrypt

Suppose:

```text
Input password:

Secret123
```

Database:

```text
$2b$12$...
```

This comparison:

```ts
"Secret123" ===
  "$2b$12$...";
```

will obviously return:

```ts
false
```

The stored value is not the original password.

Use:

```ts
bcrypt.compare(
  plaintextPassword,
  storedHash,
);
```

instead.

---

# Register Controller

```ts
import {
  NextFunction,
  Request,
  Response,
} from "express";

import {
  registerSchema,
} from "../schemas/auth.schema";

import {
  registerUser,
} from "../services/auth.service";

export async function
registerController(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  try {
    const validation =
      registerSchema.safeParse(
        req.body,
      );

    if (!validation.success) {
      return res
        .status(400)
        .json({
          message:
            "Invalid request data",

          errors:
            validation.error
              .flatten()
              .fieldErrors,
        });
    }

    const user =
      await registerUser(
        validation.data,
      );

    return res
      .status(201)
      .json({
        data: user,
      });
  } catch (error) {
    next(error);
  }
}
```

---

# Login Controller

```ts
import {
  NextFunction,
  Request,
  Response,
} from "express";

import {
  loginSchema,
} from "../schemas/auth.schema";

import {
  loginUser,
} from "../services/auth.service";

export async function
loginController(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  try {
    const validation =
      loginSchema.safeParse(
        req.body,
      );

    if (!validation.success) {
      return res
        .status(400)
        .json({
          message:
            "Invalid request data",

          errors:
            validation.error
              .flatten()
              .fieldErrors,
        });
    }

    const user =
      await loginUser(
        validation.data,
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

# Custom Errors

```ts
export class ConflictError
  extends Error {
  statusCode = 409;

  constructor(
    message: string,
  ) {
    super(message);

    this.name =
      "ConflictError";
  }
}
```

```ts
export class UnauthorizedError
  extends Error {
  statusCode = 401;

  constructor(
    message: string,
  ) {
    super(message);

    this.name =
      "UnauthorizedError";
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
  ConflictError,
} from "../errors/conflict.error";

import {
  UnauthorizedError,
} from "../errors/unauthorized.error";

export function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction,
) {
  if (
    error instanceof
    ConflictError
  ) {
    return res
      .status(409)
      .json({
        message:
          error.message,
      });
  }

  if (
    error instanceof
    UnauthorizedError
  ) {
    return res
      .status(401)
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

# Why Not Return Raw Database Errors?

Bad:

```ts
return res.status(500).json({
  error:
    error.message,
  stack:
    error.stack,
});
```

An internal database error may contain:

```text
Table names
Column names
Database host
SQL fragments
Internal file paths
Stack traces
```

That information should generally not be exposed to clients.

Instead:

```json
{
  "message": "Internal server error"
}
```

is returned externally.

Detailed error information belongs in internal logs.

---

# Password Must Never Be Returned

Even this is wrong:

```json
{
  "id": 1,
  "email": "mulia@example.com",
  "passwordHash": "$2b$12$..."
}
```

The fact that the password is hashed does not mean the hash should be exposed.

The client does not need it.

Use explicit Prisma selection:

```ts
select: {
  id: true,
  name: true,
  email: true,
  createdAt: true,
}
```

instead of returning the complete database object.

---

# Status Codes

| Situation | Status |
|---|---:|
| Registration successful | `201 Created` |
| Login successful | `200 OK` |
| Invalid request | `400 Bad Request` |
| Email already registered | `409 Conflict` |
| Invalid login credentials | `401 Unauthorized` |
| Unexpected server/database failure | `500 Internal Server Error` |

---

# Why 409 for Duplicate Email?

The request itself may be syntactically valid:

```json
{
  "name": "Mulia",
  "email": "mulia@example.com",
  "password": "Secret123"
}
```

but conflicts with existing server state:

```text
mulia@example.com
already exists
```

Therefore:

```http
409 Conflict
```

is a reasonable choice.

---

# Authentication vs Authorization

These are different concepts.

## Authentication

Authentication answers:

```text
Who are you?
```

Example:

```text
Mulia enters:
email + password
```

The system verifies the credentials.

That is:

```text
Authentication
```

---

## Authorization

Authorization answers:

```text
What are you allowed to do?
```

Example:

```text
Mulia is authenticated

but only ADMIN
may delete users
```

Checking whether Mulia has the ADMIN role is:

```text
Authorization
```

Conceptually:

```text
Authentication
↓
Who are you?

Authorization
↓
What can you access?
```

---

# Code Review

Original code:

```ts
async function login(
  email: string,
  password: string,
) {
  const user =
    await prisma.user.findUnique({
      where: {
        email,
      },
    });

  if (!user) {
    throw new Error(
      "Email does not exist",
    );
  }

  if (
    user.password !== password
  ) {
    throw new Error(
      "Wrong password",
    );
  }

  return user;
}
```

There are several problems.

---

## Problem 1: Plain Password Comparison

Incorrect:

```ts
user.password !== password
```

Passwords should be verified using:

```ts
bcrypt.compare()
```

---

## Problem 2: User Enumeration

These separate messages:

```text
Email does not exist
Wrong password
```

allow attackers to determine which accounts exist.

Prefer:

```text
Invalid email or password
```

---

## Problem 3: Returning Sensitive Data

```ts
return user;
```

may return:

```text
passwordHash
```

The service should return a safe object.

---

## Problem 4: Generic Error Type

```ts
throw new Error(...)
```

does not clearly communicate whether the error should become:

```text
400
401
409
500
```

Use application-specific errors or another structured error mechanism.

---

## Problem 5: Missing Validation

The function assumes:

```text
email is valid
password exists
```

without validation.

Validation should happen before business logic.

---

## Problem 6: Missing Email Normalization

Without normalization:

```text
MULIA@example.com
```

and:

```text
mulia@example.com
```

may behave inconsistently.

---

# Architecture Decision

Suppose another developer proposes:

```text
OAuth
Redis session
JWT
Access token
Refresh token
Rotating refresh token
Microservice authentication
```

for an internal application with roughly 500 users.

I would not automatically implement all of them.

The engineering process should be:

```text
Requirement
↓
Threat model
↓
Simplest correct solution
↓
Measure limitations
↓
Add complexity when justified
```

For Day 5, the immediate requirements are:

```text
Secure password storage
Credential verification
Input validation
Safe API responses
Correct error handling
```

Adding Redis, microservices, OAuth, and refresh-token rotation does not fix plaintext password storage.

Those technologies solve different problems.

Start with correct authentication fundamentals first.

---

# Project Structure

```text
fullstack-interview-lab/
└── day-05-authentication-basics/
    ├── prisma/
    │   └── schema.prisma
    │
    ├── src/
    │   ├── controllers/
    │   │   └── auth.controller.ts
    │   │
    │   ├── services/
    │   │   └── auth.service.ts
    │   │
    │   ├── schemas/
    │   │   └── auth.schema.ts
    │   │
    │   ├── routes/
    │   │   └── auth.route.ts
    │   │
    │   ├── errors/
    │   │   ├── conflict.error.ts
    │   │   └── unauthorized.error.ts
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
    │   └── auth.test.ts
    │
    ├── package.json
    ├── tsconfig.json
    └── README.md
```

---

# Routes

```ts
import {
  Router,
} from "express";

import {
  loginController,
  registerController,
} from "../controllers/auth.controller";

const router =
  Router();

router.post(
  "/register",
  registerController,
);

router.post(
  "/login",
  loginController,
);

export default router;
```

Mounted with:

```ts
app.use(
  "/auth",
  authRoutes,
);
```

Result:

```text
POST /auth/register

POST /auth/login
```

---

# Testing

Important authentication tests include:

```text
Register success
Invalid email
Short password
Duplicate email
Password hashing
Login success
Unknown email
Wrong password
Safe response
Database failure
```

---

# Register Test

```ts
it(
  "registers a new user",
  async () => {
    const response =
      await request(app)
        .post(
          "/auth/register",
        )
        .send({
          name: "Mulia",
          email:
            "mulia@example.com",
          password:
            "Secret123",
        });

    expect(
      response.status,
    ).toBe(201);

    expect(
      response.body.data.email,
    ).toBe(
      "mulia@example.com",
    );

    expect(
      response.body.data
        .passwordHash,
    ).toBeUndefined();
  },
);
```

---

# Invalid Email Test

```ts
it(
  "rejects invalid email",
  async () => {
    const response =
      await request(app)
        .post(
          "/auth/register",
        )
        .send({
          name: "Mulia",
          email: "abc",
          password:
            "Secret123",
        });

    expect(
      response.status,
    ).toBe(400);
  },
);
```

---

# Short Password Test

```ts
it(
  "rejects short password",
  async () => {
    const response =
      await request(app)
        .post(
          "/auth/register",
        )
        .send({
          name: "Mulia",
          email:
            "mulia@example.com",
          password: "123",
        });

    expect(
      response.status,
    ).toBe(400);
  },
);
```

---

# Duplicate Email Test

```ts
it(
  "rejects duplicate email",
  async () => {
    await request(app)
      .post(
        "/auth/register",
      )
      .send({
        name: "Mulia",
        email:
          "mulia@example.com",
        password:
          "Secret123",
      });

    const response =
      await request(app)
        .post(
          "/auth/register",
        )
        .send({
          name:
            "Another Mulia",
          email:
            "mulia@example.com",
          password:
            "Another123",
        });

    expect(
      response.status,
    ).toBe(409);
  },
);
```

---

# Password Hash Test

A useful test should verify two things.

First:

```text
The stored password is not plaintext.
```

Second:

```text
The hash still verifies against
the original password.
```

Example:

```ts
it(
  "stores password as a bcrypt hash",
  async () => {
    await request(app)
      .post(
        "/auth/register",
      )
      .send({
        name: "Mulia",
        email:
          "mulia@example.com",
        password:
          "Secret123",
      });

    const user =
      await prisma.user.findUnique({
        where: {
          email:
            "mulia@example.com",
        },
      });

    expect(user)
      .not.toBeNull();

    expect(
      user?.passwordHash,
    ).not.toBe(
      "Secret123",
    );

    const matches =
      await bcrypt.compare(
        "Secret123",
        user!.passwordHash,
      );

    expect(
      matches,
    ).toBe(true);
  },
);
```

This is stronger than only checking:

```ts
passwordHash !== password
```

because it also proves that the stored hash actually represents the original password.

---

# Login Success Test

```ts
it(
  "logs in with valid credentials",
  async () => {
    await request(app)
      .post(
        "/auth/register",
      )
      .send({
        name: "Mulia",
        email:
          "mulia@example.com",
        password:
          "Secret123",
      });

    const response =
      await request(app)
        .post(
          "/auth/login",
        )
        .send({
          email:
            "mulia@example.com",
          password:
            "Secret123",
        });

    expect(
      response.status,
    ).toBe(200);

    expect(
      response.body.data.email,
    ).toBe(
      "mulia@example.com",
    );

    expect(
      response.body.data
        .passwordHash,
    ).toBeUndefined();
  },
);
```

---

# Wrong Password Test

```ts
it(
  "rejects incorrect password",
  async () => {
    const response =
      await request(app)
        .post(
          "/auth/login",
        )
        .send({
          email:
            "mulia@example.com",
          password:
            "WrongPassword",
        });

    expect(
      response.status,
    ).toBe(401);

    expect(
      response.body.message,
    ).toBe(
      "Invalid email or password",
    );
  },
);
```

---

# Unknown Email Test

```ts
it(
  "does not reveal whether the email exists",
  async () => {
    const response =
      await request(app)
        .post(
          "/auth/login",
        )
        .send({
          email:
            "unknown@example.com",
          password:
            "Secret123",
        });

    expect(
      response.status,
    ).toBe(401);

    expect(
      response.body.message,
    ).toBe(
      "Invalid email or password",
    );
  },
);
```

The external response should be the same as the wrong-password case.

---

# Database Failure

If PostgreSQL becomes unavailable, the client should not receive:

```text
Prisma stack trace
database hostname
raw SQL
internal connection details
```

Expected:

```http
500 Internal Server Error
```

```json
{
  "message": "Internal server error"
}
```

Internally, the real error should still be logged.

---

# Security Boundaries

This challenge provides basic authentication fundamentals.

A production system may additionally require:

```text
Rate limiting
HTTPS
Login attempt monitoring
Password reset flow
Email verification
Session or token management
Secret management
Audit logging
MFA
```

depending on the application's threat model.

These should be introduced because requirements justify them, not simply because they are common security terms.

---

# Better Interview Answer

> The first major issue is that the application stores passwords as plaintext. I would never store the original password. During registration, I would validate and normalize the input, hash the password using a password hashing algorithm such as bcrypt, and store only the resulting hash.
>
> The email column should have a database-level unique constraint. I may check for an existing email before hashing for efficiency, but I would still handle the database unique constraint because two concurrent registration requests could otherwise race.
>
> During login, I would find the user using the normalized email and verify the submitted password using `bcrypt.compare()`. I would not directly compare the submitted password with the stored hash.
>
> I would also return the same `Invalid email or password` response for both an unknown email and an incorrect password to reduce account enumeration.
>
> Password hashes should never be returned to the frontend. I would explicitly select the fields that belong in the API response.
>
> Validation errors should return a client error, duplicate email can return `409 Conflict`, invalid credentials should return `401 Unauthorized`, and unexpected database failures should be handled centrally as `500 Internal Server Error` without leaking internal database details.
>
> I would keep the design simple at this stage. JWT, refresh tokens, OAuth, Redis sessions, and microservices solve additional requirements, but they do not replace correct password hashing, validation, and error handling.

---

# Interview Follow-Up Questions

## 1. Why can't we decrypt a bcrypt hash?

Because bcrypt is a password hashing algorithm, not an encryption algorithm.

Authentication verifies the password using:

```ts
bcrypt.compare()
```

instead of recovering the original password.

---

## 2. Why isn't checking duplicate email before INSERT enough?

Because of race conditions.

Two requests can both observe:

```text
email does not exist
```

before either inserts it.

The database unique constraint guarantees uniqueness.

---

## 3. Why not return the password hash?

Because the client does not need it.

Sensitive information should not be exposed unnecessarily, even if it is not plaintext.

---

## 4. Authentication vs authorization?

```text
Authentication
→ Who are you?

Authorization
→ What are you allowed to do?
```

Login is authentication.

Checking whether the authenticated user is an ADMIN is authorization.

---

## 5. Should every authentication system immediately use Redis?

No.

Redis may be useful for specific session, caching, token, or rate-limiting requirements.

It is not required simply to hash and verify passwords.

---

# What I Learned

The registration flow is:

```text
Validate
↓
Normalize
↓
Hash
↓
Store
↓
Return safe response
```

The login flow is:

```text
Validate
↓
Normalize
↓
Find user
↓
Compare password against hash
↓
Authenticate
↓
Return safe response
```

The most important principles from this challenge are:

```text
Never store plaintext passwords

Hash passwords using a password
hashing algorithm

Never compare plaintext directly
with the stored hash

Never expose password hashes

Use database constraints for
important invariants

Do not reveal unnecessary
authentication information

Separate validation, business logic,
database access, and HTTP handling
```

---

# Interview Key Takeaway

A weak answer is:

> "Use bcrypt and JWT."

A stronger answer is:

> "I would first secure the credential lifecycle. Validate and normalize registration input, store only a bcrypt hash, enforce email uniqueness at the database level, verify login using `bcrypt.compare()`, avoid account enumeration, never expose the hash, and map expected failures to appropriate HTTP responses. Token or session architecture comes after credential verification is implemented correctly."

The engineering process should remain:

```text
Requirement
↓
Threat
↓
Simplest correct solution
↓
Implementation
↓
Testing
↓
Add complexity when justified
```

---

# Git Workflow

Create branch:

```bash
git checkout -b feat/day-05-authentication-basics
```

Suggested commits:

```bash
git commit -m "feat: add user registration validation"
```

```bash
git commit -m "feat: hash user passwords with bcrypt"
```

```bash
git commit -m "feat: add secure credential verification"
```

```bash
git commit -m "fix: prevent sensitive authentication data exposure"
```

```bash
git commit -m "test: add authentication integration tests"
```

---

# Progress Summary

```text
Day:
05

Topic:
Authentication Basics

Difficulty:
Fundamental

Main Skill:
Secure Registration and Login

Secondary Skills:
bcrypt
Password Hashing
Zod Validation
HTTP Status Codes
Error Handling
Prisma Unique Constraints
Authentication vs Authorization
Security

Project:
Authentication API

GitHub Folder:
day-05-authentication-basics

Suggested Branch:
feat/day-05-authentication-basics

Suggested Commits:
feat: add user registration validation
feat: hash user passwords with bcrypt
feat: add secure credential verification
fix: prevent sensitive authentication data exposure
test: add authentication integration tests
```