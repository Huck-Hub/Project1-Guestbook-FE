# Project 1: Community Guestbook & Feedback Wall (Full-Stack MEVN)

A complete **Full-Stack MEVN (MongoDB, Express.js, Vue.js 3, Node.js)** application built using the exact vanilla patterns from **CST3144 Full Stack Development**, styled with a modern **Digital Lavender & Soft Tech Glassmorphism** UI/UX theme.

---

## 1. Folder Structure

```text
Project1_Guestbook/
├── README.md                 # Complete Project & Line-by-Line Code Guide
├── frontend/                 # Separate Git Repository for GitHub Pages
│   └── index.html            # Vue 3 CDN Single-Page Application + Glassmorphic CSS
└── backend/                  # Separate Git Repository for Render.com
    ├── .gitignore            # Excludes node_modules/ and config.env from Git
    ├── config.env            # Private MongoDB Atlas credentials (local only)
    ├── package.json          # Node.js dependencies (express, cors, dotenv, mongodb)
    ├── reviews.json          # Seed data for MongoDB Atlas "reviews" collection
    ├── server.js             # Express REST API + Middlewares + MongoDB Native Driver
    └── staticFiles/          # Avatar PNG images served by Express static middleware
        ├── avatar1.png ... avatar6.png
```

---

## 2. UI/UX Theme: "Digital Lavender & Soft Tech Glassmorphism"

Researched from modern 2026 SaaS & testimonial wall UI/UX trends:
* **Color Palette:**
  * Deep Slate Background (`#0F1117`) with soft radial lavender/indigo ambient light glows to prevent eye strain.
  * Frosted Glass Cards (`rgba(255, 255, 255, 0.035)`) with `backdrop-filter: blur(12px)` and subtle 1px translucent borders (`rgba(255, 255, 255, 0.08)`).
  * Digital Lavender & Electric Indigo Gradient Accents (`linear-gradient(135deg, #8E8BFF 0%, #C084FC 100%)`).
  * Golden Star Ratings (`#FACC15`) and Emerald Success Banners (`#34D399`).
* **Micro-Interactions:**
  * Hovering over any guestbook card lifts it smoothly (`transform: translateY(-4px)`) and illuminates its border with a soft lavender glow.
  * 5-star reviews automatically receive a `.featured` gradient highlight via Vue's `v-bind:class="{ featured: review.rating === 5 }"`.

---

## 3. Line-by-Line Code Explanation: Back-End (`backend/server.js`)

### Lines 1–5: Loading Core Modules & Environment Variables
```javascript
var express = require("express");
var cors = require("cors");
var path = require("path");
var fs = require("fs");
require("dotenv").config({ path: "./config.env" });
```
* **`require("express")`**: Loads the Express.js web server framework.
* **`require("cors")`**: Loads Cross-Origin Resource Sharing middleware so the Vue Front-End (on a different port or GitHub Pages) is allowed to call this server.
* **`require("path")` & `require("fs")`**: Built-in Node.js modules for resolving file paths and reading our local `reviews.json` seed file.
* **`require("dotenv").config({ path: "./config.env" })`**: Reads sensitive variables (`MONGODB_USERNAME`, `MONGODB_PASSWORD`, etc.) from `config.env` into `process.env` so passwords are never hardcoded in `server.js`.

### Lines 8–18: Setting Up the Native MongoDB Driver (& Local Seed Fallback)
```javascript
const { MongoClient, ObjectId } = require("mongodb");
const hasAtlasConfig = process.env.MONGODB_USERNAME && process.env.MONGODB_USERNAME !== "your_username";
const uri = process.env.MONGODB_PREFIX +
    process.env.MONGODB_USERNAME + ":" +
    process.env.MONGODB_PASSWORD +
    process.env.MONGODB_URL;
const client = hasAtlasConfig ? new MongoClient(uri) : null;
const db = client ? client.db(process.env.MONGODB_DB || "GuestbookDB") : null;

let localReviews = JSON.parse(fs.readFileSync(path.resolve(__dirname, "reviews.json"), "utf8"));
```
* **`MongoClient`**: Connects to your MongoDB Atlas cluster using the connection string `uri`.
* **`ObjectId`**: Converts a 24-character string ID from a URL parameter into a MongoDB `_id` object for `PUT` updates.
* **`localReviews`**: Reads `reviews.json` into memory so if you run `node server.js` before configuring MongoDB Atlas, the app still works 100% out of the box! Once you put your real Atlas credentials in `config.env`, `hasAtlasConfig` is `true` and all routes read/write directly to MongoDB Atlas.

### Lines 20–37: Express App & Middleware Pipeline
```javascript
var app = express();

app.use(cors());
app.use(express.json());

// Logger Middleware
app.use(function(request, response, next) {
    console.log("Incoming " + request.method + " request to " + request.url + " [" + new Date().toISOString() + "]");
    next();
});

// Static File Middleware
var publicPath = path.resolve(__dirname, "staticFiles");
app.use("/images", express.static(publicPath));
app.use("/images", function(request, response) {
    response.status(404).send("Image not found!");
});
```
1. **`app.use(cors())`**: Adds `Access-Control-Allow-Origin` headers to every response.
2. **`app.use(express.json())`**: Parses incoming JSON request bodies (for `POST` and `PUT` requests) into `request.body`.
3. **Logger Middleware**: Prints the HTTP method (`GET`, `POST`, `PUT`), the URL path (`/reviews`, `/search?q=...`), and the timestamp for every request, then calls `next()` to continue down the pipeline.
4. **Static File Middleware (`/images`)**:
   * First checks if the requested file exists inside `backend/staticFiles/` (e.g. `avatar1.png`) and sends it.
   * If the file does not exist, `express.static` calls `next()`, which hits the second `app.use("/images", ...)` and returns HTTP `404` with `"Image not found!"`.

### Lines 48–116: REST API Routing Functions (`GET`, `POST`, `PUT`)
* **`GET /reviews` (Lines 48–55)**: Queries `db.collection("reviews").find({}).toArray()` and returns all guestbook reviews as JSON.
* **`GET /search?q=...` (Lines 58–73)**: Reads `request.query.q`, fetches all reviews from the database, and filters them in the Back-End across `name`, `role`, `comment`, and `rating` (converting numbers to strings with `String(...)` so searching `"5"` matches 5-star reviews).
* **`POST /reviews` (Lines 76–91)**: Takes the new review object from `request.body` and inserts it into the MongoDB `"reviews"` collection using `collection.insertOne(body)`.
* **`PUT /reviews/:_id` (Lines 94–115)**: Reads the document ID from `request.params._id`, builds `{ _id: new ObjectId(tempid) }`, and updates the fields sent in `request.body` (such as `{ likes: 15 }`) using `collection.updateOne(query, { $set: request.body })`.
* **404 Fallback & `app.listen` (Lines 118–125)**: Any unrecognized route returns `"Page not found!"` with status `404`, and the server listens on `process.env.PORT || 3000`.

---

## 4. Line-by-Line Code Explanation: Front-End (`frontend/index.html`)

### Head & Libraries (Lines 1–13)
* Loads **Font Awesome 4.7** for star, thumbs-up, check-circle, and pencil icons.
* Loads **Vue 3** from the official CDN (`https://unpkg.com/vue@3/dist/vue.global.js`).

### Header & Live Statistics Bar (Lines 269–288)
* **`<h1 v-text="sitename"></h1>`**: Binds the page title to `sitename` in Vue `data()`.
* **`{{ totalReviews }}` and `{{ averageRating }}`**: Display live statistics calculated automatically by Vue `computed` properties whenever reviews are fetched, searched, or added.
* **`<button class="btn-primary" v-on:click="toggleForm">`**: Toggles the Boolean `showForm` between `false` (Guestbook Wall view) and `true` (Sign Guestbook form view).

### View 1: Guestbook Wall (`<div v-if="!showForm">`, Lines 292–368)
* **Toolbar Controls (Lines 294–333)**:
  * `<input v-model="searchQuery">` — 2-way bound to `searchQuery`.
  * `<select v-model="filterRating">` — Filters by All Ratings, 5 Stars, 4 Stars, or 3 Stars.
  * `<select v-model="sortBy">` & `<select v-model="sortOrder">` — Lets the user sort by Likes, Star Rating, Author Name, or Date Posted in Ascending or Descending order.
* **Guestbook Card Loop (`v-for="review in sortedReviews"`, Lines 336–367)**:
  * **`v-bind:class="{ featured: review.rating === 5 }"`**: Dynamically adds the `.featured` CSS class to 5-star reviews.
  * **`<span v-for="n in review.rating">★</span>`**: Repeats the gold star symbol `review.rating` times (`1` to `5`).
  * **`<img v-bind:src="imageAddress(review)">`**: Loads the reviewer's avatar image directly from the Express Back-End's `/images/...` static middleware.
  * **`<button class="like-btn" v-on:click="likeReview(review)">`**: Increments `review.likes` by 1 and sends an asynchronous `PUT` request to `/reviews/:_id` to persist the new like count in MongoDB.

### View 2: Sign Guestbook Form (`<div v-else>`, Lines 371–422)
* Uses **`v-model`** on Name, Role, Star Rating (`v-model.number`), Feedback Comment, and Verified checkbox.
* **`<button v-on:click="submitReview" v-bind:disabled="!isValidReview">`**:
  * Stays visible at all times, but is **disabled** until `isValidReview` returns `true` (checking via JavaScript Regular Expressions `/^[A-Za-z\s]+$/` that `name` and `role` contain letters only, and `comment` is at least 5 characters long).

### Vue 3 Application Script (Lines 426–564)
* **`data()` (Lines 430–449)**: Stores reactive state (`sitename`, `showForm`, `reviews`, `searchQuery`, `filterRating`, `sortBy`, `sortOrder`, `newReview`, `submitSuccess`).
* **`created()` (Lines 450–452)**: Vue lifecycle hook that automatically calls `this.fetchReviews()` when the page loads.
* **`watch: { searchQuery: "searchReviews" }` (Lines 453–455)**: Watches `searchQuery` and calls `searchReviews()` (`GET /search?q=...`) every time the user types in the search box.
* **`methods` (Lines 456–519)**:
  * `fetchReviews()`: Uses `await fetch(BACKEND_URL + "/reviews")` to load all reviews.
  * `searchReviews()`: Uses `await fetch(BACKEND_URL + "/search?q=" + ...)` for full-stack search.
  * `likeReview(review)`: Uses `await fetch(BACKEND_URL + "/reviews/" + id, { method: "PUT", ... })` to update likes.
  * `submitReview()`: Uses `await fetch(BACKEND_URL + "/reviews", { method: "POST", ... })` to save a new guestbook entry and refresh the list.
* **`computed` (Lines 520–560)**:
  * `totalReviews()`, `averageRating()`, `sortedReviews()`, and `isValidReview()`.

---

## 5. How to Run Locally & Deploy to GitHub, MongoDB Atlas, and Render

### A. Run Locally
1. Start the Back-End server:
   ```bash
   cd /home/j30/Documents/Project1_Guestbook/backend
   node server.js
   ```
2. Open `/home/j30/Documents/Project1_Guestbook/frontend/index.html` with VS Code **Live Preview** (or in your browser).

### B. Connect MongoDB Atlas
1. In MongoDB Atlas, create a database named **`GuestbookDB`** with a collection named **`reviews`**.
2. Import `backend/reviews.json` into the `reviews` collection.
3. Update `backend/config.env` with your MongoDB Atlas username, password, and cluster URL.

### C. Push to GitHub & Host Online
1. **Back-End (`backend/`) → GitHub + Render.com:**
   * Create a public GitHub repo (e.g. `project1-guestbook-backend`) and push `backend/`.
   * Connect that repo on **Render.com** as a Web Service (`node server.js`) and add your `config.env` variables under Render's **Environment Variables**.
2. **Front-End (`frontend/`) → GitHub + GitHub Pages:**
   * Update `BACKEND_URL` in `frontend/index.html` to your Render URL (when deploying live).
   * Create a public GitHub repo (e.g. `project1-guestbook-frontend`), push `frontend/`, and enable **GitHub Pages** on the `main` branch!
