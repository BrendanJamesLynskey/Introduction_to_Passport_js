# Introduction to Passport.js — Presentation Notes

---

## Slide 01 — Title

**Introduction to Passport.js**

Strategies, Sessions & OAuth — Authentication for Node.js

19 slides · strategies, OAuth, JWT, sessions, security · 2026

---

## Slide 02 — Agenda

### Foundations
- What Is Passport.js?
- Core Concepts — Strategies, Serialize, Deserialize
- Session Setup
- Local Strategy — Username & Password
- Registration & Login Flow

### OAuth & JWT
- OAuth 2.0 Concepts
- Google OAuth Strategy
- GitHub OAuth Strategy
- JWT Strategy

### Patterns & Security
- Protecting Routes
- User Model & Database
- Multiple Strategies
- Security Best Practices

### Production
- Error Handling & Flash Messages
- Testing Authentication
- Complete Working Example
- Summary & Next Steps

---

## Slide 03 — What Is Passport.js?

### Philosophy: "Simple, Unobtrusive Authentication"

Passport.js is authentication middleware for Node.js created by Jared Hanson in 2011. It uses a strategy pattern to support 500+ authentication mechanisms — from local username/password to OAuth providers like Google, GitHub, Facebook, and more.

### Key Characteristics

- **Strategy pattern** — each auth method is a pluggable strategy
- **500+ strategies** — local, OAuth, SAML, LDAP, JWT, and more
- **Express-native** — integrates seamlessly as middleware
- **~2.5M weekly npm downloads** — the most popular Node auth library
- **Session support** — built-in serialization/deserialization

### Install & Basic Setup

```bash
npm install passport express-session
```

```javascript
const express = require('express');
const passport = require('passport');
const session = require('express-session');

const app = express();

app.use(session({
  secret: 'keyboard cat',
  resave: false,
  saveUninitialized: false
}));

app.use(passport.initialize());
app.use(passport.session());
```

### How It Works

Passport delegates authentication to strategies. You configure one or more strategies, then call `passport.authenticate('strategy')` as route middleware.

---

## Slide 04 — Core Concepts — Strategies, Serialize, Deserialize

### The Strategy Pattern

```javascript
const LocalStrategy = require('passport-local').Strategy;

passport.use(new LocalStrategy(
  (username, password, done) => {
    User.findOne({ username }, (err, user) => {
      if (err) return done(err);
      if (!user) return done(null, false, { message: 'Unknown user' });
      if (!user.validPassword(password))
        return done(null, false, { message: 'Wrong password' });
      return done(null, user);
    });
  }
));
```

Each strategy provides a **verify callback** that receives credentials and calls `done()` with the result.

### serializeUser

```javascript
// Store user ID in the session
passport.serializeUser((user, done) => {
  done(null, user.id);
});
```

Called after successful authentication. Determines what data is stored in `req.session.passport.user`.

### deserializeUser

```javascript
// Retrieve full user from session ID
passport.deserializeUser((id, done) => {
  User.findById(id, (err, user) => {
    done(err, user);
  });
});
```

Called on every request. Fetches the full user object and attaches it to `req.user`.

### Flow Summary

**Login:** verify callback → serializeUser → session
**Each request:** session → deserializeUser → req.user

---

## Slide 05 — Session Setup

### express-session Configuration

```javascript
const session = require('express-session');
const MongoStore = require('connect-mongo');

app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  store: MongoStore.create({
    mongoUrl: process.env.MONGO_URI
  }),
  cookie: {
    maxAge: 1000 * 60 * 60 * 24, // 1 day
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax'
  }
}));
```

### Cookie Options

| Option | Purpose |
|--------|---------|
| `httpOnly` | Prevents JavaScript access to cookie |
| `secure` | Cookie sent only over HTTPS |
| `sameSite` | CSRF protection (`lax` or `strict`) |
| `maxAge` | Cookie expiration in milliseconds |

### Session Stores

| Store | Package | Use Case |
|-------|---------|----------|
| Memory | Built-in | Dev only (leaks!) |
| MongoDB | `connect-mongo` | Mongo-based apps |
| Redis | `connect-redis` | High performance |
| PostgreSQL | `connect-pg-simple` | SQL-based apps |

### Secret Management

```bash
# .env
SESSION_SECRET=a1b2c3d4e5f6...
MONGO_URI=mongodb://localhost:27017/myapp
```

```javascript
require('dotenv').config();
// Never hard-code secrets!
// Use crypto.randomBytes(64).toString('hex') to generate strong secrets
```

**Never** use the default MemoryStore in production. It leaks memory and does not scale across multiple processes.

---

## Slide 06 — Local Strategy — Username & Password

### Install & Configure

```bash
npm install passport-local bcrypt
```

```javascript
const LocalStrategy = require('passport-local').Strategy;

passport.use(new LocalStrategy(
  { usernameField: 'email' },
  async (email, password, done) => {
    try {
      const user = await User.findOne({ email });
      if (!user) return done(null, false, { message: 'No account found' });

      const match = await bcrypt.compare(password, user.password);
      if (!match) return done(null, false, { message: 'Wrong password' });

      return done(null, user);
    } catch (err) {
      return done(err);
    }
  }
));
```

### The Verify Callback

The verify callback receives credentials and must call `done()` with one of three outcomes:

- `done(null, user)` — **success**
- `done(null, false, { message })` — **auth failure**
- `done(err)` — **server error**

### bcrypt Password Hashing

```javascript
const bcrypt = require('bcrypt');
const SALT_ROUNDS = 12;

// Hash on registration
const hash = await bcrypt.hash(plainPassword, SALT_ROUNDS);

// Compare on login
const isMatch = await bcrypt.compare(plainPassword, hash);
```

### Custom Field Names

```javascript
// Default: username + password
// Override with options:
new LocalStrategy({
  usernameField: 'email',
  passwordField: 'passwd'
}, verifyCallback);
```

---

## Slide 07 — Registration & Login Flow

### Registration Route

```javascript
app.post('/register', async (req, res) => {
  const { name, email, password } = req.body;

  const existing = await User.findOne({ email });
  if (existing) {
    req.flash('error', 'Email in use');
    return res.redirect('/register');
  }

  const hash = await bcrypt.hash(password, 12);
  const user = await User.create({ name, email, password: hash });

  req.login(user, (err) => {
    if (err) return next(err);
    res.redirect('/dashboard');
  });
});
```

### Flash Messages

```bash
npm install connect-flash
```

```javascript
const flash = require('connect-flash');
app.use(flash());

app.use((req, res, next) => {
  res.locals.success = req.flash('success');
  res.locals.error = req.flash('error');
  next();
});
```

### Login Route

```javascript
app.post('/login',
  passport.authenticate('local', {
    successRedirect: '/dashboard',
    failureRedirect: '/login',
    failureFlash: true
  })
);
```

Passport handles the entire flow: parse credentials, call verify, serialize user, create session.

### Logout Route

```javascript
app.post('/logout', (req, res, next) => {
  req.logout((err) => {
    if (err) return next(err);
    req.flash('success', 'Logged out');
    res.redirect('/');
  });
});
```

### Login Form

```html
<form action="/login" method="POST">
  <input name="email" type="email" placeholder="Email" required>
  <input name="password" type="password" placeholder="Password" required>
  <button type="submit">Log In</button>
</form>
<% if (error && error.length) { %>
  <p class="alert"><%= error %></p>
<% } %>
```

---

## Slide 08 — OAuth 2.0 Concepts

### Authorization Code Flow

1. User clicks "Login with Google"
2. App redirects to Google's authorization server
3. User consents and grants permissions
4. Google redirects back with an authorization code
5. App exchanges code for access token (server-to-server)
6. App receives access token and user profile

### Key Terms

- **Client ID** — public identifier for your app
- **Client Secret** — private key (never expose client-side)
- **Redirect URI** — where provider sends the user back
- **Scopes** — permissions your app requests (email, profile, etc.)
- **Access Token** — short-lived credential to call provider APIs
- **Refresh Token** — long-lived token to obtain new access tokens

### Tokens & Scopes

```javascript
// Scopes define what data you request
{ scope: ['profile', 'email'] }

// Access token (short-lived) — used to call provider APIs
// Refresh token (long-lived) — exchange for new access token when current one expires
```

### Redirect URIs

```text
Development:  http://localhost:3000/auth/google/callback
Production:   https://myapp.com/auth/google/callback
```

Must be registered exactly in the provider's developer console. Mismatches cause errors.

**Never** expose your Client Secret in front-end code or public repositories. Always store in environment variables.

---

## Slide 09 — Google OAuth Strategy

### Install & Configure

```bash
npm install passport-google-oauth20
```

```javascript
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback'
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      let user = await User.findOne({ googleId: profile.id });
      if (!user) {
        user = await User.create({
          googleId: profile.id,
          name: profile.displayName,
          email: profile.emails[0].value,
          avatar: profile.photos[0].value
        });
      }
      return done(null, user);
    } catch (err) {
      return done(err);
    }
  }
));
```

### Routes

```javascript
// Redirect to Google
app.get('/auth/google',
  passport.authenticate('google', { scope: ['profile', 'email'] })
);

// Google callback
app.get('/auth/google/callback',
  passport.authenticate('google', { failureRedirect: '/login', failureFlash: true }),
  (req, res) => { res.redirect('/dashboard'); }
);
```

### Google Profile Object

```javascript
{
  id: '1234567890',
  displayName: 'Jane Doe',
  name: { givenName: 'Jane', familyName: 'Doe' },
  emails: [{ value: 'jane@gmail.com', verified: true }],
  photos: [{ value: 'https://...' }],
  provider: 'google'
}
```

### Setup in Google Console

- Go to `console.cloud.google.com`
- Create OAuth 2.0 credentials
- Add authorized redirect URIs
- Enable Google+ API or People API

---

## Slide 10 — GitHub OAuth Strategy

### Install & Configure

```bash
npm install passport-github2
```

```javascript
const GitHubStrategy = require('passport-github2').Strategy;

passport.use(new GitHubStrategy({
    clientID: process.env.GITHUB_CLIENT_ID,
    clientSecret: process.env.GITHUB_CLIENT_SECRET,
    callbackURL: '/auth/github/callback'
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      let user = await User.findOne({ githubId: profile.id });
      if (!user) {
        user = await User.create({
          githubId: profile.id,
          name: profile.displayName || profile.username,
          email: profile.emails?.[0]?.value,
          avatar: profile.photos?.[0]?.value
        });
      }
      return done(null, user);
    } catch (err) {
      return done(err);
    }
  }
));
```

### Routes

```javascript
app.get('/auth/github',
  passport.authenticate('github', { scope: ['user:email'] })
);

app.get('/auth/github/callback',
  passport.authenticate('github', { failureRedirect: '/login', failureFlash: true }),
  (req, res) => { res.redirect('/dashboard'); }
);
```

### GitHub Profile Mapping

```javascript
{
  id: '12345',
  username: 'janedoe',
  displayName: 'Jane Doe',
  profileUrl: 'https://github.com/janedoe',
  emails: [{ value: 'jane@example.com' }],
  photos: [{ value: 'https://avatars...' }],
  provider: 'github'
}
```

### GitHub Scopes

| Scope | Access |
|-------|--------|
| `user:email` | Read email addresses |
| `read:user` | Read profile info |
| `repo` | Read/write repositories |
| `read:org` | Read org membership |

GitHub emails can be private. Always request `user:email` scope and handle the case where `profile.emails` is empty.

---

## Slide 11 — JWT Strategy

### Install & Configure

```bash
npm install passport-jwt jsonwebtoken
```

```javascript
const JwtStrategy = require('passport-jwt').Strategy;
const ExtractJwt = require('passport-jwt').ExtractJwt;

const opts = {
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  secretOrKey: process.env.JWT_SECRET
};

passport.use(new JwtStrategy(opts,
  async (jwt_payload, done) => {
    try {
      const user = await User.findById(jwt_payload.sub);
      if (user) return done(null, user);
      return done(null, false);
    } catch (err) {
      return done(err, false);
    }
  }
));
```

### Issuing Tokens

```javascript
const jwt = require('jsonwebtoken');

app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });

  if (!user || !await bcrypt.compare(password, user.password)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const token = jwt.sign(
    { sub: user.id, email: user.email },
    process.env.JWT_SECRET,
    { expiresIn: '1h' }
  );

  res.json({ token });
});
```

### Protecting API Routes

```javascript
app.get('/api/profile',
  passport.authenticate('jwt', { session: false }),
  (req, res) => { res.json({ user: req.user }); }
);
```

Note: `session: false` — JWT is **stateless**. No session needed.

### Token Extraction Methods

| Method | Extracts From |
|--------|---------------|
| `fromAuthHeaderAsBearerToken()` | `Authorization: Bearer <token>` |
| `fromHeader('x-token')` | Custom header |
| `fromUrlQueryParameter('token')` | Query string |
| `fromBodyField('token')` | Request body |

---

## Slide 12 — Protecting Routes

### ensureAuthenticated Middleware

```javascript
function ensureAuth(req, res, next) {
  if (req.isAuthenticated()) {
    return next();
  }
  req.flash('error', 'Please log in');
  res.redirect('/login');
}

// Use on individual routes
app.get('/dashboard', ensureAuth, (req, res) => {
  res.render('dashboard', { user: req.user });
});

// Use on route groups
app.use('/admin', ensureAuth, adminRouter);
```

### Guest-Only Middleware

```javascript
function ensureGuest(req, res, next) {
  if (!req.isAuthenticated()) {
    return next();
  }
  res.redirect('/dashboard');
}

app.get('/login', ensureGuest, (req, res) => {
  res.render('login');
});
```

### Role-Based Access Control

```javascript
function authorise(...roles) {
  return (req, res, next) => {
    if (!req.isAuthenticated()) return res.redirect('/login');
    if (!roles.includes(req.user.role)) {
      return res.status(403).render('403', { message: 'Access denied' });
    }
    next();
  };
}

app.get('/admin/users', authorise('admin'), adminController.listUsers);
app.get('/posts/edit/:id', authorise('admin', 'editor'), postController.edit);
```

### Return-To Pattern

```javascript
function ensureAuth(req, res, next) {
  if (req.isAuthenticated()) return next();
  req.session.returnTo = req.originalUrl;
  res.redirect('/login');
}

app.post('/login', passport.authenticate('local', { failureRedirect: '/login' }),
  (req, res) => {
    const url = req.session.returnTo || '/';
    delete req.session.returnTo;
    res.redirect(url);
  }
);
```

---

## Slide 13 — User Model & Database

### Mongoose User Schema

```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true, lowercase: true },
  password: String,
  googleId: String,
  githubId: String,
  avatar: String,
  role: { type: String, enum: ['user', 'editor', 'admin'], default: 'user' },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('User', userSchema);
```

### Sequelize User Model

```javascript
const { DataTypes } = require('sequelize');

module.exports = (sequelize) => {
  return sequelize.define('User', {
    name: { type: DataTypes.STRING, allowNull: false },
    email: { type: DataTypes.STRING, allowNull: false, unique: true },
    password: DataTypes.STRING,
    googleId: DataTypes.STRING,
    githubId: DataTypes.STRING,
    avatar: DataTypes.STRING,
    role: { type: DataTypes.ENUM('user', 'editor', 'admin'), defaultValue: 'user' }
  });
};
```

### findOrCreate Pattern

```javascript
let user = await User.findOne({ googleId: profile.id });
if (!user) {
  user = await User.create({
    googleId: profile.id,
    name: profile.displayName,
    email: profile.emails[0].value
  });
}
```

**Tip:** Make `password`, `googleId`, and `githubId` optional. Users who register via OAuth won't have a password; users who register locally won't have provider IDs.

---

## Slide 14 — Multiple Strategies — Linking Accounts

### Linking OAuth to Existing Accounts

```javascript
// In Google strategy verify callback
async (accessToken, refreshToken, profile, done) => {
  // 1. Check if already linked
  let user = await User.findOne({ googleId: profile.id });
  if (user) return done(null, user);

  // 2. Check if email matches existing user
  const email = profile.emails[0].value;
  user = await User.findOne({ email });
  if (user) {
    user.googleId = profile.id;
    user.avatar = user.avatar || profile.photos[0].value;
    await user.save();
    return done(null, user);
  }

  // 3. Create new user
  user = await User.create({
    googleId: profile.id,
    name: profile.displayName,
    email, avatar: profile.photos[0].value
  });
  return done(null, user);
}
```

### Account Settings Page

```javascript
app.get('/account/link/github',
  ensureAuth,
  passport.authorize('github', { scope: ['user:email'] })
);

app.get('/account/link/github/callback',
  ensureAuth,
  passport.authorize('github', { failureRedirect: '/account' }),
  async (req, res) => {
    req.user.githubId = req.account.id;
    await req.user.save();
    req.flash('success', 'GitHub linked!');
    res.redirect('/account');
  }
);
```

Note: `passport.authorize()` does not affect `req.user` — it populates `req.account` instead.

### Unlink Provider

```javascript
app.post('/account/unlink/github', ensureAuth, async (req, res) => {
  if (!req.user.password && !req.user.googleId) {
    req.flash('error', 'Keep at least one login method');
    return res.redirect('/account');
  }
  req.user.githubId = undefined;
  await req.user.save();
  res.redirect('/account');
});
```

---

## Slide 15 — Security Best Practices

### Session Fixation

```javascript
app.post('/login', (req, res, next) => {
  passport.authenticate('local', (err, user, info) => {
    if (err) return next(err);
    if (!user) return res.redirect('/login');
    req.session.regenerate((err) => {
      if (err) return next(err);
      req.login(user, (err) => {
        if (err) return next(err);
        res.redirect('/dashboard');
      });
    });
  })(req, res, next);
});
```

Prevents attackers from pre-setting a session ID before the user logs in.

### Rate Limiting Login Attempts

```javascript
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 mins
  max: 5,
  message: 'Too many login attempts',
  standardHeaders: true,
  legacyHeaders: false
});

app.post('/login', loginLimiter, passport.authenticate('local', { ... }));
```

### Secure Cookie Configuration

```javascript
app.use(session({
  secret: process.env.SESSION_SECRET,
  cookie: {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 3600000,
    domain: '.myapp.com'
  }
}));
```

### CSRF Protection

```javascript
const csrf = require('csurf');
app.use(csrf());

app.use((req, res, next) => {
  res.locals.csrfToken = req.csrfToken();
  next();
});
```

```html
<form method="POST" action="/login">
  <input type="hidden" name="_csrf" value="<%= csrfToken %>">
  ...
</form>
```

### Security Checklist

- **bcrypt** with 12+ salt rounds for passwords
- **Helmet.js** for HTTP security headers
- **HTTPS** in production (always)
- **Input validation** with express-validator
- **Account lockout** after repeated failures
- **Audit logging** for login events

---

## Slide 16 — Error Handling & Flash Messages

### Built-In Redirect Options

```javascript
passport.authenticate('local', {
  successRedirect: '/dashboard',
  failureRedirect: '/login',
  failureFlash: true,
  failureFlash: 'Invalid credentials',
  successFlash: 'Welcome back!'
});
```

`failureFlash: true` uses the message from the strategy's `done(null, false, { message })` call.

### Custom Callback Pattern

```javascript
app.post('/login', (req, res, next) => {
  passport.authenticate('local', (err, user, info) => {
    if (err) return next(err);
    if (!user) {
      return res.status(401).render('login', {
        error: info.message,
        email: req.body.email
      });
    }
    req.logIn(user, (err) => {
      if (err) return next(err);
      return res.redirect('/dashboard');
    });
  })(req, res, next);
});
```

### Flash Message Template

```html
<!-- views/partials/flash.ejs -->
<% if (success && success.length) { %>
  <div class="alert alert-success"><%= success %></div>
<% } %>
<% if (error && error.length) { %>
  <div class="alert alert-danger"><%= error %></div>
<% } %>
```

### Common Error Scenarios

| Error | Cause |
|-------|-------|
| `Failed to serialize user` | Missing `serializeUser` |
| `Failed to deserialize user` | User deleted from DB |
| `No auth token found` | Missing/expired JWT |
| `OAuth callback error` | Mismatched redirect URI |
| `InternalOAuthError` | Provider API down or wrong credentials |

**Tip:** Use the custom callback pattern when you need fine-grained control (API JSON responses, custom status codes, form repopulation). Use redirect options for simple page-based flows.

---

## Slide 17 — Testing Authentication

### Supertest with Sessions

```javascript
const request = require('supertest');
const app = require('../app');

describe('Auth Routes', () => {
  const agent = request.agent(app);

  it('should log in', async () => {
    await agent
      .post('/login')
      .send({ email: 'test@test.com', password: 'password123' })
      .expect(302)
      .expect('Location', '/dashboard');
  });

  it('should access protected route', async () => {
    const res = await agent.get('/dashboard').expect(200);
    expect(res.text).toContain('Dashboard');
  });
});
```

### Mocking Passport

```javascript
function mockAuth(user) {
  return (req, res, next) => {
    req.isAuthenticated = () => true;
    req.user = user;
    next();
  };
}

app.use(mockAuth({
  id: '1', name: 'Test User',
  email: 'test@test.com', role: 'admin'
}));
```

### Integration Test Checklist

- Login with valid credentials → 302 to dashboard
- Login with invalid credentials → 302 to login + flash
- Access protected route without auth → 302 to login
- Access protected route with auth → 200
- Logout → session cleared, 302 to home
- Role-restricted route → 403 for wrong role

### Test Database Strategy

Use a separate test database or in-memory MongoDB (`mongodb-memory-server`). Seed a test user before each suite. Clean up after tests.

---

## Slide 18 — Complete Working Example

### app.js — Express + Passport + EJS

```javascript
const express = require('express');
const session = require('express-session');
const passport = require('passport');
const flash = require('connect-flash');
const path = require('path');

require('./config/passport')(passport);

const app = express();
app.set('view engine', 'ejs');
app.set('views', path.join(__dirname, 'views'));
app.use(express.urlencoded({ extended: true }));
app.use(express.static('public'));

app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false
}));
app.use(passport.initialize());
app.use(passport.session());
app.use(flash());

app.use((req, res, next) => {
  res.locals.currentUser = req.user;
  res.locals.success = req.flash('success');
  res.locals.error = req.flash('error');
  next();
});

app.use('/', require('./routes/index'));
app.use('/auth', require('./routes/auth'));

app.listen(3000);
```

### config/passport.js

```javascript
const LocalStrategy = require('passport-local').Strategy;
const GoogleStrategy = require('passport-google-oauth20').Strategy;
const bcrypt = require('bcrypt');
const User = require('../models/User');

module.exports = (passport) => {
  passport.use(new LocalStrategy(
    { usernameField: 'email' },
    async (email, password, done) => {
      const user = await User.findOne({ email });
      if (!user) return done(null, false, { message: 'No account found' });
      const match = await bcrypt.compare(password, user.password);
      if (!match) return done(null, false, { message: 'Wrong password' });
      return done(null, user);
    }
  ));

  passport.use(new GoogleStrategy({ ... },
    async (access, refresh, profile, done) => {
      // findOrCreate logic
    }
  ));

  passport.serializeUser((user, done) => done(null, user.id));
  passport.deserializeUser(async (id, done) => done(null, await User.findById(id)));
};
```

### Project Structure

```text
passport-app/
├── app.js
├── .env
├── config/
│   └── passport.js
├── models/
│   └── User.js
├── routes/
│   ├── index.js
│   └── auth.js
├── middleware/
│   └── auth.js
└── views/
    ├── layouts/main.ejs
    ├── partials/nav.ejs
    ├── login.ejs
    ├── register.ejs
    └── dashboard.ejs
```

---

## Slide 19 — Summary & Next Steps

### Core Takeaways

- Passport.js = pluggable authentication via the strategy pattern
- 500+ strategies: local, Google, GitHub, JWT, and more
- serializeUser stores, deserializeUser restores from session
- OAuth 2.0 for third-party login without handling passwords
- JWT for stateless API authentication

### Best Practices

- Always hash passwords with bcrypt (12+ rounds)
- Use a production session store (Redis, Mongo)
- Secure cookies: httpOnly, secure, sameSite
- Rate-limit login endpoints
- Regenerate sessions after login
- Validate all input with express-validator

### Next Steps

- Build a full CRUD app with Passport auth
- Add Google + GitHub OAuth to an existing app
- Implement JWT auth for a REST API
- Add two-factor authentication (2FA)
- Explore Passport with TypeScript
- Try alternative libraries: Auth.js, Lucia

### Essential Resources

- **Official Docs:** passportjs.org
- **GitHub:** github.com/jaredhanson/passport
- **npm:** npmjs.com/package/passport
- **Strategies:** passportjs.org/packages/

### Key Packages

| Package | Purpose |
|---------|---------|
| `passport` | Core authentication framework |
| `passport-local` | Username/password strategy |
| `passport-google-oauth20` | Google OAuth 2.0 |
| `passport-github2` | GitHub OAuth |
| `passport-jwt` | JSON Web Token strategy |
