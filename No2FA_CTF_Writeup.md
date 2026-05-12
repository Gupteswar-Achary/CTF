# 🔐 CTF Writeup — No 2FA (Medium)
**Platform:** [PicoCTF](https://learn.cylabacademy.org/library/765?page=2&category=1)  
**Category:** Web Exploitation  
**Difficulty:** Medium  

---

## 📋 Challenge Overview

The challenge provided:
- Python source code of a vulnerable web application
- A leaked database in **SQLite3** format

The goal: retrieve the flag from an admin-protected route.

---

## 🧩 Solution Walkthrough

### Step 1 — Database Recon

Opened the SQLite3 dump and extracted the admin credentials:

```sql
SELECT * FROM users;
-- Result: username = 'admin', password = <SHA-256 hash>
```

Used **hash-identifier** to confirm the hashing algorithm: `SHA-256`.

Since the hash was **unsalted**, ran it through [CrackStation](https://crackstation.net) — cracked instantly.

---

### Step 2 — Login & Source Code Analysis

Authenticated with the recovered credentials. Hit a **2FA / OTP prompt**.

Before panicking, went back to the Python source. Found this:

```python
if session.get('username') == 'admin':
    flag = os.getenv('FLAG')
```

> 💡 The flag gate checks **only the session** — not the password, not the OTP.  
> The real attack surface is the session cookie.

---

### Step 3 — Session Cookie Forensics

Inspected the session cookie in the browser dev tools.

- Encoding: **Base64**
- Compression: **zlib**

Wrote a Python script to decode it:

```python
import base64
import zlib
import json

cookie = "<paste_cookie_here>"

decoded = base64.urlsafe_b64decode(cookie + "==")
decompressed = zlib.decompress(decoded)
session_data = json.loads(decompressed)

print(session_data)
```

**Output revealed the OTP stored inside the client-side cookie.**  
Used it to pass 2FA → flag captured. ✅

---

## 🚩 Flag

```
picoCTF{...}
```

---

## 🛡️ Vulnerabilities Identified

| Vulnerability | Description |
|---|---|
| Unsalted password hash | SHA-256 without salt is trivially crackable via rainbow tables |
| Client-side session trust | OTP stored in a client-accessible, decodable session cookie |
| Weak 2FA implementation | OTP embedded in session rather than validated server-side |

---

## 💡 Key Takeaway

> The 2FA was theater.

The actual vulnerability was trusting the client with sensitive session data — including the OTP itself. Security isn't about how many layers you add. It's about whether those layers actually hold.

---

## 🛠️ Tools Used

- `hash-identifier` — Hash algorithm detection
- [CrackStation](https://crackstation.net) — Hash cracking
- Browser DevTools — Cookie inspection
- Python (`base64`, `zlib`, `json`) — Cookie decoding

---

## 📚 Skills Practiced

- SQLite3 forensics
- Hash identification & cracking
- Base64 / zlib decoding
- Python scripting
- Web source code auditing
- Web exploitation methodology

---

*Part of my ongoing CTF practice log. Follow along for more writeups.*
