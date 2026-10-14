# AGENTS.md — Mobius / Corsac Diameter (local fork)

**Tree:** `ethiopia-working-dir/worktrees/diameter/corsac-diameter`  
Upstream: [mobius-software-ltd/corsac-diameter](https://github.com/mobius-software-ltd/corsac-diameter)

Local fork of **Mobius Diameter 10** used as the **connection / codec** library behind micro-jainslee `vendor-ras/ra-diameter`. Do not treat this tree as an application.

## JDK

**Java 25 only** (mise `zulu-25`). Default Maven profile is `java25`. Do not re-enable Java 8/11 as the host baseline.

```bash
mise exec -- mvn -Pjava25 -DskipTests install
```

Log4j2 **2.24.3** — match `jainslee-core`.

## Layering (non-negotiable)

| Layer | Owns | Does not own |
|-------|------|----------------|
| **Corsac stack** (`DiameterStack` / `NetworkManager` / `DiameterLink`) | TCP + SCTP sockets, CER/CEA, DWR/DWA, DPR/DPA, encode/decode | Cx / S6a / Gx / Gy business, HSS SQN, charging |
| **`diameter-ra`** | 3-port RA, peer truth (`isPeerReady()` = link `OPEN`), fire `DiameterRequestEvent` / `DiameterAnswerEvent` | Application answers |
| **SBB** (`DiameterCxSbb`, `DiameterGatewaySbb`, …) | Application logic, CMP fields, sealed `SendDiameterRequest` / `SendDiameterAnswer` | Sockets |
| **ProfileFacility** | Provisioned identity (`EpsSubscriber`, `ApnProfile`, IMS subscriber) | Live sessions, money counters |
| **CMP + PG dual-write** | In-flight session / bearer checkpoint | Ki / OPc |

`NetworkListener.onMessage` in the RA is a **bridge to SLEE**, not a place to implement AIR/ULR/MAR.

Corsac `DiameterSessionStorage` is **not** the micro-jainslee failover SoT. SLEE CMP + Postgres are.

## Peer truth

`DiameterLink.isUp()` / `PeerStateEnum.OPEN` = CER/CEA completed. LISTEN / `isStarted()` / `isConnected()` is not traffic-ready.

## Authorship

Commits that land in nhanth87 trees: nhanth87 / Tran Nhan only. Do not push this fork to Digicom-ET or public remotes unless asked.

## Resource hygiene (workplace-wide rule, 2026-08-23)

- When done (tests/smoke/dev): stop everything you started — `docker compose down` (keep volumes), kill dev servers/JVMs you spawned. Never leave them running "for later"; RAM is shared across ALL worktrees on this machine.
- Before ending a session verify: `docker ps` shows nothing from this tree; no stray `java`/`node` processes left (`ps -eo pid,rss,args --sort=-rss | head`).
- Long-lived services (EPC / FreeSWITCH / PG / app servers) run only while their session needs them. If the owner asks to keep one up, note which and why in the session handoff.
- DB/app port binds use loopback (`127.0.0.1:`) unless explicitly public; never expose default credentials beyond lab.
