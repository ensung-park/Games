# Sweet Match v2: Cloud Save, Google Auth, Graphics Polish & Public Deploy

## Overview

Transform the Sweet Match game from a local single-player experience into a publicly accessible web game with Google account login, cloud-synced progress, improved visual animations, and a leaderboard. Deploy on GitHub Pages with Firebase for auth and data.

## Architecture

```mermaid
graph TB
    subgraph "Client (GitHub Pages)"
        HTML[index.html<br/>Game + UI]
        AUTH[Firebase Auth SDK<br/>Google Sign-In]
        DB[Firestore SDK<br/>Cloud Save/Load]
        CANVAS[Canvas Renderer<br/>Improved Animations]
    end

    subgraph "Firebase (Google Cloud)"
        FA[Firebase Authentication<br/>Google OAuth 2.0]
        FS[Cloud Firestore<br/>User Progress + Leaderboard]
        SEC[Security Rules<br/>Users own their data]
    end

    subgraph "Hosting"
        GHP[GitHub Pages<br/>ensung-park.github.io/Games]
    end

    HTML --> AUTH
    HTML --> DB
    HTML --> CANVAS
    AUTH --> FA
    DB --> FS
    FS --> SEC
    GHP --> HTML
```

## Authentication Flow

```mermaid
sequenceDiagram
    participant U as User
    participant G as Game (Browser)
    participant FA as Firebase Auth
    participant FS as Firestore

    U->>G: Opens game
    G->>G: Check localStorage for cached user
    G->>G: Show game with "Sign In" button
    U->>G: Clicks "Sign In with Google"
    G->>FA: signInWithPopup(GoogleAuthProvider)
    FA-->>G: User credential (uid, displayName, photoURL)
    G->>FS: getDoc(users/{uid})
    alt User exists
        FS-->>G: Saved progress (levels, stars, highScores)
        G->>G: Merge with local progress (keep best stars)
    else New user
        G->>FS: setDoc(users/{uid}, defaultProgress)
    end
    G->>G: Show level select with synced progress

    Note over U,FS: On level complete:
    U->>G: Completes level
    G->>FS: updateDoc(users/{uid}, newProgress)
    G->>FS: Update leaderboard entry
```

## Data Model

```mermaid
erDiagram
    USERS {
        string uid PK "Firebase Auth UID"
        string displayName "Google display name"
        string photoURL "Google profile photo"
        map progress "level_num -> stars (1-3)"
        int totalStars "Sum of all stars"
        timestamp lastPlayed "Last activity"
        timestamp createdAt "Account creation"
    }

    LEADERBOARD {
        string odcId PK "Auto-generated"
        string uid FK "References USERS"
        string displayName "Cached for fast reads"
        int totalStars "Ranking metric"
        int highestLevel "Highest completed level"
        timestamp updatedAt "Last update"
    }

    USERS ||--o| LEADERBOARD : "has entry"
```

### Firestore Collections

**`users/{uid}`** document:
```json
{
  "displayName": "John Doe",
  "photoURL": "https://...",
  "progress": { "1": 3, "2": 2, "3": 1 },
  "totalStars": 6,
  "highestLevel": 3,
  "lastPlayed": "<timestamp>",
  "createdAt": "<timestamp>"
}
```

**`leaderboard/{uid}`** document (denormalized for fast reads):
```json
{
  "uid": "abc123",
  "displayName": "John Doe",
  "photoURL": "https://...",
  "totalStars": 42,
  "highestLevel": 15,
  "updatedAt": "<timestamp>"
}
```

### Security Rules
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.auth.uid == userId;
    }
    match /leaderboard/{userId} {
      allow read: if true;  // Public leaderboard
      allow write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

## Graphics Improvements

### Current Problems
- No swap animation (candies teleport)
- No gravity/fall animation (candies appear instantly)
- No scale-in for new candies
- Match removal is instant (no satisfying dissolve)
- Screen shake uses random noise instead of smooth oscillation

### Animation System

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Swapping: User swaps
    Swapping --> MatchCheck: Swap complete (200ms)
    MatchCheck --> Matching: Matches found
    MatchCheck --> SwapBack: No matches
    SwapBack --> Idle: Reversed (200ms)
    Matching --> Falling: Matched cleared (300ms)
    Falling --> MatchCheck: Gravity settled (250ms)
    Matching --> Falling: Cascade
```

**Tweened animations to add:**

| Animation | Duration | Easing | Description |
|-----------|----------|--------|-------------|
| Swap slide | 180ms | easeInOut | Two candies slide to each other's positions |
| Swap back | 180ms | easeInOut | Invalid swap reversal |
| Match dissolve | 300ms | easeOut | Scale to 1.3x then to 0, brightness increases |
| Gravity fall | 250ms per row | bounce | Candies drop with slight bounce at bottom |
| New candy enter | 200ms | easeOut | Scale from 0 to 1.05 to 1.0 |
| Special create | 350ms | bounce | Flash white, pulse scale 1.0 -> 1.3 -> 1.0 |
| Score popup | 800ms | easeOut | Float upward with fade, slight scale |

**Implementation approach:** Each cell gets optional `drawX`, `drawY`, `scale`, `alpha` properties. The game loop tweens these properties over time. When all animations complete, the next game phase begins.

## Game Screens (Updated)

```mermaid
graph LR
    TITLE[Title Screen<br/>Logo + Sign In] --> LS[Level Select<br/>+ Leaderboard Tab]
    LS --> PLAY[Playing]
    PLAY --> WIN[Level Complete]
    PLAY --> FAIL[Level Failed]
    WIN --> LS
    FAIL --> LS
    LS --> TITLE
    PLAY --> LS
```

### Title Screen (New)
- Game logo with animated candy background
- "Sign in with Google" button (styled, not the default Google button)
- "Play as Guest" option (uses localStorage only, no cloud save)
- If already signed in, auto-redirect to level select

### Level Select (Updated)
- User avatar + name in top-left (or "Guest" badge)
- "Leaderboard" tab showing top 20 players by total stars
- Sign out button
- Current grid layout preserved

### Leaderboard
- Top 20 players sorted by totalStars descending
- Shows rank, avatar, name, total stars, highest level
- Current user highlighted
- Refreshed on each view (Firestore query, ordered, limited to 20)

## File Structure

```
Games/
  sweet-match/
    index.html          (game - loads Firebase from CDN)
  firebase.json         (Firebase hosting config, optional)
  firestore.rules       (Firestore security rules)
  .firebaserc           (Firebase project config)
  README.md             (updated with game link + setup)
```

The game remains a single `index.html` file. Firebase SDKs are loaded from CDN (`https://www.gstatic.com/firebasejs/...`). No build step needed.

## Deployment

### GitHub Pages
- Repo must be **public** (or GitHub Pro for private Pages)
- Enable Pages in repo Settings -> Pages -> Source: "Deploy from a branch" -> `master` / `root`
- Game accessible at `https://ensung-park.github.io/Games/sweet-match/`

### Firebase Setup (One-time)
1. Create Firebase project at console.firebase.google.com
2. Enable Google Authentication provider
3. Create Firestore database
4. Deploy security rules
5. Add GitHub Pages domain to authorized domains
6. Copy Firebase config object into index.html

## Guest Mode
- Users who don't sign in play as "Guest"
- Progress saved to localStorage only (current behavior)
- Prompt to sign in on level complete: "Sign in to save your progress!"
- When guest signs in, merge local progress with cloud (keep higher stars)

## Non-Goals
- No real-time multiplayer gameplay
- No social features beyond leaderboard
- No payment/IAP
- No custom avatars (use Google profile photo)
- No push notifications
- No offline-first with sync queue (simple online-only cloud save)
