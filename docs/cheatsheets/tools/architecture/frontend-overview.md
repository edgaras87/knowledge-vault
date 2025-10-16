---
title: frontend-system-overview
tags: [architecture, frontend, react, vite, nodejs, npm, docker, nginx, api, state-management, testing, ci-cd]
summary: Comprehensive overview of modern frontend system architecture using Node.js, npm, React, Vite, Docker, and Nginx. Covers development flow, folder structure, state management, API communication, testing, and deployment strategies.
---


# 💡 Frontend System Architecture Overview  

---

### (Node.js → npm → React/Vite → API → Build & Deployment)

The frontend layer is the bridge between users and your backend.  
It manages UI, application state, and communication with your APIs — built, bundled, and served through Node.js tooling.

This document shows how the frontend development pipeline fits into your full-stack environment.

---

## 🧱 1. The Big Picture

```

User → Browser → Frontend App (React/Vite)
│
▼
REST / GraphQL API (Nginx → Backend)
│
PostgreSQL & Redis under the hood

````

**Frontend role:**  
Present data, handle input, manage state, call backend endpoints, and render updates instantly.

---

## ⚙️ 2. Core Components of Modern Frontend

| Component | Purpose | Tool |
|------------|----------|------|
| **Runtime** | JavaScript engine for tooling & builds | Node.js |
| **Package Manager** | Installs dependencies | npm / pnpm / yarn |
| **Framework** | UI logic and components | React, Vue, Svelte |
| **Bundler/Dev Server** | Hot reload, build output | Vite / Webpack |
| **State Management** | Handle UI data flow | Redux, Zustand, Context API |
| **API Layer** | Communicate with backend | Fetch / Axios |
| **Build Output** | Static assets for Nginx | `dist/` folder |

---

## 🧩 3. Local Development Flow

```bash
# Install dependencies
npm install

# Start dev server
npm run dev
````

Default:

* Vite dev server on `http://localhost:5173`
* Auto-reloads when files change
* Proxy API calls to backend (e.g., `http://localhost:8080`)

**Example Vite proxy config:**

```js
// vite.config.js
export default {
  server: {
    proxy: {
      '/api': 'http://localhost:8080'
    }
  }
};
```

---

## 🧠 4. Folder Structure

```
frontend/
 ├─ src/
 │   ├─ components/
 │   ├─ pages/
 │   ├─ hooks/
 │   ├─ context/
 │   ├─ services/        # API calls, Axios configs
 │   ├─ assets/          # images, styles
 │   └─ main.jsx
 ├─ public/
 │   └─ index.html
 ├─ package.json
 ├─ vite.config.js
 └─ .env
```

---

## ⚡ 5. Environment Variables

Stored in `.env` (never committed).

```
VITE_API_URL=http://localhost:8080/api
VITE_APP_ENV=development
```

Access in React:

```js
const apiUrl = import.meta.env.VITE_API_URL;
```

✅ Prefix with `VITE_` for access in client-side code.

---

## 🧰 6. Communication with Backend

### REST Example (Axios)

```js
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});

export const fetchUsers = async () => {
  const res = await api.get('/users');
  return res.data;
};
```

### GraphQL Example

```js
import { request, gql } from 'graphql-request';
const API = import.meta.env.VITE_API_URL + '/graphql';

const query = gql`
  query Users { users { id name } }
`;

export async function getUsers() {
  return await request(API, query);
}
```

---

## 🧱 7. State and Caching Strategy

| Layer             | Example       | Purpose                     |
| ----------------- | ------------- | --------------------------- |
| React Context     | `AuthContext` | Manage global state         |
| LocalStorage      | `token`       | Persist login               |
| SWR / React Query | `useQuery()`  | Cache API calls             |
| Redux / Zustand   | store slice   | Predictable state container |

Rule of thumb:

> Global state for global data, local state for local UI.

---

## 🧩 8. Building for Production

```bash
npm run build
```

Outputs optimized static assets to:

```
frontend/dist/
```

Files include:

* `index.html`
* `assets/*.js`, `*.css`

You serve these via **Nginx**:

```nginx
server {
    listen 80;
    server_name example.com;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://backend:8080;
    }
}
```

---

## 🚀 9. Dockerizing the Frontend

**Dockerfile:**

```dockerfile
# Build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Serve stage
FROM nginx:latest
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

**docker-compose.yml:**

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
```

---

## 💻 10. Frontend in IDEs (JetBrains / VS Code)

### JetBrains (WebStorm / IntelliJ Ultimate)

* Built-in React + TypeScript tooling.
* Integrated terminal → run `npm run dev`.
* Live linting, code formatting, and Git integration.
* Debugger for browser + Node processes.

**Shortcut tips:**

* Shift+F10 — run dev server
* Ctrl+B — jump to component definition
* Ctrl+Shift+R — run npm script

### VS Code Setup

Recommended extensions:

* **ESLint** — static analysis
* **Prettier** — formatting
* **Vite** — syntax + run configs
* **Tailwind CSS IntelliSense**
* **REST Client** — test API endpoints

---

## 🧠 11. Testing and Quality

**Unit tests**

```bash
npm run test
```

Tools: Vitest / Jest.

**E2E tests**

```bash
npx playwright test
```

Define flows:

* User logs in
* Calls `/api/users`
* Checks UI update

**Lint & format**

```bash
npm run lint
npm run format
```

Automation ensures consistent builds across machines.

---

## ⚙️ 12. Build → Deploy Pipeline

### Local → Staging → Production

| Step         | Tool              | Description             |
| ------------ | ----------------- | ----------------------- |
| Code changes | Git               | Track & commit          |
| Build        | Vite              | Bundle assets           |
| Test         | Jest / Playwright | Validate functionality  |
| Package      | Docker            | Create deployable image |
| Deploy       | Nginx / CI/CD     | Serve to users          |

CI/CD example (GitHub Actions):

```yaml
- name: Build and Push Frontend
  run: |
    docker build -t ghcr.io/user/frontend:${{ github.sha }} .
    docker push ghcr.io/user/frontend:${{ github.sha }}
```

---

## 🧩 13. Communication with Backend Stack

| Interaction           | Description                                      |
| --------------------- | ------------------------------------------------ |
| **API Requests**      | `fetch()` or Axios calls to `/api/`              |
| **Auth Tokens**       | Sent via `Authorization: Bearer <token>` headers |
| **Rate Limiting**     | Controlled by Nginx or Redis                     |
| **CORS**              | Managed by backend or Nginx                      |
| **Real-Time Updates** | WebSockets or Redis Pub/Sub bridges              |

All HTTP requests pass through **Nginx**, which proxies to the backend and ensures security.

---

## 🧰 14. Developer Workflow Summary

1. Pull latest code:
   `git pull`
2. Run dev server:
   `npm run dev`
3. Build assets:
   `npm run build`
4. Serve with Docker or Nginx.
5. Deploy via CI/CD or manually.

---

## 🧭 15. Frontend ↔ Backend Integration Diagram

```
   ┌──────────────┐
   │   Browser     │
   │ (React/Vite)  │
   └──────┬────────┘
          │
          │ REST/GraphQL calls
          ▼
   ┌──────────────┐
   │   Nginx Proxy │
   └──────┬────────┘
          │ forwards requests
          ▼
   ┌──────────────┐
   │ Backend API  │
   └──────┬────────┘
     │           │
     ▼           ▼
 PostgreSQL     Redis
 (storage)     (cache)
```

---

## ✅ 16. Summary

* **Node.js** – runtime & package manager.
* **Vite/React** – fast UI development.
* **Axios/Fetch** – API communication.
* **Docker & Nginx** – consistent build + deployment.
* **Git & CI/CD** – version control and automation.

Together, these tools mirror the backend architecture — the same principles of reproducibility, versioning, and service boundaries apply.

