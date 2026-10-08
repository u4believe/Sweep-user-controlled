# Sweep UCW — MVP design & reference code

A non-custodial version of Sweep built on **Circle user-controlled wallets (UCW)**.
Users hold their own keys (Circle PIN), see two balances — **Wallet** and **Platform
Account** (their Circle Gateway unified balance) — and every money movement is signed
by the user. Sweep only builds the transactions, sponsors gas, relays Gateway mints
and charges a transparent fee.

> **Status:** design + reference code. Nothing here is wired into the Sweep repo yet,
> and parts of it depend on behaviour that must be confirmed on testnet first — see
> [`docs/09-verify-first.md`](docs/09-verify-first.md). Every such spot in the code is
> marked `// VERIFY:`.

## The product in one table

| Rule | Where it lives |
|---|---|
| Two balances: **Wallet** (USDC per chain, where deposits land) and **Platform Account** (Gateway unified balance) | `api/src/lib/balances.ts`, `web/src/screens/Home.tsx` |
| On-ramp USDC lands in the **Wallet** | `api/src/lib/ramps/bridge.ts` (USD), `yellowcard.ts` (NGN) |
| Wallet can only **send to an external address** or **move to the Platform Account** | `api/src/routes/wallet.ts` |
| Moving Wallet → Platform Account is **free** (gas sponsored), **PIN only**, with a minimum amount and daily cap | `routes/wallet.ts`, `lib/limits.ts` |
| Everything else needs **Passkey + PIN** | `api/src/lib/stepUp.ts`, `web/src/lib/approve.ts` |
| User → user payments only **from the Platform Account**, always **delivered on Arc**, into the recipient's **Wallet** | `api/src/routes/platform.ts` |
| Fees = **actual Circle cost × 1.6** for user → user and Platform → external | `api/src/lib/fees.ts` |
| Off-ramp (Naira) and on-ramp through **Yellow Card** and **Bridge**, different fee model | `api/src/lib/ramps/*`, `routes/ramps.ts` |
| **P2P marketplace** (buy/sell USDC for Naira between users) with an on-chain escrow on Arc | `contracts/P2PEscrow.sol`, `routes/p2p.ts` |
| **USDC → EURC** — research only (StableFX), not built | `docs/07-eurc-conversion.md` |
| No recurring payments, no subscriptions | — (removed) |

## Folder map

```
sweep-ucw-mvp/
├── README.md                     ← you are here
├── .env.example                  ← every setting the API needs
├── package.json                  ← dependencies to add to Sweep (api + web + contracts)
├── docs/
│   ├── 01-product-spec.md        ← features, rules, screens, limits
│   ├── 02-architecture.md        ← components, custody, sequence diagrams for every flow
│   ├── 03-fees.md                ← fee formulas with worked examples
│   ├── 04-security-passkey-pin.md← Passkey + PIN step-up design
│   ├── 05-ramps.md               ← Bridge (USD) + Yellow Card (NGN)
│   ├── 06-p2p-marketplace.md     ← Naira marketplace + escrow
│   ├── 07-eurc-conversion.md     ← StableFX research notes
│   ├── 08-reuse-map.md           ← what to copy from the current Sweep code, file by file
│   ├── 09-verify-first.md        ← testnet spikes to run BEFORE building
│   └── 10-build-plan.md          ← milestones
├── db/
│   ├── schema.ts                 ← Drizzle tables (drop into lib/db/src/schema)
│   └── migrations/0001_ucw.sql   ← same, as SQL
├── contracts/
│   └── P2PEscrow.sol             ← escrow for the Naira marketplace (Arc)
├── api/src/
│   ├── config.ts                 ← env + chain table (extends Sweep's gatewayConfig.ts)
│   ├── lib/
│   │   ├── circle/userWallets.ts ← UCW: users, PIN, wallets, challenges
│   │   ├── circle/relayer.ts     ← Sweep's developer-controlled relayer (gatewayMint, P2P escrow calls, float)
│   │   ├── gateway.ts            ← burn intents, Gateway API, unified balance
│   │   ├── balances.ts           ← Wallet + Platform Account balances
│   │   ├── fees.ts               ← the 60% markup fee engine
│   │   ├── platformSend.ts       ← quote → passkey → PIN signature → Gateway (every Platform Account send)
│   │   ├── limits.ts             ← minimums, daily caps
│   │   ├── stepUp.ts             ← Passkey step-up tokens (reuses Sweep's lib/webauthn.ts)
│   │   ├── ramps/types.ts        ← provider interface
│   │   ├── ramps/bridge.ts       ← USD virtual accounts
│   │   └── ramps/yellowcard.ts   ← NGN payouts & collections
│   ├── routes/
│   │   ├── onboarding.ts         ← create Circle user, PIN, wallets
│   │   ├── wallet.ts             ← Wallet balance actions
│   │   ├── platform.ts           ← Platform Account: send to user / external (+ /api/stepup)
│   │   ├── ramps.ts              ← KYC, USD account, Naira cash-out
│   │   ├── p2p.ts                ← marketplace
│   │   └── webhooks.ts           ← Circle, Bridge, Yellow Card
│   └── workers/deliveryWorker.ts ← mints Gateway transfers, Naira payouts, escrow open/refund
└── web/src/
    ├── lib/circleSdk.ts          ← W3SSdk wrapper (PIN challenges)
    ├── lib/approve.ts            ← Passkey + PIN, or PIN only
    ├── hooks/useBalances.ts
    └── screens/                  ← Home, MoveToPlatform, SendToUser, SendExternal, CashOutNaira, P2PMarket
```

## Code conventions

- `api/src/**` mirrors Sweep's `artifacts/api-server/src/**`: imports are written as if the
  files already live there (`./auth.js`, `./webauthn.js` are Sweep's existing modules).
- `db/schema.ts` goes to `lib/db/src/schema/ucw.ts` and is re-exported from `@workspace/db`.
- `web/src/**` mirrors `artifacts/usdc-send/src/**` (`@/…` aliases, Sweep's UI kit).
- `// VERIFY:` marks behaviour to confirm on testnet (docs/09).

## Wiring into Sweep's app.ts

```ts
import onboarding from "./routes/onboarding.js";
import wallet from "./routes/wallet.js";
import platform, { stepUpRouter } from "./routes/platform.js";
import ramps from "./routes/ramps.js";
import p2p from "./routes/p2p.js";
import webhooks from "./routes/webhooks.js";
import { startDeliveryWorker } from "./workers/deliveryWorker.js";

app.use("/api/webhooks", webhooks);          // BEFORE express.json() — needs the raw body
app.use(express.json());
app.use("/api/ucw", onboarding);
app.use("/api/wallet", wallet);
app.use("/api/platform", platform);
app.use("/api/stepup", stepUpRouter);
app.use("/api/ramps", ramps);
app.use("/api/p2p", p2p);
startDeliveryWorker();
```

## What has been checked

- `api/src/**` typechecks (`tsc --strict`) against the real `@circle-fin/user-controlled-wallets`
  and `@circle-fin/developer-controlled-wallets` type definitions, with Sweep's `auth`/`webauthn`
  modules stubbed.
- `web/src/**` typechecks against Sweep's actual UI kit (`components/sweep/*`), `@circle-fin/w3s-pw-web-sdk`
  and `@simplewebauthn/browser`.
- `contracts/P2PEscrow.sol` compiles with solc 0.8.24 + OpenZeppelin 5 (`evmVersion: cancun`).
- `lib/fees.ts` reproduces the worked examples in `docs/03-fees.md` (B $0.0168, C $0.0456, D $0.048).
- **Not** run against Circle, Bridge or Yellow Card — that's what `docs/09-verify-first.md` is for.

## How to read this

1. Start with `docs/01-product-spec.md`, then `docs/02-architecture.md`.
2. Run the spikes in `docs/09-verify-first.md` — two of them decide details of the code.
3. Use `docs/08-reuse-map.md` to move pieces into the Sweep repo; most of the plumbing
   (auth, passkeys, Gateway types, chain config, workers, UI kit) already exists there.

## Stack (same as Sweep)

Node 20+/Express 5 · TypeScript · Drizzle/Postgres · React 19 + Vite + Tailwind 4 ·
`@circle-fin/user-controlled-wallets` (server) · `@circle-fin/w3s-pw-web-sdk` (web) ·
`@circle-fin/developer-controlled-wallets` (relayer, already used by Sweep) ·
`@simplewebauthn/server` + `/browser` (already used by Sweep) · `viem` (ABI encoding) ·
Solidity 0.8.24 + OpenZeppelin 5 (escrow).
# Sweep-user-controlled
