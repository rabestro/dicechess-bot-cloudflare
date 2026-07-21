# Dice Chess bot — Cloudflare Workers (engine-powered)

A Dice Chess webhook bot that runs the **real game engine** —
[`dicechess-engine-scala`](https://github.com/rabestro/dicechess-engine-scala) — on **Cloudflare
Workers**, through the engine's **Scala.js** build. It plays the engine's **aggressive** king-hunt
search behind the exported **opening book** (`aggressive-book`). No container, no cold start: the
same Scala engine the browser client runs, on the edge, woken only when it's this bot's turn.

This is the fourth runtime for one engine — alongside GraalVM-native on Azure, JVM in a container,
and the browser. The Worker itself is ~40 lines of TypeScript glue; all the chess is the engine.

## Licensing

**AGPL-3.0**, because it links the AGPL engine. If you want a **closed-source** bot, the legal
moves are already on the wire — use a transport-only MIT starter
([TypeScript](https://github.com/rabestro/dicechess-bot-typescript),
[Python](https://github.com/rabestro/dicechess-bot-python)) and no engine linkage is ever required.

## Does the engine fit the free plan?

Cloudflare's free plan allows **~10 ms CPU per request**. Measured on the shipped engine build
(Node V8, the same family as `workerd`), per `getBestMove` call:

| | typical (p50) | worst seen |
| --- | --- | --- |
| `aggressive-book` | ~0.4 ms | ~3.4 ms |

Comfortable at the median with a wide margin. The tail is honest, though: a turn with the maximum
number of legal micro-moves (three high-mobility dice in a dense position) can approach or exceed
10 ms. If a single turn's request is killed on the CPU limit, **that one turn is forfeited on the
clock — the game continues**; it is not a crash. For a demo bot that trade-off is fine. `monte-carlo`
does **not** fit (hundreds of ms) — that algorithm belongs on a JVM/native platform with real time.

## Layout

| Path | Role |
| --- | --- |
| `src/strategy.ts` | Registers `aggressive-book` and turns a DFEN into UCI moves via the engine. **Swap the algorithm here.** |
| `src/webhook.ts` | Pure delivery logic: WebCrypto HMAC verify (±5 min replay window), handshake echo. No engine — directly unit-tested. |
| `src/index.ts` | The Workers `fetch` handler — reads the signing secret from the environment, relays to `handleDelivery`. |
| `opening_book.json` | The exported opening book, bundled into the Worker. |

## Local development

Requires Node 24+ and a GitHub token with `read:packages` for the engine
(`export NODE_AUTH_TOKEN=$(gh auth token)` before installing — see `.npmrc`).

```bash
npm install
npm test          # HMAC vector, handshake, 401/400 paths, and a real engine-legal move
npm run typecheck
npm run dev       # wrangler dev — runs the Worker locally in workerd
```

## Deploy to Cloudflare Workers

```bash
npm install                         # needs NODE_AUTH_TOKEN (read:packages)
npx wrangler login                  # one-time, opens the browser
npx wrangler deploy                 # publishes to https://dicechess-bot-cloudflare.<subdomain>.workers.dev
```

Then wire it to the platform (any HTTP client; `curl` shown):

```bash
BASE=https://play-api.jc.id.lv
URL=https://dicechess-bot-cloudflare.<subdomain>.workers.dev

# 1. Claim a durable identity. Token shown ONCE.
curl -X POST "$BASE/bot/register" -H "Content-Type: application/json" \
  -d '{"team":"cloudflare","name":"scala-aggressive-book"}'

# 2. Register the webhook (the deployed Worker must already answer — ownership handshake).
#    The response carries the signing secret, shown ONCE.
curl -X POST "$BASE/bot/webhook" -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" -d "{\"url\":\"$URL\"}"

# 3. Give the Worker its secret (never commit it):
npx wrangler secret put DICECHESS_WEBHOOK_SECRET   # paste the secret from step 2

# 4. Join the rating ladder — passive from here; watch /bots/cloudflare/scala-aggressive-book converge.
curl -X POST "$BASE/bot/ladder/join" -H "Authorization: Bearer <token>"
```

The `workers.dev` URL is HTTPS and public, which is all the webhook registration requires — no
custom domain needed. Before step 4 you can [play against it
yourself](https://jc.id.lv/dicechess-play-api/play-your-bot/) from the lobby to confirm it plays a
legal game. Full platform reference: <https://jc.id.lv/dicechess-play-api/>.
