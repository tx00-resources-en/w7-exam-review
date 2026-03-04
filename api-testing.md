
## Testing Product API Endpoints

In this lab you will add automated **integration tests** to the **products** API. You will use **Jest** as the test runner and **Supertest** to make HTTP requests against your Express app without starting a real server.

By the end of the lab you will have tests covering:

- User signup and login (Part 2)
- Full CRUD on the protected Product endpoints (Part 3)

-------

## PART 1 — Project & Test Setup

Before writing any tests, we need to install the required tooling and configure the project so Jest can run properly alongside our Express server.

### 1.1 Install test dependencies

We need two packages, both as **dev dependencies** (`-D`) since they are only used during development/testing:

| Package | Purpose |
|---------|---------|
| **jest** | JavaScript test runner — discovers test files, runs them, and reports results |
| **supertest** | Lets you make HTTP requests (GET, POST, PUT, DELETE) against an Express `app` object directly, without starting the server on a port |

```bash
npm install jest supertest cross-env -D
```

### 1.2 Add a test script

Open `package.json` and make sure you have a `test` script. We use **cross-env** to set `NODE_ENV=test` so the app can connect to a separate test database. The `--runInBand` flag tells Jest to run test files sequentially (important when tests share a database).

```json
"scripts": {
  "test": "cross-env NODE_ENV=test node --experimental-vm-modules node_modules/jest/bin/jest.js --verbose --runInBand"
}
```

> **Why `cross-env`?** Setting environment variables differs between Windows (`set VAR=val`) and Unix (`VAR=val`). `cross-env` handles this for you on any OS. If you don't already have it, install it: `npm install cross-env`

### 1.3 Configure Jest

Add a `"jest"` section to your `package.json` (or create a `jest.config.js` file). Two settings are important:

| Setting | Why |
|---------|-----|
| `testEnvironment: "node"` | Tells Jest we are testing a Node.js app (not a browser app) |
| `globalTeardown` | Points to a script that runs **after all test suites finish**, giving us a place to cleanly exit the process |

```json
"jest": {
  "testEnvironment": "node",
  "globalTeardown": "./tests/teardown.js"
}
```

Now create the teardown file:

**`tests/teardown.js`**

```js
module.exports = () => {
  process.exit(0);
};
```

> **Why do we need this?** Without it, Jest may hang after tests complete because the MongoDB connection (or other async handles) keeps the Node.js process alive. The teardown forces a clean exit.

### 1.4 Suppress deprecation warnings (optional)

Some transitive dependencies still use older versions of `glob` and `minimatch`, which print deprecation warnings. You can silence them by adding `overrides` to `package.json` (see the [package.json](./backend/package.json) in this repo for reference):

```json
"overrides": {
  "glob": "^13.0.0",
  "minimatch": "^10.2.1"
}
```

After adding overrides, run `npm install` once more so npm resolves the new versions.

### 1.5 Verify the setup

Create a quick sanity-check test to make sure Jest is wired up:

**`tests/mock.test.js`**

```js
describe("Sanity check", () => {
  it("1 + 1 = 2", () => {
    expect(1 + 1).toBe(2);
  });
});
```

Run:

```bash
npm test
```

You should see **1 test passed**. If so, the tooling is ready and you can move on to Part 2.


-------

### Part 2 — User Signup & Login Tests

In this part you will write tests for the **User** authentication endpoints. These are the first tests that hit a real database, so pay attention to how we clean up between tests to keep them isolated.

**Step 1:** Navigate to the `backend` directory if you're not there already.

**Step 2:** Create a `.env` file by copying `.env.example`:

```bash
cp .env.example .env
```

Edit `.env` and fill in your MongoDB connection strings and a strong `SECRET` for JWT signing.

> **Tip:** Use a **separate** MongoDB database for testing (e.g. `myapp_test`) so test data never interferes with your development data. You can check `NODE_ENV` inside your connection config to pick the right URI.

**Step 3:** Install dependencies and make sure the server starts:

```bash
npm install
npm run dev
```

Stop the server with `Ctrl+C` once you see "Connected to MongoDB".

**Step 4:** Verify Jest is still working:

```bash
npm test
```

You should see `mock.test.js` pass from Part 1. Leave that file as-is.

**Step 5:** Create a new test file:

```
tests/users.test.js
```

**Step 6:** Familiarize yourself with the **User model**. It has the following required fields:

| Field | Type | Notes |
|-------|------|-------|
| `fullName` | String | User's full name |
| `email` | String | Must be unique |
| `password` | String | Will be hashed by the controller |
| `phoneNumber` | String | e.g. `"+358401234567"` |
| `gender` | String | e.g. `"female"`, `"male"`, `"other"` |
| `date_of_birth` | Date | ISO date string, e.g. `"1995-06-15"` |
| `accountType` | String | e.g. `"regular"`, `"admin"` |

Every field above is **required**. Sending a request with any of them missing should result in a `400` error.

**Step 7:** At the top of your test file, import everything you'll need:

```js
const mongoose = require("mongoose");
const supertest = require("supertest");
const app = require("../app");       // your Express app (must NOT call app.listen)
const api = supertest(app);          // supertest wraps the app
const User = require("../models/userModel");
```

> **Important:** `supertest` needs the `app` object exported **without** calling `.listen()`. Make sure your `app.js` exports the app, and `index.js` (or `server.js`) is the file that calls `app.listen()`.

Add a `beforeEach` that clears the users collection before every test so each test starts with a clean slate:

```js
beforeEach(async () => {
  await User.deleteMany({});
});
```

And an `afterAll` to close the database connection when all tests finish:

```js
afterAll(async () => {
  await mongoose.connection.close();
});
```

**Step 8:** Write tests for these two endpoints:

| Method | Endpoint             | Purpose                              |
|--------|----------------------|--------------------------------------|
| POST   | `/api/users/signup`  | Register a new user (returns a JWT token) |
| POST   | `/api/users/login`   | Log in an existing user (returns a JWT token) |

For **signup** tests — use `describe("POST /api/users/signup")` with **nested** describes:

- `"when the payload is valid"` → send all required fields → expect status `201`, response body contains `token` and `email`, and user exists in the database
- `"when the payload is invalid"` → omit required fields (e.g. only send `email`) → expect `400`, response body has `error`, and user is **not** saved in the DB
- `"when the email is already taken"` → sign up the same email twice → second request should return `400`

For **login** tests — use `describe("POST /api/users/login")` with a `beforeEach` that signs up a user first so there is someone to log in:

- `"when the credentials are valid"` → send correct email & password → expect `200`, response has `token`
- `"when the credentials are invalid"` → wrong password → `400`; non-existing email → `400`

**Step 9:** Run your tests:

```bash
npm test
```

All 6 tests should pass.

---

> **Try writing the solution on your own first**, then expand to compare your work.

<details>
<summary>Sample Solution — Part 2</summary>

```js
const mongoose = require("mongoose");
const supertest = require("supertest");
const app = require("../app");
const api = supertest(app);
const User = require("../models/userModel");

// Clean the users collection before each test
beforeEach(async () => {
  await User.deleteMany({});
});

// ────────────────── POST /api/users/signup ──────────────────
describe("POST /api/users/signup", () => {
  describe("when the payload is valid", () => {
    it("should signup a new user with status 201 and return a token", async () => {
      const userData = {
        fullName: "Jane Doe",
        email: "jane@example.com",
        password: "R3g5T7#gh",
        phoneNumber: "+358401234567",
        gender: "female",
        date_of_birth: "1995-06-15",
        accountType: "regular",
      };

      const result = await api
        .post("/api/users/signup")
        .send(userData)
        .expect(201)
        .expect("Content-Type", /application\/json/);

      expect(result.body).toHaveProperty("token");
      expect(result.body.email).toBe(userData.email);

      // Verify the user was actually saved in the database
      const savedUser = await User.findOne({ email: userData.email });
      expect(savedUser).not.toBeNull();
      expect(savedUser.fullName).toBe(userData.fullName);
    });
  });

  describe("when the payload is invalid", () => {
    it("should return 400 if required fields are missing", async () => {
      const userData = {
        email: "incomplete@example.com",
      };

      const result = await api
        .post("/api/users/signup")
        .send(userData)
        .expect(400);

      expect(result.body).toHaveProperty("error");

      // Verify the user was NOT created
      const savedUser = await User.findOne({ email: userData.email });
      expect(savedUser).toBeNull();
    });
  });

  describe("when the email is already taken", () => {
    it("should return 400 for duplicate email", async () => {
      const userData = {
        fullName: "First User",
        email: "duplicate@example.com",
        password: "R3g5T7#gh",
        phoneNumber: "+358401234567",
        gender: "male",
        date_of_birth: "1990-01-20",
        accountType: "regular",
      };

      // Create the first user
      await api.post("/api/users/signup").send(userData).expect(201);

      // Try to create a second user with the same email
      const result = await api
        .post("/api/users/signup")
        .send({ ...userData, fullName: "Second User" })
        .expect(400);

      expect(result.body).toHaveProperty("error");
    });
  });
});

// ────────────────── POST /api/users/login ──────────────────
describe("POST /api/users/login", () => {
  // Sign up a user before each login test
  beforeEach(async () => {
    await api.post("/api/users/signup").send({
      fullName: "Login Tester",
      email: "login@example.com",
      password: "R3g5T7#gh",
      phoneNumber: "+358409876543",
      gender: "other",
      date_of_birth: "1988-11-30",
      accountType: "regular",
    });
  });

  describe("when the credentials are valid", () => {
    it("should login and return a token with status 200", async () => {
      const result = await api
        .post("/api/users/login")
        .send({
          email: "login@example.com",
          password: "R3g5T7#gh",
        })
        .expect(200)
        .expect("Content-Type", /application\/json/);

      expect(result.body).toHaveProperty("token");
      expect(result.body.email).toBe("login@example.com");
    });
  });

  describe("when the credentials are invalid", () => {
    it("should return 400 with a wrong password", async () => {
      const result = await api
        .post("/api/users/login")
        .send({
          email: "login@example.com",
          password: "WrongPassword1!",
        })
        .expect(400);

      expect(result.body).toHaveProperty("error");
    });

    it("should return 400 with a non-existing email", async () => {
      const result = await api
        .post("/api/users/login")
        .send({
          email: "nobody@example.com",
          password: "R3g5T7#gh",
        })
        .expect(400);

      expect(result.body).toHaveProperty("error");
    });
  });
});

// Close DB connection once after all tests
afterAll(async () => {
  await mongoose.connection.close();
});
```

**Expected output:**

```
 PASS  tests/users.test.js
  POST /api/users/signup
    when the payload is valid
      ✓ should signup a new user with status 201 and return a token
    when the payload is invalid
      ✓ should return 400 if required fields are missing
    when the email is already taken
      ✓ should return 400 for duplicate email
  POST /api/users/login
    when the credentials are valid
      ✓ should login and return a token with status 200
    when the credentials are invalid
      ✓ should return 400 with a wrong password
      ✓ should return 400 with a non-existing email
```

</details>

<!-- 
#### Commit

```bash
git add .
git commit -m "feat: add signup and login tests for users API"
``` 
-->

---

### Part 3 — Protected Product Endpoint Tests

Now you will test **all five CRUD endpoints** for products. Unlike the user endpoints, most product routes are **protected**. They require a valid JWT token in the `Authorization` header. This means your tests must first obtain a token by signing up a user, and then include it with each protected request.

**Step 1:** Create a new test file:

```
tests/products.test.js
```

**Step 2:** Study the route protection in `routes/productRouter.js`. Notice the pattern:

```js
// Public routes (no auth required)
router.get("/",      getAllProducts);
router.get("/:productId", getProductById);

// Auth middleware — everything below requires a valid token
router.use(requireAuth);

// Protected routes
router.post("/",           createProduct);
router.put("/:productId",  updateProduct);
router.delete("/:productId", deleteProduct);
```

This means: **no token is needed for GET requests**, but POST, PUT, and DELETE will return `401 Unauthorized` if you don't send one.

**Step 3:** Set up the file structure. At the top import everything you need:

```js
const mongoose = require("mongoose");
const supertest = require("supertest");
const app = require("../app");
const api = supertest(app);
const Product = require("../models/productModel");
const User = require("../models/userModel");
```

Then prepare your test infrastructure:

- **Seed data** — Create an array of two product objects that match the Product model. Each product needs: `productName`, `category`, `description`, `price`, `inventoryCount`, and a `supplier` object with `name`, `contactEmail`, `contactPhone`, and `isVerified`.
- **`productsInDb()` helper** — A small async function that reads all products from the DB and returns them as plain JSON. This is useful for checking the database state after operations:
  ```js
  const productsInDb = async () => {
    const allProducts = await Product.find({});
    return allProducts.map((p) => p.toJSON());
  };
  ```
- **`beforeAll`** — Delete all users, then sign up one user and store the returned `token` in a variable accessible to all tests.
- **Wrap tests in `describe("Product Routes", () => { ... })`** — Inside, add a `beforeEach` that deletes all products and re-seeds two products via the API (using the token).
- **`afterAll`** — Close the mongoose connection.

**Step 4:** Write tests for all five endpoints:

| Method | Endpoint                    | Auth required? | Success status |
|--------|-----------------------------|----------------|----------------|
| GET    | `/api/products`             | No (public)    | `200` |
| GET    | `/api/products/:productId`  | No (public)    | `200` |
| POST   | `/api/products`             | Yes            | `201` |
| PUT    | `/api/products/:productId`  | Yes            | `200` |
| DELETE | `/api/products/:productId`  | Yes            | `204` |

For **GET /api/products** (public):
- Expect `200` and JSON content type
- Response body length should equal the number of seeded products

For **GET /api/products/:productId** (public):
- Valid ID → expect `200`, response `productName` matches the DB record
- Non-existing valid ObjectId → expect `404`
- Invalid ID format (e.g. `"12345"`) → expect `404`

For each **protected endpoint** (POST, PUT, DELETE), write nested describes:

- `"when the user is authenticated"` — include the token via `.set("Authorization", "Bearer " + token)`, expect success
- `"when the user is not authenticated"` — omit the token, expect `401`

Additional checks:
- **POST**: After a successful create, the DB should have `seeds.length + 1` products
- **POST** without auth: The DB count should remain unchanged
- **PUT**: After a successful update, re-read the product from the DB and verify the fields changed
- **DELETE**: After a successful delete, the DB should have one fewer product, and the deleted product's name should no longer appear

For **invalid IDs** on PUT and DELETE, send a malformed ID with the token and expect `404`.

**Step 5:** Run your tests:

```bash
npm test
```

You should see 12 passing tests for products (plus the 6 user tests from Part 2).

---

> **Try writing the solution on your own first**, then expand to compare your work.

<details>
<summary>Sample Solution — Part 3</summary>

```js
const mongoose = require("mongoose");
const supertest = require("supertest");
const app = require("../app");
const api = supertest(app);
const Product = require("../models/productModel");
const User = require("../models/userModel");

// Seed data
const products = [
  {
    productName: "Wireless Mouse",
    category: "Electronics",
    description: "Ergonomic wireless mouse with USB receiver",
    price: 29.99,
    inventoryCount: 150,
    supplier: {
      name: "TechSupply Co.",
      contactEmail: "sales@techsupply.com",
      contactPhone: "+358401112233",
      isVerified: true,
    },
  },
  {
    productName: "Standing Desk",
    category: "Furniture",
    description: "Adjustable height standing desk, 160cm wide",
    price: 499.95,
    inventoryCount: 30,
    supplier: {
      name: "OfficePro Ltd.",
      contactEmail: "orders@officepro.com",
      contactPhone: "+358409998877",
      isVerified: false,
    },
  },
];

// Helper: read all products straight from DB
const productsInDb = async () => {
  const allProducts = await Product.find({});
  return allProducts.map((p) => p.toJSON());
};

let token = null;

// Create a user and get a token before all tests
beforeAll(async () => {
  await User.deleteMany({});
  const result = await api.post("/api/users/signup").send({
    fullName: "John Doe",
    email: "john@example.com",
    password: "R3g5T7#gh",
    phoneNumber: "+358401234567",
    gender: "male",
    date_of_birth: "1992-03-10",
    accountType: "regular",
  });
  token = result.body.token;
});

describe("Product Routes", () => {
  // Seed products via the API before each test
  beforeEach(async () => {
    await Product.deleteMany({});
    await api
      .post("/api/products")
      .set("Authorization", "Bearer " + token)
      .send(products[0]);

    await api
      .post("/api/products")
      .set("Authorization", "Bearer " + token)
      .send(products[1]);
  });

  // ────────────────── GET /api/products (public) ──────────────────
  describe("GET /api/products", () => {
    it("should return all products as JSON with status 200", async () => {
      const response = await api
        .get("/api/products")
        .expect(200)
        .expect("Content-Type", /application\/json/);

      expect(response.body).toHaveLength(products.length);
    });
  });

  // ────────────────── GET /api/products/:productId (public) ──────────────────
  describe("GET /api/products/:productId", () => {
    it("should return one product by ID", async () => {
      const product = await Product.findOne();
      const response = await api
        .get(`/api/products/${product._id}`)
        .expect(200)
        .expect("Content-Type", /application\/json/);

      expect(response.body.productName).toBe(product.productName);
    });

    it("should return 404 for a non-existing product ID", async () => {
      const nonExistentId = new mongoose.Types.ObjectId();
      await api.get(`/api/products/${nonExistentId}`).expect(404);
    });

    it("should return 404 for an invalid product ID format", async () => {
      const invalidId = "12345";
      await api.get(`/api/products/${invalidId}`).expect(404);
    });
  });

  // ────────────────── POST /api/products (protected) ──────────────────
  describe("POST /api/products", () => {
    describe("when the user is authenticated", () => {
      it("should create a new product with status 201", async () => {
        const newProduct = {
          productName: "Mechanical Keyboard",
          category: "Electronics",
          description: "RGB mechanical keyboard with Cherry MX switches",
          price: 129.99,
          inventoryCount: 75,
          supplier: {
            name: "KeyboardWorld",
            contactEmail: "info@keyboardworld.com",
            contactPhone: "+358405556677",
            isVerified: true,
          },
        };

        const response = await api
          .post("/api/products")
          .set("Authorization", "Bearer " + token)
          .send(newProduct)
          .expect(201);

        expect(response.body.productName).toBe(newProduct.productName);

        const productsAtEnd = await productsInDb();
        expect(productsAtEnd).toHaveLength(products.length + 1);
      });
    });

    describe("when the user is not authenticated", () => {
      it("should return 401 if no token is provided", async () => {
        const newProduct = {
          productName: "Ghost Product",
          category: "Unknown",
          description: "Should not be created",
          price: 0,
          inventoryCount: 0,
          supplier: {
            name: "Nobody",
            contactEmail: "no@one.com",
            contactPhone: "+000000000",
            isVerified: false,
          },
        };

        await api.post("/api/products").send(newProduct).expect(401);

        const productsAtEnd = await productsInDb();
        expect(productsAtEnd).toHaveLength(products.length);
      });
    });
  });

  // ────────────────── PUT /api/products/:productId (protected) ──────────────────
  describe("PUT /api/products/:productId", () => {
    describe("when the user is authenticated", () => {
      it("should update the product and return the updated document", async () => {
        const product = await Product.findOne();
        const updates = {
          productName: "Updated Product Name",
          price: 39.99,
        };

        const response = await api
          .put(`/api/products/${product._id}`)
          .set("Authorization", "Bearer " + token)
          .send(updates)
          .expect(200)
          .expect("Content-Type", /application\/json/);

        expect(response.body.productName).toBe(updates.productName);

        const updatedProduct = await Product.findById(product._id);
        expect(updatedProduct.price).toBe(updates.price);
      });
    });

    describe("when the user is not authenticated", () => {
      it("should return 401 if no token is provided", async () => {
        const product = await Product.findOne();
        await api
          .put(`/api/products/${product._id}`)
          .send({ productName: "Nope" })
          .expect(401);
      });
    });

    describe("when the id is invalid", () => {
      it("should return 404 for an invalid ID format", async () => {
        const invalidId = "12345";
        await api
          .put(`/api/products/${invalidId}`)
          .set("Authorization", "Bearer " + token)
          .send({})
          .expect(404);
      });
    });
  });

  // ────────────────── DELETE /api/products/:productId (protected) ──────────────────
  describe("DELETE /api/products/:productId", () => {
    describe("when the user is authenticated", () => {
      it("should delete the product and return status 204", async () => {
        const productsAtStart = await productsInDb();
        const productToDelete = productsAtStart[0];

        await api
          .delete(`/api/products/${productToDelete.id}`)
          .set("Authorization", "Bearer " + token)
          .expect(204);

        const productsAtEnd = await productsInDb();
        expect(productsAtEnd).toHaveLength(productsAtStart.length - 1);
        expect(productsAtEnd.map((p) => p.productName)).not.toContain(
          productToDelete.productName
        );
      });
    });

    describe("when the user is not authenticated", () => {
      it("should return 401 if no token is provided", async () => {
        const product = await Product.findOne();
        await api.delete(`/api/products/${product._id}`).expect(401);
      });
    });

    describe("when the id is invalid", () => {
      it("should return 404 for an invalid ID format", async () => {
        const invalidId = "12345";
        await api
          .delete(`/api/products/${invalidId}`)
          .set("Authorization", "Bearer " + token)
          .expect(404);
      });
    });
  });
});

// Close DB connection once after all tests
afterAll(async () => {
  await mongoose.connection.close();
});
```

**Expected output:**

```
 PASS  tests/products.test.js
  Product Routes
    GET /api/products
      ✓ should return all products as JSON with status 200
    GET /api/products/:productId
      ✓ should return one product by ID
      ✓ should return 404 for a non-existing product ID
      ✓ should return 404 for an invalid product ID format
    POST /api/products
      when the user is authenticated
        ✓ should create a new product with status 201
      when the user is not authenticated
        ✓ should return 401 if no token is provided
    PUT /api/products/:productId
      when the user is authenticated
        ✓ should update the product and return the updated document
      when the user is not authenticated
        ✓ should return 401 if no token is provided
      when the id is invalid
        ✓ should return 404 for an invalid ID format
    DELETE /api/products/:productId
      when the user is authenticated
        ✓ should delete the product and return status 204
      when the user is not authenticated
        ✓ should return 401 if no token is provided
      when the id is invalid
        ✓ should return 404 for an invalid ID format
```

</details>

<!--
 #### Commit

```bash
git add .
git commit -m "feat: add protected product API tests with auth"
``` 
-->

---

### Full Solution

<details>
<summary>Full Solution — tests/users.test.js</summary>

```js
const mongoose = require("mongoose");
const supertest = require("supertest");
const app = require("../app");
const api = supertest(app);
const User = require("../models/userModel");

beforeEach(async () => {
  await User.deleteMany({});
});

describe("POST /api/users/signup", () => {
  describe("when the payload is valid", () => {
    it("should signup a new user with status 201 and return a token", async () => {
      const userData = {
        fullName: "Jane Doe",
        email: "jane@example.com",
        password: "R3g5T7#gh",
        phoneNumber: "+358401234567",
        gender: "female",
        date_of_birth: "1995-06-15",
        accountType: "regular",
      };

      const result = await api
        .post("/api/users/signup")
        .send(userData)
        .expect(201)
        .expect("Content-Type", /application\/json/);

      expect(result.body).toHaveProperty("token");
      expect(result.body.email).toBe(userData.email);

      const savedUser = await User.findOne({ email: userData.email });
      expect(savedUser).not.toBeNull();
      expect(savedUser.fullName).toBe(userData.fullName);
    });
  });

  describe("when the payload is invalid", () => {
    it("should return 400 if required fields are missing", async () => {
      const userData = {
        email: "incomplete@example.com",
      };

      const result = await api
        .post("/api/users/signup")
        .send(userData)
        .expect(400);

      expect(result.body).toHaveProperty("error");

      const savedUser = await User.findOne({ email: userData.email });
      expect(savedUser).toBeNull();
    });
  });

  describe("when the email is already taken", () => {
    it("should return 400 for duplicate email", async () => {
      const userData = {
        fullName: "First User",
        email: "duplicate@example.com",
        password: "R3g5T7#gh",
        phoneNumber: "+358401234567",
        gender: "male",
        date_of_birth: "1990-01-20",
        accountType: "regular",
      };

      await api.post("/api/users/signup").send(userData).expect(201);

      const result = await api
        .post("/api/users/signup")
        .send({ ...userData, fullName: "Second User" })
        .expect(400);

      expect(result.body).toHaveProperty("error");
    });
  });
});

describe("POST /api/users/login", () => {
  beforeEach(async () => {
    await api.post("/api/users/signup").send({
      fullName: "Login Tester",
      email: "login@example.com",
      password: "R3g5T7#gh",
      phoneNumber: "+358409876543",
      gender: "other",
      date_of_birth: "1988-11-30",
      accountType: "regular",
    });
  });

  describe("when the credentials are valid", () => {
    it("should login and return a token with status 200", async () => {
      const result = await api
        .post("/api/users/login")
        .send({
          email: "login@example.com",
          password: "R3g5T7#gh",
        })
        .expect(200)
        .expect("Content-Type", /application\/json/);

      expect(result.body).toHaveProperty("token");
      expect(result.body.email).toBe("login@example.com");
    });
  });

  describe("when the credentials are invalid", () => {
    it("should return 400 with a wrong password", async () => {
      const result = await api
        .post("/api/users/login")
        .send({
          email: "login@example.com",
          password: "WrongPassword1!",
        })
        .expect(400);

      expect(result.body).toHaveProperty("error");
    });

    it("should return 400 with a non-existing email", async () => {
      const result = await api
        .post("/api/users/login")
        .send({
          email: "nobody@example.com",
          password: "R3g5T7#gh",
        })
        .expect(400);

      expect(result.body).toHaveProperty("error");
    });
  });
});

afterAll(async () => {
  await mongoose.connection.close();
});
```

</details>

<details>
<summary>Full Solution — tests/products.test.js</summary>

```js
const mongoose = require("mongoose");
const supertest = require("supertest");
const app = require("../app");
const api = supertest(app);
const Product = require("../models/productModel");
const User = require("../models/userModel");

const products = [
  {
    productName: "Wireless Mouse",
    category: "Electronics",
    description: "Ergonomic wireless mouse with USB receiver",
    price: 29.99,
    inventoryCount: 150,
    supplier: {
      name: "TechSupply Co.",
      contactEmail: "sales@techsupply.com",
      contactPhone: "+358401112233",
      isVerified: true,
    },
  },
  {
    productName: "Standing Desk",
    category: "Furniture",
    description: "Adjustable height standing desk, 160cm wide",
    price: 499.95,
    inventoryCount: 30,
    supplier: {
      name: "OfficePro Ltd.",
      contactEmail: "orders@officepro.com",
      contactPhone: "+358409998877",
      isVerified: false,
    },
  },
];

const productsInDb = async () => {
  const allProducts = await Product.find({});
  return allProducts.map((p) => p.toJSON());
};

let token = null;

beforeAll(async () => {
  await User.deleteMany({});
  const result = await api.post("/api/users/signup").send({
    fullName: "John Doe",
    email: "john@example.com",
    password: "R3g5T7#gh",
    phoneNumber: "+358401234567",
    gender: "male",
    date_of_birth: "1992-03-10",
    accountType: "regular",
  });
  token = result.body.token;
});

describe("Product Routes", () => {
  beforeEach(async () => {
    await Product.deleteMany({});
    await api
      .post("/api/products")
      .set("Authorization", "Bearer " + token)
      .send(products[0]);

    await api
      .post("/api/products")
      .set("Authorization", "Bearer " + token)
      .send(products[1]);
  });

  describe("GET /api/products", () => {
    it("should return all products as JSON with status 200", async () => {
      const response = await api
        .get("/api/products")
        .expect(200)
        .expect("Content-Type", /application\/json/);

      expect(response.body).toHaveLength(products.length);
    });
  });

  describe("GET /api/products/:productId", () => {
    it("should return one product by ID", async () => {
      const product = await Product.findOne();
      const response = await api
        .get(`/api/products/${product._id}`)
        .expect(200)
        .expect("Content-Type", /application\/json/);

      expect(response.body.productName).toBe(product.productName);
    });

    it("should return 404 for a non-existing product ID", async () => {
      const nonExistentId = new mongoose.Types.ObjectId();
      await api.get(`/api/products/${nonExistentId}`).expect(404);
    });

    it("should return 404 for an invalid product ID format", async () => {
      const invalidId = "12345";
      await api.get(`/api/products/${invalidId}`).expect(404);
    });
  });

  describe("POST /api/products", () => {
    describe("when the user is authenticated", () => {
      it("should create a new product with status 201", async () => {
        const newProduct = {
          productName: "Mechanical Keyboard",
          category: "Electronics",
          description: "RGB mechanical keyboard with Cherry MX switches",
          price: 129.99,
          inventoryCount: 75,
          supplier: {
            name: "KeyboardWorld",
            contactEmail: "info@keyboardworld.com",
            contactPhone: "+358405556677",
            isVerified: true,
          },
        };

        const response = await api
          .post("/api/products")
          .set("Authorization", "Bearer " + token)
          .send(newProduct)
          .expect(201);

        expect(response.body.productName).toBe(newProduct.productName);

        const productsAtEnd = await productsInDb();
        expect(productsAtEnd).toHaveLength(products.length + 1);
      });
    });

    describe("when the user is not authenticated", () => {
      it("should return 401 if no token is provided", async () => {
        const newProduct = {
          productName: "Ghost Product",
          category: "Unknown",
          description: "Should not be created",
          price: 0,
          inventoryCount: 0,
          supplier: {
            name: "Nobody",
            contactEmail: "no@one.com",
            contactPhone: "+000000000",
            isVerified: false,
          },
        };

        await api.post("/api/products").send(newProduct).expect(401);

        const productsAtEnd = await productsInDb();
        expect(productsAtEnd).toHaveLength(products.length);
      });
    });
  });

  describe("PUT /api/products/:productId", () => {
    describe("when the user is authenticated", () => {
      it("should update the product and return the updated document", async () => {
        const product = await Product.findOne();
        const updates = {
          productName: "Updated Product Name",
          price: 39.99,
        };

        const response = await api
          .put(`/api/products/${product._id}`)
          .set("Authorization", "Bearer " + token)
          .send(updates)
          .expect(200)
          .expect("Content-Type", /application\/json/);

        expect(response.body.productName).toBe(updates.productName);

        const updatedProduct = await Product.findById(product._id);
        expect(updatedProduct.price).toBe(updates.price);
      });
    });

    describe("when the user is not authenticated", () => {
      it("should return 401 if no token is provided", async () => {
        const product = await Product.findOne();
        await api
          .put(`/api/products/${product._id}`)
          .send({ productName: "Nope" })
          .expect(401);
      });
    });

    describe("when the id is invalid", () => {
      it("should return 404 for an invalid ID format", async () => {
        const invalidId = "12345";
        await api
          .put(`/api/products/${invalidId}`)
          .set("Authorization", "Bearer " + token)
          .send({})
          .expect(404);
      });
    });
  });

  describe("DELETE /api/products/:productId", () => {
    describe("when the user is authenticated", () => {
      it("should delete the product and return status 204", async () => {
        const productsAtStart = await productsInDb();
        const productToDelete = productsAtStart[0];

        await api
          .delete(`/api/products/${productToDelete.id}`)
          .set("Authorization", "Bearer " + token)
          .expect(204);

        const productsAtEnd = await productsInDb();
        expect(productsAtEnd).toHaveLength(productsAtStart.length - 1);
        expect(productsAtEnd.map((p) => p.productName)).not.toContain(
          productToDelete.productName
        );
      });
    });

    describe("when the user is not authenticated", () => {
      it("should return 401 if no token is provided", async () => {
        const product = await Product.findOne();
        await api.delete(`/api/products/${product._id}`).expect(401);
      });
    });

    describe("when the id is invalid", () => {
      it("should return 404 for an invalid ID format", async () => {
        const invalidId = "12345";
        await api
          .delete(`/api/products/${invalidId}`)
          .set("Authorization", "Bearer " + token)
          .expect(404);
      });
    });
  });
});

afterAll(async () => {
  await mongoose.connection.close();
});
```

</details>

---------
