# 📓 Zomansta Development Journal

# 📅 Day 02

## 🎯 Goal
Build the **data and error-handling foundation** of the backend:
1. Create the **User model** (the first MongoDB schema).
2. Create **helper utilities** so every success and error response looks the same (`ApiError`, `ApiResponse`, `asyncHandler`).
3. Create **error middleware** so all errors are handled in one place.
4. Test that it works and push to GitHub.

> Day 01 gave me the empty house (folders, server, database connection). Day 02 gives it the **rules**: how data is stored and how errors are handled.

> ⚠️ **How to use this page:** the code below is a **generic example** that shows the concept. It is not copied from my repo. After each block, paste my real code from the file named there. Files marked ✅ exist in my GitHub repo.

---

## 🧭 Today's Flow (what connects to what)

```
Request
   │
   ▼
Route ──► Controller (wrapped in asyncHandler)
                │
                ▼
             Service ──► Model (User) ──► MongoDB Atlas
                │
        error thrown? ──► ApiError
                │
                ▼
         errorMiddleware ──► clean JSON error to client

Success ──► ApiResponse ──► clean JSON to client
```

---

## 📁 Files Created / Changed Today

| File | What it does |
|---|---|
| `src/models/User.js` ✅ | Shape of a user in MongoDB |
| `src/utils/ApiError.js` ✅ | Custom error class with a status code |
| `src/utils/ApiResponse.js` ✅ | Standard success response |
| `src/utils/asyncHandler.js` ✅ | Catches errors from async functions |
| `src/middleware/errorMiddleware.js` ✅ | One place that handles every error |
| `src/app.js` ✅ | Express setup, where the error middleware is connected |

---

## 1️⃣ User Model: `src/models/User.js`

### What is a Model?
- **Schema** = the blueprint (which fields, which types, which rules).
- **Model** = the tool that uses the blueprint to create, read, update and delete users.

### Generic example
```js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, 'Name is required'],
      trim: true
    },
    email: {
      type: String,
      required: [true, 'Email is required'],
      unique: true,
      lowercase: true,
      trim: true
    },
    password: {
      type: String,
      required: [true, 'Password is required'],
      minlength: 6,
      select: false
    },
    avatar: {
      type: String,
      default: ''
    },
    role: {
      type: String,
      enum: ['user', 'admin'],
      default: 'user'
    }
  },
  { timestamps: true }
);

module.exports = mongoose.model('User', userSchema);
```

### Line-by-line meaning
| Option | Meaning |
|---|---|
| `type: String` | The value must be text |
| `required: [true, 'msg']` | The field is compulsory, and `msg` is the error message |
| `trim: true` | Removes spaces at the start and end (`" a@b.com "` becomes `"a@b.com"`) |
| `lowercase: true` | `A@B.com` and `a@b.com` are stored the same way |
| `unique: true` | No two users can have the same email (creates a **unique index**) |
| `minlength: 6` | Password must be at least 6 characters |
| `select: false` | Password is **hidden** when I read users (safe by default) |
| `enum: [...]` | The value must be one from the list |
| `default` | Value used if none is given |
| `timestamps: true` | Adds `createdAt` and `updatedAt` automatically |

### Important points
- `unique: true` is **not** a validator. It creates a database index. A duplicate email throws a **MongoServerError with code 11000**.
- The password will be **hashed with bcryptjs** before saving (Day 03). It is never stored as plain text.
- `mongoose.model('User', ...)` creates a collection named **`users`** (lowercase and plural) in MongoDB.

**My code:** paste `src/models/User.js` here.

---

## 2️⃣ `ApiError`: `src/utils/ApiError.js`

### Why?
Normal `Error` has only a message. My API also needs a **status code** (404, 401 and so on). `ApiError` extends `Error` and adds it.

### Generic example
```js
class ApiError extends Error {
  constructor(statusCode, message = 'Something went wrong', errors = []) {
    super(message);
    this.statusCode = statusCode;
    this.errors = errors;
    this.success = false;
  }
}

module.exports = ApiError;
```

### How I will use it
```js
throw new ApiError(404, 'User not found');
throw new ApiError(400, 'Email already exists');
```

### Concept learned
- `class ApiError extends Error` = **inheritance**. ApiError gets everything Error has, plus my extra fields.
- `super(message)` calls the parent class constructor. It must come **before** `this`.

**My code:** paste `src/utils/ApiError.js` here.

---

## 3️⃣ `ApiResponse`: `src/utils/ApiResponse.js`

### Why?
Every successful reply has the **same shape**. The frontend can always read `data`, `message` and `success` without guessing.

### Generic example
```js
class ApiResponse {
  constructor(statusCode, data, message = 'Success') {
    this.statusCode = statusCode;
    this.data = data;
    this.message = message;
    this.success = statusCode < 400;
  }
}

module.exports = ApiResponse;
```

### How I will use it
```js
res.status(201).json(new ApiResponse(201, user, 'User created'));
```

### Output the client sees
```json
{
  "statusCode": 201,
  "data": { "name": "Sanjana" },
  "message": "User created",
  "success": true
}
```

**My code:** paste `src/utils/ApiResponse.js` here.

---

## 4️⃣ `asyncHandler`: `src/utils/asyncHandler.js`

### The problem it solves
Controllers are `async` functions. In Express 4, if an `await` fails inside one **without try/catch**, the error is not passed to Express, and the request can hang or the server can crash with an unhandled rejection.

**Without asyncHandler (repeated in every controller):**
```js
const getUser = async (req, res, next) => {
  try {
    // logic
  } catch (err) {
    next(err);
  }
};
```

**With asyncHandler (clean):**
```js
const getUser = asyncHandler(async (req, res) => {
  // logic. If something fails, the error goes to errorMiddleware automatically
});
```

### Generic example
```js
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

module.exports = asyncHandler;
```

### How to read this line
1. `asyncHandler(fn)` takes my controller `fn` and returns a **new function**.
2. That new function runs `fn` inside `Promise.resolve(...)`.
3. If `fn` fails, `.catch(next)` sends the error to Express's error handler.

**My code:** paste `src/utils/asyncHandler.js` here.

---

## 5️⃣ Error Middleware: `src/middleware/errorMiddleware.js`

### Key rules
- Error middleware has **4 parameters**: `(err, req, res, next)`. Express identifies it by that.
- It must be added **after all routes** in `app.js`.
- **Two parts:** `notFound` (no route matched) and `errorHandler` (everything else).

### Generic example
```js
const ApiError = require('../utils/ApiError');

const notFound = (req, res, next) => {
  next(new ApiError(404, `Route not found: ${req.originalUrl}`));
};

const errorHandler = (err, req, res, next) => {
  const statusCode = err.statusCode || 500;

  res.status(statusCode).json({
    success: false,
    message: err.message || 'Internal Server Error',
    errors: err.errors || [],
    ...(process.env.NODE_ENV !== 'production' && { stack: err.stack })
  });
};

module.exports = { notFound, errorHandler };
```

### Connect it in `src/app.js`
```js
const express = require('express');
const { notFound, errorHandler } = require('./middleware/errorMiddleware');

const app = express();
app.use(express.json());

// routes go here (Day 03 onwards)

app.use(notFound);      // after all routes
app.use(errorHandler);  // always last

module.exports = app;
```

### Concepts learned
- The **order of middleware** matters. Error handlers go last.
- `next(err)` skips all normal middleware and jumps to the error handler.
- The **stack trace** helps me while developing but is **hidden in production** so attackers can't see internal details.

**My code:** paste `src/middleware/errorMiddleware.js` and `src/app.js` here.

---

## 🧪 How I Tested It

### Temporary test route (add in `app.js` before `notFound`, and remove after testing)
```js
const asyncHandler = require('./utils/asyncHandler');
const ApiError = require('./utils/ApiError');

app.get('/api/test-error', asyncHandler(async (req, res) => {
  throw new ApiError(400, 'This is a test error');
}));
```

### Checks
| Open this URL | Expected result |
|---|---|
| `http://localhost:5000/api/test-error` | `{ "success": false, "message": "This is a test error" ... }` with status **400** |
| `http://localhost:5000/api/abc` | `Route not found: /api/abc` with status **404** |

✅ If both give clean JSON, the error system works.

---

## 🐞 Problems Faced (common ones. Replace with my own)

| Problem | Reason | Solution |
|---|---|---|
| `Cannot find module '../utils/ApiError'` | Wrong file path or spelling (capital letters matter) | Check the folder and file name exactly |
| `ApiError is not a constructor` | Forgot `module.exports = ApiError` | Add the export line |
| Error handler never runs | It was placed **above** the routes | Move it to the very end of `app.js` |
| Request keeps loading forever | An async error was not caught | Wrap the controller in `asyncHandler` |
| Error handler ignored | Wrote only 3 params `(err, req, res)` | Keep all 4: `(err, req, res, next)` |

---

## 💾 Git: Saving Today's Work

Run only inside `C:\Users\kyasa\zomansta\backend`:

```bash
git pull
git status
git add .
git commit -m "Add User model, error utilities and error middleware"
git push
```

✅ `git status` must **not** show `.env` or `node_modules`.

---

## ✅ Achievements
- Created the **User schema and model** with validation rules
- Understood schema vs model, `unique`, `select: false` and `timestamps`
- Built a custom **ApiError** class using inheritance
- Built a standard **ApiResponse** so every reply has the same shape
- Wrote **asyncHandler** to avoid try/catch in every controller
- Built **error middleware** with a 404 handler
- Tested the error flow and pushed to GitHub

---

## 💡 Concepts Learned (quick revision)

| Concept | One-line meaning |
|---|---|
| Schema | Blueprint of a document |
| Model | Tool to create, read, update and delete documents |
| ODM | Library that maps JS objects to MongoDB documents (Mongoose) |
| ObjectId | Unique 12-byte ID MongoDB gives every document (`_id`) |
| Inheritance | A class reusing another class (`extends`) |
| Middleware | A function that runs between request and response |
| `next()` | Passes control to the next middleware |
| `next(err)` | Jumps straight to the error handler |
| HTTP status codes | 200 OK, 201 Created, 400 Bad request, 401 Unauthorized, 403 Forbidden, 404 Not found, 500 Server error |

---

# ❓ Interview Questions (Most Important → Least Important)

**How many to learn?**

| Tier | Questions | What to do |
|---|---|---|
| 🔴 **Tier 1: Mandatory** | Q1 to Q12 (12 questions) | Must answer these **confidently and without notes**. |
| 🟠 **Tier 2: Important** | Q13 to Q22 (10 questions) | Learn these too. They are asked in many technical rounds. |
| 🟢 **Tier 3: Bonus** | Q23 to Q30 (8 questions) | These impress the interviewer. Learn them after Tier 1 and 2. |

> 💬 **Note:** an HR round mostly asks about your project ("Explain your project", "What problem did you face?"). The **technical round** asks the questions below. Learn both: Q28 to Q30 are project-style questions that work in HR rounds too.

---

## 🔴 Tier 1: Mandatory (12)

**Q1. What is Mongoose and why do you use it with MongoDB?**
Mongoose is an **ODM (Object Data Modeling)** library for Node.js. MongoDB itself has no fixed structure, so Mongoose adds **schemas, validation, and easy methods** (`create`, `find`, `save`) on top of it. It keeps my data consistent and my code cleaner.

**Q2. What is the difference between a schema and a model?**
A **schema** defines the *structure* of a document (fields, types, rules). A **model** is created from the schema (`mongoose.model('User', userSchema)`) and gives me the methods to *work with* the collection, like `User.find()` and `User.create()`.

**Q3. SQL vs NoSQL: why did you choose MongoDB?**
SQL stores data in **tables with fixed columns**. MongoDB stores **JSON-like documents** in collections, so the structure is flexible. For Zomansta, data such as reels, comments and likes changes often, and MongoDB scales easily and works naturally with JavaScript objects.

**Q4. What is middleware in Express?**
A function that runs **between the request and the response**. It receives `(req, res, next)`, can change the request or response, end the request, or call `next()` to pass control forward. Examples: `express.json()`, authentication, error handling.

**Q5. How does error-handling middleware work in Express?**
It has **4 parameters**: `(err, req, res, next)`. Express recognizes it by that. It is placed **after all routes**. When any route or middleware calls `next(err)` or throws, Express skips the normal middleware and runs this one, which sends a clean error response.

**Q6. Why did you create an `asyncHandler`?**
In Express 4, an error inside an `async` function is not caught automatically unless I use `try/catch`. `asyncHandler` wraps the controller with `Promise.resolve(fn(...)).catch(next)`, so **any error is passed to the error middleware**. It removes repeated try/catch blocks and keeps controllers clean.

**Q7. Why create a custom `ApiError` class?**
The built-in `Error` has no HTTP status code. `ApiError extends Error` and adds `statusCode` (and optional `errors`), so I can write `throw new ApiError(404, 'User not found')` and the error middleware knows exactly what status to send. All errors then have the **same structure**.

**Q8. Why use a standard `ApiResponse` class?**
For **consistency**. Every success response has the same fields (`statusCode`, `data`, `message`, `success`), so the frontend code stays simple and predictable.

**Q9. Explain these status codes: 200, 201, 400, 401, 403, 404, 500.**
- **200** OK: request succeeded.
- **201** Created: a new resource was created.
- **400** Bad Request: invalid input from the client.
- **401** Unauthorized: not logged in or invalid token.
- **403** Forbidden: logged in but not allowed.
- **404** Not Found: resource or route doesn't exist.
- **500** Internal Server Error: a server-side failure.

**Q10. What does `next()` do? What is the difference between `next()` and `next(err)`?**
`next()` passes control to the **next normal middleware or route**. `next(err)` passes an error and **skips all normal middleware**, jumping directly to the error-handling middleware.

**Q11. Does `unique: true` validate the email in Mongoose?**
No. `unique` is **not a validator**. It creates a **unique index** in MongoDB. If I insert a duplicate, MongoDB throws an error with **code 11000**, which I should catch and turn into a friendly message like "Email already exists".

**Q12. What does `timestamps: true` do?**
Mongoose automatically adds **`createdAt`** and **`updatedAt`** fields to every document and keeps `updatedAt` current whenever the document is updated.

---

## 🟠 Tier 2: Important (10)

**Q13. Why use `select: false` on the password field?**
So the password is **not returned by default** in queries such as `User.find()`. It prevents accidentally leaking the hash in an API response. When I need it (for login), I ask for it explicitly with `.select('+password')`.

**Q14. Why store the password hash and not the plain password?**
If the database leaks, plain passwords would be exposed immediately. A **hash** (bcrypt) is one-way and can't be reversed. I compare using `bcrypt.compare()`. (I implement this on Day 03.)

**Q15. What is the difference between Mongoose validation (`required`, `minlength`) and `express-validator`?**
`express-validator` checks the **incoming request** *before* it reaches my logic. Mongoose validation checks the data *just before it is saved to the database*. I use both: request-level for a fast, friendly error and schema-level as the last safety net.

**Q16. What is an ObjectId?**
A **unique 12-byte identifier** MongoDB generates for the `_id` field of every document. It includes a timestamp, so IDs are roughly time-ordered. It is used to link documents (for example, a reel storing its creator's `_id`).

**Q17. What do `trim` and `lowercase` do? Why use them on email?**
`trim` removes extra spaces and `lowercase` converts to lowercase. Without them, `"A@gmail.com"` and `"a@gmail.com "` could be treated as two different users, which defeats the `unique` rule.

**Q18. What is `Promise.resolve(fn(req, res, next)).catch(next)` doing?**
`fn(...)` runs my async controller and returns a promise. `Promise.resolve()` makes sure the result is treated as a promise. `.catch(next)` sends any rejection to Express's error handler by calling `next(error)`.

**Q19. Why must the error middleware be the last `app.use()`?**
Express runs middleware **in order**. Error handlers only catch errors from middleware and routes registered **before** them. If I put it first, it never sees any error.

**Q20. Why hide the stack trace in production?**
A stack trace shows internal file paths and code structure, which helps attackers. In development I show it for debugging, and in production I send only a safe message. I control this with `NODE_ENV`.

**Q21. What is the difference between operational errors and programmer errors?**
**Operational errors** are expected runtime problems (user not found, invalid input, DB down) and should be handled and answered properly with `ApiError`. **Programmer errors** are bugs (typo, `undefined` variable) and should be fixed in code, not "handled".

**Q22. What is `module.exports` and how is it different from `import/export`?**
`module.exports` and `require()` are **CommonJS**, Node's original module system. `import/export` is **ES Modules**, the newer standard. My project uses CommonJS (`require`), so I write `module.exports = ...`.

---

## 🟢 Tier 3: Bonus (8)

**Q23. What does `extends Error` and `super(message)` mean in `ApiError`?**
`extends Error` makes `ApiError` a child of the built-in `Error` class, so it inherits features like `message` and `stack`. `super(message)` calls the parent constructor to set them up. It must be called **before** using `this`.

**Q24. What is the difference between `throw` and `next(err)`?**
Inside an **async** route wrapped by `asyncHandler`, `throw` works because the rejection is caught and forwarded with `next`. Inside **synchronous** code Express also catches a thrown error. In **callbacks** or unwrapped async code, I must call `next(err)` explicitly.

**Q25. What are database indexes and why does `email` need one?**
An index is a special data structure that makes searching **faster**, like a book's index. Searching users by `email` at login happens constantly, so a (unique) index on `email` makes lookups fast and prevents duplicates.

**Q26. What is the difference between `Model.create()` and `new Model().save()`?**
Both insert a document. `create()` builds and saves in one step. `new Model(data)` creates the object first, so I can change it before calling `.save()`. Both run schema validation and pre-save hooks.

**Q27. Mongoose is "schema-based" but MongoDB is "schema-less". Why still use schemas?**
MongoDB doesn't force a structure, which can lead to messy, inconsistent data. Mongoose schemas add **validation, defaults and consistency** at the application level, so bugs are caught early.

**Q28. 🧑‍💼 (Project) How do you handle errors in your project?**
"I created a custom `ApiError` class with status codes, an `asyncHandler` to forward async errors, and a central error middleware that sends a consistent JSON response. There's also a 404 handler for unknown routes. In production, stack traces are hidden."

**Q29. 🧑‍💼 (Project) Why did you separate your code into controllers, services and models?**
"Each layer has one job. Routes map URLs, controllers handle HTTP, services hold business logic, and models talk to the database. It makes the code easier to read, test, debug and scale."

**Q30. 🧑‍💼 (Project) Tell me about a bug you faced and how you solved it.**
Use a **real** one from your Problems Faced table. Structure it as: **Problem → Reason → What I tried → Solution → What I learned.** Example: "My error handler wasn't running. I realized I had placed it above my routes, so I moved it to the end of `app.js`, and now I know middleware order matters."

---

## 📌 Revision Plan for This Day
1. **Today:** read Q1 to Q12 and say each answer **out loud** in your own words.
2. **Tomorrow:** Q13 to Q22.
3. **Weekend:** Q23 to Q30 and practice the project story (Q28 to Q30).
4. Tick this list: I can explain the flow *Route → Controller → Service → Model → DB* and the error flow *throw → asyncHandler → next(err) → errorMiddleware*.
