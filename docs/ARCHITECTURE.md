# Architecture

Conceptual view of the Android TV app. Names are descriptive, not production identifiers.

![System overview](../assets/diagrams/system-overview.svg)

## Layers

| Layer | Responsibility |
|---|---|
| Input | Focusable widgets handle Select and Enter; Flutter's directional traversal moves between them; one root hook plays click sounds |
| Screens | Splash, home, how to play, settings, other apps, category, rounds, player count, player names, vote, results, scoreboard, winner |
| State | Riverpod notifiers for the game session, settings and categories; Freezed models |
| Data | Bundled JSON question files, preferences for settings and purchase flags, a Hive box for scores, counters and seen question ids |
| Services | Ads, billing bridge, audio bridge and analytics in Dart |
| Native | Kotlin billing manager, sound manager, IME listener and URL opener on the activity host |

## Startup

```mermaid
sequenceDiagram
    participant App as main()
    participant Sp as Splash
    participant Sv as Services (parallel)
    participant FB as Firebase
    participant Rt as App + router
    App->>Sp: run splash immediately
    Sp->>Sp: lock landscape, immersive mode
    par
        Sp->>Sv: Hive, preferences, audio, billing
        Sp->>FB: init, attach analytics backend
        Sp->>Sp: minimum 3 s display time
    end
    Sp->>Sv: load sound effects
    Sp->>Rt: crossfade into the app
    Rt->>Rt: home screen starts the ad SDK after 800 ms
```

If Firebase fails to initialise, a no-op analytics backend stays attached and the app continues.

## A game

```mermaid
stateDiagram-v2
    [*] --> Category
    Category --> Rounds: free or unlocked
    Category --> UnlockDialog: locked
    UnlockDialog --> Rounds: purchased, or rewarded ad completed
    UnlockDialog --> Category: cancelled, failed or skipped
    Rounds --> Players
    Players --> Names
    Names --> Vote
    Vote --> Vote: next player votes
    Vote --> Paused: pause control
    Paused --> Vote: resume
    Paused --> [*]: quit to menu
    Vote --> TimeUp: timer reaches zero
    TimeUp --> Vote: random option recorded
    Vote --> Results: last player voted
    Results --> Vote: next question
    Results --> Winner: rounds finished
    Vote --> Scoreboard: scoreboard control
    Scoreboard --> Vote: continue
    Scoreboard --> Winner: end game
    Winner --> Category: play again
    Winner --> [*]: main menu
```

## Voting and scoring

```mermaid
flowchart LR
    Q[Question: option A, option B] --> V1[Player 1 votes]
    V1 --> V2[Player 2 votes]
    V2 --> Vn[... last player votes]
    Vn --> C[Count A and B on the device]
    C --> T{Equal?}
    T -- yes --> N[No points]
    T -- no --> M[One point to each player on the larger side]
    M --> R[Results screen shows both counts]
    N --> R
```

There is no network step between the votes and the results screen. Counts are those of the players in the room.

## Question loading

```mermaid
flowchart TD
    L[loadQuestions category] --> K{In memory cache?}
    K -- no --> F[Read JSON from the first folder, else the second]
    F --> D[Drop repeated option pairs]
    D --> CA[Cache parsed list]
    K -- yes --> S
    CA --> S[Load seen ids for the category]
    S --> A[Filter out seen ids]
    A --> E{Anything left?}
    E -- no --> RS[Reset seen ids, use the full list]
    E -- yes --> SH
    RS --> SH[Shuffle with a secure random source]
    SH --> G[Game session]
```

Seen ids are written when a game is ended through the normal finish or the scoreboard's end action. Quitting from the pause menu discards the session without recording them.

## Keyboard closing on a TV

```mermaid
flowchart LR
    A[Native insets listener:<br/>keyboard hidden] --> X[One close method]
    B[View inset change in Dart] --> X
    C[App resumed] --> X
    X --> H[Hide keyboard, clear text client]
    H --> U[Drop primary focus]
    U --> P[Restore default name if blank]
    P --> R[Request focus on the player row]
```

## Platform channels

| Channel | Direction | Purpose |
|---|---|---|
| Billing methods | Dart to Kotlin | Initialise, query products, purchase, list purchases |
| Billing events | Kotlin to Dart | Connection changes and purchase updates |
| Audio methods | Dart to Kotlin | Initialise, load and play effects, play and stop music, volume and mute |
| IME | Kotlin to Dart | Keyboard hidden |
| Utilities | Dart to Kotlin | Open a URL with a view intent |

## Network use

- Ad network (banner, interstitial, rewarded) and Play Billing.
- Firebase Analytics.
- Runtime font download, by default, through the fonts package.
- No service stores or shares game data, questions or votes.
