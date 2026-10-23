# 🛡️ TryHackMe Writeups

Writeups documenting my hands-on work in **web application & client-side security** — stored/DOM XSS, CSRF, CORS misconfigurations, cookie manipulation, and account-takeover chains.

Maintained by **Shahadat Fahim** — aspiring penetration tester & bug bounty hunter.

---

## 📂 Writeups

| Room | Topics | Link |
|------|--------|------|
| **WorldWAP** | Stored XSS → cookie theft → same-origin exfiltration → admin takeover | [Read »](./WorldWAP-Writeup.md) |
| **El Bandito** | nginx `..;` ACL bypass · Spring Boot actuator leak · HTTP/2 H2.CL request smuggling | [Read »](./El-Bandito-Writeup.md) |
*(more coming soon)*

---

## 🧰 Common Techniques Covered

- **Cross-Site Scripting (XSS)** — reflected, stored, DOM-based; weaponisation beyond `alert()`
- **Cross-Site Request Forgery (CSRF)** — token bypass, SameSite (Lax+POST), double-submit cookie
- **CORS misconfigurations** — arbitrary origin, bad regex, null origin
- **Session & cookie attacks** — theft, manipulation, HttpOnly/SameSite impact

---

## 🔗 Connect

- 🐙 GitHub: [github.com/shahadatfahim](https://github.com/shahadatfahim)
- 💼 LinkedIn: [linkedin.com/in/shahadatfahim](https://linkedin.com/in/shahadatfahim)
- 🎯 HackerOne: [@shahadatfahim](https://hackerone.com/shahadatfahim)

---

> ⚠️ These writeups are for **educational purposes** only. All testing was performed in authorized lab environments (TryHackMe).
