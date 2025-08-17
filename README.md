# AzureAd-Msal-Login-Authorization

Perfect ✅ thanks for sharing your **folder structure** and the **Azure AD credentials** from `.env`.

We’ll implement **Azure AD login using MSAL (Microsoft Authentication Library)** in your **Node.js + Express + TypeScript + Prisma** project.
From your structure, the flow should fit into:

* `src/controllers/auth.controller.ts` → handles routes `/auth/login` and `/auth/callback`
* `src/services/msalClient.ts` → MSAL config & client
* `src/routes/auth.routes.ts` → define login & callback endpoints
* `src/store/authStore.ts` (optional) → keep tokens in memory or DB
* `.env` → already has `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `REDIRECT_URI`

---

### Step 1: Install required packages

Run:

```bash
npm install @azure/msal-node
```

---

### Step 2: MSAL Client (src/services/msalClient.ts)

```ts
import { ConfidentialClientApplication, Configuration } from "@azure/msal-node";
import dotenv from "dotenv";

dotenv.config();

const msalConfig: Configuration = {
  auth: {
    clientId: process.env.AZURE_CLIENT_ID!,
    authority: `https://login.microsoftonline.com/${process.env.AZURE_TENANT_ID}`,
    clientSecret: process.env.AZURE_CLIENT_SECRET!,
  },
};

const cca = new ConfidentialClientApplication(msalConfig);

export default cca;
```

---

### Step 3: Auth Controller (src/controllers/auth.controller.ts)

```ts
import { Request, Response } from "express";
import cca from "../services/msalClient";
import { AuthorizationUrlRequest, AuthorizationCodeRequest } from "@azure/msal-node";

const SCOPES = ["user.read"]; // Microsoft Graph permission

// Redirect user to Microsoft login
export const loginHandler = async (req: Request, res: Response) => {
  try {
    const authCodeUrlParams: AuthorizationUrlRequest = {
      scopes: SCOPES,
      redirectUri: process.env.REDIRECT_URI!,
    };

    const authCodeUrl = await cca.getAuthCodeUrl(authCodeUrlParams);
    res.redirect(authCodeUrl);
  } catch (error) {
    console.error("Error generating auth URL:", error);
    res.status(500).send("Error during login");
  }
};

// Handle callback and get token
export const callbackHandler = async (req: Request, res: Response) => {
  const code = req.query.code as string;

  if (!code) {
    return res.status(400).send("Authorization code not found");
  }

  try {
    const tokenRequest: AuthorizationCodeRequest = {
      code,
      scopes: SCOPES,
      redirectUri: process.env.REDIRECT_URI!,
    };

    const tokenResponse = await cca.acquireTokenByCode(tokenRequest);

    if (!tokenResponse) {
      return res.status(401).send("Failed to acquire token");
    }

    // Save user info / tokens to DB if needed
    res.json({
      message: "Login successful",
      accessToken: tokenResponse.accessToken,
      account: tokenResponse.account,
    });
  } catch (error) {
    console.error("Error handling callback:", error);
    res.status(500).send("Error during callback");
  }
};
```

---

### Step 4: Routes (src/routes/auth.routes.ts)

```ts
import { Router } from "express";
import { loginHandler, callbackHandler } from "../controllers/auth.controller";

const router = Router();

router.get("/login", loginHandler);
router.get("/callback", callbackHandler);

export default router;
```

---

### Step 5: Register Routes in `app.ts`

```ts
import express from "express";
import authRoutes from "./routes/auth.routes";

const app = express();

app.use("/auth", authRoutes);

export default app;
```

---

### Step 6: Run & Test

Start your server:

```bash
npm run dev
```

In browser or Postman:

1. `http://localhost:3000/auth/login` → Redirects to Microsoft login
2. After login → Redirects to `http://localhost:3000/auth/callback?code=...`
3. Returns token + account info

---

👉 Do you want me to also **store the Azure AD user in your PostgreSQL (via Prisma)** after login, or just authenticate & return tokens for now?




# So How the Jwt Work with Azure ad and when the jwt role come in picture

Perfect 🚀 I’ll be here to help you step by step when you’re ready.

When you want to start coding, just tell me what part you want to tackle first (login flow, callback handling, token validation middleware, or role-based authorization).

Shall I keep a **ready-to-go plan** for you so we can code smoothly when you come back?
