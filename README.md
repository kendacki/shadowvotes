<div align="center" style="font-family: 'Times New Roman', Times, serif;">

<img src="public/shadowvote-emblem.svg" alt="ShadowVote emblem" width="96" />

<h1 style="font-family: 'Times New Roman', Times, serif; letter-spacing: 0.04em; margin-bottom: 0.35em;">ShadowVote</h1>

<p style="font-family: 'Times New Roman', Times, serif; font-size: 1.15em; font-style: italic; max-width: 42em; margin: 0 auto 1.25em;">
Private governance on Midnight — people and DAOs vote with zero-knowledge proofs, so ballots never attach to wallet addresses on-chain.
</p>

<p>
<strong><a href="https://shadowvotesapp.vercel.app">Live app — shadowvotesapp.vercel.app</a></strong>
</p>

<p>
<a href="#how-it-works">How it works</a> ·
<a href="#features">Features</a> ·
<a href="#network-deployments">Live deployment</a> ·
<a href="#local-development">Develop</a> ·
<a href="#deploying-to-vercel">Deploy</a>
</p>

</div>

<div style="font-family: 'Times New Roman', Times, serif; line-height: 1.7; font-size: 16px;">

---

**ShadowVote** is an open DAO on [Midnight](https://midnight.network/): token-gated participation, zero-knowledge votes through **Lace**, and live tallies — without linking a vote to a wallet on the public ledger. Use the hosted app at **[shadowvotesapp.vercel.app](https://shadowvotesapp.vercel.app)**.

There is no voter registry and no admin allowlist. Anyone who meets the unshielded **tNIGHT** gate can prove a ballot in the browser, submit it through Lace, and leave only a nullifier and a count on-chain.

---

## How it works

ShadowVote is a frictionless **public DAO**. You do not pre-register, and no operator approves your wallet. Voting is three phases: a token gate in the app, a local ZK proof, then an on-chain tally that never records your address.

### Phase 1 — Connection and token gate

To keep spam and cheap Sybil identities out of the UI, ShadowVote enforces a minimum **unshielded tNIGHT** balance before a local voter secret is used for proofs.

1. Connect **Midnight Lace** to the dApp.
2. The app reads unshielded balances on Midnight.
3. **Rule:** at least **1,000 tNIGHT** (configurable via `NEXT_PUBLIC_SHADOWVOTE_MIN_TNIGHT`). Below that, governance actions stay locked.

*The gate is enforced in the application (`utils/tNightGate.ts`). The Compact contract records proofs and nullifiers — not a Merkle membership list or a Supabase `registered_voters` table.*

### Phase 2 — Private proof generation (off-chain)

After the gate, you vote without putting your wallet on the ballot.

1. Open a proposal and choose **Yes** or **No**.
2. The frontend generates a **zero-knowledge proof** locally (Midnight JS + Compact artifacts), not a transaction that lists your address as the voter.
3. The proof attests that a valid secret is entitled to this ballot **without revealing who you are**.
4. A **nullifier** is derived from your secret and the proposal — a one-time mark so the same secret cannot vote twice on that proposal.

### Phase 3 — Cast on-chain

1. Lace asks you to sign a transaction that carries the proof and nullifier.
2. The Midnight contract checks the ZK statement and rejects a reused nullifier.
3. If valid, **Yes** / **No** counts update in public state. The ledger does not store a mapping from that vote to your wallet.

```
Lace wallet  →  tNIGHT gate (app)  →  ZK proof + nullifier (browser)
                                              ↓
                              Compact contract (Midnight)
                                              ↓
                         Public tally  ·  hidden voter
```

---

## Features

| | |
| :--- | :--- |
| **ZK voting** | Compact circuit in `contracts/shadowvote.compact`. Proofs via Midnight JS and published ZK artifacts. |
| **Open DAO** | No Merkle allowlist. Eligibility is the unshielded tNIGHT check, then a local voter secret. |
| **Sybil / double-vote control** | Per-(secret, proposal) nullifiers on the public ledger. The app computes yours locally and shows **Vote cast** when it appears (`utils/crypto.ts`). |
| **Live sync** | Indexer contract stream (RxJS) plus polling fallback in `useShadowVote` for tallies and nullifier sets. |
| **Network awareness** | Amber banner on non-mainnet builds (`components/NetworkBanner.tsx`, `config/network.ts`). |
| **Polished UX** | Stitches tokens, Framer Motion, global toasts. |

---

## Network deployments

ShadowVote is live on **Midnight Preprod**. Use Lace and testnet tNIGHT (faucet) to transact.

| | |
| :--- | :--- |
| **App** | [https://shadowvotesapp.vercel.app](https://shadowvotesapp.vercel.app) |
| **Network** | Midnight Preprod |
| **Contract** | `b1eb2448c2164288361542720e1b8a822a28c5f05bd1a1456fb24fa293536a65` |
| **Wallet** | [Lace Midnight Preview](https://chromewebstore.google.com/detail/lace-midnight-preview/hgeekaiplokcnmakghbdfbgnlfheichg) |

---

## Tech stack

| Layer | Choice |
| :--- | :--- |
| App | **Next.js 15** (App Router) |
| Style | **@stitches/react** (`stitches.config.ts`) |
| Motion | **framer-motion** |
| On-chain / ZK | **@midnight-ntwrk/midnight-js**, compact-js, compact-runtime, ledger-v8, Lace **dapp-connector-api** |
| Contract | **Compact** (`*.compact`) → managed JS in `build/contract` |
| State | **RxJS** (indexer contract observables) |

Versions are pinned in `package.json`.

**Prerequisites:** Node.js **20+**, Lace on Preprod for public networks, **Docker** recommended for the local proof server.

---

## Local development

### 1. Clone and install

```bash
git clone https://github.com/kendacki/shadowvotes.git
cd shadowvotes
npm install
npm run setup
```

`npm run setup` creates `.env` from `.env.example` if needed, fills empty private-state passwords, and prints a short health report (including whether the proof-server port responds). Read-only: `npm run setup:check`.

### 2. Environment

Copy `.env.example` to `.env`, or let `npm run setup` bootstrap it. Minimum:

| Variable | Purpose |
| :--- | :--- |
| `NEXT_PUBLIC_SHADOWVOTE_CONTRACT_ADDRESS` | Deployed ledger address (hex). |
| `NEXT_PUBLIC_MIDNIGHT_PRIVATE_STATE_PASSWORD` | ≥16 characters — Level.js private-state store in the browser. |
| `NEXT_PUBLIC_MIDNIGHT_NETWORK_ID` or `NEXT_PUBLIC_MIDNIGHT_NETWORK` | e.g. `preprod` (banner uses the logical network). |
| `NEXT_PUBLIC_SHADOWVOTE_ZK_BASE` | URL or path to ZK assets (default `/shadowvote-zk` after `npm run zk:public`). |
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Optional off-chain proposal waiting room (`public.proposals`). |
| `NEXT_PUBLIC_MIDNIGHT_USE_PROOF_PROXY` | `1` to prove via `/api/midnight-proof` (Vercel + ngrok/Pinggy). Pair with server-only `PROOF_SERVER_URL`. |
| `PROOF_SERVER_URL` | **Server-only** — do not prefix `NEXT_PUBLIC_`. Tunnel to `http://localhost:6300`. Also used by `npm run deploy`. |

Indexer URLs, deploy seed, and proof/proxy notes: `.env.example`.

### 3. Compile the contract and publish ZK artifacts

```bash
npm run compile:contract
npm run zk:public
```

`public/shadowvote-zk` (or your custom base URL) must contain the prover artifacts from the Compact build.

### 4. Proof server

Needed for CLI deploy and some local proving flows:

```bash
npm run start-proof-server
# docker compose when available, else docker-compose
# http://127.0.0.1:6300 — set PROOF_SERVER_URL if needed
```

**WSL2:** enable Docker Desktop → Settings → Resources → **WSL integration**. If `docker version` fails inside WSL, run `npm run start-proof-server` from **Windows PowerShell**, or fix integration and open a new WSL terminal.

### 5. Run the app

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Connect Lace, then use **Dashboard** for proposals and votes.

If you see **CACHE MISMATCH ERROR** or a witness error mentioning **`voterMembershipPath`**, Next.js is still serving the old Merkle contract. This repo is **Open DAO** (`voterSecret` only). Recompile, republish ZK assets, and clear `.next`:

```bash
npm run compile:contract
npm run zk:public
# Windows PowerShell:
Remove-Item -Recurse -Force .next
npm run dev
```

`npm run build` checks that `build/contract` matches Open DAO before a production bundle.

**HTTPS hosts (e.g. Vercel):** the browser cannot call `http://127.0.0.1:6300` (mixed content; that host is *your* machine). `PROOF_SERVER_URL` alone helps **Node** (`npm run deploy`), not visitors’ browsers. See [Proving on Vercel](#proving-zk-on-vercel).

### 6. Deploy the contract (optional)

With wallet seed and proof server running:

```bash
npm run deploy
```

Writes `deployment.json` (address, `model: open-dao`, timestamp).

---

## Production build

`next.config.ts` is the source of truth. It enables WASM-friendly experiments (`asyncWebAssembly`, `layers`, `topLevelAwait`), `output.environment.asyncFunction`, `serverExternalPackages` for selected `@midnight-ntwrk/*` packages, and aliases for `@shadowvote/contract` and the `isomorphic-ws` shim.

```bash
npm run build
npm start
```

---

## Deploying to Vercel

1. Import this Git repository. Set **Root Directory** if the app is not the repo root.
2. **Settings → Environment Variables** — same `NEXT_PUBLIC_*` values as local (Production and Preview as needed). Never commit secrets. For a tunnel to a local proof server:

| Name | Example | Visibility |
| :--- | :--- | :--- |
| `NEXT_PUBLIC_MIDNIGHT_USE_PROOF_PROXY` | `1` | Browser (`NEXT_PUBLIC_*`). |
| `PROOF_SERVER_URL` | `https://your-tunnel.ngrok-free.dev` | **Server-only.** Vercel `/api/midnight-proof` → tunnel → `http://localhost:6300`. |

Do **not** put the ngrok URL in `NEXT_PUBLIC_MIDNIGHT_PROVER_SERVER_URI` when using the proxy (CORS).

3. Build: `npm run build` (install: `npm install`).
4. Output: Next.js default — no static export.
5. WASM: `vercel.json` sets `Content-Type: application/wasm` for `*.wasm`. Do **not** add an SPA catch-all rewrite to `index.html` (it breaks App Router).
6. ZK assets: run `npm run zk:public` in CI, or host artifacts on a CDN and set `NEXT_PUBLIC_SHADOWVOTE_ZK_BASE`.
7. Mainnet: set `NEXT_PUBLIC_MIDNIGHT_NETWORK=mainnet` only when you intend production; the network banner hides on mainnet.

### Proving (ZK) on Vercel

Midnight’s default proof server is **local HTTP** (`127.0.0.1:6300`). A public HTTPS app cannot call that from the browser. CORS can also block a raw HTTPS prover.

**What works**

1. **Lace in-wallet proving** — if the connector exposes `getProvingProvider`, the app uses it first (no extra env).
2. **Same-origin proxy** — `NEXT_PUBLIC_MIDNIGHT_USE_PROOF_PROXY=1`. The browser calls `/api/midnight-proof/check` and `/prove` on your app; the server forwards to `PROOF_SERVER_URL` (base URL, no `/check`). Keep ngrok/Pinggy **out** of `NEXT_PUBLIC_MIDNIGHT_PROVER_SERVER_URI` — the proof server does not send CORS headers.
3. **Direct HTTPS prover** — `NEXT_PUBLIC_MIDNIGHT_PROVER_SERVER_URI` to an origin the browser can call (and that allows your site if cross-origin).

`PROOF_SERVER_URL` is also used by `npm run deploy`.

**Example — ngrok → local proof server (port 6300)**

```bash
npm run start-proof-server
# other terminal:
ngrok http 6300
```

On Vercel (or in `.env`):

- `NEXT_PUBLIC_MIDNIGHT_USE_PROOF_PROXY=1`
- `PROOF_SERVER_URL=https://<your-ngrok-host>` (no `/check`)

Free ngrok URLs change on restart unless you reserve a domain. Leave ngrok running while you vote.

---

## Project layout

```
app/              Landing, dashboard, proposal detail (App Router)
components/       UI (Stitches + Motion)
config/           Network helpers
contexts/         Toast provider
contracts/        Compact sources
hooks/            Wallet, identity, indexer sync
lib/              Providers, contract loader
public/           Static assets + shadowvote-zk after zk:public
scripts/          Deploy and compile helpers
utils/            Nullifier / voting crypto
build/            Generated contract + zkir (after compile)
```

---

## Security

- Never commit `.env`, seeds, mnemonics, or `deployment.json` tied to funded keys. `.gitignore` excludes them; use `.env.example` only as a template.
- If a seed or mnemonic was ever shared or committed, **rotate**: move funds to a new wallet and treat the old material as compromised.
- `get-key.mjs` reads `MIDNIGHT_MNEMONIC` from the environment only — never paste phrases into source.
- `midnight.config.json` in this repo holds **public** endpoints. Keep wallet material in env or private config.

---

## License and support

Midnight ledger, Compact, and JS SDK packages in `node_modules` follow **Midnight Foundation** licensing. This README is documentation only — audit contracts and deployment before mainnet or high-value use.

| | |
| :--- | :--- |
| Midnight | [midnight.network](https://midnight.network/) |
| Contract | `contracts/shadowvote.compact` |
| Builds | Keep `next.config.ts` free of stale SPA rewrites in `vercel.json`. WASM must be reachable at `NEXT_PUBLIC_SHADOWVOTE_ZK_BASE`. |

</div>
