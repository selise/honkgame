---
name: blocks-html-self-registration
description: Add self-registration (user-initiated signup) to a SELISE Blocks HTML tool. Use when users should be able to sign themselves up rather than only receiving admin invitations. Covers enabling self-signup in Blocks Cloud, configuring Google reCAPTCHA v2, and adding the signup UI to an existing single-file HTML tool.
argument-hint: "[x-blocks-key] [app-domain]"
disable-model-invocation: true
---

You are helping the user add self-registration to a SELISE Blocks HTML tool. Follow these steps in order. At each step, confirm with the user before proceeding to the next.

## What you will build

- A signup tab on the existing login screen that accepts an email address
- A Google reCAPTCHA v2 challenge to protect the endpoint
- A call to the Blocks Identifier API (`People/Signup`) that emails the user an activation link
- After clicking the link the user activates their account through the existing activation flow

This skill is designed to be used alongside `deploy-blocks-html-tool`. The activation flow (`/activate?code=`) must already be in place before self-registration is useful.

---

## Step 1 — Enable Self-Registration in Blocks Cloud

Tell the user to do this manually in the browser:

1. Open **Blocks Cloud → Access Manager → Settings**
2. Find the **Self Registration** or **Allow Self Signup** toggle and enable it
3. Save

Without this step the `People/Signup` API returns a 403 regardless of the request payload.

---

## Step 2 — Create a Google reCAPTCHA v2 Key

Tell the user to do this manually:

1. Open [https://www.google.com/recaptcha/admin](https://www.google.com/recaptcha/admin) and sign in
2. Click **Create** (or the **+** button)
3. Fill in:
   - **Label**: any descriptive name
   - **reCAPTCHA type**: **Challenge (v2) → "I'm not a robot" Checkbox** — this is the only type supported by the Blocks captcha config; v3 and Enterprise keys will fail with "Invalid key type"
   - **Domains**: add the app domain without the protocol (e.g. `myapp-abc.seliseblocks.com`)
4. Accept the Terms of Service and click **Submit**
5. Copy both the **Site Key** and **Secret Key**

---

## Step 3 — Configure Captcha in Blocks Cloud

Tell the user to do this manually in the browser:

1. Open **Blocks Cloud → Project Settings → Captcha** (or **Authentication → Captcha Config**)
2. Create or edit the captcha configuration for the **Signup** action
3. Set:
   - **Provider**: Google reCAPTCHA
   - **Site Key**: the site key from Step 2
   - **Secret Key**: the secret key from Step 2
4. Save and ensure the config is **enabled**

The backend uses the secret key to verify each token with Google. A mismatch between the site key used in the frontend and the secret key saved here causes a `400 {"CaptchaCode": "Captcha doesn't match"}` error.

---

## Step 4 — Add Signup UI to index.html

Ask the user for:
1. Their **x-blocks-key** (from Blocks Cloud → Project Settings), if not already known
2. The **reCAPTCHA site key** from Step 2

Then make the following changes to `index.html`.

### 4a — Add the reCAPTCHA script

Add before the closing `</body>` tag, above any existing `<script>` blocks:

```html
<script src="https://www.google.com/recaptcha/api.js" async defer></script>
```

### 4b — Add the signup tab and form

Add a tab toggle next to the existing login tab and a signup form alongside the login form. The signup form needs only an email field and the reCAPTCHA widget:

```html
<!-- Tab buttons — add alongside existing login tab -->
<button onclick="showLogin()">Login</button>
<button onclick="showRegister()">Sign Up</button>

<!-- Signup form — hidden by default -->
<form id="register-form" style="display:none" onsubmit="handleRegister(event)">
  <p>Enter your email and we'll send you an invitation link to activate your account.</p>
  <input type="email" id="register-email" placeholder="your@email.com" required>
  <div class="g-recaptcha" data-sitekey="{site-key}"></div>
  <div id="register-error" style="color:red"></div>
  <div id="register-success" style="color:green"></div>
  <button type="submit" id="register-submit">Send Invitation</button>
</form>
```

### 4c — Add the Identifier API constant

Add alongside the existing `BLOCKS_API` constant:

```javascript
const IDENTIFIER_API = 'https://api.seliseblocks.com/identifier/v1';
```

Never use the app domain as the API base URL.

### 4d — Add tab-switching helpers

```javascript
function showLogin() {
  document.getElementById('login-form').style.display = 'block';
  document.getElementById('register-form').style.display = 'none';
}

function showRegister() {
  document.getElementById('login-form').style.display = 'none';
  document.getElementById('register-form').style.display = 'block';
}
```

### 4e — Add the signup handler

```javascript
async function handleRegister(event) {
  event.preventDefault();
  const email = document.getElementById('register-email').value.trim().toLowerCase();
  const errorEl = document.getElementById('register-error');
  const successEl = document.getElementById('register-success');
  const submitBtn = document.getElementById('register-submit');

  errorEl.textContent = '';
  successEl.textContent = '';

  const captchaToken = grecaptcha.getResponse();
  if (!captchaToken) {
    errorEl.textContent = 'Please complete the captcha.';
    return;
  }

  submitBtn.disabled = true;
  submitBtn.textContent = 'Sending...';

  try {
    const res = await fetch(`${IDENTIFIER_API}/People/Signup`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-blocks-key': BLOCKS_KEY
      },
      credentials: 'include',
      body: JSON.stringify({
        email: email,
        captchaCode: captchaToken,
        projectKey: BLOCKS_KEY
      })
    });

    if (res.ok) {
      successEl.textContent = 'Invitation sent! Check your email to activate your account.';
      document.getElementById('register-form').reset();
      grecaptcha.reset();
    } else {
      const text = await res.text().catch(() => '');
      let message = 'Signup failed. Please try again.';
      try { const j = JSON.parse(text); message = j.message || j.title || message; } catch {}
      errorEl.textContent = message;
      grecaptcha.reset();
    }
  } catch {
    errorEl.textContent = 'Connection error. Please try again.';
    grecaptcha.reset();
  } finally {
    submitBtn.disabled = false;
    submitBtn.textContent = 'Send Invitation';
  }
}
```

**Critical constants in the request body:**
- `captchaCode` — the token from `grecaptcha.getResponse()`; must be the v2 checkbox response, not a v3 score token
- `projectKey` — must equal `BLOCKS_KEY`; omitting this causes a `500 NullReferenceException` on the server

---

## Step 5 — Test the Flow

Walk the user through a full end-to-end test:

1. Open the app and click **Sign Up**
2. Enter a valid email and solve the reCAPTCHA
3. Click **Send Invitation** — expect the success message
4. Check the email inbox for an activation link
5. Click the link → lands on `https://{app-domain}/activate?code=...`
6. Set a password → account is active and the user can log in

---

## Common Mistakes — Check for These

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Self-registration not enabled in Blocks Cloud | `403` on `People/Signup` | Enable in Access Manager → Settings |
| reCAPTCHA key type is v3 or Enterprise | "ERROR for site owner: Invalid key type" in widget | Recreate key as v2 Checkbox |
| App domain not listed in reCAPTCHA console | `400 {"CaptchaCode": "Captcha doesn't match"}` | Add domain in Google reCAPTCHA admin |
| Site key / secret key mismatch | `400 {"CaptchaCode": "Captcha doesn't match"}` | Ensure both keys are from the same reCAPTCHA entry |
| Captcha config type in Blocks set to wrong provider | `500 NullReferenceException` | Set provider to Google reCAPTCHA in Blocks captcha config |
| `projectKey` missing from request body | `500 NullReferenceException` | Always include `projectKey: BLOCKS_KEY` |
| `credentials: 'include'` missing | CORS / cookie errors | Required for all Blocks IDP and Identifier calls |
| Activation flow not yet implemented | User receives email but cannot activate | Implement activation screen first using `deploy-blocks-html-tool` |
