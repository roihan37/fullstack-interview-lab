# Full Stack Interview Lab

A collection of hands-on software engineering challenges created as part of my preparation for Software Developer and Full Stack Developer technical interviews.

This repository focuses on solving practical engineering problems, understanding technical trade-offs, writing production-oriented code, and documenting what I learn throughout the process.

The challenges gradually progress from fundamental programming concepts to more advanced topics such as database optimization, caching, payment systems, concurrency, messaging systems, security, scalability, and system design.

---

## Goals

The main objectives of this repository are to:

* Improve problem-solving skills for technical interviews
* Practice real-world software engineering scenarios
* Strengthen Full Stack development fundamentals
* Improve code quality and software architecture decisions
* Practice debugging and code review
* Understand engineering trade-offs
* Build consistent testing habits
* Learn production-oriented engineering concepts
* Document my learning progress publicly

---

## Main Tech Stack

The technologies used may vary depending on each challenge.

### Frontend

* JavaScript
* TypeScript
* React.js
* Next.js
* Tailwind CSS
* TanStack Query

### Backend

* Node.js
* Express.js
* TypeScript

### Database

* PostgreSQL
* Prisma
* Sequelize

### Caching & Messaging

* Redis
* BullMQ
* RabbitMQ
* Kafka

### Testing

* Vitest
* Jest
* Supertest

### Infrastructure & Tools

* Git
* GitHub
* Docker
* Docker Compose
* GitHub Actions
* Postman

Not every challenge uses all of these technologies.

The project structure and technology choices are determined by the actual problem being solved.

---

## Repository Structure

Each challenge is organized by day.

```text
fullstack-interview-lab/
│
├── README.md
├── LICENSE
├── .gitignore
│
├── docs/
│   ├── progress.md
│   └── interview-notes.md
│
├── day-01-topic-name/
├── day-02-topic-name/
├── day-03-topic-name/
│
└── ...
```

Each challenge may have a different internal structure depending on its complexity.

### Simple Challenge

```text
day-01-array-problem/
├── README.md
├── solution.ts
└── solution.test.ts
```

### Backend Challenge

```text
day-10-rest-api/
├── README.md
├── package.json
├── tsconfig.json
├── src/
└── tests/
```

### Database Challenge

```text
day-15-database-transaction/
├── README.md
├── package.json
├── prisma/
├── src/
└── tests/
```

### Full Stack Challenge

```text
day-35-payment-checkout/
├── README.md
│
├── client/
│   └── src/
│
├── server/
│   ├── prisma/
│   ├── src/
│   └── tests/
│
└── docker-compose.yml
```

Project structure is intentionally kept proportional to the problem.

A simple problem should have a simple implementation. Additional architecture is introduced only when it provides a clear engineering benefit.

---

## Learning Progression

The challenges gradually increase in complexity.

### Fundamentals

Topics may include:

* JavaScript
* TypeScript
* Async/Await
* Promises
* REST API
* SQL
* CRUD
* Validation
* Error handling
* React fundamentals
* Basic testing

### Backend Engineering

Topics may include:

* Authentication
* Authorization
* JWT
* Refresh tokens
* Database transactions
* Pagination
* Database indexing
* Query optimization
* API security
* Logging

### Production Engineering

Topics may include:

* Redis
* Caching
* Rate limiting
* Background jobs
* WebSocket
* BullMQ
* RabbitMQ
* Kafka
* Idempotency
* Retry mechanisms
* Race conditions
* Database locking

### Payment Systems

Topics may include:

* Payment gateways
* Payment webhooks
* Webhook verification
* Duplicate payment prevention
* Idempotency keys
* Inventory reservation
* Refund handling
* Payment expiration
* Transaction consistency

### Scalability & Architecture

Topics may include:

* Horizontal scaling
* Load balancing
* Distributed caching
* Database replication
* Event-driven architecture
* Circuit breakers
* Microservices
* Modular monoliths
* System design

---

## Challenge Workflow

Each challenge follows a problem-solving process similar to a technical interview.

```text
Problem
   ↓
Requirement Analysis
   ↓
Solution Proposal
   ↓
Trade-off Analysis
   ↓
Implementation
   ↓
Testing
   ↓
Review
   ↓
Documentation
```

I try to understand the problem before choosing a technology.

The goal is not to use the most complex technology available, but to choose an appropriate solution based on the actual requirements.

---

## Challenge Documentation

Each challenge contains its own `README.md` whenever appropriate.

A typical challenge README contains:

```text
Problem
Requirements
Solution
Architecture
Tech Stack
API
Testing
How to Run
Trade-offs
What I Learned
```

This allows each challenge to serve as both a coding exercise and a small engineering case study.

---

## Example Challenges

| Day | Challenge                 | Main Topic   | Difficulty   |
| --- | ------------------------- | ------------ | ------------ |
| 01  | REST API Pagination       | Backend      | Fundamental  |
| 02  | JWT Authentication        | Security     | Fundamental  |
| 03  | Database Transaction      | Database     | Junior       |
| 04  | React Performance         | Frontend     | Junior       |
| 05  | Redis Caching             | Caching      | Intermediate |
| 06  | API Rate Limiting         | Security     | Intermediate |
| 07  | Payment Webhook           | Payment      | Intermediate |
| 08  | Concurrent Checkout       | Concurrency  | Intermediate |
| 09  | Background Job Processing | Queue        | Intermediate |
| 10  | E-commerce System Design  | Architecture | Advanced     |

The actual table will continue to grow as new challenges are completed.

---

## Git Workflow

The `main` branch contains completed challenges.

Each new challenge is developed using a temporary feature branch.

Example:

```bash
git checkout main

git pull origin main

git checkout -b feature/day-07-payment-webhook
```

After completing and testing the challenge:

```bash
git add .

git commit -m "feat: implement idempotent payment webhook"

git push origin feature/day-07-payment-webhook
```

The feature branch can then be merged into `main`.

---

## Commit Convention

This repository follows Conventional Commit-style messages where appropriate.

Examples:

```text
feat: implement cursor-based pagination

feat: add JWT authentication

fix: prevent duplicate payment processing

perf: optimize product search query

refactor: extract payment logic into service layer

test: add payment webhook integration tests

docs: document Redis caching challenge

chore: add Docker Compose configuration
```

Commit history is intended to show how each solution evolves instead of combining an entire challenge into a single large commit.

---

## Testing Philosophy

Testing is part of the implementation rather than an optional final step.

Depending on the challenge, tests may include:

* Happy path
* Validation failure
* Authentication failure
* Authorization failure
* Edge cases
* Duplicate requests
* Database failures
* Concurrent requests
* External service failures

Testing tools may include:

```text
Vitest
Jest
Supertest
```

---

## Environment Variables

Sensitive information is never committed to the repository.

Files such as:

```text
.env
.env.local
```

are excluded through `.gitignore`.

An `.env.example` file may be provided when a challenge requires configuration.

Example:

```env
PORT=

DATABASE_URL=

JWT_SECRET=

REDIS_URL=

PAYMENT_API_KEY=
```

Real credentials and API keys should never be committed.

---

## Engineering Principles

Throughout this repository, I try to follow several principles:

* Understand the problem before choosing technology
* Avoid unnecessary complexity
* Keep responsibilities clear
* Write readable and maintainable code
* Consider failure scenarios
* Validate external input
* Protect sensitive operations
* Write meaningful tests
* Consider performance when necessary
* Document engineering decisions
* Understand trade-offs instead of memorizing solutions

---

## What I Want to Improve

This repository is also used to track areas that I want to continuously improve, including:

* Technical interview communication
* Backend architecture
* Database optimization
* Frontend performance
* Testing strategies
* Application security
* Distributed systems
* Payment architecture
* System design
* Production debugging

---

## Disclaimer

This repository is primarily created for learning, technical interview preparation, experimentation, and portfolio purposes.

Some implementations intentionally simplify infrastructure or business requirements so that each challenge can focus on a specific engineering concept.

Production systems may require additional security, monitoring, infrastructure, compliance, and reliability considerations.

---

## License

This project is licensed under the MIT License.

See the [LICENSE](./LICENSE) file for details.
