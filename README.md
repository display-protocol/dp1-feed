# DP-1 Feed Operator API

[![Build Status](https://img.shields.io/github/actions/workflow/status/display-protocol/dp1-feed/test.yaml?branch=main&label=build%20status&logo=github)](https://github.com/display-protocol/dp1-feed/actions/workflows/test.yaml)
[![Linter](https://img.shields.io/github/actions/workflow/status/display-protocol/dp1-feed/lint.yaml?branch=main&label=linter&logo=github)](https://github.com/display-protocol/dp1-feed/actions/workflows/lint.yaml)
[![Code Coverage](https://img.shields.io/codecov/c/github/display-protocol/dp1-feed/main?label=code%20coverage&logo=codecov)](https://codecov.io/gh/display-protocol/dp1-feed)
[![Benchmark](https://img.shields.io/github/actions/workflow/status/display-protocol/dp1-feed/benchmark.yaml?branch=main&label=benchmark%20status&logo=github)](https://github.com/display-protocol/dp1-feed/actions/workflows/benchmark.yaml)

A modern API server implementing the DP-1 Feed Operator specification for blockchain-native digital art playlists. Supports both **Cloudflare Workers** (serverless) and **Node.js** (self-hosted) deployments.

## 🚀 Features

- **DP-1 Compliant**: Full OpenAPI 3.1.0 implementation of DP-1 v1.0.0
- **Dual Deployment**: Cloudflare Workers (serverless) + Node.js (self-hosted)
- **Type Safety**: End-to-end TypeScript with Zod validation
- **Modern Stack**: Hono framework, Ed25519 signatures, async processing
- **Production Ready**: KV storage, queues, authentication, CORS, monitoring

## 📦 Quick Start

### Prerequisites

- Node.js 22+ (download from [nodejs.org](https://nodejs.org/) if you don't have it)
- npm (comes with Node.js) or yarn
- For Cloudflare Workers: Cloudflare account + Wrangler CLI (`npm install -g wrangler`)
- For Node.js: etcd (for key-value storage) + NATS JetStream (for messaging/queues)

If you're new to this, start with the Node.js local setup below—it's the easiest way to run things on your machine without any cloud accounts.

### Installation

```bash
git clone https://github.com/display-protocol/dp1-feed.git
cd dp1-feed
npm install
```

### Local Setup for Node.js

This gets you running the server locally on your machine (e.g., Mac, Linux, or Windows) for testing or development. No cloud needed—great for OSS tinkering.

1. **Install etcd and NATS JetStream** (these handle storage and messaging):
   - On Mac (using Homebrew—install it first from [brew.sh](https://brew.sh) if needed):
     ```
     brew install etcd
     brew install nats-server
     ```
   - On Linux (e.g., Ubuntu):
     ```
     sudo apt update
     sudo apt install etcd nats-server
     ```
   - On Windows: Download binaries from [etcd.io](https://etcd.io/docs/v3.5/install/) and [nats.io](https://docs.nats.io/nats-server/installation/windows), then add to your PATH.
   
   Run them in separate terminal windows (they need to stay open):
   ```
   etcd  # Starts on localhost:2379
   ```
   ```
   nats-server -js  # Starts on localhost:4222 with JetStream enabled
   ```

2. **Set Required Environment Variables**:
   The server needs an `API_SECRET` for secure write operations (like creating playlists). This is like a password—use a strong one for real use, but anything works locally.
   ```
   export API_SECRET=your-secret-here  # e.g., a random string like 'supersecret123'
   ```
   (On Windows, use `set API_SECRET=your-secret-here` instead.)
   
   Tip: For convenience, create a `.env` file in the project root with `API_SECRET=your-secret-here`. The server will load it automatically.

   In addition to `API_SECRET`, the server needs an `ED25519_PRIVATE_KEY` for signing playlists (a security feature in the DP-1 spec to verify authenticity). This is a 64-character hex string representing a private key seed. Generate one locally—never share it or commit it to code.

   1. **Generate the Key (Mac/Linux)**:
      - Install OpenSSL if needed: `brew install openssl` (Mac) or `sudo apt install openssl` (Ubuntu).
      - Run:
        ```
        openssl genpkey -algorithm ed25519 -out ed_private.pem
        ```
      - Extract the hex seed:
        ```
        openssl pkey -in ed_private.pem -outform DER | tail -c 32 | xxd -p -c 32
        ```
        (If `xxd` is missing, install with `brew install vim` or use: `openssl pkey -in ed_private.pem -outform DER | tail -c 32 | od -An -tx1 | tr -d ' \n'`.)
      - Output: A string like `2f8c6129d816cf51c374bc7f08c3e63ed156cf78aefb4a6550d97b87997977ee`.

   2. **Set the Env Var**:
      ```
      export ED25519_PRIVATE_KEY=your64charhexstringhere  # Paste from above
      ```
      (On Windows: `set ED25519_PRIVATE_KEY=your64charhexstringhere`.)
      
      Add to your `.env` file for persistence: `ED25519_PRIVATE_KEY=your64charhexstringhere`.

      Note: For production, generate securely and store in a secret manager (e.g., Vault). You may also need a matching `ED25519_PUBLIC_KEY` if the code checks for it—generate with `openssl pkey -in ed_private.pem -pubout -outform DER | tail -c 32 | xxd -p -c 32` and set similarly.

3. **Build and Run**:
   ```
   npm run node:build
   npm run node:start:dev
   ```
   It should start on http://localhost:8787. Open that in your browser or test with curl (see examples below).

If you hit issues, check the Troubleshooting section at the end.

### Quick Deploy to Cloudflare Workers

For serverless hosting (scales automatically, no servers to manage):

```bash
npm run worker:setup:kv
npm run worker:setup:secrets
npm run worker:deploy
```

## 🛠️ Development

```bash
# Cloudflare Workers development (simulates locally with Miniflare)
npm run worker:dev

# Node.js development (watches for changes and restarts)
npm run node:dev

# Run tests
npm test

# Code quality
npm run lint && npm run format
```

📖 **For detailed development guide, see [DEVELOPMENT.md](DEVELOPMENT.md)**

## 📡 API Reference

### Base URL

- **Cloudflare Workers**: `https://your-worker.your-subdomain.workers.dev`
- **Node.js (local)**: `http://localhost:8787` (default; configurable via env vars)

### Authentication

All write operations (POST/PUT) require Bearer token authentication using your `API_SECRET`:

```bash
Authorization: Bearer YOUR_API_SECRET
```

Read operations (GET) are open—no auth needed.

### Core Endpoints

| Method | Endpoint                      | Description     | Auth Required |
| ------ | ----------------------------- | --------------- | ------------- |
| `GET`  | `/api/v1`                     | API information | No            |
| `GET`  | `/api/v1/health`              | Health check    | No            |
| `GET`  | `/api/v1/playlists`           | List playlists  | No            |
| `GET`  | `/api/v1/playlists/{id}`      | Get playlist    | No            |
| `POST` | `/api/v1/playlists`           | Create playlist | Yes           |
| `PUT`  | `/api/v1/playlists/{id}`      | Update playlist | Yes           |
| `GET`  | `/api/v1/playlist-items`      | List items      | No            |
| `GET`  | `/api/v1/playlist-items/{id}` | Get item        | No            |

### Example Requests

#### Create Playlist (Requires Auth)

```bash
curl -X POST http://localhost:8787/api/v1/playlists \
  -H "Authorization: Bearer your-secret-here" \
  -H "Content-Type: application/json" \
  -d '{
    "dpVersion": "1.0.0",
    "title": "my-first-playlist",
    "items": [
      {
        "source": "https://example.com/art.html",
        "duration": 300,
        "license": "open"
      }
    ]
  }'
```

#### Get Playlists (No Auth)

```bash
curl -X GET "http://localhost:8787/api/v1/playlists?sort=desc&limit=10"
```

#### Health Check (No Auth)

```bash
curl -X GET http://localhost:8787/api/v1/health
```

### Response Format

**Success Response:**

```json
{
  "dpVersion": "1.0.0",
  "id": "playlist-id",
  "items": [...],
  "signature": "ed25519-signature",
  "created": "2024-01-15T14:30:00.000Z"
}
```

**Error Response:**

```json
{
  "error": "validation_error",
  "message": "Invalid playlist data: items.0.duration: Expected number, received string"
}
```

### Error Codes

| Code  | Description                          |
| ----- | ------------------------------------ |
| `400` | Bad Request (validation error)       |
| `401` | Unauthorized (missing/invalid token) |
| `404` | Not Found                            |
| `500` | Internal Server Error                |

## 🧪 Testing

```bash
# Unit tests
npm test

# Integration tests
npm run test:api

# Performance benchmarks
npm run benchmark

# Coverage report
npm run test:coverage
```

📖 **For detailed testing guide, see [TESTING.md](TESTING.md)**

## 📚 Documentation

- **[DEVELOPMENT.md](DEVELOPMENT.md)** - Local development setup and project structure
- **[TESTING.md](TESTING.md)** - Testing strategies and performance benchmarking
- **[k6/README.md](k6/README.md)** - Performance testing with K6

## 🔗 References

- [DP-1 Specification](https://github.com/display-protocol/dp1/blob/main/docs/spec.md)
- [OpenAPI Schema](https://github.com/display-protocol/dp1/blob/main/docs/feed-api.yaml)
- [Hono Framework](https://hono.dev/)
- [Zod Validation](https://zod.dev/)
- [Cloudflare Workers](https://developers.cloudflare.com/workers/)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes with proper TypeScript types
4. Add Zod validation for new fields
5. Update tests and documentation
6. Run validation: `npm run validate`
7. Submit a pull request

### Code Standards

- **TypeScript**: Strict mode with comprehensive typing
- **Zod Schemas**: All API inputs must be validated
- **ESLint + Prettier**: Consistent code formatting
- **DP-1 Compliance**: Maintain DP-1 specification compliance

## 📄 License

Mozilla Public License 2.0

Copyright (c) 2025 Feral File

## 🔧 Troubleshooting

- **"Missing required environment variable: API_SECRET"**: Set it with `export API_SECRET=your-secret` (or in `.env`) and restart the server.
- **"Missing required environment variable: ED25519_PRIVATE_KEY"**: Follow the key generation steps above and set the env var.
- **etcd or NATS not connecting**: Ensure they're running (check with `ps aux | grep etcd`). Defaults to localhost—override with env vars like `ETCD_ENDPOINTS=localhost:2379` if needed.
- **Deprecation warnings during install**: Update packages like Miniflare to v4 in package.json, then `npm install`. (Only matters if using Workers mode.)
- **Port in use**: Change the port with `PORT=3000 npm run node:start:dev`.
- Still stuck? Check server logs, or open an issue with your OS/Node version and error details.

---

**Built with ❤️ by Feral File for the DP-1 ecosystem.**
