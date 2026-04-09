# ◇ Introduction to Passport.js

An interactive Reveal.js presentation covering Passport.js — from strategies and sessions through to OAuth, JWT, multi-provider auth, security, and Express integration.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Introduction_to_Passport_js/)

## 📄 [Markdown Version](https://github.com/BrendanJamesLynskey/Introduction_to_Passport_js/blob/main/presentation.md)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | Introduction to Passport.js — Express Authentication Middleware |
| 02 | Agenda | Overview of all topics covered |
| 03 | What Is Passport.js? | Philosophy, strategy pattern, 500+ strategies, install & setup |
| 04 | Core Concepts | Strategies, serializeUser, deserializeUser, verify callback |
| 05 | Session Setup | express-session, cookie config, session stores, secret management |
| 06 | Local Strategy | passport-local, bcrypt hashing, verify callback, custom fields |
| 07 | Registration & Login Flow | Signup route, login route, flash messages, logout |
| 08 | OAuth 2.0 Concepts | Authorization code flow, tokens, scopes, redirect URIs |
| 09 | Google OAuth Strategy | passport-google-oauth20, credentials, callback, profile object |
| 10 | GitHub OAuth Strategy | passport-github2, setup, scopes, profile mapping |
| 11 | JWT Strategy | passport-jwt, token extraction, stateless auth, issuing tokens |
| 12 | Protecting Routes | ensureAuthenticated, role-based access, return-to pattern |
| 13 | User Model & Database | Mongoose/Sequelize schemas, findOrCreate, flexible fields |
| 14 | Multiple Strategies | Linking OAuth to existing accounts, account merging, unlinking |
| 15 | Security Best Practices | Session fixation, CSRF, secure cookies, rate limiting login |
| 16 | Error Handling & Flash Messages | failureRedirect, failureFlash, custom callbacks |
| 17 | Testing Authentication | supertest with sessions, mocking passport, integration tests |
| 18 | Complete Working Example | Express + Passport + EJS app with local + Google auth |
| 19 | Summary & Next Steps | Core takeaways, best practices, resources, key packages |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

- [Passport.js Official Site](https://www.passportjs.org/) — documentation and API reference
- [Passport.js GitHub Repository](https://github.com/jaredhanson/passport) — source code and issues
- [Express.js Session Guide](https://expressjs.com/en/resources/middleware/session.html) — official session middleware docs
- [passport-local](https://www.passportjs.org/packages/passport-local/) — local username/password strategy
- [passport-google-oauth20](https://www.passportjs.org/packages/passport-google-oauth20/) — Google OAuth 2.0 strategy
- [passport-github2](https://github.com/cfsghost/passport-github) — GitHub OAuth strategy
- [passport-jwt](https://www.passportjs.org/packages/passport-jwt/) — JSON Web Token strategy
- [bcrypt](https://www.npmjs.com/package/bcrypt) — password hashing library
- [MDN Web Docs: OAuth 2.0](https://developer.mozilla.org/en-US/docs/Web/Security) — security background

## License

Educational use. Code examples provided as-is.
