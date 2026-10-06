# Technical Case Study

Would You Rather TV is a Flutter app for Android TV with a small Kotlin layer. This document walks through it in the order that mattered most during development: the remote first, then the game, then everything that supports it.

Statements are drawn from reading the production code, manifests and build files. Nothing was run to measure performance, and no numbers beyond counts of files, lines and tests are given.

## 1. The input model comes first

A TV has no pointer. Every control has to be reachable with four arrows and a confirm key, and a group of people share one remote.

**How the app does it**

- Every interactive element is a `Focus` widget. It owns a boolean "focused" state, redraws itself when that changes (a scale to about 1.08, a gold border and a glow) and handles Select and Enter in `onKeyEvent`.
- Movement between elements is left to Flutter's directional focus traversal. The first sensible control on each screen sets `autofocus`. This is a different choice from a hand-written cursor model: the layouts here (a centred column, a wrapped grid, two side-by-side option cards) suit geometric traversal.
- A single `HardwareKeyboard` handler registered at the app root plays a navigation click for arrows and a select click for Select and Enter. It returns `false`, so it never consumes a key that a widget needs.
- Screens that scroll (the category grid, settings) call `Scrollable.ensureVisible` when a card gains focus, so the focused card is always on screen.
- Every control still has a `GestureDetector` for taps, which costs nothing and helps when the app is driven from a debug build on a phone.

**What this does not do**

There is no custom focus order. Behaviour depends on widget geometry, and nothing in the test suite exercises it.

## 2. Dialogs, overlays and the Back key

Three kinds of layered UI exist: the unlock dialog, the "time's up" dialog and the pause overlay.

- The pause overlay is a `FocusScope` with autofocus, drawn over the vote screen. Its primary button, Resume, has autofocus, so the overlay is usable immediately.
- The unlock dialog is a `FocusScope` with autofocus on Buy, wrapped in a `PopScope` that routes Back to a pop. It also mixes in a helper written to ignore Back for the first 300 ms, because the code comments say key repeat in release builds could close a dialog the instant it opened. The helper's key handler is never attached, so the guard is not active; this is noted as a gap.
- The "time's up" dialog cannot be dismissed by tapping outside and has one focused button.
- Back on the pause overlay is not intercepted: the vote screen's `PopScope` allows the pop and logs the game as abandoned.
- The vote screen watches `isPaused` and `_navigatedAway`. While a results screen or the scoreboard is on top, or while the game is paused, its timer ticks but does nothing.

## 3. Naming players with a TV keyboard

The name screen uses real text fields. On Android TV the system keyboard runs as a separate activity, and pressing Back closes it without a clear focus event in Flutter.

The screen listens for three things and funnels all of them into one method:

1. A native `ViewCompat` window-insets listener in `MainActivity` that, when the keyboard goes from visible to hidden, calls a method channel into Dart.
2. A view-inset change observed in `didChangeMetrics`.
3. The app returning to the foreground.

That method hides the keyboard, clears the text-input client so the cursor stops blinking, drops primary focus, restores the default name if the field was left blank and hands focus back to the player's row. Analytics for this screen record only which slot was opened, never the name.

## 4. Content: bundled files, no backend

Questions are JSON arrays of `{id, optionA, optionB}` in the app bundle: four files for the free categories in one folder and ten in another. The loader tries one folder, then the other.

- **Counts.** 14 files with 200 entries each, 2,800 entries in all. Thirteen categories are wired up, so 2,600 entries are reachable. One category has a file in both folders; the loader reads the first folder it tries, so the other file is never read.
- **De-duplication.** On first load, entries with the same pair of options (case-insensitive, trimmed) are dropped. That leaves about 2,500 distinct entries across the thirteen loaded categories.
- **Shuffle.** `Random.secure()` shuffles the available list for every new game.
- **Anti-repeat.** The ids of questions shown in a finished game are stored per category in a Hive box. The next game filters those out. When the pool is exhausted, the record for that category resets and the full pool is used again.
- **Caching.** Parsed lists are cached in memory per category.

The questions are never fetched, so the game's content is fully available offline.

## 5. The game session

The game state is one immutable `GameSession` (Freezed): the category, the players with scores, the shuffled questions, the round limit, indices for the question and the current player, the round number, the votes cast so far and a pause flag. A Riverpod notifier holds it.

- `submitVote` appends 'A' or 'B'. If more players remain, it advances the player index. Otherwise the round is complete.
- `calculateRoundResult` counts votes, finds the majority, awards one point to each player on that side (none on a tie) and returns the totals for the results screen.
- `nextQuestion` advances the question, resets the player index and clears the votes.
- `endGame` records the high score, increments a games-played counter and stores the seen question ids.

A game ends when the chosen number of rounds is reached or the questions run out.

## 6. About the numbers on the results screen

The results screen shows, for each option, the number of players who voted for it. The counts are produced from the votes list of the current round, which was built by the players on that TV. The code contains no network client for votes, no database write for votes, no seeded or simulated percentage and no "other players chose" figure. The app reports what the room chose, and nothing else.

## 7. Ads

- **Initialisation.** The home screen starts the Unity SDK 800 ms after it opens, in the background. A ten-second timeout resolves the init future as a failure.
- **Interstitials.** Three call sites: after each round's results, before the final score, and when ending from the scoreboard. Each call site checks the remove-ads setting first. The service skips the ad if the SDK is not initialised or a cooldown (90 seconds since the last interstitial, 240 seconds since a rewarded ad) is active. Every outcome (skip, complete, fail) calls the continuation.
- **Rewarded.** Granted only on completion. A skip or a failure calls a failure callback and unlocks nothing. The unlock is held in memory and ends when the app closes.
- **Banner.** Shown on the vote screen, hidden while settings are loading or when ads are removed.
- **Default entitlement.** Ads-removed defaults to false in preferences. Ads are on until a purchase or a restore sets it.

## 8. Billing

A Kotlin `BillingManager` is a method channel plus an event channel. It starts the billing client with pending one-time purchases enabled, queries product details, launches the purchase flow, acknowledges unacknowledged purchases and lists existing purchases. Dart exposes a broadcast stream of events. The category screen and the settings screen subscribe just before they start a purchase and cancel the subscription after a result.

Entitlements are written to `shared_preferences` on purchase or when Restore Purchases is used. Nothing in the code restores them automatically at launch, and there is no server-side verification of purchase tokens.

## 9. Audio

A second Kotlin class wraps a `SoundPool` for short effects (WAV bytes are loaded from Flutter assets, written to a cache file natively and loaded) and a looping `MediaPlayer` for music (MP3). Dart calls it through a method channel. The app stops music when it goes to the background and restarts the saved track when it resumes. Sound effects are fire-and-forget, and load failures are ignored.

## 10. Analytics

A validating layer sits between the app and Firebase Analytics. It enforces the event-name, parameter-name and value limits and drops reserved names. Until Firebase initialises, a no-op backend is attached, so failures to initialise never reach the player. There are 55 named events, covering screens, controls, game lifecycle, ads and purchases. Input is summarised rather than logged per press: arrow presses are counted and flushed in a throttled way. Player names are explicitly excluded from events.

## 11. Manifest and build

- Leanback required, touchscreen not required, gamepad optional.
- TV banner declared; launcher and Leanback launcher categories on the main activity.
- Sensor landscape, single-top launch mode, immersive full-screen mode.
- Internet and billing permissions.
- Release builds: minification and resource shrinking on, ProGuard rules for Flutter, Play Billing and Unity, core library desugaring, Java 17.
- Release signing reads a local properties file when present and falls back to debug signing when it is missing.

## 12. Tests

38 test cases in 5 files: four analytics suites (validator rules, service behaviour, input tracking, screen tracking) and one smoke test. There are no tests for scoring, question loading or focus behaviour.

## 13. Limits worth stating

- The scoring and loading code is small but untested.
- Fonts come from the `google_fonts` package with no bundled font files, which by default fetches fonts at runtime and caches them, so a first launch with no connection depends on the package's fallback behaviour. This was not tested.
- Entitlements are trusted from local storage, and billing has no reconnect loop.
- One data file is shadowed and unused.
