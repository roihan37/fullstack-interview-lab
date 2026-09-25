# Day 03 - React Rendering, Memoization, and Search Performance

## Problem

A Product Management page receives around **5,000 products** from the backend.

Users can:

- Search products by name
- Filter by category
- Select a product
- See search results immediately while typing

Initial implementation:

```tsx
import { useState } from "react";

type Product = {
  id: number;
  name: string;
  category: string;
  price: number;
};

type ProductListProps = {
  products: Product[];
};

export function ProductList({
  products,
}: ProductListProps) {
  const [search, setSearch] = useState("");
  const [category, setCategory] =
    useState("all");

  const [
    selectedProduct,
    setSelectedProduct,
  ] = useState<Product | null>(null);

  const filteredProducts =
    products.filter((product) => {
      const matchesSearch =
        product.name
          .toLowerCase()
          .includes(
            search.toLowerCase(),
          );

      const matchesCategory =
        category === "all" ||
        product.category === category;

      return (
        matchesSearch &&
        matchesCategory
      );
    });

  return (
    <div>
      <input
        value={search}
        onChange={(event) =>
          setSearch(event.target.value)
        }
        placeholder="Search product"
      />

      <select
        value={category}
        onChange={(event) =>
          setCategory(
            event.target.value,
          )
        }
      >
        <option value="all">
          All
        </option>

        <option value="electronics">
          Electronics
        </option>

        <option value="fashion">
          Fashion
        </option>
      </select>

      <p>
        Found:
        {filteredProducts.length}
      </p>

      {filteredProducts.map(
        (product) => (
          <div
            key={product.id}
            onClick={() =>
              setSelectedProduct(
                product,
              )
            }
          >
            <h3>
              {product.name}
            </h3>

            <p>
              {product.price}
            </p>
          </div>
        ),
      )}

      {selectedProduct && (
        <div>
          Selected:
          {selectedProduct.name}
        </div>
      )}
    </div>
  );
}
```

The application starts showing several symptoms:

- Search feels slower
- Filtering runs on every component render
- Selecting a product causes filtering to run again
- Parent re-renders also cause filtering again
- Thousands of product cards may be rendered into the DOM

The goal is to improve the component without introducing unnecessary optimization.

---

# Interview Question

> What is wrong with this component, and how would you improve its performance without overengineering it?

Follow-up:

> Would you immediately use `useMemo`, `useCallback`, debounce, `React.memo`, or server-side search?

---

# Interview Answer

The first thing I would do is identify the actual bottleneck instead of immediately adding memoization or another technology.

The component currently performs filtering directly during render:

```tsx
const filteredProducts =
  products.filter(...);
```

React function components execute their function body again when they re-render.

Therefore, changes to:

```text
search
category
selectedProduct
```

can all cause this code to execute again.

Even though selecting a product does not change `search` or `category`, this:

```tsx
setSelectedProduct(product);
```

updates component state.

That causes a new render.

During that render:

```tsx
products.filter(...)
```

runs again.

The filtering result is therefore recalculated even though its real dependencies have not changed.

For around 5,000 products this may or may not actually be a serious performance issue.

I would profile the component first before optimizing aggressively.

---

# Why setSelectedProduct Causes Filtering Again

Consider:

```tsx
setSelectedProduct(product);
```

React schedules a state update.

The component function executes again:

```text
ProductList()
    ↓
useState()
    ↓
products.filter()
    ↓
JSX
```

Therefore:

```tsx
const filteredProducts =
  products.filter(...);
```

is recalculated.

React does not automatically understand that the filtering result only depends on:

```text
products
search
category
```

Without explicit memoization, JavaScript simply executes the expression again.

---

# Using useMemo

An optimization could be:

```tsx
const filteredProducts =
  useMemo(() => {
    const normalizedSearch =
      search.toLowerCase();

    return products.filter(
      (product) => {
        const matchesSearch =
          product.name
            .toLowerCase()
            .includes(
              normalizedSearch,
            );

        const matchesCategory =
          category === "all" ||
          product.category ===
            category;

        return (
          matchesSearch &&
          matchesCategory
        );
      },
    );
  }, [
    products,
    search,
    category,
  ]);
```

Now React will reuse the previously calculated result unless one of these dependencies changes:

```text
products
search
category
```

Therefore changing only:

```text
selectedProduct
```

does not require recalculating the filtered products.

---

# Benefits of useMemo

For this case, `useMemo` may help when:

```text
products.filter(...)
```

is sufficiently expensive and the component re-renders frequently for unrelated reasons.

Benefits:

- Avoid unnecessary recalculation
- Useful for expensive derived data
- Can help preserve references used by memoized child components
- Keeps filtering tied to its actual dependencies

---

# Downsides of useMemo

`useMemo` is not free.

It adds:

- Dependency tracking
- Additional code complexity
- Memory usage for cached values
- Risk of incorrect dependency arrays
- Harder code maintenance

Therefore I would not write:

```tsx
useMemo(() => products.filter(...))
```

for every array operation.

A simple operation on a small collection may be cheaper than the complexity introduced by memoization.

The correct approach is:

```text
Measure
→ Identify expensive calculation
→ Optimize if useful
```

not:

```text
Array method
→ useMemo
```

---

# Incorrect Dependency Array

This implementation is wrong:

```tsx
const filteredProducts =
  useMemo(() => {
    return products.filter(
      (product) =>
        product.name
          .toLowerCase()
          .includes(
            search.toLowerCase(),
          ),
    );
  }, []);
```

The empty dependency array means:

```text
Calculate once
→ reuse forever
```

But the calculation depends on:

```text
products
search
```

If the user changes:

```text
iphone
```

to:

```text
samsung
```

React may continue returning the old filtered result.

This is a stale value bug.

The correct dependencies are:

```tsx
[
  products,
  search,
]
```

or:

```tsx
[
  products,
  search,
  category,
]
```

if category filtering is also included.

---

# useCallback

Suppose we have:

```tsx
const handleSearch = (
  event:
    React.ChangeEvent<HTMLInputElement>,
) => {
  setSearch(
    event.target.value,
  );
};
```

It is possible to write:

```tsx
const handleSearch =
  useCallback(
    (
      event:
        React.ChangeEvent<HTMLInputElement>,
    ) => {
      setSearch(
        event.target.value,
      );
    },
    [],
  );
```

But that does not automatically improve performance.

The original function is usually inexpensive to recreate.

Using `useCallback` becomes useful when function identity matters.

For example:

```tsx
<MemoizedProductCard
  onSelect={handleSelect}
/>
```

If `ProductCard` uses `React.memo`, a stable callback reference may help prevent unnecessary child renders.

But for a simple input:

```tsx
<input
  onChange={handleSearch}
/>
```

`useCallback` is usually unnecessary unless profiling shows a real benefit.

---

# React.memo

Suppose the product card is:

```tsx
type ProductCardProps = {
  product: Product;
  onSelect:
    (product: Product) => void;
};

function ProductCard({
  product,
  onSelect,
}: ProductCardProps) {
  return (
    <button
      onClick={() =>
        onSelect(product)
      }
    >
      <h3>
        {product.name}
      </h3>

      <p>
        {product.price}
      </p>
    </button>
  );
}

export const
  MemoizedProductCard =
    React.memo(ProductCard);
```

`React.memo` compares props using shallow equality.

This can prevent unnecessary renders if props remain referentially equal.

However, this parent code creates a new function every render:

```tsx
<MemoizedProductCard
  product={product}
  onSelect={(product) =>
    setSelectedProduct(product)
  }
/>
```

The callback:

```tsx
(product) =>
  setSelectedProduct(product)
```

is a new function object every render.

Therefore:

```text
previous onSelect
!== new onSelect
```

From `React.memo`'s perspective, the props changed.

As a result, the component may still re-render.

A better approach:

```tsx
const handleSelect =
  useCallback(
    (product: Product) => {
      setSelectedProduct(
        product,
      );
    },
    [],
  );
```

Then:

```tsx
<MemoizedProductCard
  product={product}
  onSelect={handleSelect}
/>
```

Now `onSelect` has a stable reference.

---

# Important Note About React.memo

This does not mean:

```text
React.memo
+
useCallback
=
always faster
```

Memoization itself has a cost.

It is most useful when:

- Child rendering is expensive
- There are many child components
- Parent renders frequently
- Props usually remain unchanged

For a tiny component rendered only a few times, memoization might add unnecessary complexity.

---

# Search Debounce

Suppose the user types:

```text
i
ip
iph
ipho
iphon
iphone
```

Without debounce, filtering can run six times.

For local filtering across approximately 5,000 simple records, debounce may not be necessary.

A JavaScript filter over 5,000 simple objects can often be fast enough.

Adding debounce also changes user experience because the result is intentionally delayed.

Therefore I would first measure.

---

# When Debounce Is More Useful

Debounce becomes much more useful when every keystroke triggers:

```http
GET /products?search=i
GET /products?search=ip
GET /products?search=iph
GET /products?search=ipho
```

Without debounce, typing quickly could create many unnecessary network requests.

Example:

```tsx
useEffect(() => {
  const timeout =
    setTimeout(() => {
      searchProducts(search);
    }, 300);

  return () =>
    clearTimeout(timeout);
}, [search]);
```

Now rapid typing:

```text
i
ip
iph
ipho
iphone
```

may only produce one final request:

```http
GET /products?search=iphone
```

after the user pauses.

---

# Client-Side Filtering vs Server-Side Filtering

For:

```text
5,000 products
```

client-side filtering may still be acceptable depending on:

- Payload size
- Device performance
- Filtering complexity
- Rendering cost
- Frequency of data updates

But if the dataset grows to:

```text
500,000 products
```

I would not download everything into the browser.

Problems would include:

```text
Large network payload
High memory usage
Slow initial load
Expensive filtering
Large DOM rendering
Outdated data
```

At that point, filtering should generally move to the backend.

Example:

```http
GET /products
  ?search=iphone
  &category=electronics
  &page=1
  &limit=20
```

The database should perform:

```text
Filtering
Sorting
Pagination
```

and the frontend should receive only the required subset.

---

# Example Server-Side Flow

```text
User types search
        ↓
Debounce
        ↓
GET /products?search=iphone
        ↓
Backend
        ↓
Database filtering
        ↓
Pagination
        ↓
20 products
        ↓
Frontend
```

This is much more scalable than sending hundreds of thousands of products to the browser.

---

# DOM Performance

Even if filtering is optimized, there may still be another bottleneck.

Suppose filtering produces:

```text
4,000 products
```

React still has to create and reconcile:

```text
4,000 ProductCard components
```

The browser also needs to manage thousands of DOM nodes.

Therefore:

```tsx
useMemo(...)
```

only optimizes the filtering computation.

It does not reduce:

```text
DOM size
React reconciliation
Layout work
Painting
```

Possible solutions include:

```text
Pagination
Virtualization
Server-side pagination
Infinite scrolling
```

depending on the product requirements.

---

# Recommended Implementation

```tsx
import {
  memo,
  useCallback,
  useMemo,
  useState,
} from "react";

type Product = {
  id: number;
  name: string;
  category: string;
  price: number;
};

type ProductCardProps = {
  product: Product;
  onSelect:
    (product: Product) => void;
};

const ProductCard = memo(
  function ProductCard({
    product,
    onSelect,
  }: ProductCardProps) {
    return (
      <button
        type="button"
        onClick={() =>
          onSelect(product)
        }
      >
        <h3>
          {product.name}
        </h3>

        <p>
          {product.price}
        </p>
      </button>
    );
  },
);

type ProductListProps = {
  products: Product[];
};

export function ProductList({
  products,
}: ProductListProps) {
  const [search, setSearch] =
    useState("");

  const [category, setCategory] =
    useState("all");

  const [
    selectedProduct,
    setSelectedProduct,
  ] =
    useState<Product | null>(
      null,
    );

  const filteredProducts =
    useMemo(() => {
      const normalizedSearch =
        search
          .trim()
          .toLowerCase();

      return products.filter(
        (product) => {
          const matchesSearch =
            product.name
              .toLowerCase()
              .includes(
                normalizedSearch,
              );

          const matchesCategory =
            category === "all" ||
            product.category ===
              category;

          return (
            matchesSearch &&
            matchesCategory
          );
        },
      );
    }, [
      products,
      search,
      category,
    ]);

  const handleSelect =
    useCallback(
      (product: Product) => {
        setSelectedProduct(
          product,
        );
      },
      [],
    );

  return (
    <section>
      <input
        value={search}
        onChange={(event) =>
          setSearch(
            event.target.value,
          )
        }
        placeholder="Search product"
      />

      <select
        value={category}
        onChange={(event) =>
          setCategory(
            event.target.value,
          )
        }
      >
        <option value="all">
          All
        </option>

        <option value="electronics">
          Electronics
        </option>

        <option value="fashion">
          Fashion
        </option>
      </select>

      <p>
        Found:
        {" "}
        {filteredProducts.length}
      </p>

      {filteredProducts.map(
        (product) => (
          <ProductCard
            key={product.id}
            product={product}
            onSelect={
              handleSelect
            }
          />
        ),
      )}

      {selectedProduct && (
        <p>
          Selected:
          {" "}
          {
            selectedProduct.name
          }
        </p>
      )}
    </section>
  );
}
```

---

# Why This Implementation Is Better

The filtering operation only recalculates when:

```text
products changes
search changes
category changes
```

Changing:

```text
selectedProduct
```

still causes the parent component to render, but the memoized filtering value can be reused.

The `handleSelect` callback also has a stable reference:

```tsx
const handleSelect =
  useCallback(...);
```

This allows:

```tsx
React.memo(ProductCard)
```

to be more effective.

---

# Code Review

Given:

```tsx
function Products({
  products,
}: {
  products: Product[];
}) {
  const [search, setSearch] =
    useState("");

  const [selected, setSelected] =
    useState<Product | null>(
      null,
    );

  const filtered =
    useMemo(() => {
      return products.filter(
        (product) =>
          product.name.includes(
            search,
          ),
      );
    }, [products]);

  const handleSelect =
    useCallback(
      (product: Product) => {
        setSelected(product);
      },
      [selected],
    );

  return (
    <>
      <input
        value={search}
        onChange={(e) =>
          setSearch(
            e.target.value,
          )
        }
      />

      {filtered.map(
        (product) => (
          <ProductCard
            key={product.id}
            product={product}
            onSelect={
              handleSelect
            }
          />
        ),
      )}
    </>
  );
}
```

There are several issues.

---

## Issue 1: Missing search Dependency

Current code:

```tsx
useMemo(() => {
  return products.filter(
    (product) =>
      product.name.includes(
        search,
      ),
  );
}, [products]);
```

The calculation depends on:

```text
products
search
```

but only `products` is listed.

Therefore when search changes, React may continue returning the previous cached result.

Correct:

```tsx
}, [
  products,
  search,
]);
```

---

## Issue 2: Incorrect useCallback Dependency

Current:

```tsx
const handleSelect =
  useCallback(
    (product: Product) => {
      setSelected(product);
    },
    [selected],
  );
```

The callback does not read:

```text
selected
```

Therefore it should not depend on it.

Current behavior creates a new callback every time `selected` changes.

Correct:

```tsx
const handleSelect =
  useCallback(
    (product: Product) => {
      setSelected(product);
    },
    [],
  );
```

---

## Issue 3: Search Is Case-Sensitive

Current:

```tsx
product.name.includes(search)
```

Searching:

```text
iphone
```

will not match:

```text
iPhone
```

A better implementation:

```tsx
product.name
  .toLowerCase()
  .includes(
    search.toLowerCase(),
  )
```

Ideally normalize the search only once:

```tsx
const normalizedSearch =
  search
    .trim()
    .toLowerCase();
```

---

## Issue 4: Selected State Is Never Used

The code stores:

```tsx
const [
  selected,
  setSelected,
] = useState(...)
```

but `selected` is never rendered or used for business logic.

If selection is actually unnecessary, the state itself should be removed.

Unused state creates unnecessary renders and complexity.

---

## Issue 5: Memoization May Be Premature

Even after fixing the dependency array, we should still ask:

```text
Is filtering actually slow?
```

For a small dataset, this:

```tsx
products.filter(...)
```

may not require memoization at all.

Optimization should be based on measurement.

---

# Architecture Decision: Should We Add Redis?

If the interviewer says:

> The page is slow. Let's add Redis.

I would not immediately agree.

Redis primarily helps with server-side data access and caching.

The current performance problem may be:

```text
React rendering
DOM size
Client-side filtering
Large JavaScript workload
Unnecessary re-renders
```

Redis would not directly solve those problems.

I would first investigate:

```text
Problem
↓
React Profiler
↓
Browser Performance tools
↓
Identify bottleneck
↓
Choose solution
```

If the bottleneck is:

```text
4,000 DOM nodes
```

Redis does not fix that.

If the bottleneck is:

```text
slow database query
```

then server-side optimization may be relevant.

---

# Should We Immediately Move Search to the Backend?

Not necessarily.

For:

```text
5,000 small products
```

client-side search might provide:

- Instant search
- No network round-trip
- Simple implementation
- Reduced backend requests

Server-side filtering becomes more appropriate when:

```text
Dataset becomes large
Payload becomes expensive
Data changes frequently
Security prevents sending all records
Database filtering is required
Pagination is required
```

Therefore the decision should depend on measurement and requirements.

---

# Decision Process

Use:

```text
Problem
↓
Measurement
↓
Bottleneck
↓
Possible Solutions
↓
Trade-off
↓
Implementation
```

Example:

```text
Slow search
↓
Profiler shows filtering = 2 ms
but rendering = 120 ms
↓
Filtering is not the bottleneck
↓
useMemo provides little benefit
↓
Optimize rendered list
↓
Pagination or virtualization
```

This is better than:

```text
Slow
→ useMemo
```

---

# Testing

## 1. Search Works

Given:

```ts
[
  {
    id: 1,
    name: "iPhone",
    category: "electronics",
    price: 1000,
  },

  {
    id: 2,
    name: "T-Shirt",
    category: "fashion",
    price: 100,
  },
]
```

When user searches:

```text
iphone
```

Expected:

```text
iPhone
```

is displayed.

---

## 2. Category Filtering

Select:

```text
electronics
```

Expected:

```text
Only electronics products
```

---

## 3. Search + Category

Search:

```text
iphone
```

Category:

```text
electronics
```

Expected:

```text
Product must match both conditions
```

---

## 4. Empty Result

Search:

```text
nonexistent-product
```

Expected:

```text
Found: 0
```

and no product cards.

---

## 5. Case-Insensitive Search

Search:

```text
IPHONE
```

should still match:

```text
iPhone
```

---

## 6. Selecting Product Does Not Change Filter Result

Given:

```text
search = iphone
```

When a product is selected:

```tsx
setSelectedProduct(product);
```

the visible product list should remain unchanged.

---

## 7. Search Does Not Become Stale

If search changes from:

```text
iphone
```

to:

```text
samsung
```

the result must update.

This verifies that the dependency array includes:

```text
search
```

---

## 8. Products Prop Changes

If the parent provides a new products array:

```text
5 products
→
6 products
```

the filtered result must be recalculated.

This verifies that:

```text
products
```

exists in the dependency array.

---

# React Testing Library Example

```tsx
import {
  fireEvent,
  render,
  screen,
} from "@testing-library/react";

import {
  describe,
  expect,
  it,
} from "vitest";

import {
  ProductList,
} from "../src/components/ProductList";

const products = [
  {
    id: 1,
    name: "iPhone 17",
    category:
      "electronics",
    price: 1000,
  },

  {
    id: 2,
    name: "Samsung Galaxy",
    category:
      "electronics",
    price: 900,
  },

  {
    id: 3,
    name: "T-Shirt",
    category:
      "fashion",
    price: 50,
  },
];

describe(
  "ProductList",
  () => {
    it(
      "filters products by search",
      () => {
        render(
          <ProductList
            products={products}
          />,
        );

        const input =
          screen.getByPlaceholderText(
            "Search product",
          );

        fireEvent.change(
          input,
          {
            target: {
              value:
                "iphone",
            },
          },
        );

        expect(
          screen.getByText(
            "iPhone 17",
          ),
        ).toBeInTheDocument();

        expect(
          screen.queryByText(
            "Samsung Galaxy",
          ),
        ).not.toBeInTheDocument();
      },
    );
  },
);
```

---

# Category Test

```tsx
it(
  "filters products by category",
  () => {
    render(
      <ProductList
        products={products}
      />,
    );

    const select =
      screen.getByRole(
        "combobox",
      );

    fireEvent.change(
      select,
      {
        target: {
          value: "fashion",
        },
      },
    );

    expect(
      screen.getByText(
        "T-Shirt",
      ),
    ).toBeInTheDocument();

    expect(
      screen.queryByText(
        "iPhone 17",
      ),
    ).not.toBeInTheDocument();
  },
);
```

---

# Search + Category Test

```tsx
it(
  "combines search and category filters",
  () => {
    render(
      <ProductList
        products={products}
      />,
    );

    fireEvent.change(
      screen.getByPlaceholderText(
        "Search product",
      ),
      {
        target: {
          value: "samsung",
        },
      },
    );

    fireEvent.change(
      screen.getByRole(
        "combobox",
      ),
      {
        target: {
          value:
            "electronics",
        },
      },
    );

    expect(
      screen.getByText(
        "Samsung Galaxy",
      ),
    ).toBeInTheDocument();

    expect(
      screen.queryByText(
        "iPhone 17",
      ),
    ).not.toBeInTheDocument();
  },
);
```

---

# Performance Testing

Performance optimization should not be based only on assumptions.

Useful tools include:

```text
React DevTools Profiler
Chrome DevTools Performance
Browser Performance API
Lighthouse where relevant
```

Measure:

```text
Component render duration
Number of renders
Filtering duration
Number of DOM nodes
Interaction latency
```

For example:

```ts
const start =
  performance.now();

const result =
  products.filter(...);

const end =
  performance.now();

console.log(
  `Filtering took ${
    end - start
  }ms`,
);
```

This is useful for local experimentation, although production profiling should use better tooling.

---

# Example Performance Investigation

Suppose profiling shows:

```text
Filtering 5,000 products:
2 ms

Rendering 4,000 cards:
150 ms
```

Then adding:

```tsx
useMemo
```

will not solve the main problem.

The bottleneck is rendering.

A more meaningful solution could be:

```text
Pagination
or
List virtualization
```

This demonstrates why performance work should be measurement-driven.

---

# When to Use Virtualization

Virtualization is useful when the application must display a large scrolling list.

Instead of rendering:

```text
4,000 DOM nodes
```

the application might render only:

```text
20-40 visible rows
```

Libraries such as:

```text
TanStack Virtual
react-window
```

can implement this pattern.

However, virtualization should only be introduced when the product actually needs a large continuous list.

For a normal product management table, pagination might be simpler.

---

# Project Structure

```text
fullstack-interview-lab/
└── day-03-react-performance/
    ├── src/
    │   ├── components/
    │   │   ├── ProductList.tsx
    │   │   └── ProductCard.tsx
    │   │
    │   ├── types/
    │   │   └── product.ts
    │   │
    │   ├── App.tsx
    │   └── main.tsx
    │
    ├── tests/
    │   └── ProductList.test.tsx
    │
    ├── package.json
    ├── tsconfig.json
    ├── vite.config.ts
    └── README.md
```

---

# Tech Stack

```text
React
TypeScript
Vite
Vitest
React Testing Library
```

---

# How To Run

Install dependencies:

```bash
npm install
```

Start application:

```bash
npm run dev
```

Run tests:

```bash
npm test
```

---

# Better Interview Answer

> React function components execute again whenever their state or props cause a re-render. In this component, updating `selectedProduct` causes the component to render again, so `products.filter()` also runs again even though the search and category have not changed.
>
> I would first profile the component instead of immediately adding `useMemo`, `useCallback`, and `React.memo`.
>
> If filtering 5,000 products is actually expensive and unrelated state updates happen frequently, I could memoize the filtered result using `useMemo` with `products`, `search`, and `category` as dependencies.
>
> I would not use `useCallback` for every event handler. It becomes useful when maintaining function identity matters, for example when passing a callback into a memoized child component.
>
> I would also investigate rendering cost. If filtering returns 4,000 products, memoizing the filtering operation does not solve the cost of rendering thousands of DOM elements. Pagination or virtualization may provide a much larger improvement.
>
> For 5,000 local records I would not automatically move search to the server or add debounce. But if the dataset grows to hundreds of thousands of products, I would move filtering, sorting, and pagination to the backend and use debounce to reduce unnecessary API requests.
>
> The main principle is to measure the bottleneck first and then choose the smallest optimization that solves the actual problem.

---

# Follow-Up Interview Questions

## 1. Why not use useMemo everywhere?

Because memoization itself has cost and increases code complexity.

It should primarily be used when recalculating a value is more expensive than caching and checking its dependencies.

---

## 2. Why does React.memo sometimes fail to prevent re-rendering?

Because React performs shallow prop comparison.

New object or function references are considered different.

Example:

```tsx
<ProductCard
  onSelect={() =>
    selectProduct(product)
  }
/>
```

creates a new function on every render.

---

## 3. Would useCallback improve every function?

No.

Creating ordinary JavaScript functions is generally cheap.

`useCallback` is useful when function reference stability affects another optimization or dependency.

---

## 4. What if 500,000 products exist?

Do not send all of them to the browser.

Use:

```text
Server-side filtering
Pagination
Database indexes
Debounced search
```

Example:

```http
GET /products
  ?search=iphone
  &category=electronics
  &page=1
  &limit=20
```

---

## 5. What if filtering is fast but rendering is slow?

Optimize rendering instead.

Possible approaches:

```text
Pagination
Virtualization
Reducing DOM complexity
Memoizing expensive children when justified
```

---

# Common Mistakes

## Mistake 1

```text
Page slow
→ useMemo everywhere
```

Wrong because the bottleneck may not be filtering.

---

## Mistake 2

```text
Every function
→ useCallback
```

This creates unnecessary complexity.

---

## Mistake 3

```text
Every component
→ React.memo
```

Memoization only helps under specific rendering patterns.

---

## Mistake 4

```text
Search
→ always debounce
```

Local filtering may not require debounce.

Network-based search often benefits more from it.

---

## Mistake 5

```text
Large frontend list
→ Redis
```

Redis does not solve browser rendering problems.

---

# What I Learned

This challenge taught me that React performance should be approached systematically.

The process should be:

```text
Identify symptom
↓
Profile
↓
Find bottleneck
↓
Understand render behavior
↓
Choose optimization
↓
Measure again
```

Important concepts:

```text
React re-render
Derived state
useMemo
useCallback
React.memo
Dependency arrays
Stale values
Debounce
Client-side filtering
Server-side filtering
DOM performance
Virtualization
```

The most important lesson is:

```text
Optimization should solve
a measured problem.
```

Not:

```text
Optimization because
a hook exists.
```

---

# Git Workflow

Create branch:

```bash
git checkout -b feat/day-03-react-performance
```

Suggested commits:

```bash
git commit -m "feat: add product search and category filtering"
```

```bash
git commit -m "perf: memoize derived product filtering"
```

```bash
git commit -m "perf: stabilize product selection callback"
```

```bash
git commit -m "test: add product filtering component tests"
```

---

# Progress Summary

```text
Day:
03

Topic:
React Rendering and Performance

Difficulty:
Fundamental

Main Skill:
React Rendering Optimization

Secondary Skills:
useMemo
useCallback
React.memo
Search
Filtering
Debounce
DOM Performance
Testing

Project:
Product Management UI

GitHub Folder:
day-03-react-performance

Suggested Branch:
feat/day-03-react-performance

Suggested Commits:
feat: add product search and category filtering
perf: memoize derived product filtering
perf: stabilize product selection callback
test: add product filtering component tests
```