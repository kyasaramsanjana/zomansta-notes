# 📓 Zomansta Development Journal

## 📅 Day 01 

---

# 🎯 Goal

Set up the backend foundation for **Zomansta** by planning the architecture, creating the folder structure, connecting MongoDB Atlas, and preparing the project for future development.

---

# 🍽️ What is Zomansta?

**Zomansta** is an AI-powered social food discovery platform that combines:

- 🍴 Zomato-style restaurant discovery
- 🎥 Instagram-style food reels
- 🤖 AI-powered food recommendations
- ❤️ Likes, comments, bookmarks, and follows
- 🔔 Real-time notifications using Socket.io
- 📍 Location-based restaurant discovery
- ⭐ Restaurant reviews and ratings

---

# 🏗️ Technology Stack

| Layer | Technology |
|--------|------------|
| Frontend | Next.js, Tailwind CSS, Framer Motion |
| State Management | Redux Toolkit |
| Backend | Node.js, Express.js |
| Database | MongoDB Atlas |
| Authentication | JWT, bcryptjs |
| Media Storage | Cloudinary |
| AI | Gemini API |
| Real-Time | Socket.io |
| Security | Helmet, Express Rate Limit, Express Validator |
| Deployment | Vercel, Render |

---

# 📂 Backend Folder Structure

```text
backend/
│
├── src/
│   ├── config/
│   ├── constants/
│   ├── controllers/
│   ├── middleware/
│   ├── models/
│   ├── routes/
│   ├── services/
│   ├── socket/
│   ├── utils/
│   └── validators/
│
├── .env
├── .gitignore
├── package.json
└── server.js
```

---

# 📁 Why This Folder Structure?

I decided to follow a scalable project structure instead of placing everything inside one file.

### config/

Stores configuration files.

Example:

- MongoDB connection
- Cloudinary configuration

---

### controllers/

Receives requests from routes and sends responses back to the client.

Example:

- Login Controller
- Register Controller

---

### services/

Contains business logic.

Controllers remain small because all processing happens inside services.

---

### models/

Contains MongoDB schemas.

Example:

- User Schema
- Restaurant Schema
- Review Schema

---

### routes/

Defines API endpoints.

Example:

```
POST /login
GET /restaurants
```

---

### middleware/

Contains reusable middleware.

Examples:

- Authentication
- Error Handling
- Authorization

---

### validators/

Validates incoming request data.

Example:

- Email validation
- Password validation

---

### socket/

Contains Socket.io events.

Examples:

- Notifications
- Live Chat
- Real-time updates

---

### utils/

Contains helper functions.

Examples:

- Generate JWT
- Upload Image
- Format Date

---

# 📦 Packages Installed

## Production Dependencies

- express
- mongoose
- dotenv
- bcryptjs
- jsonwebtoken
- cors
- helmet
- express-rate-limit
- express-validator
- multer
- cloudinary
- socket.io

---

## Development Dependency

- nodemon

---

# 📄 Files Created

## server.js

Project entry point.

Responsibilities:

- Load environment variables
- Connect MongoDB
- Create Express app
- Parse JSON
- Start server

---

## src/config/db.js

Responsible for connecting the application to MongoDB Atlas.

---

## .env

Stores secret environment variables.

Example:

```
PORT=5000
MONGO_URI=Your MongoDB Atlas Connection String
```

This file should never be pushed to GitHub.

---

## .gitignore

Used to prevent Git from tracking sensitive or unnecessary files.

Ignored files:

```
node_modules/
.env
```

---

# 🔄 Request Flow

```
Client
   │
   ▼
Express Server
   │
   ▼
Routes
   │
   ▼
Controllers
   │
   ▼
Services
   │
   ▼
Models
   │
   ▼
MongoDB Atlas
```

---

# 💡 Concepts Learned

## Express.js

A minimal Node.js framework used to build APIs quickly.

---

## MongoDB Atlas

A cloud-hosted MongoDB database that can be accessed from anywhere.

---

## dotenv

Loads environment variables from the `.env` file.

Used to protect:

- Database passwords
- API Keys
- JWT Secret

---

## Helmet

Adds HTTP security headers to improve application security.

---

## Express Rate Limit

Protects APIs from excessive requests and brute-force attacks.

---

## .gitignore

Prevents Git from uploading unnecessary or sensitive files.

---

# 🐞 Problems Faced

### Problem

Initially needed to ensure MongoDB Atlas credentials were not exposed.

### Solution

Stored sensitive values inside the `.env` file and added `.env` to `.gitignore`.

---

# ✅ Achievements

- Planned complete backend architecture
- Created scalable folder structure
- Installed all required backend packages
- Created Express server
- Connected MongoDB Atlas
- Successfully started backend server
- Protected secret credentials
- Pushed backend project to GitHub

---

# ❓ Interview Questions

## Why did you use Express.js?

Express simplifies routing, middleware, request handling, and API development compared to the built-in Node.js HTTP module.

---

## Why use MongoDB Atlas?

Because it is cloud-hosted, production-ready, scalable, and accessible from anywhere.

---

## Why use dotenv?

To keep sensitive information outside the source code.

---

## Why keep .env inside .gitignore?

To prevent passwords, API keys, and secrets from being uploaded to GitHub.

---

## Why use Helmet?

Helmet helps secure Express applications by setting various HTTP security headers.

---

## Why create separate folders?

Separating responsibilities makes the application easier to maintain, debug, test, and scale.

---

# 📝 Git Progress

Completed the initial backend setup and successfully pushed the project to GitHub.

---

# 🎯 Tomorrow's Goal

- Create User Model
- Design User Schema
- Build Register API
- Build Login API
- Learn Password Hashing with bcryptjs

---

# ⭐ Biggest Learning Today

A well-planned architecture makes future development much easier. Spending time organizing the project at the beginning helps keep the codebase clean, scalable, and maintainable as new features are added.
