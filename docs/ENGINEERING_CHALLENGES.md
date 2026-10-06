# Engineering Challenges

Seven problems that stand out in the Android TV app. Each is drawn from behaviour visible in the code. Outcomes describe what the code does; no metrics are claimed.

---

## 1. A group game driven by one remote

**Problem**
A remote has no pointer, and several people share it. Menus, a wrapped grid of categories, a number picker, a text entry screen and a two-option vote all need to work with four arrows and Select, and the highlighted control has to be unmistakable from across a room.

**Context**
Flutter provides directional focus traversal, which suits layouts that are centred columns, wrapped grids and side-by-side cards. All of this app's screens fit that description.

**Approach**
- Every interactive element is a small stateful `Focus` widget. It tracks whether it is focused, animates scale (about 1.08), border and glow over 200 ms, and handles Select and Enter in its own key handler.
- The first sensible control on each screen has `autofocus`. For example, the first option card on the vote screen and the first category card.
- Screens that scroll call `Scrollable.ensureVisible` on focus so the highlighted card is never off screen.
- A single `HardwareKeyboard` handler at the app root plays a navigation click for arrows and a select click for Select and Enter. It returns `false`, so it never swallows a key.

**Trade-offs**
Movement depends on widget geometry rather than a written-down order, so a layout change can change where an arrow goes. No automated test covers focus movement.

**Outcome**
Every screen is operated with arrows and Select, with a visible highlight and a click for each move.

---

## 2. Dialogs and overlays that hold focus

**Problem**
A dialog that does not trap focus lets the arrows reach the screen underneath, and on a TV remote a held key repeats. The code comments record that in release builds a repeated Back could dismiss a dialog the moment it opened.

**Context**
Three layers appear over the vote and category screens: an unlock dialog, a "time's up" dialog and the pause overlay.

**Approach**
- The pause overlay and dialogs are focus scopes with autofocus on the safest default (Resume, Buy, or the single OK button).
- The unlock dialog also wraps itself in a pop handler so Back closes it cleanly.
- The "time's up" dialog cannot be dismissed from outside and has one focused button.
- A helper mixin was written to ignore Back for 300 ms after a dialog opens and to consume every key inside it.

**Trade-offs**
The mixin's key handler is never attached to a widget, so the 300 ms guard does not run; the unlock dialog closes on the first Back. The pause overlay is a stack child rather than a route, so Back there is not intercepted: the vote screen allows the pop and logs the game as abandoned.

**Outcome**
Dialogs open on a focused default and hold the arrows. The key-repeat protection exists as code but is not active, which is the part of this area that still needs finishing.

---

## 3. Typing a name when the keyboard is another activity

**Problem**
On Android TV the on-screen keyboard is a separate activity. When the user presses Back to close it, Flutter may receive no focus change. The text field can keep a blinking cursor, the remote appears dead and the player cannot reach Start.

**Context**
Players need to type their own names, and the first-run experience of the party game depends on it.

**Approach**
- A native listener on the window's insets calls a method channel into Dart when the keyboard goes from visible to hidden.
- Dart also watches view-inset changes and the app resuming, and routes all three signals into one close method.
- The close method hides the keyboard, clears the text-input client, drops primary focus, restores the default name if the field is blank and requests focus on the player's row on the next frame.
- When a field opens, the default name is cleared so the player can type fresh, and Select on the row opens the keyboard.

**Trade-offs**
Three redundant signals means the close method can run more than once, so it is written to be safe to repeat. It depends on the behaviour of specific TV firmware keyboards, and no test covers it.

**Outcome**
However the keyboard is closed, the player lands on a focused row with a valid name.

---

## 4. Honest results in a game that is usually remembered for statistics

**Problem**
In many "would you rather" apps the payoff is a percentage of other people who agreed. That implies a backend, moderation and abuse handling, and it would be misleading to fake it.

**Context**
This product is a room-sized party game. The point of each round is the discussion between the people holding the remote.

**Approach**
- Votes are stored in the session as 'A' or 'B' per player and counted locally.
- The results screen shows a vote count for each option, the majority and the players who matched it. It does not show percentages of any wider audience.
- A tie awards no points, which is stated on screen.
- A vote-side event is logged to analytics for usage understanding. The game does not read it back.

**Trade-offs**
There is no sense of "the world's answer", and there is no content feedback loop from players. In return there is no server to run, no latency and no content that depends on a connection.

**Outcome**
What a player sees after voting is exactly what the room chose. The results screen is always consistent with the scores.

---

## 5. Fresh questions without a server

**Problem**
Families play the same category repeatedly. A random pick from a fixed list will repeat questions, and a list that is too short is noticed.

**Context**
All questions ship in the app bundle. There is no account to remember what a household has seen.

**Approach**
- Each category is a JSON file parsed once and cached. Repeated option pairs are removed at load.
- Each new game takes the list minus the ids already seen in finished games for that category, and shuffles it with a secure random source.
- When nothing is left, the record for that category resets and the whole list is used again.
- Seen ids are stored in a local Hive box per category.

**Trade-offs**
Seen ids are recorded when a game ends normally or from the scoreboard. A game abandoned through the pause menu records nothing, so those questions can come back. One category has a stale second data file that is never read.

**Outcome**
Within a category, a household sees each question once before any repeat, as long as games are finished.

---

## 6. Ads and purchases on a screen with no touch

**Problem**
A full-screen ad that never calls back would trap a TV on a loading state, and a purchase flow cannot be completed unless the right screen is listening for its result.

**Context**
Interstitials sit between rounds, before the winner and when a game is ended from the scoreboard. Unlocks come from a purchase or a rewarded ad.

**Approach**
- Interstitials are skipped when the SDK is not ready or a cooldown is running, and every outcome (complete, skip, fail, not ready) calls the continuation.
- Rewarded ads grant only on completion; a skip or failure calls a failure callback and unlocks nothing. The unlock is held in memory for the session.
- Every ad call site checks the remove-ads setting before asking for an ad. The banner is hidden while settings load or when ads are removed.
- A Kotlin manager owns the billing client. Dart screens subscribe to a broadcast event stream just before starting a purchase and cancel after the result.
- The ad SDK initialises 800 ms after the home screen opens and gives up after ten seconds.

**Trade-offs**
There is no safety timeout around the show call itself, only around SDK initialisation. Entitlements are saved to local preferences and are restored only through the Restore Purchases control, not automatically at launch. The billing manager reports a dropped connection but does not reconnect.

**Outcome**
Ad failures never block the game, and a rewarded unlock is granted only for a completed ad.

---

## 7. Keeping analytics valid and private

**Problem**
Firebase Analytics silently drops events that break its naming and size limits, and a party game collects personal text (player names) that must not be sent.

**Context**
Controls, screens, ad outcomes and purchase steps are all instrumented, and many events fire in bursts on a remote.

**Approach**
- All events pass through one validator that enforces name and value length limits, a parameter cap and reserved-name rules, drops blank values and never throws.
- A no-op backend is attached until Firebase has initialised, and stays if initialisation fails.
- D-pad activity is recorded as throttled counts rather than per key press.
- Event helpers take identifiers and counts only. A comment in the code forbids free text, and the player-name screen reports only which slot was opened.

**Trade-offs**
The validator is well tested, but nothing tests that call sites obey the no-free-text rule; it relies on the helper signatures.

**Outcome**
Events that reach Firebase are valid by construction, and names stay on the device.
