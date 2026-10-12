# Kickoff 3v3

Real-time online 3v3 arcade football for phones. Up to six people on six separate devices, one footballer each.
The server is authoritative; there are no npm dependencies.

## Run it

```
node server.js          # needs Node 22+, serves on http://0.0.0.0:3000
```

- Same Wi-Fi: open `http://<your-computer-LAN-IP>:3000` on each phone.
- Over the internet: deploy this folder to any Node host that supports WebSockets (Render, Railway, Fly.io, a VPS).
  The host sets `PORT`; HTTPS gives you `wss://` automatically. Phones just open the URL.
- Solo/offline mode needs no server at all: open `public/index.html` from any static host (it only needs `game.js` beside it).
- Client hosted separately from the server? Put the server address in Settings (or open `/?server=wss://your-server`).
- The 3D engine (three.js r128) loads from cdnjs. To run with no internet, put `three.min.js` in `public/vendor/`.

## How to play

| Control | Action |
|---|---|
| Left thumb, anywhere on the left half | Floating joystick. Up always runs toward the goal you attack |
| Shoot (hold to charge, release) | Quick tap = soft shot, full charge = hard shot. Aim assist near the goal |
| Pass | Goes to the teammate in your stick direction (nearest one if the stick is idle). The ball is aimed at where they will be, and they run to meet it |
| Sprint (hold) | Faster, drains the blue bar |
| Tackle | A lunge at the ball carrier. Face to face you win the ball on contact. From behind or the side it fails and you stumble |

The game always shows in landscape: if the phone is held upright the screen is turned sideways for you (tilt the phone to match).

Kits: Sky (cyan and white stripes, purple socks) vs Fire (red, navy shorts).

Desktop test keys: WASD/arrows, Shift sprint, Space shoot, E pass, Q tackle.

Menu: Quick match (public matchmaking), Create room (4-letter code, host starts), Join with code, Play solo.
Empty slots and anyone who disconnects are played by bots. A dropped player can reconnect and gets their footballer back
(same browser); a new player can also take over a bot slot in a running match.

## Architecture

```
public/game.js    shared simulation (pitch, ball, possession, tackles, goals, clock, bot AI)
server.js         HTTP static files + minimal WebSocket + rooms/matchmaking + 60 Hz sim, 30 Hz snapshots
public/index.html UI, three.js scene, touch controls, solo mode, online netcode
```

- **Authority:** clients send only input (stick, sprint held, and counters for pass/shoot-release/tackle so a fast tap is never lost).
  The server owns ball, positions, goals, clock, and bots.
- **Client:** interpolates other players and the ball ~110 ms behind the newest snapshot. Your own footballer is predicted locally
  with the same `movePlayer` code, then reconciled against the server using the input sequence number the server acknowledges.
  While you dribble, the ball is attached to your predicted foot so it never trails you.
- **Solo mode** runs the very same `Game` class in the browser.
- Slow connections: the server skips snapshots to a client whose socket buffer is backed up instead of queueing them.

Messages (JSON): client `hello quick create join team start leave in ping`; server `welcome room start s pong err`.

Making an Android app (APK): see APK.md.

## Tuning

All gameplay numbers are in the `C` block at the top of `public/game.js` (speeds, stamina, pitch size, tackle cooldown, friction).
Server env vars: `PORT`, `MATCH_SECONDS` (default 180), `FULLTIME_HOLD` (seconds before returning to the lobby).
Graphics: Settings has Auto (adapts resolution to keep ~60 fps), Battery saver, Sharp.

## Tests

`npm test` runs: a WebSocket end-to-end test against the real server (rooms, teams, input, bot takeover, reconnect, full time,
matchmaking), the real client script in a stubbed DOM against the real server, and a latency/jitter simulation of the prediction.

## Known limits

- Single server process, rooms in memory. No accounts, no anti-cheat beyond server authority.
- Not yet measured on physical low-end Android hardware. The scene is ~130 draw calls with no shadow maps and adaptive resolution,
  but profile on your target device.
- Bots are simple (roles, pressing, shooting, passing, keeper dives), not tactical.
