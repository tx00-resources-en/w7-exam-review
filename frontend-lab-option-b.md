# Frontend Activity — Product Store (Beginner-Friendly)

> [!NOTE]  
> This is **option-B** of the frontend lab. It contains the same goals and specifications as *option-A*, with the difference that this version provides code solutions.

----

**Part 1 (Iterations 0–5)** — Build a full CRUD front-end:
- A clear, **step‑by‑step walkthrough** for building the app.
- By the end of Part 1, you will have a fully functional React Frontend connected to an API.

**Part 2 (Iterations 6–7)** — Add authentication and route protection:
- **Iteration 6:** User signup & login with inline `useState` and `fetch`, JWT stored in `localStorage`.
- **Iteration 7:** Protected routes using `<Navigate>`, `Authorization: Bearer <token>` headers on POST/PUT/DELETE, and conditional rendering based on authentication state.

**Part 3 (Iteration 8)** — Refactor with custom hooks:
- **Iteration 8:** Extract reusable logic into custom hooks (`useField`, `useSignup`, `useLogin`). Replace repetitive `useState` + `onChange` pairs with `useField` and inline `fetch` calls with `useSignup` / `useLogin`.

----

## Part 1: Overview

In this part you will connect a **React** front-end to an **Express + MongoDB** back-end.  
By the end you will have a working app that can **Create, Read, Update and Delete** (CRUD) products — all through regular (non-protected) API routes. No authentication is involved.

> **Prerequisite:** Complete the **backend lab** first. This frontend lab uses the backend you built there. You will need **two versions** of your backend:
>
> 1. **`backend-crud/`** — Your backend after completing **Iteration 6** of the backend lab (has user signup/login but **no** route protection). Use this for **Part 1** and **Part 2 Iteration 6**.
> 2. **`backend/`** — Your backend after completing **Iteration 7** of the backend lab (has signup/login **and** `requireAuth` middleware on product routes). Use this for **Part 2 Iteration 7** and **Part 3**.
>
> If you followed the backend lab instructions, you saved a copy of your backend before adding route protection. That copy is `backend-crud/`.

**Getting started:**

1. Make sure your backend copy (`backend-crud/`) is running on `http://localhost:4000`.
2. Open a **second terminal** for the frontend, navigate to the `frontend/` folder, run `npm install`, then `npm run dev`.


### What You Will Learn

- How to send HTTP requests (`GET`, `POST`, `PUT`, `DELETE`) from React to an Express API using `fetch`.
- How to use **controlled inputs** (form values stored in React state with `useState`).
- How to fetch data when a component mounts using `useEffect`.
- How to use React Router (`Routes`, `Route`, `Link`, `useParams`, `useNavigate`) for navigation.
- How to structure a React app with **pages** and **components**.

### Activity Structure

There are **5 iterations** (plus a setup step). Each iteration adds one CRUD feature:

| Iteration | Feature | HTTP Method | Files You Will Change |
|---|---|---|---|
| 0 | Setup | — | — |
| 1 | Add a product | `POST` | `AddProductPage.jsx` |
| 2 | List all products | `GET` | `HomePage.jsx`, `ProductListings.jsx`, `ProductListing.jsx` |
| 3 | View one product | `GET` | `App.jsx`, `ProductListing.jsx`, `ProductPage.jsx` |
| 4 | Delete a product | `DELETE` | `ProductPage.jsx` |
| 5 | Edit a product | `PUT` | `App.jsx`, `ProductPage.jsx`, `EditProductPage.jsx` |

> **Important:** Commit your work after each iteration.


### Commit Messages (Best Practice)

Use small commits that describe *what* changed. Recommended format:

- `feat(add-product): send POST request from AddProductPage form`
- `feat(list-products): fetch and display all products on HomePage`
- `refactor(product-listing): accept product prop and display data`
- `chore: install dependencies`

Rule of thumb: one commit = one idea you can explain in one sentence.


## The Backend API (Reference)

The backend is already built. Here is everything you need to know about it.

**Base URL:** `http://localhost:4000`  
(The Vite proxy in `vite.config.js` forwards any request starting with `/api` to this URL, so in your React code you only write `/api/products`.)

| Method | Endpoint | Description | Request Body |
|---|---|---|---|
| `POST` | `/api/products` | Create a new product | JSON (see below) |
| `GET` | `/api/products` | Get all products | — |
| `GET` | `/api/products/:productId` | Get a single product by ID | — |
| `PUT` | `/api/products/:productId` | Update a product by ID | JSON (see below) |
| `DELETE` | `/api/products/:productId` | Delete a product by ID | — |

**Product JSON shape** (what the API expects and returns):

```json
{
  "productName": "Wireless Headphones",
  "category": "Electronics",
  "description": "High-quality wireless headphones with noise cancellation",
  "price": 149.99,
  "inventoryCount": 50,
  "supplier": {
    "name": "TechSupplies Inc.",
    "contactEmail": "sales@techsupplies.com",
    "contactPhone": "555-123-4567",
    "isVerified": true
  }
}
```

> **Tip:** You can test the API with a tool such as VS Code REST Client, Postman, or `curl` before writing any React code. That way you will know exactly what the API returns.


## Instructions

### Iteration 1: Add a Product (`POST`)

**Goal:** Make the "Add Product" form actually save a new product to the database.

**File to change:** `src/pages/AddProductPage.jsx`

Right now the form is there but nothing happens when you press "Add Product" — the `submitForm` function only logs to the console and the inputs are not connected to state.

**What to do:**

1. **Create state for each form field** using `useState`:
   ```jsx
   const [productName, setProductName] = useState("");
   const [category, setCategory] = useState("Electronics");
   const [description, setDescription] = useState("");
   const [price, setPrice] = useState("");
   const [inventoryCount, setInventoryCount] = useState("");
   const [supplierName, setSupplierName] = useState("");
   const [contactEmail, setContactEmail] = useState("");
   const [contactPhone, setContactPhone] = useState("");
   const [isVerified, setIsVerified] = useState("true");
   ```

2. **Connect each input to its state** (controlled inputs). For example:
   ```jsx
   <input
     type="text"
     required
     value={productName}
     onChange={(e) => setProductName(e.target.value)}
   />
   ```
   Do the same for every `<input>`, `<select>`, and `<textarea>` in the form.

3. **Write an `addProduct` function** that sends a POST request:
   ```jsx
   const addProduct = async (newProduct) => {
     try {
       const res = await fetch("/api/products", {
         method: "POST",
         headers: { "Content-Type": "application/json" },
         body: JSON.stringify(newProduct),
       });
       if (!res.ok) throw new Error("Failed to add product");
     } catch (error) {
       console.error(error);
     }
   };
   ```

4. **Update `submitForm`** to build a product object from the state and call `addProduct`, then navigate home:
   ```jsx
   import { useNavigate } from "react-router-dom";
   // inside the component:
   const navigate = useNavigate();

   const submitForm = (e) => {
     e.preventDefault();
     const newProduct = {
       productName,
       category,
       description,
       price: parseFloat(price),
       inventoryCount: parseInt(inventoryCount, 10),
       supplier: {
         name: supplierName,
         contactEmail,
         contactPhone,
         isVerified: isVerified === "true",
       },
     };
     addProduct(newProduct);
     navigate("/");
   };
   ```

   > **Note:** The `price` and `inventoryCount` state are strings because HTML input values are always strings. We convert them to numbers with `parseFloat()` and `parseInt()` when building the request body. The `isVerified` field is converted from a string to a boolean.


**You are done with Iteration 1 when:**

- You fill in the form and click "Add Product".
- The page navigates back to Home.
- If you check MongoDB (e.g., MongoDB Compass or the API with `GET /api/products`), the new product is there.

> **Note:** The home page does not show products yet — that is the next iteration.


### Iteration 2: Fetch and Display All Products (`GET`)

**Goal:** When the Home page loads, fetch all products from the API and display them as a list.

**Files to change:** `src/pages/HomePage.jsx`, `src/components/ProductListings.jsx`, `src/components/ProductListing.jsx`

#### Step A — Fetch products in `HomePage.jsx`

1. Import `useState` and `useEffect` from React.
2. Create three pieces of state:
   ```jsx
   const [products, setProducts] = useState(null);
   const [isPending, setIsPending] = useState(true);
   const [error, setError] = useState(null);
   ```
3. Use `useEffect` to fetch products when the component mounts:
   ```jsx
   useEffect(() => {
     const fetchProducts = async () => {
       try {
         const res = await fetch("/api/products");
         if (!res.ok) throw new Error("Could not fetch products");
         const data = await res.json();
         setProducts(data);
         setIsPending(false);
       } catch (err) {
         setError(err.message);
         setIsPending(false);
       }
     };
     fetchProducts();
   }, []);
   ```
4. Render loading, error, and success states:
   ```jsx
   return (
     <div className="home">
       {error && <div>{error}</div>}
       {isPending && <div>Loading...</div>}
       {products && <ProductListings products={products} />}
     </div>
   );
   ```

#### Step B — Accept and map products in `ProductListings.jsx`

The component currently renders a single hard-coded `<ProductListing />`. Change it to accept a `products` prop and map over the array:

```jsx
const ProductListings = ({ products }) => {
  return (
    <div className="product-list">
      {products.map((product) => (
        <ProductListing key={product.id} product={product} />
      ))}
    </div>
  );
};
```

#### Step C — Display product data in `ProductListing.jsx`

Accept a `product` prop and display the actual values:

```jsx
const ProductListing = ({ product }) => {
  return (
    <div className="product-preview">
      <h2>{product.productName}</h2>
      <p>Category: {product.category}</p>
      <p>Price: ${product.price.toFixed(2)}</p>
      <p>Inventory Count: {product.inventoryCount}</p>
    </div>
  );
};
```


**Discuss:**

- Where is the products state stored?
- When does `fetch` run? (Hint: only once, on mount — because of the `[]` dependency array.)
- How do you handle loading and error states?

**You are done with Iteration 2 when:**

- The Home page shows all products from the database.
- When you add a new product (Iteration 1) and navigate back to Home, the new product appears in the list (the page re-fetches on mount).

### Iteration 3: View a Single Product (`GET` one)

**Goal:** Click a product in the list to open a detail page that shows all its information.

**Files to change:** `src/App.jsx`, `src/components/ProductListing.jsx`, `src/pages/ProductPage.jsx`

#### Step A — Add a new route in `App.jsx`

Import `ProductPage` and add a route for it:

```jsx
import ProductPage from "./pages/ProductPage";
// inside <Routes>:
<Route path="/products/:id" element={<ProductPage />} />
```

The `:id` is a **URL parameter**. React Router will match URLs like `/products/abc123` and make `abc123` available via the `useParams` hook.

#### Step B — Link each product to its detail page in `ProductListing.jsx`

Import `Link` from `react-router-dom` and wrap the product name:

```jsx
import { Link } from "react-router-dom";

// inside the return:
<Link to={`/products/${product.id}`}>
  <h2>{product.productName}</h2>
</Link>
```

> **Why `Link` instead of `<a href>`?** `<Link>` navigates without a full page reload, keeping React state alive. `<a href>` reloads the entire page.

#### Step C — Fetch and display the product in `ProductPage.jsx`

1. Import `useParams`, `useNavigate`, `useEffect`, and `useState`.
2. Get the `id` from the URL and fetch the single product:
   ```jsx
   const { id } = useParams();
   const [product, setProduct] = useState(null);
   const [loading, setLoading] = useState(true);
   const [error, setError] = useState(null);

   useEffect(() => {
     const fetchProduct = async () => {
       try {
         const res = await fetch(`/api/products/${id}`);
         if (!res.ok) throw new Error("Network response was not ok");
         const data = await res.json();
         setProduct(data);
       } catch (err) {
         setError(err.message);
       } finally {
         setLoading(false);
       }
     };
     fetchProduct();
   }, [id]);
   ```
3. Display loading, error, and product data (including supplier details).
4. Add a "Back" button:
   ```jsx
   const navigate = useNavigate();
   // inside the return:
   <button onClick={() => navigate("/")}>Back</button>
   ```


**You are done with Iteration 3 when:**

- You can click a product title on the Home page and see all its details on a dedicated page.
- The "Back" button returns you to the Home page.

**Discussion Questions:**

- What is the difference between a **page** and a **component** in this app?
- Why do we pass `[id]` as the dependency array in `useEffect`?


### Iteration 4: Delete a Product (`DELETE`)

**Goal:** Add a "Delete" button to the product detail page that removes the product from the database.

**File to change:** `src/pages/ProductPage.jsx`

The route and the detail page already exist from Iteration 3. You only need to add delete functionality.

**What to do:**

1. **Write a `deleteProduct` function:**
   ```jsx
   const deleteProduct = async (productId) => {
     try {
       const res = await fetch(`/api/products/${productId}`, {
         method: "DELETE",
       });
       if (!res.ok) throw new Error("Failed to delete product");
     } catch (error) {
       console.error("Error deleting product:", error);
     }
   };
   ```

2. **Write a click handler** with a confirmation dialog:
   ```jsx
   const onDeleteClick = (productId) => {
     const confirm = window.confirm("Are you sure you want to delete this product?");
     if (!confirm) return;
     deleteProduct(productId);
     navigate("/");
   };
   ```

3. **Add a "Delete" button** in the JSX (next to or instead of the "Back" button):
   ```jsx
   <button onClick={() => onDeleteClick(product._id)}>Delete</button>
   ```


**You are done with Iteration 4 when:**

- You click "Delete" on a product detail page.
- A confirmation dialog appears.
- After confirming, the app navigates to Home and the deleted product is no longer in the list.

### Iteration 5: Edit a Product (`PUT`)

**Goal:** Add an "Edit" button on the product detail page that opens a pre-filled form. Submitting the form updates the product in the database.

**Files to change:** `src/App.jsx`, `src/pages/ProductPage.jsx`, `src/pages/EditProductPage.jsx`

#### Step A — Add a new route in `App.jsx`

Import `EditProductPage` and add a route:

```jsx
import EditProductPage from "./pages/EditProductPage";
// inside <Routes>:
<Route path="/edit-product/:id" element={<EditProductPage />} />
```

#### Step B — Add an "Edit" button in `ProductPage.jsx`

Add a button that navigates to the edit page:

```jsx
<button onClick={() => navigate(`/edit-product/${product._id}`)}>Edit</button>
```

#### Step C — Build the edit form in `EditProductPage.jsx`

This is the most complex page. It combines patterns you already used in earlier iterations:

1. **Get the `id` from the URL** with `useParams` (same pattern as `ProductPage`).
2. **Fetch the existing product** with `useEffect` (same pattern as `ProductPage`).
3. **Create state for each form field** with `useState` (same pattern as `AddProductPage`).
4. **Pre-fill the form** — after fetching, set each state variable to the fetched value:
   ```jsx
   useEffect(() => {
     const fetchProduct = async () => {
       const res = await fetch(`/api/products/${id}`);
       const data = await res.json();
       setProductName(data.productName);
       setCategory(data.category);
       setDescription(data.description);
       setPrice(data.price.toString());
       setInventoryCount(data.inventoryCount.toString());
       setSupplierName(data.supplier.name);
       setContactEmail(data.supplier.contactEmail);
       setContactPhone(data.supplier.contactPhone);
       setIsVerified(data.supplier.isVerified ? "true" : "false");
     };
     fetchProduct();
   }, [id]);
   ```
5. **Write an `updateProduct` function** that sends a PUT request:
   ```jsx
   const updateProduct = async (updatedProduct) => {
     try {
       const res = await fetch(`/api/products/${id}`, {
         method: "PUT",
         headers: { "Content-Type": "application/json" },
         body: JSON.stringify(updatedProduct),
       });
       if (!res.ok) throw new Error("Failed to update product");
       return true;
     } catch (error) {
       console.error("Error updating product:", error);
       return false;
     }
   };
   ```
6. **Handle form submission:**
   ```jsx
   const submitForm = (e) => {
     e.preventDefault();
     const updatedProduct = {
       productName,
       category,
       description,
       price: parseFloat(price),
       inventoryCount: parseInt(inventoryCount, 10),
       supplier: {
         name: supplierName,
         contactEmail,
         contactPhone,
         isVerified: isVerified === "true",
       },
     };
     updateProduct(updatedProduct);
     navigate(`/products/${id}`);
   };
   ```
7. **Render the form** — this is almost identical to `AddProductPage`, but with an "Update Product" button and a loading check before the form.


**You are done with Iteration 5 when:**

- You can click "Edit" on a product detail page.
- The edit form opens with the current values pre-filled.
- After submitting, you are redirected to the detail page showing the updated data.
- The updated data also appears correctly in the products list on the Home page.


--------

## Part 2: Part 2 (Authentication & Route Protection)

In Part 1 you built a full CRUD front-end for products using the **unprotected** backend.
In Part 2 you will add **user registration & login** (Iteration 6) and then **protect routes** so that only logged-in users can create, edit, or delete products (Iteration 7).

**Backend versions you will use:**

| Version | Description | Used In |
|---|---|---|
| `backend-crud/` | Your backend after backend lab Iteration 6 — has user signup/login but **no** route protection. | Iteration 6 |
| `backend/` | Your backend after backend lab Iteration 7 — has signup/login **and** `requireAuth` middleware on POST/PUT/DELETE product routes. | Iteration 7 |

### What You Will Learn

- How to manage form state with `useState` and handle form submissions with `fetch`.
- How to store a JWT token in `localStorage` and read it back on page load.
- How to add `Authorization: Bearer <token>` headers to protected API requests.
- How to **conditionally render** UI elements (Navbar links, Edit/Delete buttons) based on authentication state.
- How to **protect client-side routes** using `<Navigate>` from React Router.

### Activity Structure

| Iteration | Feature | Backend Used | New / Changed Files |
|---|---|---|---|
| 6 | User signup & login | `backend-crud/` | `Signup.jsx`, `Login.jsx`, `Navbar.jsx`, `App.jsx` |
| 7 | Route protection & token headers | `backend/` | `App.jsx`, `Navbar.jsx`, `Signup.jsx`, `Login.jsx`, `ProductPage.jsx`, `AddProductPage.jsx`, `EditProductPage.jsx` |

> **Important:** Commit your work after each iteration.


## The Backend APIs (Reference)

### `backend-crud/` — Authentication Endpoints (Iteration 6)

This is the copy of your backend that you saved before adding route protection in the backend lab. It has **user routes** (signup/login) but all product routes remain **unprotected**.

**Base URL:** `http://localhost:4000`

| Method | Endpoint | Description | Request Body |
|---|---|---|---|
| `POST` | `/api/users/signup` | Register a new user | JSON (see below) |
| `POST` | `/api/users/login` | Log in an existing user | JSON (see below) |
| `GET` | `/api/products` | Get all products | — |
| `GET` | `/api/products/:productId` | Get a single product | — |
| `POST` | `/api/products` | Create a product | JSON |
| `PUT` | `/api/products/:productId` | Update a product | JSON |
| `DELETE` | `/api/products/:productId` | Delete a product | — |

**Signup body:**

```json
{
  "fullName": "Jane Doe",
  "email": "jane@example.com",
  "password": "secret123",
  "phoneNumber": "040-1234567",
  "gender": "Female",
  "date_of_birth": "1995-06-15",
  "accountType": "Active"
}
```

**Login body:**

```json
{
  "email": "jane@example.com",
  "password": "secret123"
}
```

**Successful response (both signup and login):**

```json
{
  "email": "jane@example.com",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

The `token` is a JWT. You will store it in `localStorage` and send it in request headers in Iteration 7.

### `backend/` — Protected Product Routes (Iteration 7)

This is your final backend from the backend lab. It is identical to `backend-crud/` except that **creating, updating, and deleting** products requires a valid JWT in the `Authorization` header.

| Method | Endpoint | Protected? | Required Header |
|---|---|---|---|
| `GET` | `/api/products` | No | — |
| `GET` | `/api/products/:productId` | No | — |
| `POST` | `/api/products` | **Yes** | `Authorization: Bearer <token>` |
| `PUT` | `/api/products/:productId` | **Yes** | `Authorization: Bearer <token>` |
| `DELETE` | `/api/products/:productId` | **Yes** | `Authorization: Bearer <token>` |
| `POST` | `/api/users/signup` | No | — |
| `POST` | `/api/users/login` | No | — |

If a protected route is called without a valid token, the API responds with:

```json
{ "error": "Authorization token required" }
```


## Instructions

### Iteration 6: User Signup & Login (`POST`)

**Goal:** Add signup, login, and logout functionality. After signing up or logging in, the user's email and JWT token are saved to `localStorage`. The Navbar shows links to Login and Signup pages and a Log out button.

**Backend to use:** You should still be running `backend-crud/` (the copy you saved before adding route protection). If it is not running, start it:

```bash
cd backend-crud
npm run dev
```

**New files to create:** `src/pages/Signup.jsx`, `src/pages/Login.jsx`  
**Files to change:** `src/App.jsx`, `src/components/Navbar.jsx`

#### Step A — Create the Signup page

Create `src/pages/Signup.jsx`. This page uses `useState` for each form field and sends a POST request to the signup endpoint. On success, the returned user (email + token) is saved to `localStorage`:

```jsx
import { useState } from "react";
import { useNavigate } from "react-router-dom";

const Signup = () => {
  const navigate = useNavigate();
  const [fullName, setFullName] = useState("");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [phoneNumber, setPhoneNumber] = useState("");
  const [gender, setGender] = useState("");
  const [dateOfBirth, setDateOfBirth] = useState("");
  const [accountType, setAccountType] = useState("");
  const [error, setError] = useState(null);

  const handleFormSubmit = async (e) => {
    e.preventDefault();
    setError(null);

    const response = await fetch("/api/users/signup", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        email,
        password,
        fullName,
        phoneNumber,
        gender,
        date_of_birth: dateOfBirth,
        accountType,
      }),
    });
    const user = await response.json();

    if (!response.ok) {
      setError(user.error);
      return;
    }

    localStorage.setItem("user", JSON.stringify(user));
    console.log("success");
    navigate("/");
  };

  return (
    <div className="create">
      <h2>Sign Up</h2>
      <form onSubmit={handleFormSubmit}>
        <label>Full Name:</label>
        <input type="text" value={fullName} onChange={(e) => setFullName(e.target.value)} />
        <label>Email address:</label>
        <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} />
        <label>Password:</label>
        <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
        <label>Phone Number:</label>
        <input type="text" value={phoneNumber} onChange={(e) => setPhoneNumber(e.target.value)} />
        <label>Gender:</label>
        <input type="text" value={gender} onChange={(e) => setGender(e.target.value)} />
        <label>Date of Birth:</label>
        <input type="date" value={dateOfBirth} onChange={(e) => setDateOfBirth(e.target.value)} />
        <label>Account Type:</label>
        <input type="text" value={accountType} onChange={(e) => setAccountType(e.target.value)} />
        <button>Sign up</button>
        {error && <p className="error">{error}</p>}
      </form>
    </div>
  );
};

export default Signup;
```

> **Key concept:** `localStorage.setItem("user", JSON.stringify(user))` saves the object `{ email, token }` so it persists across page refreshes. You can read it back later with `JSON.parse(localStorage.getItem("user"))`.

#### Step B — Create the Login page

Create `src/pages/Login.jsx`:

```jsx
import { useState } from "react";
import { useNavigate } from "react-router-dom";

const Login = () => {
  const navigate = useNavigate();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState(null);

  const handleFormSubmit = async (e) => {
    e.preventDefault();
    setError(null);

    const response = await fetch("/api/users/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password }),
    });
    const user = await response.json();

    if (!response.ok) {
      setError(user.error);
      return;
    }

    localStorage.setItem("user", JSON.stringify(user));
    console.log("success");
    navigate("/");
  };

  return (
    <div className="create">
      <h2>Login</h2>
      <form onSubmit={handleFormSubmit}>
        <label>Email address:</label>
        <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} />
        <label>Password:</label>
        <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
        <button>Log in</button>
        {error && <p className="error">{error}</p>}
      </form>
    </div>
  );
};

export default Login;
```

#### Step C — Update `App.jsx` to add auth routes

Import the new pages and add routes for `/signup` and `/login`:

```jsx
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";

// pages & components
import Navbar from "./components/Navbar";
import Home from "./pages/HomePage";
import AddProductPage from "./pages/AddProductPage";
import ProductPage from "./pages/ProductPage";
import EditProductPage from "./pages/EditProductPage";
import NotFoundPage from "./pages/NotFoundPage";
import Login from "./pages/Login";
import Signup from "./pages/Signup";

const App = () => {
  return (
    <div className="App">
      <BrowserRouter>
        <Navbar />
        <div className="content">
          <Routes>
            <Route path="/" element={<Home />} />
            <Route path="/products/:id" element={<ProductPage />} />
            <Route path="/add-product" element={<AddProductPage />} />
            <Route path="/edit-product/:id" element={<EditProductPage />} />
            <Route path="/signup" element={<Signup />} />
            <Route path="/login" element={<Login />} />
            <Route path="*" element={<NotFoundPage />} />
          </Routes>
        </div>
      </BrowserRouter>
    </div>
  );
};

export default App;
```

> **What changed compared to step 5?**
> - Imported `Navigate`, `Login`, and `Signup`.
> - Added two new `<Route>` entries for `/signup` and `/login`.

#### Step D — Update `Navbar.jsx` with Login, Signup, and Logout links

Replace the old Navbar with one that shows Login/Signup links and a Log out button:

```jsx
import { Link } from "react-router-dom";

const Navbar = () => {
  const handleClick = () => {
    localStorage.removeItem("user");
  };

  return (
    <nav className="navbar">
      <Link to="/">
        <h1>Product Store</h1>
      </Link>
      <div className="links">
        <div>
          <Link to="/add-product">Add Product</Link>
          <Link to="/login">Login</Link>
          <Link to="/signup">Signup</Link>
          <button onClick={handleClick}>Log out</button>
        </div>
      </div>
    </nav>
  );
};

export default Navbar;
```

> **What changed compared to step 5?**
> - Changed `<a>` tags to `<Link>` components (no full page reload).
> - Added Login and Signup links.
> - Added a Log out button that removes the user from `localStorage`.


**You are done with Iteration 6 when:**

- You can open the Signup page, fill in all fields, and click "Sign up".
- After signing up, you are redirected to the Home page.
- If you open your browser's DevTools → Application → Local Storage, you can see a `user` key containing `{ "email": "...", "token": "..." }`.
- You can open the Login page, enter the email and password, and log in successfully.
- Clicking "Log out" removes the `user` entry from Local Storage.
- All CRUD operations (add, list, view, edit, delete products) still work as before.

**Discussion Questions:**

- Why do we store the token in `localStorage` instead of React state?
- What happens if the signup or login `fetch` call fails — how does the component communicate the error to the user?

### Iteration 7: Protect Routes (`Authorization` header + conditional rendering)

**Goal:** Only logged-in users can add, edit, or delete products. The Navbar adapts based on authentication state — showing the user's email and a Log out button when logged in, or Login/Signup links when logged out. Protected pages redirect unauthenticated users to the Signup page.

**Backend to use:** Stop `backend-crud/` and start your final `backend/` (the version with route protection):

```bash
cd backend
npm run dev
```

> **What is different in `backend/`?** The product router now uses a `requireAuth` middleware on POST, PUT, and DELETE. Any request to these endpoints **must** include an `Authorization: Bearer <token>` header, or it will be rejected with a 401 error.

**Files to change:** `src/App.jsx`, `src/components/Navbar.jsx`, `src/pages/Signup.jsx`, `src/pages/Login.jsx`, `src/pages/ProductPage.jsx`, `src/pages/AddProductPage.jsx`, `src/pages/EditProductPage.jsx`

#### Step A — Add `isAuthenticated` state to `App.jsx`

The app needs to know whether a user is logged in. We store this in a top-level state variable, initialized from `localStorage` so it survives page refreshes:

```jsx
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import { useState } from "react";

// pages & components
import Navbar from "./components/Navbar";
import Home from "./pages/HomePage";
import AddProductPage from "./pages/AddProductPage";
import ProductPage from "./pages/ProductPage";
import EditProductPage from "./pages/EditProductPage";
import NotFoundPage from "./pages/NotFoundPage";
import Login from "./pages/Login";
import Signup from "./pages/Signup";

const App = () => {
  const [isAuthenticated, setIsAuthenticated] = useState(() => {
    const user = JSON.parse(localStorage.getItem("user"));
    return user && user.token ? true : false;
  });

  return (
    <div className="App">
      <BrowserRouter>
        <Navbar
          isAuthenticated={isAuthenticated}
          setIsAuthenticated={setIsAuthenticated}
        />
        <div className="content">
          <Routes>
            <Route path="/" element={<Home />} />
            <Route
              path="/products/:id"
              element={<ProductPage isAuthenticated={isAuthenticated} />}
            />
            <Route
              path="/add-product"
              element={
                isAuthenticated ? <AddProductPage /> : <Navigate to="/signup" />
              }
            />
            <Route
              path="/edit-product/:id"
              element={
                isAuthenticated ? (
                  <EditProductPage />
                ) : (
                  <Navigate to="/signup" />
                )
              }
            />
            <Route
              path="/signup"
              element={
                isAuthenticated ? (
                  <Navigate to="/" />
                ) : (
                  <Signup setIsAuthenticated={setIsAuthenticated} />
                )
              }
            />
            <Route
              path="/login"
              element={
                isAuthenticated ? (
                  <Navigate to="/" />
                ) : (
                  <Login setIsAuthenticated={setIsAuthenticated} />
                )
              }
            />
            <Route path="*" element={<NotFoundPage />} />
          </Routes>
        </div>
      </BrowserRouter>
    </div>
  );
};

export default App;
```

> **What changed compared to step 6?**
> - Added `isAuthenticated` state initialized from `localStorage`.
> - `Navbar` receives `isAuthenticated` and `setIsAuthenticated` as props.
> - `ProductPage` receives `isAuthenticated` to conditionally show Edit/Delete buttons.
> - `AddProductPage` and `EditProductPage` routes redirect to `/signup` if not authenticated.
> - `Signup` and `Login` routes redirect to `/` if already authenticated.
> - `Signup` and `Login` receive `setIsAuthenticated` so they can update it on success.

> **Why `<Navigate to="/signup" />`?** This is React Router's way of doing a redirect. If a non-authenticated user tries to visit `/add-product`, they are immediately taken to the Signup page instead.

#### Step B — Update `Navbar.jsx` for conditional rendering

The Navbar should show different links depending on whether the user is logged in:

```jsx
import { Link } from "react-router-dom";

const Navbar = ({ isAuthenticated, setIsAuthenticated }) => {
  const handleClick = () => {
    setIsAuthenticated(false);
    localStorage.removeItem("user");
  };

  return (
    <nav className="navbar">
      <Link to="/">
        <h1>Product Store</h1>
      </Link>
      <div className="links">
        {isAuthenticated && (
          <div>
            <Link to="/add-product">Add Product</Link>
            <span>{JSON.parse(localStorage.getItem("user")).email}</span>
            <button onClick={handleClick}>Log out</button>
          </div>
        )}
        {!isAuthenticated && (
          <div>
            <Link to="/login">Login</Link>
            <Link to="/signup">Signup</Link>
          </div>
        )}
      </div>
    </nav>
  );
};

export default Navbar;
```

> **What changed compared to step 6?**
> - Accepts `isAuthenticated` and `setIsAuthenticated` props.
> - When authenticated: shows "Add Product" link, the user's email, and a "Log out" button.
> - When not authenticated: shows "Login" and "Signup" links only.
> - `handleClick` now also calls `setIsAuthenticated(false)` to re-render the entire app.

#### Step C — Update `Signup.jsx` to set authentication state

Accept `setIsAuthenticated` as a prop and call it on successful signup:

```jsx
const Signup = ({ setIsAuthenticated }) => {
  // ... same as step 6, but add this line after signup succeeds:
```

Inside `handleFormSubmit`, after `localStorage.setItem("user", JSON.stringify(user))`, add:

```jsx
    localStorage.setItem("user", JSON.stringify(user));
    setIsAuthenticated(true);   // <-- ADD THIS LINE
    console.log("success");
    navigate("/");
```

#### Step D — Update `Login.jsx` to set authentication state

Same pattern — accept `setIsAuthenticated` as a prop:

```jsx
const Login = ({ setIsAuthenticated }) => {
  // ... same as step 6, but add this line after login succeeds:
```

Inside `handleFormSubmit`, after `localStorage.setItem("user", JSON.stringify(user))`, add:

```jsx
    localStorage.setItem("user", JSON.stringify(user));
    setIsAuthenticated(true);   // <-- ADD THIS LINE
    console.log("success");
    navigate("/");
```

#### Step E — Send the token when adding a product (`AddProductPage.jsx`)

The `backend/` API now requires a JWT in the `Authorization` header for POST requests. Read the token from `localStorage` and include it in the `fetch` call:

1. **Read the token** at the top of the component:
   ```jsx
   const user = JSON.parse(localStorage.getItem("user"));
   const token = user ? user.token : null;
   ```

2. **Add the `Authorization` header** to the `addProduct` function:
   ```jsx
   const addProduct = async (newProduct) => {
     try {
       const res = await fetch("/api/products", {
         method: "POST",
         headers: {
           "Content-Type": "application/json",
           Authorization: `Bearer ${token}`,    // <-- ADD THIS
         },
         body: JSON.stringify(newProduct),
       });
       if (!res.ok) throw new Error("Failed to add product");
       return true;
     } catch (error) {
       console.error("Error adding product:", error);
       return false;
     }
   };
   ```

#### Step F — Send the token when deleting a product and conditionally show buttons (`ProductPage.jsx`)

1. **Accept the `isAuthenticated` prop:**
   ```jsx
   const ProductPage = ({ isAuthenticated }) => {
   ```

2. **Read the token** at the top of the component:
   ```jsx
   const user = JSON.parse(localStorage.getItem("user"));
   const token = user ? user.token : null;
   ```

3. **Add the `Authorization` header** to the `deleteProduct` function:
   ```jsx
   const deleteProduct = async (productId) => {
     try {
       const res = await fetch(`/api/products/${productId}`, {
         method: "DELETE",
         headers: {
           Authorization: `Bearer ${token}`,    // <-- ADD THIS
         },
       });
       if (!res.ok) throw new Error("Failed to delete product");
       navigate("/");
     } catch (error) {
       console.error("Error deleting product:", error);
     }
   };
   ```

4. **Conditionally render** the Edit and Delete buttons — only show them when authenticated:
   ```jsx
   {isAuthenticated && (
     <>
       <button onClick={() => navigate(`/edit-product/${product._id}`)}>Edit</button>
       <button onClick={() => onDeleteClick(product._id)}>Delete</button>
     </>
   )}
   ```

#### Step G — Send the token when updating a product (`EditProductPage.jsx`)

Same pattern as AddProductPage:

1. **Read the token:**
   ```jsx
   const user = JSON.parse(localStorage.getItem("user"));
   const token = user ? user.token : null;
   ```

2. **Add the `Authorization` header** to the `updateProduct` function:
   ```jsx
   const updateProduct = async (product) => {
     try {
       const res = await fetch(`/api/products/${id}`, {
         method: "PUT",
         headers: {
           "Content-Type": "application/json",
           Authorization: `Bearer ${token}`,    // <-- ADD THIS
         },
         body: JSON.stringify(product),
       });
       if (!res.ok) throw new Error("Failed to update product");
       return true;
     } catch (error) {
       console.error("Error updating product:", error);
       return false;
     }
   };
   ```

**You are done with Iteration 7 when:**

- **Logged out:** The Navbar shows only "Login" and "Signup" links.
- **Logged out:** Visiting `/add-product` or `/edit-product/:id` redirects to `/signup`.
- **Logged out:** The product detail page does **not** show Edit or Delete buttons.
- **Logged out:** Already-authenticated users visiting `/signup` or `/login` are redirected to `/`.
- **Logged in:** The Navbar shows "Add Product", the user's email, and "Log out".
- **Logged in:** You can add, edit, and delete products (the API accepts the token).
- **Logged in:** Clicking "Log out" clears the user from `localStorage`, updates the Navbar, and re-applies route protection.
- Viewing and listing products (GET) still works for everyone — no token needed.

**Discussion Questions:**

- What does the `Authorization: Bearer <token>` header do? Why doesn't the GET endpoint need it?
- What is the difference between **client-side route protection** (`<Navigate>`) and **server-side route protection** (`requireAuth` middleware)? Do you need both?
- Why do we initialize `isAuthenticated` with a function (`useState(() => { ... })`) instead of just `useState(false)`?
- What happens if a user manually deletes the token from `localStorage` while the app is open?


------

## Part 3 (Refactoring with Custom Hooks)

In Part 2 you added user registration, login, and route protection. The Signup and Login components use inline `useState` for each form field and inline `fetch` calls for the API requests.

In Part 3 you will **refactor** the Signup and Login pages by extracting reusable logic into **custom hooks**:

- `useField` — manages a single form input's state (`type`, `value`, `onChange`).
- `useSignup` — encapsulates the POST request to the signup endpoint.
- `useLogin` — encapsulates the POST request to the login endpoint.

**Backend to use:** Keep using `backend/` from Part 2.

| Folder | Purpose |
|---|---|
| `backend/` | Same Express API from Part 2 (signup, login, protected routes). |

### What You Will Learn

- What a **custom hook** is and how it differs from a regular function.
- How to create `useField` to eliminate repetitive `useState` + `onChange` pairs.
- How to create `useSignup` and `useLogin` to extract API call logic from components.
- How the **spread operator** (`{...field}`) simplifies `<input>` bindings.

### Activity Structure

| Iteration | Feature | Backend Used | New / Changed Files |
|---|---|---|---|
| 8 | Custom hooks for forms & auth | `backend/` | `useField.jsx`, `useSignup.jsx`, `useLogin.jsx`, `Signup.jsx`, `Login.jsx` |

> **Important:** Commit your work after completing the iteration.

## Iteration 8: Refactor with Custom Hooks

**Goal:** Extract repeated logic from `Signup.jsx` and `Login.jsx` into three custom hooks. The app behaviour stays exactly the same — this is a pure refactoring exercise.

**Backend to use:** Keep using `backend/` from Part 2:

```bash
cd backend
npm run dev
```

**New files to create:** `src/hooks/useField.jsx`, `src/hooks/useSignup.jsx`, `src/hooks/useLogin.jsx`

**Files to update:** `src/pages/Signup.jsx`, `src/pages/Login.jsx`

### Step A — Create the `useField` custom hook

Create a new file `src/hooks/useField.jsx`. This hook manages a single form input's state so you don't have to repeat `useState` + `onChange` for every field:

```jsx
import { useState } from "react";

export default function useField(type) {
  const [value, setValue] = useState("");

  const onChange = (e) => {
    setValue(e.target.value);
  };

  return { type, value, onChange };
}
```

> **Why a custom hook?** In step 7 you wrote a separate `useState` + `onChange` for every form field. `useField` bundles that logic into one line per field. You can spread the returned object directly onto an `<input>`: `<input {...email} />` — this sets `type`, `value`, and `onChange` all at once.

### Step B — Create the `useSignup` custom hook

Create `src/hooks/useSignup.jsx`. This hook sends a POST request to the signup endpoint and stores the returned user (email + token) in `localStorage`:

```jsx
import { useState } from "react";

export default function useSignup(url) {
  const [error, setError] = useState(null);
  const [isLoading, setIsLoading] = useState(null);

  const signup = async (object) => {
    setIsLoading(true);
    setError(null);
    const response = await fetch(url, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(object),
    });
    const user = await response.json();

    if (!response.ok) {
      setError(user.error);
      setIsLoading(false);
      return;
    }

    localStorage.setItem("user", JSON.stringify(user));
    setIsLoading(false);
  };

  return { signup, isLoading, error };
}
```

> **What changed compared to step 7?** The `fetch` call, error state, and `localStorage` write that were inside `Signup.jsx` are now inside this hook. The component just calls `await signup({...})` and reads `error` from the hook's return value.


### Step C — Create the `useLogin` custom hook

Create `src/hooks/useLogin.jsx`. This is almost identical to `useSignup`:

```jsx
import { useState } from "react";

export default function useLogin(url) {
  const [error, setError] = useState(null);
  const [isLoading, setIsLoading] = useState(null);

  const login = async (object) => {
    setIsLoading(true);
    setError(null);
    const response = await fetch(url, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(object),
    });
    const user = await response.json();

    if (!response.ok) {
      setError(user.error);
      setIsLoading(false);
      return;
    }

    localStorage.setItem("user", JSON.stringify(user));
    setIsLoading(false);
  };

  return { login, isLoading, error };
}
```

### Step D — Refactor `Signup.jsx` to use the custom hooks

Replace the contents of `src/pages/Signup.jsx`. Instead of multiple `useState` calls and an inline `fetch`, use `useField` and `useSignup`:

```jsx
import useField from "../hooks/useField";
import useSignup from "../hooks/useSignup";
import { useNavigate } from "react-router-dom";

const Signup = ({ setIsAuthenticated }) => {
  const navigate = useNavigate();
  const fullName = useField("text");
  const email = useField("email");
  const password = useField("password");
  const phoneNumber = useField("text");
  const gender = useField("text");
  const dateOfBirth = useField("date");
  const accountType = useField("text");

  const { signup, error } = useSignup("/api/users/signup");

  const handleFormSubmit = async (e) => {
    e.preventDefault();
    await signup({
      email: email.value,
      password: password.value,
      fullName: fullName.value,
      phoneNumber: phoneNumber.value,
      gender: gender.value,
      date_of_birth: dateOfBirth.value,
      accountType: accountType.value,
    });
    if (!error) {
      console.log("success");
      setIsAuthenticated(true);
      navigate("/");
    }
  };

  return (
    <div className="create">
      <h2>Sign Up</h2>
      <form onSubmit={handleFormSubmit}>
        <label>Full Name:</label>
        <input {...fullName} />
        <label>Email address:</label>
        <input {...email} />
        <label>Password:</label>
        <input {...password} />
        <label>Phone Number:</label>
        <input {...phoneNumber} />
        <label>Gender:</label>
        <input {...gender} />
        <label>Date of Birth:</label>
        <input {...dateOfBirth} />
        <label>Account Type:</label>
        <input {...accountType} />
        <button>Sign up</button>
      </form>
    </div>
  );
};

export default Signup;
```

> **What changed?**
>
> | Step 7 (before) | Step 8 (after) |
> |---|---|
> | `const [email, setEmail] = useState("")` | `const email = useField("email")` |
> | `<input type="email" value={email} onChange={(e) => setEmail(e.target.value)} />` | `<input {...email} />` |
> | Inline `fetch("/api/users/signup", ...)` | `const { signup, error } = useSignup("/api/users/signup")` then `await signup({...})` |
> | `const [error, setError] = useState(null)` | `error` comes from the hook |

### Step E — Refactor `Login.jsx` to use the custom hooks

Replace the contents of `src/pages/Login.jsx`:

```jsx
import useField from "../hooks/useField";
import useLogin from "../hooks/useLogin";
import { useNavigate } from "react-router-dom";

const Login = ({ setIsAuthenticated }) => {
  const navigate = useNavigate();
  const email = useField("email");
  const password = useField("password");

  const { login, error } = useLogin("/api/users/login");

  const handleFormSubmit = async (e) => {
    e.preventDefault();
    await login({ email: email.value, password: password.value });
    if (!error) {
      console.log("success");
      setIsAuthenticated(true);
      navigate("/");
    }
  };

  return (
    <div className="create">
      <h2>Login</h2>
      <form onSubmit={handleFormSubmit}>
        <label>Email address:</label>
        <input {...email} />
        <label>Password:</label>
        <input {...password} />
        <button>Log in</button>
      </form>
    </div>
  );
};

export default Login;
```



**You are done with Iteration 8 when:**

- The app behaves exactly the same as step 7 — signup, login, logout, and all protected routes still work.
- You have three new files in `src/hooks/`: `useField.jsx`, `useSignup.jsx`, `useLogin.jsx`.
- `Signup.jsx` and `Login.jsx` no longer contain any `useState` calls or inline `fetch` calls.
- Each form input uses the spread syntax: `<input {...fieldName} />`.

**Discussion Questions:**

- What is a **custom hook**? How is it different from a regular function?
- Why must custom hook names start with `use`?
- What does the spread operator (`{...email}`) do when applied to an `<input>` element?
- What are the **advantages** of extracting `useSignup` and `useLogin` into hooks instead of keeping the `fetch` logic inside each component?
