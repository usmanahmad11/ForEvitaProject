# MoodTunes Full Stack Project Setup Guide

Set up your MoodTunes project from scratch, including backend, frontend, and the database.

---

## 1. Prerequisites

- **Node.js** and **npm** ([Download Node.js](https://nodejs.org/))
- **MongoDB** (local or remote)  
  [Install MongoDB locally](https://docs.mongodb.com/manual/installation/)
- **Google OAuth Credentials**  
  - Get these at [Google Cloud Console](https://console.developers.google.com/)
  - Set `Authorized redirect URI` to: `http://localhost:3001/auth/google/callback`
- (Optional) **Git** for cloning the repo

---

## 2. Project Structure

```
MoodTunes/
├── backend/
│   ├── .env.example
│   ├── app.js
│   ├── models/
│   │   └── User.js
│   └── routes/
│       ├── auth.js
│       └── user.js
├── frontend/
│   ├── package.json
│   ├── public/
│   │   └── index.html
│   └── src/
│       ├── App.js
│       ├── index.js
│       └── App.css
├── README.md
```

---

## 3. Backend Setup

### 3.1. Install Dependencies

```bash
cd backend
npm install
```

### 3.2. Create `.env` File

Copy `.env.example` to `.env` and fill in your credentials:

```env
MONGODB_URI=mongodb://localhost:27017/moodtunes
SESSION_SECRET=your-session-secret
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
```

### 3.3. Start MongoDB

If running locally, start your MongoDB server:

```bash
mongod
```

### 3.4. Start the Backend Server

```bash
npm start
# or
node app.js
```
Visit [http://localhost:3001/](http://localhost:3001/) to check backend health.

---

## 4. Frontend Setup

```bash
cd ../frontend
npm install
npm start
```
Visit [http://localhost:3000/](http://localhost:3000/) in your browser.

---

## 5. Key Backend Files

### backend/app.js

```js
const express = require('express');
const session = require('express-session');
const mongoose = require('mongoose');
const passport = require('passport');
const cors = require('cors');
const dotenv = require('dotenv');
const authRoutes = require('./routes/auth');
const userRoutes = require('./routes/user');
dotenv.config();

const app = express();

mongoose.connect(process.env.MONGODB_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log('MongoDB Connected'))
  .catch(err => console.error(err));

app.use(cors({ origin: 'http://localhost:3000', credentials: true }));
app.use(express.json());
app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
}));
app.use(passport.initialize());
app.use(passport.session());

app.use('/auth', authRoutes);
app.use('/user', userRoutes);

app.get('/', (req, res) => res.send('MoodTunes backend running!'));

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### backend/models/User.js

```js
const mongoose = require('mongoose');

const MoodSchema = new mongoose.Schema({
  mood: String,
  genre: String,
  tracks: [String],
  date: { type: Date, default: Date.now }
});

const UserSchema = new mongoose.Schema({
  googleId: { type: String, unique: true },
  name: String,
  email: String,
  moodHistory: [MoodSchema]
});

module.exports = mongoose.model('User', UserSchema);
```

### backend/routes/auth.js

```js
const express = require('express');
const passport = require('passport');
const User = require('../models/User');
const router = express.Router();
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
  clientID: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  callbackURL: '/auth/google/callback',
}, async (accessToken, refreshToken, profile, done) => {
  try {
    let user = await User.findOne({ googleId: profile.id });
    if (!user) {
      user = new User({
        googleId: profile.id,
        name: profile.displayName,
        email: profile.emails[0].value
      });
      await user.save();
    }
    return done(null, user);
  } catch (err) {
    return done(err);
  }
}));

passport.serializeUser((user, done) => done(null, user.id));
passport.deserializeUser(async (id, done) => {
  try {
    const user = await User.findById(id);
    done(null, user);
  } catch (err) {
    done(err, null);
  }
});

router.get('/google', passport.authenticate('google', { scope: ['profile', 'email'] }));

router.get('/google/callback',
  passport.authenticate('google', { failureRedirect: '/' }),
  (req, res) => {
    res.redirect('http://localhost:3000'); // Frontend URL
  }
);

router.get('/me', (req, res) => {
  if (req.user) {
    res.json(req.user);
  } else {
    res.status(401).json({ error: 'Not authenticated' });
  }
});

router.get('/logout', (req, res) => {
  req.logout(() => res.redirect('http://localhost:3000'));
});

module.exports = router;
```

### backend/routes/user.js

```js
const express = require('express');
const User = require('../models/User');
const router = express.Router();

// Save a new mood entry
router.post('/add-mood', async (req, res) => {
  const { userId, mood, genre, tracks } = req.body;
  try {
    const user = await User.findById(userId);
    if (!user) return res.status(404).json({ error: 'User not found' });

    user.moodHistory.push({ mood, genre, tracks });
    await user.save();

    res.json({ success: true, moodHistory: user.moodHistory });
  } catch (err) {
    res.status(500).json({ error: 'Failed to save mood.' });
  }
});

// Get mood history
router.get('/moods/:userId', async (req, res) => {
  try {
    const user = await User.findById(req.params.userId);
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json({ moodHistory: user.moodHistory });
  } catch (err) {
    res.status(500).json({ error: 'Failed to get mood history.' });
  }
});

module.exports = router;
```

---

## 6. Key Frontend Files

### frontend/package.json

```json
{
  "name": "frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0",
    "react-scripts": "5.0.0"
  },
  "scripts": {
    "start": "react-scripts start"
  }
}
```

### frontend/public/index.html

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%PUBLIC_URL%/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>MoodTunes</title>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
  </body>
</html>
```

### frontend/src/index.js

```js
import React from 'react';
import ReactDOM from 'react-dom/client';
import './App.css';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### frontend/src/App.js

```jsx
import React, { useState, useEffect } from 'react';

function App() {
  const [user, setUser] = useState(null);
  const [mood, setMood] = useState('');
  const [genre, setGenre] = useState('');
  const [tracks, setTracks] = useState('');
  const [history, setHistory] = useState([]);

  useEffect(() => {
    fetch('http://localhost:3001/auth/me', { credentials: 'include' })
      .then(res => res.ok ? res.json() : null)
      .then(data => data && setUser(data));
  }, []);

  const handleSave = () => {
    fetch('http://localhost:3001/user/add-mood', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include',
      body: JSON.stringify({
        userId: user._id,
        mood, genre,
        tracks: tracks.split(',').map(t => t.trim())
      })
    })
      .then(res => res.json())
      .then(data => setHistory(data.moodHistory));
  };

  useEffect(() => {
    if (user) {
      fetch(`http://localhost:3001/user/moods/${user._id}`)
        .then(res => res.json())
        .then(data => setHistory(data.moodHistory));
    }
  }, [user]);

  return (
    <div>
      <h1>MoodTunes</h1>
      {!user ? (
        <a href="http://localhost:3001/auth/google">
          <button>Login with Google</button>
        </a>
      ) : (
        <div>
          <p>Welcome, {user.name}!</p>
          <button onClick={() => window.location = 'http://localhost:3001/auth/logout'}>Logout</button>
          <div>
            <h2>Save Mood</h2>
            <input placeholder="Mood" value={mood} onChange={e => setMood(e.target.value)} />
            <input placeholder="Genre" value={genre} onChange={e => setGenre(e.target.value)} />
            <input placeholder="Tracks (comma separated)" value={tracks} onChange={e => setTracks(e.target.value)} />
            <button onClick={handleSave}>Save</button>
          </div>
          <div>
            <h2>Mood History</h2>
            <ul>
              {history.map((entry, idx) => (
                <li key={idx}>{entry.date?.substring(0,10)}: {entry.mood} - {entry.genre} [{entry.tracks?.join(', ')}]</li>
              ))}
            </ul>
          </div>
        </div>
      )}
    </div>
  );
}

export default App;
```

### frontend/src/App.css

```css
body {
  font-family: Arial, sans-serif;
  background: #f7f7f7;
  margin: 0;
  padding: 0;
}
h1 {
  color: #222;
}
button {
  margin: 8px;
  padding: 8px 18px;
  font-size: 1em;
  background: #2196f3;
  color: #fff;
  border: none;
  border-radius: 6px;
}
input {
  margin: 4px;
  padding: 6px;
}
ul {
  list-style: none;
  padding: 0;
}
li {
  margin: 6px 0;
}
```

---


---

**You now have a full-stack MoodTunes project!**
