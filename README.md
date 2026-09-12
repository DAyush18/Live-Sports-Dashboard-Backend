# Live Scoreboard — Backend

A Node.js backend that serves match data and broadcasts live commentary over WebSocket, backed by PostgreSQL.

## Tech stack

- **Node.js** + **Express** — HTTP API
- **ws** — WebSocket server for live commentary/match updates
- **PostgreSQL** (via `pg`) — persistent storage for matches and commentary
- **Zod** — request validation
- **Arcjet** — bot detection and rate limiting on HTTP routes and WebSocket upgrades

## Getting started

### Prerequisites

- Node.js 18+
- A PostgreSQL database (local or hosted)
- An [Arcjet](https://arcjet.com) account and API key (free tier available)

### Installation

```bash
git clone <your-repo-url>
cd <repo-folder>
npm install
```

### Environment variables

Copy `.env.example` to `.env` and fill in your own values:

```dotenv
PORT=8000

# PostgreSQL connection string
DATABASE_URL=postgres://user:password@localhost:5432/scoreboard

# Arcjet
ARCJET_KEY=your_arcjet_key
# Required for local development — without this, Arcjet can't fingerprint
# requests on localhost and will throw on every request/WS upgrade.
ARCJET_ENV=development

# Allowed origin(s) for CORS — should match your frontend's dev/prod URL
CORS_ORIGIN=http://localhost:5173
```

### Running the server

```bash
npm run dev     # starts with auto-reload (nodemon)
npm start       # starts in production mode
```

On success you should see:

```
Server is running on http://localhost:8000
WebSocket Server is running on ws://localhost:8000/ws
```

## API

> Adjust paths/fields below to match your actual route definitions if they differ.

### `GET /matches`

Returns a list of matches.

Query params:
| Param | Type | Description |
|-------|------|-------------|
| `limit` | number (1–100) | Max number of matches to return |

### `GET /matches/:id/commentary`

Returns commentary entries for a given match, oldest first.

Query params:
| Param | Type | Description |
|-------|------|-------------|
| `limit` | number (1–100) | Max number of entries to return |

### `POST /matches`

Creates a new match. Body:

```json
{
  "sport": "football",
  "homeTeam": "Manchester United",
  "awayTeam": "Chelsea",
  "startTime": "2026-09-12T19:30:00.000Z",
  "endTime": "2026-09-12T21:30:00.000Z",
  "homeScore": 0,
  "awayScore": 0
}
```

`endTime` must be chronologically after `startTime`.

### `PATCH /matches/:id/score`

Updates a match's score. Body:

```json
{ "homeScore": 1, "awayScore": 0 }
```

### `POST /matches/:id/commentary`

Adds a commentary entry and broadcasts it to subscribed WebSocket clients. Body:

```json
{
  "minute": 27,
  "sequence": 1,
  "period": "2H",
  "eventType": "yellow_card",
  "actor": "Casemiro",
  "team": "Manchester United",
  "message": "Yellow card shown to Casemiro for a late challenge.",
  "metadata": {},
  "tags": []
}
```

Only `minute` and `message` are required.

## WebSocket protocol

Connect to `ws://localhost:8000/ws`.

### Client → server messages

| Type | Payload | Description |
|------|---------|--------------|
| `subscribe` | `{ "type": "subscribe", "matchId": 1 }` | Start receiving commentary for a match |
| `unsubscribe` | `{ "type": "unsubscribe", "matchId": 1 }` | Stop receiving commentary for a match |

### Server → client messages

| Type | Description |
|------|--------------|
| `welcome` | Sent once, right after the connection opens |
| `subscribed` / `unsubscribed` | Acknowledges a subscribe/unsubscribe request |
| `match_created` | Broadcast to all clients when a new match is created |
| `commentary` | Broadcast to clients subscribed to that match when a new commentary entry is added |
| `error` | Sent when an incoming message can't be parsed or is invalid |

The server pings every 30s and terminates connections that don't respond, to clear out dead sockets.

## Notes on Arcjet in development

Arcjet fingerprints requests using the client's IP address. On `localhost`, there's no real public IP to use, so **`ARCJET_ENV=development` must be set** or every request (including WebSocket upgrades) will fail fingerprinting and be rejected. In production, this isn't needed since real client IPs are available.

## Project structure

```
src/
├── index.js          # App entry point — starts HTTP + WebSocket servers
├── arcjet.js          # Arcjet client configuration (HTTP + WS)
├── ws.js              # WebSocket server, subscriptions, broadcast helpers
├── routes/            # Express route handlers
├── schemas/           # Zod validation schemas
└── db/                # Database connection and queries
```

> Update this tree to reflect your actual folder layout.

## License

Add your license here (MIT, ISC, etc).
