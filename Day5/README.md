📓 Zomansta Development Journal
📅 Day 05: The Reel model — Reel.js
🎯 Goal
Model the core content of the app: a food reel, linked to the restaurant it's from and the user who posted it, with likes, saves and a ranking score.

⚠️ Every code block below is copied exactly from your own terminal output. Nothing is invented.

📁 Files Created Today
File	Purpose
src/models/Reel.js	MongoDB schema for a food reel
1️⃣ src/models/Reel.js
const mongoose = require('mongoose');
const reelSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    default: ''
  },
  videoUrl: {
    type: String,
    required: true
  },
  thumbnail: {
    type: String,
    default: ''
  },
  restaurant: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Restaurant',
    required: true
  },
  postedBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  likes: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }],
  saves: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }],
  tags: [String],
  score: {
    type: Number,
    default: 0
  }
}, { timestamps: true });

module.exports = mongoose.model('Reel', reelSchema);
What I learned
title, videoUrl, restaurant, postedBy are required: true — a reel cannot exist without these. description and thumbnail are optional, with sensible empty defaults.
restaurant: { ref: 'Restaurant', required: true } — every reel must belong to a real restaurant. This is a relationship, not the restaurant's actual data — just its ObjectId, which can later be expanded with .populate().
postedBy: { ref: 'User', required: true } — same idea, linking the reel to the user who created it.
likes and saves are arrays of User ObjectIds, not plain numbers. This is a deliberate design choice: storing who liked or saved a reel (not just a count) lets me check "has this specific user already liked this?" and toggle the action — a plain counter couldn't do that. This is used directly in reelService.js (Day 06).
tags: [String] — a plain array of strings with no schema restrictions, for things like #spicy or #vegan.
score: { type: Number, default: 0 } — starts at 0 for every new reel. It isn't calculated automatically by the schema; it's manually recalculated in reelService.js whenever someone likes or saves a reel (likes.length * 3 + saves.length * 5).
timestamps: true — automatically adds createdAt and updatedAt. Combined with score, this lets me sort reels by { score: -1, createdAt: -1 } later — highest score first, and newest first as a tiebreaker.
🔎 Honest Observation
This schema references ref: 'Restaurant', meaning it expects a Restaurant model to exist elsewhere in the app. I have not been able to confirm Restaurant.js is actually present in my src/models folder — it only came up earlier in a planning conversation, not in my verified terminal file listing.

If it's genuinely missing: Reel.create({ ... restaurant: someId ... }) would still technically work, because Mongoose doesn't check that a referenced ID actually points to an existing Restaurant document unless I add extra validation — but .populate('restaurant', 'name') (used in reelService.js, Day 06) would just return null for that field instead of real restaurant data.

Action for me: open src/models in VS Code and confirm whether Restaurant.js exists. If it doesn't, that's a good thing to know before an interview, not after — I can either build it properly or be upfront that it's planned but not yet implemented.

🧭 How This Connects to Later Days
Day 06 (reelService.js, reelController.js, reelRoutes.js) builds the actual CRUD operations and the like/save toggle logic on top of this schema.
Day 07 (Comment.js, SavedReel.js) references this Reel model the same way this file references User and Restaurant.
💾 Git: Saving Today's Work
Run only inside C:\Users\kyasa\zomansta\backend:

git pull
git status
git add .
git commit -m "Add Reel model"
git push
✅ git status must not show .env or node_modules.

✅ Achievements
Designed the Reel schema with required fields, relationships, and defaults
Understood why likes/saves are arrays of user IDs instead of plain counters
Understood how score and timestamps work together for future sorting/ranking
Identified a real open question (whether Restaurant.js actually exists) instead of assuming
💡 Concepts Learned (quick revision)
Concept	One-line meaning
Reference field (ref)	Stores another document's ObjectId, not its full data — expanded later with .populate()
Array of ObjectIds	Lets me check membership (.includes(userId)) and toggle it — plain numbers can't do this
default: '' / default: 0	Value used automatically if the field isn't provided when creating the document
Denormalized score field	A calculated value stored directly on the document for fast sorting, instead of recalculating it on every read
Unvalidated reference	Mongoose doesn't check that a ref'd ID actually exists unless I add that logic myself
❓ Interview Questions — Day 5
How many to learn? Q1–Q5 are must-know. Q6–Q8 are strong bonus depth.

🔴 Must-know
Q1. Why did you store likes and saves as arrays of user IDs instead of just incrementing a number? A plain number can only tell me how many people liked something — not who. Storing user IDs lets me check whether a specific logged-in user already liked a reel (reel.likes.includes(userId)), which is exactly what I need to build a toggle-style like button, and to prevent the same user from being counted twice.

Q2. What does ref: 'Restaurant' actually store in the database? Just the restaurant's ObjectId — a 12-byte reference, not the restaurant's actual data. To get the full restaurant details, I'd call .populate('restaurant') when querying, which Mongoose uses to fetch and attach the referenced document.

Q3. Why are title, videoUrl, restaurant, and postedBy required, but not description or thumbnail? Those four are essential for a reel to make any sense at all — you can't have a food reel with no video or no restaurant. Description and thumbnail are nice-to-haves I chose to default to an empty string instead of blocking creation.

Q4. What is score for, and is it calculated automatically? It's a ranking number used to sort reels, similar to a simple trending algorithm. It's not automatic — the schema just defaults it to 0. It's manually recalculated in my service layer (reelService.js) as likes.length * 3 + saves.length * 5 whenever someone likes or saves a reel.

Q5. What does timestamps: true give you here, and how do you use it? It auto-adds createdAt and updatedAt. I use createdAt as a tiebreaker when sorting reels by score — .sort({ score: -1, createdAt: -1 }) — so equally popular reels show newest first.

🟢 Bonus depth
Q6. Does MongoDB or Mongoose check that a ref'd ObjectId actually points to a real document? No, not by default. I could save a Reel with a restaurant ID that doesn't exist in the Restaurant collection, and Mongoose wouldn't complain at save time. It would only become obvious later, when .populate('restaurant') returns null for that field instead of real data.

Q7. Why did you choose to weight saves higher than likes in your score formula? Saving a reel shows more intent than a quick like — someone saving it usually means they want to come back to it later, so I gave it more weight (5 points vs 3) in the ranking formula.

Q8. There's an open question about whether your Restaurant model actually exists yet — how would you answer that honestly in an interview? "My Reel schema references a Restaurant model by design, since every reel should belong to a restaurant. I need to double-check whether I've actually implemented that model yet in my current codebase, or whether it's still a planned piece — I want to give you an accurate answer rather than guess." That's a completely fine, professional answer.

📌 Revision Plan for This Day
Say Q1–Q5 out loud, pointing at the actual schema fields as you explain them.
Draw the relationship on paper: Reel → restaurant (ObjectId) → Restaurant document and Reel → postedBy (ObjectId) → User document.
Before your next interview: actually check src/models in VS Code for Restaurant.js and update this note — don't walk in still unsure.
