# 🎮 PatienceRush

A fast-paced 2D platformer for Android where patience is your greatest weapon.  
Jump, dodge, and survive through 50 handcrafted levels — each one harder than the last.

---

## 🚀 Features

- 50 progressively difficult levels
- Custom 2D physics engine (no third-party physics library)
- Procedurally parameterized map generation
- Moving obstacle blocks (horizontal & vertical)
- Google Play Games sign-in & leaderboard integration
- Firebase Firestore global leaderboard (Top 10)
- Background music & volume control
- Mirrored / swappable touch controls
- Interstitial ads via Google AdMob
- Progress persistence via local storage

---

## 🛠️ Tech Stack

| Technology | Usage |
|---|---|
| **Java** (JDK 21) | Core game logic |
| **Android SDK 36** | Platform target |
| **SurfaceView + Thread** | Custom game loop rendering |
| **Firebase Firestore** | Cloud leaderboard storage |
| **Google Play Games Services** | Authentication & player identity |
| **Google AdMob** | Interstitial ad display |
| **RecyclerView** | Leaderboard UI list |
| **MediaPlayer** | Background music |
| **Gradle (Kotlin DSL)** | Build system |

---

## ⚙️ Core Algorithms

### 1. 🔁 Game Loop
The game runs on a dedicated background thread using `SurfaceView`. Each frame cycle executes three steps in order: **update → draw → FPS control**. The loop targets a fixed tick rate and sleeps the thread for a constant duration to maintain consistent timing across devices.

### 2. 🧱 Procedural Map Generation (`MapCreator`)
Maps are not pre-designed files — they are algorithmically generated at runtime for each level. Key mechanics:
- The number of blocks **scales with level** using a power function (`blockCount ≈ base × level^0.825`), keeping early levels simple and later levels dense.
- Block **width and height** are randomized within bounds that shrink as difficulty increases, forcing more precise jumps.
- A **type roll** system decides per-block whether it becomes a standard platform, a ceiling piece, or a wall — adding vertical complexity.
- A **moving block chance** scales linearly with level (capped at 60%), randomly assigning horizontal or vertical movement to eligible blocks.
- The layout uses a **gap-based horizontal spacing** algorithm: each new block is placed after a fixed gap from the previous, with Y position varied within reachable jump height bounds.
- A `FinishBlock` is always placed at a fixed elevated Y position after all regular blocks, acting as the level goal.

### 3. ⚡ Physics Engine (`Character`)
A fully custom axis-separated physics system:
- **Gravity** is applied every frame as a constant downward Y acceleration.
- **Jump** applies a strong negative Y velocity only when the character is grounded (`jumpReady` flag).
- **Horizontal movement** uses acceleration-based speed buildup (not instant velocity), and **friction** decelerates the character when no input is held, using a multiplier + divisor formula to simulate surface drag.
- **Collision detection** is split into two independent passes per frame:
  1. **Y-axis first** — resolves vertical overlap to determine ground/ceiling contact and sets the `jumpReady` flag.
  2. **X-axis second** — resolves horizontal overlap to stop the character against walls.
  - A `SNAG_TOLERANCE` constant (5px) filters micro-intersections to prevent the character from getting stuck on block edges.
- **Moving block attachment**: when the character lands on a moving block, its current frame delta (`currentDx` / `currentDy`) is added to the character's position, creating a "riding" effect.

### 4. 📐 Responsive Viewport (`GameView`)
The game is designed against a reference resolution of **1240×720**. At runtime, `scaleFactor` and `translate` offsets are computed to map the design coordinates onto any real screen size. The camera **follows the character** with a configurable X/Y offset, keeping the player always in view.

### 5. 🏆 Leaderboard & Score Saving (`FirebaseHelper`)
Score saving uses a **two-path strategy**:
- If the player is signed into Google Play Games, their display name is fetched and saved with their level to Firestore.
- If no Google account is available, a fallback anonymous identifier is used.
- Leaderboard queries fetch the **top 10 entries** ordered by level, displayed in a `RecyclerView`.

---

## 🎮 Controls

| Action | Button |
|---|---|
| Move Left | Left arrow button |
| Move Right | Right arrow button |
| Jump | Up arrow button |

Controls can be **mirrored** from the options menu to support left/right-handed preference.

## 📄 License

© 2026 ScanGraphi. All rights reserved.

This project is proprietary software, developed entirely from scratch by Muhammed Emin Aydın within ScanGraphi. Unauthorized copying, modification, distribution, or commercial use is strictly prohibited.
