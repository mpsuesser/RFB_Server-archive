# RUN Football Server

Dedicated simulation for [RUN Football](https://github.com/mpsuesser/RFB_Client-archive). The client renders and collects input. This project is the match: clock, downs, ball, units, and the network.

Unity **2021.3.12f1**, Built-in RP, Mono, .NET Standard 2.0. One scene: `Assets/Scenes/GameServer.unity`. Product name `FootballGameServer`. Protocol version `0.4.0` (`Constants.VERSION`) — it must match the client or the handshake dies.

If you can keep a small authoritative sim honest at 30 Hz and you can read a TCP-then-UDP handshake, you can work here.

## How the pieces fit

```
Assets/Scripts/
  NetworkManager.cs      Play Mode bootstrap; starts the listen socket
  Server.cs              TcpListener + UdpClient, client slots
  Client.cs              Per-connection TCP/UDP state
  Packet.cs              Length-prefixed binary messages
  ServerHandle.cs        Incoming packet dispatch
  ServerSend.cs          Outgoing replication
  ThreadManager.cs       Socket callbacks → main thread
  Lobby.cs / PlayerInfo.cs
  GameMaster.cs          Match flow (coin flip, snap, quarters, score)
  GameState.cs           Possession, down, ball spot, clock
  Stats.cs / AutoPrevention.cs / CoroutineRunner.cs
  Constants.cs           Every gameplay number worth tuning
  ServerEntities/        Unit, Ball, CatchTrigger, position subclasses
  UnitCommands/          Move, follow, tackle, stiff
  AStar/                 Grid pathfinding
  NavMeshComponents/     Baked surfaces for the field
```

`NetworkManager.Start` is the entry: vsync off, 30 FPS, then `Server.Start(12, Constants.SERVER_LISTEN_PORT)`.

`GameMaster` owns the sport. `Server` owns the sockets. `Constants` is the design sheet. Change yards and cooldowns there before you hunt through entities.

## Authority

The server is authoritative.

Clients send *intents* (move here, throw there, tackle that unit). This process runs the units, the ball, catch windows, tackles, the play clock, and first downs. `ServerSend` tells every client what the world is.

If a rule can be cheated by trusting the client, it does not belong on the client.

## Network

Not Mirror, not Netcode, not Photon. Custom TCP + UDP in the Tom Weiland layout (`Packet`, `ServerHandle`, `ServerSend`).

- Listen: **TCP and UDP on port 7777**, `IPAddress.Any`, max **12** clients.
- `Packages/manifest.json` still lists Riptide as a git package. Sockets do not use it. Opening the project still needs `git` on PATH because UPM will fetch it.
- No command-line args. **Play Mode is the server.** There is no headless binary in this repo.

Handshake, in order:

1. Client TCP connect
2. Server `WelcomeToServer`
3. Client UDP handshake (username + version string)
4. Client `RequestToJoinLobby`
5. Server `WelcomeToLobby` (or `versionOutOfDate` / game-already-in-progress)

There is a text copy of that sequence in `Assets/Scripts/handshake.txt`.

Lobby: slots 1–6 are team 1, 7–12 team 2. `PLAYERS_TO_START` is `1`. **Start is currently gated to username `Arold`.** Anyone else who hits Go is ignored. That is a host key, not a bug you need to rediscover.

A friend on another machine needs **both** TCP 7777 and UDP 7777. There is no relay and no hole punch. If only TCP is forwarded, the handshake dies on UDP.

## Simulation (30 Hz)

`Constants.TICKS_PER_SEC = 30`. Gameplay numbers, all in `Constants.cs`:

| | |
| --- | --- |
| Quarter | 300 seconds |
| Move speed RB / WR / TE / QB / LM | 6 / 5.5 / 5 / 4 / 4 |
| Throw range QB / RB / TE / WR | 60 / 20 / 15 / 10 |
| Charge | 6.5 speed, 5s duration, 20s cooldown |
| Juke | 1.5s, 10s cooldown |
| Tackle | 2 unit min distance, 3s cooldown, strength 1.5 |
| Stiff | 3.5 min distance, 10s cooldown |
| Ball | hike speed 5 (range 10), throw speed 10 |
| Catch window | 3.5 from origin, 4.5 from destination |
| Click cap | 5 clicks per 1s window (`AutoPrevention`) |

Position classes under `ServerEntities/` specialize `Unit`. Orders become `UnitCommands` (move to a spot, move toward a unit, follow, tackle, stiff). Pathing is A* on `AStar/Grid` plus NavMesh on the field.

`GameMaster.SignalGameStart` waits five seconds (client scene load), coin-flips possession (solo test always gives red), spots the ball at ±30, and builds the snap formation.

## Where to edit

| You want to… | Start here |
| --- | --- |
| Change a cooldown, speed, throw range | `Constants.cs` |
| Change down/clock/score rules | `GameMaster.cs`, `GameState.cs` |
| Change how a position behaves | `ServerEntities/<Position>.cs`, `Unit.cs` |
| Add an order | `UnitCommands/` + a `ServerHandle` case + a `ServerSend` |
| Change who can start, or lobby size | `Lobby.cs`, `ServerHandle` (the `Arold` check), `NetworkManager` max players |
| Change listen port | `Constants.SERVER_LISTEN_PORT` — and the client's Inspector, see below |

## Landmines

- The **client** Preload scene (`ClientManager`) ships hardcoded to `162.83.244.27:26950`. This server listens on **7777**. They will not connect until you set the client's ip/port (there is no IP field in the UI). Client `Constants.SERVER_LISTEN_PORT` is 26950 and is overridden by that Inspector field.
- Windows `Library/` is committed. There is no `.gitignore`. First open on Linux/macOS should delete `Library/`, `Logs/`, `obj/` and let the editor reimport.
- No Git LFS. No GitHub Releases. Old Windows builds lived on a desktop path and are not in the repo.
- Client package `com.unity.postprocessing` `2.0.3-preview` does not compile on 2021.3 (`EditorSceneManager.IsGameObjectInScene`). Bump that client package to `3.2.2`.

## Run it from scratch

You need Unity Hub, a Personal license, and editor **2021.3.12f1** (Linux editor exists for this version).

1. Clone this repo and [the client](https://github.com/mpsuesser/RFB_Client-archive).
2. Delete `Library/`, `Logs/`, and `obj/` in both clones if they came along for the ride.
3. Open **this** project in 2021.3.12f1. Wait for UPM (git must be installed).
4. Open `Assets/Scenes/GameServer`. Press Play. Console should print `Starting server...`, `Initialized packets.`, `Server started on 7777.` Leave it playing.
5. Open the **client** project in a **second** editor. In Preload, select `ClientManager`, set `ip` to `127.0.0.1` (or the server's LAN/public IP) and `port` to `7777`.
6. If the client fails to compile on `PostProcessManager` / `IsGameObjectInScene`, set `com.unity.postprocessing` to `3.2.2` in the client's `Packages/manifest.json`.
7. Play the client. Main menu: username `Arold`. Join lobby, take a slot, **Go!**.
8. Another PC: same client, same ip/port pointing at the host, **TCP 7777 and UDP 7777** both open. Same version string `0.4.0`.

The player-facing half of the docs lives on the client: [RFB_Client-archive](https://github.com/mpsuesser/RFB_Client-archive).
