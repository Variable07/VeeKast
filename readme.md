<div align="center">

<br/>

<img src="https://img.shields.io/badge/VeeKast-Video%20Platform%20Backend-dc2626?style=for-the-badge&logo=youtube&logoColor=white" height="36"/>

<br/><br/>

# 🎬 VeeKast — Production-Grade Video Streaming Backend

**Stream · Share · Subscribe**

<br/>

[![Express](https://img.shields.io/badge/Express-5.x-000000?style=flat-square&logo=express&logoColor=white)](https://expressjs.com/)
[![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose%208-47A248?style=flat-square&logo=mongodb&logoColor=white)](https://mongodb.com/)
[![Cloudinary](https://img.shields.io/badge/Cloudinary-Media%20CDN-3448C5?style=flat-square&logo=cloudinary&logoColor=white)](https://cloudinary.com/)
[![JWT](https://img.shields.io/badge/JWT-Auth-000000?style=flat-square&logo=jsonwebtokens&logoColor=white)](https://jwt.io/)
[![Node.js](https://img.shields.io/badge/Node.js-18+-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![License: ISC](https://img.shields.io/badge/License-ISC-blue?style=flat-square)](LICENSE)

<br/>

> A production-ready REST API backend for a YouTube-inspired video sharing platform. Built with Express 5, MongoDB aggregation pipelines, JWT-based auth with refresh token rotation, Cloudinary media hosting, and smart chunked uploads for large video files.

<br/>

[📡 API Reference](#-api-reference) · [🚀 Getting Started](#-getting-started) · [🐛 Report Bug](issues)

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Data Models](#-data-models)
- [Request Flow](#-request-flow)
- [Features](#-features)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Getting Started](#-getting-started)
- [Environment Variables](#-environment-variables)
- [API Reference](#-api-reference)
- [Utility Layer](#-utility-layer)
- [Roadmap](#-roadmap)

---

## 🔭 Overview

VeeKast is a backend-only REST API that models the core functionality of a video streaming platform. It covers the full lifecycle — from user registration with avatar uploads, to video publishing on Cloudinary, channel subscriptions tracked via MongoDB aggregation, and a tweet-style social layer.

| Capability | Details |
|---|---|
| 🔐 Auth | JWT access tokens + refresh token rotation via HTTP-only cookies |
| 🔒 Security | bcrypt password hashing, token verification middleware |
| 🎥 Videos | Upload to Cloudinary, chunked support for large files, CRUD |
| 👤 Users | Register, login, channel profiles, watch history |
| 🔔 Subscriptions | Subscribe/unsubscribe, subscriber count via aggregation |
| 🐦 Tweets | Create, edit, delete, fetch all tweets per user |
| 💬 Comments | Schema defined, controller scaffolded (in progress) |
| ❤️ Likes | Polymorphic — supports videos, comments, and tweets |
| 📋 Playlists | Schema defined, endpoints planned |
| 📄 Pagination | `mongoose-aggregate-paginate-v2` for video and comment feeds |

---

## 🏛️ Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        CLIENT / FRONTEND                          │
│            (Postman · Custom Frontend · Mobile App)               │
└──────────────────────────────┬───────────────────────────────────┘
                               │ HTTP Requests
┌──────────────────────────────▼───────────────────────────────────┐
│                    Express 5 Application                           │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                     Middleware Stack                          │  │
│  │  cors() → express.json() → urlencoded() → cookieParser()    │  │
│  │  static("public") → verifyJwtToken (protected routes)       │  │
│  └──────────────────────────────┬──────────────────────────────┘  │
│                                 │                                  │
│  ┌──────────────────────────────▼──────────────────────────────┐  │
│  │                        Routers                                │  │
│  │  /api/v1/user    /api/v1/video                               │  │
│  │  /api/v1/tweet   /api/v1/comment                             │  │
│  └──────────────────────────────┬──────────────────────────────┘  │
│                                 │                                  │
│  ┌──────────────────────────────▼──────────────────────────────┐  │
│  │                      Controllers                              │  │
│  │  user · video · tweet · comment                              │  │
│  │  wrapped in expressAsyncHandler                              │  │
│  └──────┬───────────────────────────────────────┬──────────────┘  │
│         │                                       │                  │
│  ┌──────▼──────┐                       ┌────────▼──────────────┐  │
│  │   Mongoose   │                       │  Cloudinary SDK        │  │
│  │    Models    │                       │  (media CDN)           │  │
│  └──────┬──────┘                       └────────┬──────────────┘  │
└─────────┼──────────────────────────────────────┼──────────────────┘
          │                                       │
┌─────────▼──────────┐                  ┌────────▼─────────────────┐
│  MongoDB Atlas      │                  │  Cloudinary Cloud         │
│  Collections:       │                  │  Videos · Thumbnails      │
│  users · videos     │                  │  Avatars · Cover Images   │
│  tweets · comments  │                  └──────────────────────────┘
│  likes · playlists  │
│  subscriptions      │
└────────────────────┘
```

---

## 🗄️ Data Models

### User

```
┌──────────────────────────────────────────────────┐
│                     User                          │
├──────────────────────────────────────────────────┤
│  userName      : String  unique · indexed        │
│  email         : String  unique · required       │
│  fullName      : String  required                │
│  avatar        : String  (Cloudinary URL)        │
│  coverImage    : String  (Cloudinary URL)        │
│  password      : String  bcrypt hashed           │
│  refreshToken  : String                          │
│  watchHistory  : [ObjectId → Video]              │
│  createdAt     : Date    (timestamps)            │
│  updatedAt     : Date    (timestamps)            │
├──────────────────────────────────────────────────┤
│  Methods:                                         │
│  isPassCorrect(password) → Boolean               │
│  generateAccessToken()   → JWT                   │
│  generateRefreshToken()  → JWT                   │
│  Hooks: pre("save") → bcrypt hash password       │
└──────────────────────────────────────────────────┘
```

### Video

```
┌──────────────────────────────────────────────────┐
│                     Video                         │
├──────────────────────────────────────────────────┤
│  title         : String  required                │
│  videoFile     : String  (Cloudinary HLS URL)    │
│  thumbnail     : String  (Cloudinary URL)        │
│  description   : String  required                │
│  duration      : Number  (from Cloudinary)       │
│  views         : Number  default 0               │
│  isPublished   : Boolean                         │
│  owner         : ObjectId → User                 │
│  createdAt     : Date                            │
├──────────────────────────────────────────────────┤
│  Plugins: mongooseAggregatePaginate              │
└──────────────────────────────────────────────────┘
```

### Subscription (Channel Model)

```
┌──────────────────────────────────────────────────┐
│                  Subscription                     │
├──────────────────────────────────────────────────┤
│  subscriber    : ObjectId → User  (the fan)      │
│  channel       : ObjectId → User  (the creator)  │
└──────────────────────────────────────────────────┘

  One User document serves dual purpose:
  ├── As "channel"    → has subscribers
  └── As "subscriber" → subscribed to channels
```

### Complete Entity Relationships

```
         ┌─────────┐
         │  User    │◄─────────────────────────────────┐
         └────┬─────┘                                  │
              │ owns                                    │ likedBy
    ┌─────────┼──────────────┐                         │
    │         │              │                  ┌──────┴──────┐
    ▼         ▼              ▼                  │    Like      │
┌───────┐ ┌───────┐ ┌──────────────┐           │  video ─────┼─► Video
│ Video │ │ Tweet │ │  Playlist    │           │  comment ───┼─► Comment
└───┬───┘ └───┬───┘ │  videos[] ──┼──► Video  │  tweet ─────┼─► Tweet
    │         │     └──────────────┘           └─────────────┘
    │         │
    ▼         ▼
┌─────────────────┐         ┌────────────────────┐
│    Comment      │         │   Subscription      │
│  video → Video  │         │  subscriber → User  │
│  owner → User   │         │  channel → User     │
└─────────────────┘         └────────────────────┘
```

---

## 🔄 Request Flow

### Authentication Flow

```
  Client                  Express                  MongoDB
    │                        │                        │
    ├──POST /register ───────►│                        │
    │  { userName, email,     │                        │
    │    password, avatar,    │                        │
    │    coverImage }         │                        │
    │                        │                        │
    │                   ┌────▼────────────────┐       │
    │                   │  multer middleware   │       │
    │                   │  saves to /public/  │       │
    │                   │  temp/              │       │
    │                   └────┬────────────────┘       │
    │                        │                        │
    │                   ┌────▼────────────────┐       │
    │                   │  cloudinaryUpload() │       │
    │                   │  → returns URL      │       │
    │                   └────┬────────────────┘       │
    │                        │                        │
    │                        ├──User.create()─────────►│
    │                        │                        ├─ bcrypt hash
    │                        │                        │  password
    │                        │◄───────────────────────┤
    │◄───201 createdUser ────┤  (no password/token)   │
    │
    ├──POST /login ──────────►│
    │  { email, password }    │
    │                        ├──User.findOne()────────►│
    │                        │◄───────────────────────┤
    │                   ┌────▼──────────────────┐
    │                   │  isPassCorrect(bcrypt) │
    │                   │  generateAccessToken() │
    │                   │  generateRefreshToken()│
    │                   └────┬──────────────────┘
    │◄──200 + cookies ───────┤
    │  accessToken (cookie)  │
    │  refreshToken (cookie) │
    │
    ├──Protected Request──────►│
    │  Cookie: accessToken    │
    │                   ┌────▼──────────────────┐
    │                   │  verifyJwtToken       │
    │                   │  middleware           │
    │                   │  jwt.verify() → user  │
    │                   │  req.user = user      │
    │                   └────┬──────────────────┘
    │                        ├── controller executes
    │◄───────────────────────┤
```

### Video Upload Flow

```
  Client             Multer          Cloudinary         MongoDB
    │                  │                 │                  │
    ├──POST /publish───►│                 │                  │
    │  videoFile        │                 │                  │
    │  thumbnail        │                 │                  │
    │  title, desc      │                 │                  │
    │                  ├─save to──────────                   │
    │                  │  /public/temp/  │                  │
    │                  │                 │                  │
    │                  │    ┌────────────▼──────────────┐   │
    │                  │    │ fileSize check             │   │
    │                  │    │ if > threshold:            │   │
    │                  │    │   upload_large(chunkSize)  │   │
    │                  │    │ else:                      │   │
    │                  │    │   upload(resource:'auto')  │   │
    │                  │    └────────────┬──────────────┘   │
    │                  │                 │                  │
    │                  │                 ├── returns url,   │
    │                  │                 │   duration,      │
    │                  │                 │   playback_url   │
    │                  │    ┌────────────▼──────────────┐   │
    │                  │    │ fs.unlink(localFilePath)   │   │
    │                  │    │ (cleanup temp file)        │   │
    │                  │    └───────────────────────────┘   │
    │                  │                 │                  │
    │                  │                 │   Video.create()─►│
    │                  │                 │                  │
    │◄─────────────────┤─────────────────┤──200 video obj───┤
```

### Channel Profile (Aggregation Pipeline)

```
  GET /channelProfile/:userName
        │
        ▼
  User.aggregate([
    { $match: { userName } }           ← find the channel

    { $lookup: {                       ← join subscriptions
        from: "subscriptions",           where channel = user._id
        as: "subscribers"
    }}

    { $lookup: {                       ← join subscriptions
        from: "subscriptions",           where subscriber = user._id
        as: "subscribedTO"
    }}

    { $addFields: {
        subscribersCount: { $size: "$subscribers" }
        myTotalSubscriptions: { $size: "$subscribedTO" }
        isSubscribed: {                ← check if req.user is in
          $cond: {                       subscribers array
            if: { $in: [req.user._id, "$subscribers.subscriber"] }
            then: true, else: false
          }
        }
    }}

    { $project: {                      ← return only needed fields
        avatar, coverImage, fullName,
        userName, subscribersCount,
        myTotalSubscriptions, isSubscribed, email
    }}
  ])
```

---

## 🚀 Features

### 👤 User Management
- Register with avatar + cover image (Multer → Cloudinary pipeline)
- Login via email or username — flexible credential matching
- JWT access tokens (short-lived) + refresh tokens (long-lived, HTTP-only cookies)
- Refresh token rotation on `/refresh-tokens`
- Change password with old-password verification
- Update profile details, avatar, and cover image independently
- Channel profile with subscriber count via MongoDB aggregation
- `isSubscribed` flag — knows if the requesting user is subscribed

### 🎥 Video Management
- Publish videos with title, description, and thumbnail
- Smart upload: auto-switches to chunked `upload_large` for big files
- Duration automatically extracted from Cloudinary response
- Toggle publish status (draft vs. live)
- Update title, description, and thumbnail
- Fetch video by ID, delete video

### 🐦 Tweets
- Create, edit, and delete tweets
- Fetch all tweets for any user by `userId`
- Owner-linked via Mongoose ObjectId ref

### 💬 Comments *(in progress)*
- Schema: content + video ref + owner ref
- `mongoose-aggregate-paginate-v2` plugin attached
- Controller and routes scaffolded

### ❤️ Likes *(schema ready)*
- Polymorphic — one model handles video likes, comment likes, tweet likes
- `likedBy` ref links back to User

### 📋 Playlists *(schema ready)*
- Name, description, videos array, owner
- CRUD endpoints planned

### 🔔 Subscriptions
- Tracked via a dedicated `Subscription` collection
- Each document: `{ subscriber, channel }` — both refs to User
- Powers the aggregation-based channel profile stats

---

## 🏗️ Tech Stack

```
┌────────────────────────────────────────────────────────────────┐
│  RUNTIME & FRAMEWORK          SECURITY                          │
│  ─────────────────────        ──────────────────               │
│  Node.js 18+                  bcrypt 6     ← password hashing  │
│  Express 5.x                  jsonwebtoken ← JWT access/refresh │
│  ES Modules (type: module)    cookie-parser← HTTP-only cookies  │
│                                                                  │
│  DATABASE                     FILE HANDLING                     │
│  ─────────────                ─────────────────                 │
│  MongoDB Atlas                Multer 2     ← multipart uploads  │
│  Mongoose 8                   Cloudinary 2 ← media CDN          │
│  mongoose-aggregate-          fs/promises  ← temp file cleanup  │
│    paginate-v2                                                   │
│                                                                  │
│  DEV TOOLING                                                    │
│  ─────────────────                                              │
│  nodemon    ← hot reload      prettier  ← code formatting       │
│  dotenv     ← env config                                        │
└────────────────────────────────────────────────────────────────┘
```

---

## 📂 Project Structure

```
veekast/
│
├── public/
│   └── temp/                        # Multer temp uploads (gitignored)
│       └── .gitkeep
│
├── src/
│   ├── index.js                     # Entry point — dotenv, DB, server
│   ├── app.js                       # Express app, middleware, route mounting
│   ├── constant.js                  # DB name, request limits, upload thresholds
│   │
│   ├── 📁 db/
│   │   └── index.js                 # Mongoose connection (asyncTCWrapper)
│   │
│   ├── 📁 models/
│   │   ├── user.model.js            # Schema + bcrypt hooks + JWT methods
│   │   ├── video.model.js           # Schema + aggregate paginate plugin
│   │   ├── tweet.model.js           # Tweet schema
│   │   ├── comment.model.js         # Comment schema + paginate plugin
│   │   ├── like.model.js            # Polymorphic like schema
│   │   ├── playlist.model.js        # Playlist schema
│   │   └── subscription.model.js    # subscriber ↔ channel mapping
│   │
│   ├── 📁 controllers/
│   │   ├── user.controller.js       # register, login, logout, profile, etc.
│   │   ├── video.controller.js      # publish, get, update, delete, toggle
│   │   ├── tweet.controller.js      # create, edit, delete, getAllTweets
│   │   └── comment.controller.js    # (scaffolded — in progress)
│   │
│   ├── 📁 routes/
│   │   ├── user.routes.js           # /api/v1/user/*
│   │   ├── video.route.js           # /api/v1/video/*
│   │   ├── tweet.routes.js          # /api/v1/tweet/*
│   │   └── comment.routes.js        # /api/v1/comment/* (scaffolded)
│   │
│   ├── 📁 middlewares/
│   │   ├── auth.middleware.js       # verifyJwtToken — protects routes
│   │   └── multer.middleware.js     # diskStorage → /public/temp/
│   │
│   └── 📁 utils/
│       ├── ApiError.js              # Custom error class (statusCode, message)
│       ├── ApiResponse.js           # Uniform response wrapper
│       ├── cloudinaryClient.js      # Cloudinary config + export
│       ├── cloudinaryUpload.js      # Smart upload (normal vs. chunked)
│       ├── expressAsyncHandler.js   # HOF — wraps controllers in try/catch
│       └── tryCatchWrapper.js       # HOF — generic async try/catch wrapper
│
├── .env                             # Environment variables (not committed)
├── .prettierrc                      # Prettier config
├── .prettierignore
├── .gitignore
└── package.json
```

---

## ⚙️ Getting Started

### Prerequisites

- Node.js `>= 18.x`
- MongoDB Atlas account (free tier)
- Cloudinary account (free tier)

### 1. Clone the Repository

```bash
git clone https://github.com/your-username/veekast.git
cd veekast
```

### 2. Install Dependencies

```bash
npm install
```

### 3. Configure Environment Variables

Create a `.env` file in the root:

```env
# Server
PORT=8000
CORS_ORIGIN=http://localhost:3000

# MongoDB
MONGODB_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net

# JWT
ACCESS_TOKEN_SECRET_KEY=your_access_token_secret_here
ACCESS_TOKEN_EXPIRY=1d
REFRESH_TOKEN_SECRET_KEY=your_refresh_token_secret_here
REFRESH_TOKEN_EXPIRY=10d

# Cloudinary
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

# Upload thresholds (optional)
CLOUDINARY_LARGE_UPLOAD_THRESHOLD_MB=100
CLOUDINARY_CHUNK_SIZE_MB=6
```

### 4. Run the Development Server

```bash
npm run dev
```

Server starts at `http://localhost:8000`

---

## 🔐 Environment Variables

| Variable | Description | Required |
|---|---|---|
| `PORT` | Server port | ✅ |
| `CORS_ORIGIN` | Allowed frontend origin | ✅ |
| `MONGODB_URI` | MongoDB Atlas connection string | ✅ |
| `ACCESS_TOKEN_SECRET_KEY` | JWT signing secret (access) | ✅ |
| `ACCESS_TOKEN_EXPIRY` | Access token TTL (e.g. `1d`) | ✅ |
| `REFRESH_TOKEN_SECRET_KEY` | JWT signing secret (refresh) | ✅ |
| `REFRESH_TOKEN_EXPIRY` | Refresh token TTL (e.g. `10d`) | ✅ |
| `CLOUDINARY_CLOUD_NAME` | Cloudinary cloud name | ✅ |
| `CLOUDINARY_API_KEY` | Cloudinary API key | ✅ |
| `CLOUDINARY_API_SECRET` | Cloudinary API secret | ✅ |
| `CLOUDINARY_LARGE_UPLOAD_THRESHOLD_MB` | Threshold to switch to chunked upload | ☑️ |
| `CLOUDINARY_CHUNK_SIZE_MB` | Chunk size for large uploads | ☑️ |

---

## 📡 API Reference

> Base URL: `http://localhost:8000/api/v1`

### 👤 User Routes — `/user`

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/register` | Register with avatar + coverImage | — |
| `POST` | `/login` | Login, receive tokens in cookies | — |
| `POST` | `/logout` | Clear tokens, invalidate refresh | ✅ |
| `POST` | `/refresh-tokens` | Rotate access + refresh tokens | — |
| `POST` | `/password-change` | Change password (old + new) | ✅ |
| `GET` | `/get-user` | Get currently logged-in user | ✅ |
| `POST` | `/update-user` | Update userName and email | ✅ |
| `POST` | `/update-avatar` | Replace avatar image | ✅ |
| `POST` | `/update-coverImage` | Replace cover image | ✅ |
| `GET` | `/channelProfile/:userName` | Channel stats + isSubscribed | ✅ |

### 🎥 Video Routes — `/video`

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/publishVideo` | Upload video + thumbnail to Cloudinary | ✅ |
| `GET` | `/:videoId` | Fetch video by ID | — |
| `DELETE` | `/:videoId` | Delete a video | ✅ |
| `POST` | `/updateVideo/:videoId` | Update title, description, thumbnail | ✅ |
| `PATCH` | `/toggle-publish/:videoId` | Toggle isPublished status | ✅ |

### 🐦 Tweet Routes — `/tweet`

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/create-tweet` | Create a new tweet | ✅ |
| `POST` | `/edit-tweet` | Edit tweet content | ✅ |
| `GET` | `/allTweets/:userId` | Get all tweets by user | — |
| `POST` | `/delete-tweet/:tweetId` | Delete a tweet | ✅ |

---

## 🛠️ Utility Layer

VeeKast uses a clean, reusable utility layer across the entire codebase:

```
ApiError(statusCode, message)
  └── Extends Error — carries statusCode, message, errors[]
      Used to throw structured errors in controllers

ApiResponse(statusCode, data, message)
  └── Uniform JSON wrapper for all success responses
      { statusCode, data, message, success }

expressAsyncHandler(fn)
  └── HOF for controller functions
      Wraps async (req, res, next) in try/catch
      Passes errors to Express error middleware

asyncTCWrapper(fn)
  └── HOF for non-Express async functions
      Used for DB connection, token generation, etc.

cloudinaryUpload(localFilePath)
  └── Checks file size → normal or chunked upload
      Auto-detects resource_type
      Cleans up temp file with fs.unlink() in finally{}
```

---

## 🗺️ Roadmap

```
Phase 1 — Core Backend ✅
├── User auth (register, login, JWT, refresh tokens)
├── Video publish with Cloudinary (smart chunked upload)
├── Channel profiles with MongoDB aggregation
├── Tweet CRUD
└── Subscription schema

Phase 2 — Social Features 🔄
├── Comments CRUD (schema + routes scaffolded)
├── Like/unlike for videos, comments, tweets
├── Playlist create, add video, remove, delete
└── Watch history endpoint

Phase 3 — Discovery & Feed 📋
├── Paginated video feed (aggregate paginate)
├── Search videos by title or description
├── Trending videos (view count sorting)
└── Subscribed channels feed

Phase 4 — Production Hardening 🔮
├── Rate limiting (express-rate-limit)
├── Helmet.js for security headers
├── Redis caching for channel profiles
├── Full error handling middleware
└── API documentation (Swagger / Postman collection)
```

---

## 🤝 Contributing

```bash
git clone https://github.com/your-username/veekast.git
git checkout -b feature/your-feature
git commit -m "feat: your change"
git push origin feature/your-feature
# Open a Pull Request
```

**Commit Convention:** `feat:` · `fix:` · `docs:` · `refactor:` · `chore:`

---

## 📜 License

Licensed under the **ISC License** — see [LICENSE](LICENSE) for details.

---

## 🙌 Acknowledgements

| Tool | Role |
|---|---|
| [Express 5](https://expressjs.com/) | REST API framework |
| [Mongoose 8](https://mongoosejs.com/) | MongoDB ODM + aggregation |
| [Cloudinary](https://cloudinary.com/) | Video and image media hosting |
| [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) | JWT token signing and verification |
| [bcrypt](https://github.com/kelektiv/node.bcrypt.js) | Secure password hashing |
| [Multer](https://github.com/expressjs/multer) | Multipart file upload handling |
| [mongoose-aggregate-paginate-v2](https://github.com/aravindnc/mongoose-aggregate-paginate-v2) | Pagination for aggregation pipelines |

---

<div align="center">

**Built with ❤️ by Soumya Hedaoo**
</div>
