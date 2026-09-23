# 📓 Zomansta Development Journal

# 📅 Day 04: Protecting routes — `authMiddleware.js`

## 🎯 Goal
Make sure only logged-in users can access certain routes, by reading and verifying the JWT sent by the client.

> ⚠️ Every code block below is copied exactly from your own terminal output. Nothing is invented.

---

## 📁 Files Created Today
| File | Purpose |
|---|---|
| `src/middleware/authMiddleware.js` | Checks the JWT and attaches the logged-in user to `req.user` |

---

## 1️⃣ `src/middleware/authMiddleware.js`
```js
const jwt = require('jsonwebtoken');
const asyncHandler = require('../utils/asyncHandler');
const User = require('../models/User');
const protect = asyncHandler(async (req, res, next) => {
  let token;
  if (req.headers.authorization && 
      req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } else {
    res.status(401);
    throw new Error('Not authorized, no token');
  }
});
module.exports = {protect};
```

### What I learned
- The client sends the token as a header: `Authorization: Bearer <token>`. `req.headers.authorization.split(' ')[1]` pulls out just the token part, after the word "Bearer".
- `jwt.verify(token, process.env.JWT_SECRET)` checks the token's signature against my secret key. If the token was tampered with, or doesn't match the secret, `jwt.verify` **throws** automatically — I don't need to check this manually.
- `User.findById(decoded.id).select('-password')` fetches the full user document from the database using the ID stored inside the token, but **excludes** the password field from the result, even though `User.js` doesn't mark the password `select: false` by default.
- `req.user = ...` attaches the logged-in user to the request object, so every controller that runs **after** this middleware (like `uploadReel`, `toggleLike`) can use `req.user._id` directly.
- `next()` is only called **inside** the `if` block — if there's no valid `Authorization: Bearer` header at all, the `else` branch runs instead, which sets a 401 status and throws an error.
- This whole function is wrapped in `asyncHandler`, so if `jwt.verify` or `User.findById` throws, the error is automatically forwarded to `errorMiddleware.js` — no manual try/catch needed here.

---

## 🔎 Honest observation
If the token exists but is **expired or invalid**, `jwt.verify` throws inside the `if` block — **before** any status code is set. That error falls through to `errorMiddleware.js`, and because `res.statusCode` is still the default 200 at that point, my error handler's fallback logic (`res.statusCode === 200 ? 500 : res.statusCode`, from Day 2) sends back a **500 Internal Server Error** instead of the more correct **401 Unauthorized**. Only the "no token at all" case correctly gets a 401, because that's the only path where I explicitly call `res.status(401)` before throwing.

This is a real, subtle bug in my current code — not something to hide. It's actually a great "what would you improve" answer in an interview.

---

## 🧭 How This Connects to Other Files

```
Client sends request
   │
   ▼
Authorization: Bearer <token> header
   │
   ▼
authMiddleware.js (protect)
   │
   ├── No/invalid header → 401 "Not authorized, no token"
   │
   └── Valid header → jwt.verify() → User.findById() → req.user set → next()
                                                              │
                                                              ▼
                                              reelController.js (uploadReel, toggleLike, etc.)
                                              can now use req.user._id
```

**Where `protect` is actually used** (from `reelRoutes.js`, built on Day 6):
```js
router.post('/', protect, uploadReel);
router.put('/:id/like', protect, toggleLike);
router.put('/:id/save', protect, toggleSave);
router.delete('/:id', protect, removeReel);
```
Notice `GET /` and `GET /:id` do **not** have `protect` — anyone can browse reels, but only logged-in users can create, like, save, or delete them.

---

## 🧪 How to Test It (Postman)

**Test 1: No token at all**
- `GET http://localhost:5000/api/reels` — this route isn't protected, so it should work with no token.
- `POST http://localhost:5000/api/reels` with **no** Authorization header — expected: status **401**, message `"Not authorized, no token"`.

**Test 2: Valid token**
1. First call `POST /api/auth/login` to get a token (from Day 3).
2. Copy the `token` value from the response.
3. In Postman, go to the **Authorization** tab → type **Bearer Token** → paste the token.
4. Call `POST http://localhost:5000/api/reels` with a valid body — expected: status **201**, the reel is created.

**Test 3: Invalid/expired token (to see the honest bug)**
- Paste a fake or altered token string as the Bearer token and call a protected route.
- You'll currently get a **500** error instead of a 401 — this confirms the observation above.

---

## 🐞 Problems Faced

| Problem | Reason | Solution |
|---|---|---|
| Invalid/expired token gives 500 instead of 401 | `jwt.verify` throws before any `res.status()` call, so `errorMiddleware`'s fallback defaults to 500 | Planned fix: wrap `jwt.verify` in a try/catch inside `protect`, and explicitly call `res.status(401)` before re-throwing |

---

## 💾 Git: Saving Today's Work

Run only inside `C:\Users\kyasa\zomansta\backend`:
```bash
git pull
git status
git add .
git commit -m "Add authMiddleware to protect routes with JWT"
git push
```
✅ `git status` must **not** show `.env` or `node_modules`.

---

## ✅ Achievements
- Built the `protect` middleware to guard routes using JWT
- Understood how `req.headers.authorization` and the `Bearer` scheme work
- Learned how `jwt.verify` validates a token's signature
- Learned the difference between **authentication** (who are you — this middleware) and **authorization** (what are you allowed to do — the ownership check in `deleteReel`, built on Day 6)
- Connected `protect` to the reel routes that need a logged-in user
- Found and documented a real bug (wrong status code on invalid tokens) instead of hiding it

---

## 💡 Concepts Learned (quick revision)

| Concept | One-line meaning |
|---|---|
| `Authorization: Bearer <token>` | The standard HTTP header format for sending a token |
| `jwt.verify()` | Checks a token's signature against the secret key; throws if invalid |
| Authentication | Confirming *who* the user is |
| Authorization | Confirming *what* the user is allowed to do |
| `req.user` | Custom property attached by middleware, available to every later handler in the same request |
| `.select('-password')` | Excludes a specific field from a Mongoose query result |
| Middleware chaining | `router.post('/', protect, uploadReel)` runs `protect` first, then `uploadReel`, only if `protect` calls `next()` |

---

# ❓ Interview Questions — Day 4

**How many to learn?** Q1–Q6 are must-know (asked in nearly every backend/auth interview). Q7–Q9 are strong bonus depth, especially Q9 which is an honest project question.

## 🔴 Must-know

**Q1. What is the difference between authentication and authorization?**
Authentication is confirming **who** the user is — my `protect` middleware verifies the JWT to establish this. Authorization is confirming **what** that user is allowed to do — for example, in my `deleteReel` service, I separately check that the logged-in user is actually the one who posted the reel before letting them delete it.

**Q2. Walk me through what your `protect` middleware does, step by step.**
It checks if the request has an `Authorization` header starting with `"Bearer"`. If not, it responds with 401. If it does, it extracts the token, verifies it with `jwt.verify()` using my secret key, looks up the user in the database by the ID inside the token, attaches that user to `req.user`, and calls `next()` to let the request continue to the actual controller.

**Q3. Why read the token from a header instead of the request body?**
It's the standard convention (`Authorization: Bearer <token>`), and it keeps authentication data separate from the actual data being sent in the request.

**Q4. What does `jwt.verify` actually check?**
It confirms the token's signature matches what would be produced using my server's secret key — proving the token wasn't tampered with and wasn't created by someone without access to that secret. It also checks the token hasn't expired.

**Q5. Why fetch the user again from the database instead of just trusting the ID in the JWT?**
The JWT payload only stores the `id`. Fetching the user fresh from the database ensures I get their **current** role, name and other fields — not stale data from whenever the token was originally issued. It also confirms the user still exists.

**Q6. What does `.select('-password')` do here, and why is it needed?**
It excludes the password field from the returned user document. It's needed because my `User.js` schema doesn't mark the password field `select: false` by default, so without this, the hashed password would be attached to `req.user` unnecessarily.

## 🟢 Bonus depth

**Q7. What is middleware chaining, and how does `router.post('/', protect, uploadReel)` work?**
Express runs middleware functions in the order they're listed for a route. `protect` runs first; if it calls `next()`, control passes to `uploadReel`. If `protect` never calls `next()` (like in the 401 case), `uploadReel` never runs at all.

**Q8. Why is `protect` wrapped in `asyncHandler` even though it's middleware, not a typical controller?**
Because it contains `await` calls (`jwt.verify` isn't awaited but `User.findById` is) inside an async function — if either throws, `asyncHandler` ensures the error is passed to `next()` and handled centrally, instead of crashing or hanging silently.

**Q9. 🧑‍💼 (Project) Is there a bug in your current authentication code you're aware of?**
"Yes — right now, if someone sends an invalid or expired token, `jwt.verify` throws before I've set a status code, so my error middleware's fallback logic defaults to a 500 Internal Server Error instead of the more accurate 401 Unauthorized. Only the 'completely missing token' case correctly returns 401. My planned fix is to wrap the verification in a try/catch and explicitly set `res.status(401)` in that failure path too."
