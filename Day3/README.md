📓 Zomansta Development Journal
📅 Day 03: Authentication feature — service, controller, routes
🎯 Goal
Build the full signup/login flow: hash the password, create the user, generate a token, and expose it through an API endpoint.

⚠️ Every code block below is copied exactly from your own terminal output. Nothing is invented.

📁 Files Created Today
File	Purpose
src/services/authService.js	Business logic: register and login
src/controllers/authController.js	Reads req, calls the service, sends res
src/routes/authRoutes.js	Maps URLs to controller functions
1️⃣ src/services/authService.js
const User = require('../models/User');
const bcrypt = require('bcryptjs');
const generateToken = require('../utils/generateToken');
const registerUser = async (name, email, password) => {
  const userExists = await User.findOne({ email });
  if (userExists) {
    throw new Error('User already exists');
  }
  const salt = await bcrypt.genSalt(10);
  const hashedPassword = await bcrypt.hash(password, salt);
  const user = await User.create({
    name,
    email,
    password: hashedPassword
  });
  return {
    _id: user._id,
    name: user.name,
    email: user.email,
    role: user.role,
    token: generateToken(user._id)
  };
};
const loginUser = async (email, password) => {
  const user = await User.findOne({ email });
  if (!user) {
    throw new Error('Invalid email or password');
  }
  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) {
    throw new Error('Invalid email or password');
  }
  return {
    _id: user._id,
    name: user.name,
    email: user.email,
    role: user.role,
    token: generateToken(user._id)
  };
};
module.exports = { registerUser, loginUser };
What I learned
bcrypt.genSalt(10) generates a random salt with 10 rounds — more rounds means slower and harder to brute-force. 10 is a common balance between security and speed.
bcrypt.hash(password, salt) produces the one-way hash that's actually stored in the DB — never the plain password.
Login never decrypts the stored hash. bcrypt.compare(plainPassword, storedHash) re-hashes the entered password internally and compares the two hashes.
Both functions return the same shape (_id, name, email, role, token), so the controller can treat register and login the same way without extra branching.
Returning 'Invalid email or password' for both a wrong email and a wrong password (instead of "email not found" vs "wrong password") is a small security habit — it stops attackers from figuring out which emails are actually registered.
User.findOne({ email }) is a Mongoose query that returns the first matching document, or null if none exists.
2️⃣ src/controllers/authController.js
const asyncHandler = require('../utils/asyncHandler');
const { registerUser, loginUser } = require('../services/authService');
const register = asyncHandler(async (req, res) => {
  const { name, email, password } = req.body;
  if (!name || !email || !password) {
    res.status(400);
    throw new Error('Please fill all fields');
  }
  const user = await registerUser(name, email, password);
  res.status(201).json({
    success: true,
    data: user
  });
});
const login = asyncHandler(async (req, res) => {
  const { email, password } = req.body;
  if (!email || !password) {
    res.status(400);
    throw new Error('Please fill all fields');
  }
  const user = await loginUser(email, password);
  res.status(200).json({
    success: true,
    data: user
  });
});
module.exports = { register, login };
What I learned
const { name, email, password } = req.body is object destructuring — pulls the three fields straight out of the request body.
The pattern res.status(400); throw new Error('...') sets the status code before throwing, so when the error reaches errorMiddleware.js, res.statusCode is already 400 instead of falling back to the default 500.
asyncHandler wraps both register and login, so if registerUser or loginUser throws (like "User already exists"), it's automatically caught and passed to the error handler — no manual try/catch needed here.
register sends status 201 (Created) because a new user document was created. login sends status 200 (OK) because nothing new was created, just an existing resource accessed.
3️⃣ src/routes/authRoutes.js
const express = require('express');
const router = express.Router();
const { register, login } = require('../controllers/authController');
router.post('/register', register);
router.post('/login', login);
module.exports = router;
What I learned
express.Router() creates a mini, self-contained router that gets mounted in app.js later (as /api/auth).
Both routes use POST, because both actions create or process data sent by the client, not just read it.
Endpoints created: POST /api/auth/register, POST /api/auth/login

🔎 Honest observation
There's no express-validator check anywhere in this flow, even though the package is installed. Right now validation only checks "is this field empty or missing" — not things like "is this actually a valid email format" or "is the password strong enough." That's a fair, honest thing to mention as a planned improvement if an interviewer asks about input validation.

🧪 How I Tested It (Postman)
Register POST http://localhost:5000/api/auth/register Body → raw → JSON:

{ "name": "Sanjana", "email": "sanjana@example.com", "password": "123456" }
Expected: status 201, JSON with success: true and a data object containing a token.

Login POST http://localhost:5000/api/auth/login Body → raw → JSON:

{ "email": "sanjana@example.com", "password": "123456" }
Expected: status 200, same response shape as register.

Error case to also try: send login with a wrong password, and confirm you get 'Invalid email or password' with status 400 — not a 500 or a crash.

💾 Git: Saving Today's Work
Run only inside C:\Users\kyasa\zomansta\backend:

git pull
git status
git add .
git commit -m "Add auth service, controller and routes"
git push
✅ git status must not show .env or node_modules.

✅ Achievements
Built the full register → hash password → save user → generate token flow
Built the full login → find user → compare password → generate token flow
Connected auth logic to real HTTP endpoints (/api/auth/register, /api/auth/login)
Understood the difference between hashing and comparing passwords with bcrypt
Understood why register returns 201 and login returns 200
💡 Concepts Learned (quick revision)
Concept	One-line meaning
Salt	Random data mixed into a password before hashing, so identical passwords produce different hashes
Hash	A one-way scrambled version of the password; can't be reversed back to plain text
bcrypt.compare	Re-hashes the entered password and compares it to the stored hash — never decrypts
Destructuring	Pulling specific fields out of an object in one line (const { a, b } = obj)
express.Router()	A mini router that groups related routes, mounted into the main app later
201 vs 200	201 = a new resource was created; 200 = an existing resource was read or processed
❓ Interview Questions — Day 3
How many to learn? Q1–Q7 are must-know (asked in almost every Node/auth interview). Q8–Q10 are good bonus depth.

🔴 Must-know
Q1. Walk me through your signup flow, step by step. The route receives a POST request → the controller checks name, email and password are present, returning 400 if not → it calls registerUser in the service → the service checks if the email already exists, hashes the password with bcrypt, creates the User document, and generates a JWT → the controller sends back status 201 with the user data and token.

Q2. Why hash the password in the service layer and not directly in the controller? The controller's job is just to handle HTTP (read the request, send the response). Business logic like hashing belongs in the service layer, next to the other user-related rules, so it stays reusable and testable.

Q3. What is a "salt" in bcrypt, and why not just hash the password directly? A salt is random data mixed in before hashing, so two users with the identical password end up with completely different stored hashes. Without a salt, an attacker could precompute hashes for common passwords ("rainbow tables") and instantly crack them.

Q4. How does bcrypt.compare verify a password without ever decrypting the hash? Hashing is one-way — it can't be reversed. bcrypt.compare(enteredPassword, storedHash) re-hashes the entered password using the same salt embedded in the stored hash, then checks if the two resulting hashes match exactly.

Q5. Why does your code return the exact same error message for a wrong email and a wrong password? So an attacker can't tell whether a given email is even registered in the system. If "email not found" and "wrong password" gave different messages, it would leak which emails exist — a real security habit called avoiding "user enumeration."

Q6. Why does register return status 201 while login returns 200? 201 means a new resource was created (a new user document). Login doesn't create anything new, it just verifies and returns existing data, so it's a plain 200 OK.

Q7. What does asyncHandler do for your register and login functions specifically? Both are async functions using await. If anything inside throws (like 'User already exists'), asyncHandler catches that rejection with .catch(next) and forwards it to the error middleware — without me writing try/catch in every controller.

🟢 Bonus depth
Q8. Why generate the JWT inside the service instead of the controller? Keeps token creation next to where the user was just authenticated/created, so the service returns a complete, ready-to-use response object, and the controller only decides the HTTP status and shape of the reply.

Q9. What's missing from this auth flow that you'd add for production? Proper input validation with express-validator (checking real email format, minimum password strength), rate limiting specifically on login attempts to prevent brute-forcing, and possibly email verification before allowing login.

Q10. What does User.findOne({ email }) return if no user matches? It returns null, which is why the code checks if (!user) before trying to compare passwords — attempting bcrypt.compare on a null user's password would throw a different, confusing error.
