# Product Overview

Would You Rather TV is a party game for an Android TV. A question with two options is shown on the screen, each player in the room takes the remote in turn and picks one, and the players who chose the more popular side score a point. It is built for several people sharing one device, with no phones or accounts involved.

All statements describe behaviour observed in the production code. No app was run to collect measurements.

## How a game flows

1. **Splash.** An animated splash runs for a minimum of three seconds while storage, audio, billing and analytics start in parallel.
2. **Home.** Three controls: Start Game, How To Play and Settings.
3. **Category.** A wrapped grid of thirteen category cards. Locked cards open an unlock dialog.
4. **Rounds.** The player picks how many questions the game lasts: 3, 5, 7, 10 or 15.
5. **Players.** A choice of 2 to 6 players, then a name screen where each name can be edited with the TV keyboard. Defaults are "Player 1", "Player 2" and so on.
6. **Vote.** The question and its two options (A and B) are shown with the round number, a timer bar, a pause control and a scoreboard control. The current player's name is displayed, and each player votes in turn.
7. **Results.** After the last player votes, a results screen shows both options, a vote count for each, which side held the majority and which players matched it.
8. **Scoreboard and finish.** A scoreboard can be opened mid-game. When the rounds end, a winner screen ranks the players with medals and offers Play Again or Main Menu.

## Scoring

- Each player votes A or B on the same question.
- If one side has more votes, every player on that side gets one point. If the votes are tied, nobody scores.
- The highest total wins. Players are ranked by score, and a tie for first is not announced specially: one player is named as the winner.

## Where the numbers come from

The vote counts on the results screen are the counts of the players who voted on that TV, and nothing else. There is no network call that sends a vote anywhere to be pooled with other households, and no percentage of "other people" is displayed. The product is a group game, not a poll.

One analytics event does record each vote side, round number and player slot (not the name) as a usage signal. It is not read back into the game.

## Categories

Thirteen categories in a fixed order. The first four are free; the other nine are paid or can be opened for the current session with a rewarded ad.

| Group | Categories |
|---|---|
| Free | Funny and ridiculous, superpowers and fantasy, food battles, life choices |
| Paid | Gross and awkward, technology and future, adventure and travel, kids and family, movies and entertainment, money and luxury, relationships and social life, survival and extreme, random chaos (party) |

Every category file has 200 entries. The app removes repeated option pairs on load, so the playable count per category is slightly lower in some of them.

## Timer

The per-vote timer defaults to 60 seconds and can be cycled in Settings between 20 and 120 seconds in 20-second steps. The bar changes colour at 20 and 10 seconds left, and a tick sounds over the last five. At zero, a dialog names the player and a random option is chosen for them.

## Settings

- Sound on or off.
- Music: off, or one of five tracks, cycled with Select.
- Turn timer length.
- Remove Ads (one-time purchase), and Restore Purchases.
- Links to other apps by the same publisher, the Play listing for rating, terms and a privacy policy.

## Monetisation

- A banner on the vote screen.
- Interstitial ads between rounds, before the final score and when ending a game from the scoreboard, subject to a cooldown.
- A rewarded ad that opens a locked category until the app is closed.
- A one-time purchase that removes ads, and nine one-time purchases for categories.

## Audio

Seven short sound effects (select, navigate, vote, round end, win, tick, pause) and five looping music tracks. Music is off by default.

## Platform

Android TV only. The manifest requires Leanback, does not require a touchscreen, declares the TV launcher banner and locks to sensor landscape.
