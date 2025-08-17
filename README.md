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

Exactly 👍

Before login there’s **no JWT** at all — because JWT is just a **proof** that Azure AD has authenticated the user.

Here’s the flow:

---

### 🔹 1. Before Login

* User requests `/auth/login`.
* We redirect them to Azure AD (Microsoft login page).
* At this stage → **no JWT exists yet**.

---

### 🔹 2. After Login

* User enters credentials in Microsoft login page.
* Azure AD checks them → if correct, Azure AD sends back an **authorization code** to your app’s callback (`/auth/callback`).
* Your backend exchanges that code with Azure AD for **tokens** (Access Token, ID Token, Refresh Token).
* The **Access Token** is a **JWT** signed by Microsoft (contains user identity, roles, tenant info, expiry, etc.).

---

### 🔹 3. After Token is Issued

* Now the JWT exists.
* Your frontend or Postman will send this token as `Authorization: Bearer <jwt>` in every request.
* Your backend middleware validates this token.
* Only then, protected routes can be accessed.

---

👉 So to answer clearly:

* **Before login** → no JWT, just redirect flow.
* **After successful login** → Azure issues JWT.
* **From then on** → JWT is used for authentication & authorization in your app.

---

Do you want me to **decode a sample Azure JWT** for you and show exactly what fields (like `roles`, `email`, `upn`) it contains? That will make role-based auth clearer.



# The whole flow

Got it ✅ — Let’s slow down and go step by step. Think of me as your **teacher** now.
We’ll treat this like a **story of login** so it’s easier to understand.

---

## 🌍 The Story of Azure AD Login in Your Backend

When a user logs in with **Microsoft (Azure AD)**, here’s what happens:

1. **Your app asks Microsoft for permission**
   → This is `/auth/login`.
   You redirect the user to Microsoft’s login page.

2. **User signs in on Microsoft’s website**
   → Microsoft checks username + password, MFA, etc.

3. **Microsoft gives your app a "code"**
   → This is a **temporary ticket** (valid for a few minutes).
   The code is sent to `/auth/callback`.

4. **Your app exchanges the code for a real token**
   → This is the **token exchange step**.
   The token = proof that the user is who they claim to be.

5. **The token contains user information (claims)**
   → Example: name, email, roles, tenant, expiration.
   Now your backend can trust the user without asking for password again.

---

## 🛠 Why Each File Exists

### 1. `msalClient.ts`

This file is like your **key to talk to Microsoft**.
It sets up the **MSAL (Microsoft Authentication Library)** client.

```ts
const msalConfig: Configuration = {
  auth: {
    clientId: process.env.AZURE_CLIENT_ID!,
    authority: `https://login.microsoftonline.com/${process.env.AZURE_TENANT_ID}`,
    clientSecret: process.env.AZURE_CLIENT_SECRET!,
  },
};
```

* **clientId** → The ID Microsoft gave your app (like your student ID).
* **tenantId** → Which organization/tenant you belong to.
* **clientSecret** → Password for your app (never expose this!).

👉 This is why we use `.env` — to keep secrets safe.

---

### 2. `auth.controller.ts`

This is your **teacher** who controls the login flow.

* **`loginHandler`** → Makes the Microsoft login URL and redirects user there.
* **`callbackHandler`** → Handles Microsoft’s response (with `code`), asks for a **real token**, and gives it back to you.

💡 Analogy:

* `loginHandler` = sending you to the exam hall.
* Microsoft = exam invigilator.
* `callbackHandler` = collecting your marksheet from invigilator and handing it to you.

---

### 3. `auth.routes.ts`

This is the **map** of where login and callback are handled.

```ts
router.get("/login", loginHandler);     // Go to Microsoft login page
router.get("/callback", callbackHandler); // Handle Microsoft’s response
```

So when you visit:

* `http://localhost:3000/auth/login` → Starts login
* `http://localhost:3000/auth/callback` → Finishes login

---

### 4. `app.ts`

This is your **main entry point** where all routes are registered.

```ts
app.use("/auth", authRoutes);
```

So anything starting with `/auth` is handled by your **auth.routes.ts**.

---

## 🔑 Why the Code Exchange?

👉 Why don’t we just get a token directly?
Because of **security**.

* The **code** is short-lived and useless without your **clientSecret**.
* Only your backend (which has the clientSecret) can exchange the code for a token.
* This prevents hackers from stealing tokens directly in the browser.

So:

* Browser gets → `code`
* Backend exchanges → `token`

---

## 🛡 What Does the Token Contain?

The **Access Token** is a **JWT (JSON Web Token)** issued by Microsoft.
It has **claims**, for example:

```json
{
  "aud": "your-app-id",        // Who the token is for
  "iss": "https://login.microsoftonline.com/{tenantId}/v2.0",
  "name": "John Doe",
  "preferred_username": "john@example.com",
  "roles": ["Admin", "Manager"],
  "exp": 1692342334            // Expiry time
}
```

👉 This is what we will later use for **role-based authorization** in your app.

---

## 🧪 How Do You Test?

1. Start backend → `npm run dev`
2. Go to → `http://localhost:3000/auth/login`
3. Microsoft login page opens → login with your Azure AD credentials
4. After login → You get redirected to `/auth/callback`
5. Your backend exchanges code → Sends back JSON with `accessToken` + `account`

---

✅ At this point:

* You have a working **login system with Microsoft**.
* You get a **token** which proves who the user is.
* Later, we’ll use this token for **authorization** (e.g., Admin vs User).

---

👉 Now tell me:
Do you want me to **draw a simple diagram flow** (login → code → token) to make it crystal clear?

Perfect 👍 let’s **draw the Azure AD authentication flow** so you can clearly see **why we wrote each piece of code**.

Here’s a **step-by-step diagram (ASCII flow)**:

```
   ┌────────────┐          ┌──────────────┐         ┌─────────────┐
   │   Browser  │          │  Our Backend │         │   Azure AD  │
   │  (User)    │          │  (Express)   │         │ (MS Login)  │
   └─────┬──────┘          └──────┬───────┘         └──────┬──────┘
         │                         │                        │
         │ 1. GET /auth/login      │                        │
         ├────────────────────────>│                        │
         │                         │                        │
         │                         │ 2. Generate Auth URL   │
         │                         │ (using MSAL client)    │
         │                         ├───────────────────────>│
         │                         │                        │
         │                         │   3. Redirect User     │
         │                         │<───────────────────────┤
         │ 4. Redirect to Azure    │                        │
         ├─────────────────────────────────────────────────>│
         │                         │                        │
         │   5. User logs in       │                        │
         │   with Microsoft        │                        │
         │                         │                        │
         │                         │   6. Azure returns     │
         │                         │  Auth Code (short-lived│
         │                         │   one-time secret)     │
         │<─────────────────────────────────────────────────┤
         │  http://localhost:3000  │                        │
         │   /auth/callback?code=… │                        │
         │                         │                        │
         │ 7. GET /auth/callback   │                        │
         ├────────────────────────>│                        │
         │                         │                        │
         │                         │ 8. Exchange Auth Code  │
         │                         │ for Access Token       │
         │                         ├───────────────────────>│
         │                         │                        │
         │                         │   9. Azure returns     │
         │                         │   Access Token + ID    │
         │                         │<───────────────────────┤
         │                         │                        │
         │ 10. Return token + user │                        │
         │   info to browser       │                        │
         │<────────────────────────│                        │
         │                         │                        │
```

---

### 🔎 Now, why we wrote each piece of code:

1. **MSAL Client (msalClient.ts)**

   * We need a **Microsoft library** that knows how to talk to Azure AD securely.
   * It takes our `clientId`, `tenantId`, `clientSecret` → identifies our app.
   * Without this, our backend cannot **prove its identity** to Azure.

2. **Auth Controller (auth.controller.ts)**

   * `loginHandler`: builds a **login URL** → tells Azure:
     “Hey, we want permission to read user data (`user.read`). When done, redirect back to `/auth/callback`.”
   * `callbackHandler`: when Azure redirects back with an **auth code**, we **exchange that code** for a **token**.

     * Auth Code = one-time password (cannot access resources by itself).
     * Token = contains real user identity + roles + permissions.

3. **Routes (auth.routes.ts)**

   * We expose endpoints so our app has:

     * `/auth/login` → kicks off login
     * `/auth/callback` → handles Azure’s redirect

4. **Redirect URI**

   * Must **match** exactly in Azure Portal & our code.
   * This is where Azure AD “sends back” the auth code safely.

5. **Access Token (Bearer)**

   * After exchange, we get a **JWT token** that contains:

     * `sub` → unique user ID
     * `name` → username
     * `roles` / `groups` → Azure AD roles (if configured)
   * This is what we’ll later use for **role-based authorization** in your app.

---

✅ So in short:

* **Before login** → User has nothing.
* **Login flow** → Get `code`.
* **Code exchanged** → Get `token`.
* **Token contains identity + roles** → Now we can authorize.

---

Would you like me to also **decode a sample Azure AD JWT** (with explanation of each field) so you see what’s inside the token you’ll get?
