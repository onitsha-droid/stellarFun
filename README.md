# stellarFun

**A token launchpad with bonding curves, built on Stellar's Soroban smart contract platform.**

Launch a token in one transaction. Trade it instantly against a bonding curve. When it hits the graduation threshold, liquidity moves to a real AMM pool and the LP tokens are locked. No presales, no team allocations, no rug pulls.

> ⚠️ **Status: early development. Unaudited. Testnet only.** Do not deploy to mainnet or use real funds until the contracts have been independently audited.

---

## Table of Contents

- [How it works](#how-it-works)
- [Architecture](#architecture)
- [Repository structure](#repository-structure)
- [Getting started](#getting-started)
- [Build, test, deploy](#build-test-deploy)
- [Contract interface](#contract-interface)
- [Bonding curve math](#bonding-curve-math)
- [Frontend and indexer](#frontend-and-indexer)
- [Security considerations](#security-considerations)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## How it works

1. **Create**: a creator submits a name, symbol, and metadata URI. The factory deploys a new SEP-41 token and its bonding curve.
2. **Trade**: anyone can buy or sell against the curve. Price rises as supply is bought and falls as it is sold. Every trade takes a small protocol fee.
3. **Graduate**: once the curve reaches its target (market cap or tokens sold), the reserves and remaining tokens are deposited into a DEX pool. The LP tokens are burned or locked.
4. **Trade freely**: the token now trades on the AMM like any other asset.

## Architecture

```
                 ┌────────────────────┐
   create ─────► │  Factory contract  │──── registers ───► token index
                 └─────────┬──────────┘
                           │ deploys
              ┌────────────┴────────────┐
              ▼                         ▼
     ┌─────────────────┐       ┌──────────────────┐
     │ Token (SEP-41)  │◄──────│  Bonding curve   │◄── buy / sell / quote
     │ fixed supply    │ mint/ │  holds reserves  │
     │ no mint auth    │ xfer  │  (XLM / USDC)    │
     └─────────────────┘       └────────┬─────────┘
                                        │ graduate
                                        ▼
                              ┌──────────────────┐
                              │ AMM pool (DEX)   │
                              │ LP burned/locked │
                              └──────────────────┘
```

| Component | Responsibility |
|---|---|
| **Factory** | Deploys token and curve instances from stored WASM hashes; keeps a registry of launches. |
| **Token** | SEP-41 compliant token. Full supply minted once at creation, mint authority removed. |
| **Bonding curve** | Holds the reserve asset, prices trades, enforces slippage limits, triggers graduation. |
| **Graduation adapter** | Moves liquidity into the target AMM and locks the LP position. |
| **Indexer** | Reads contract events from Soroban RPC into Postgres for the UI. |
| **Frontend** | Create, trade, and browse tokens. Wallet signing via Stellar Wallets Kit. |

## Repository structure

> Adjust this to match your actual layout as the project grows.

```
stellarFun/
├── contracts/
│   ├── factory/          # Token + curve deployer and registry
│   ├── token/            # SEP-41 token
│   ├── bonding_curve/    # Buy / sell / quote / graduate
│   └── shared/           # Common math and types
├── indexer/              # Event ingestion service (TypeScript)
├── frontend/             # Web app (TypeScript)
├── scripts/              # Deploy and testnet helper scripts
├── docs/                 # Design notes and curve math
├── Cargo.toml            # Rust workspace
└── README.md
```

## Getting started

### Prerequisites

- [Rust](https://www.rust-lang.org/tools/install) with the WebAssembly target:
  ```bash
  rustup target add wasm32v1-none
  ```
- [Stellar CLI](https://developers.stellar.org/docs/tools/cli/install-cli)
- Node.js 20+ and a package manager (npm, pnpm, or yarn), for the frontend and indexer
- A Stellar wallet such as [Freighter](https://www.freighter.app/) set to Testnet
- PostgreSQL, if you are running the indexer

> Check the Stellar docs for the current recommended Wasm target and CLI version, as these change between releases.

### Clone and install

```bash
git clone https://github.com/<your-username>/stellarFun.git
cd stellarFun

# Contracts
cargo build

# Frontend and indexer
cd frontend && npm install && cd ..
cd indexer && npm install && cd ..
```

### Set up a testnet identity

```bash
stellar keys generate deployer --network testnet --fund
stellar keys address deployer
```

## Build, test, deploy

### Build contracts

```bash
stellar contract build
```

Optimized `.wasm` files land in `target/`.

### Run tests

```bash
cargo test
```

Curve math should be covered by unit tests and fuzz or property tests, especially rounding and overflow edge cases.

### Deploy to testnet

```bash
# Upload WASM for the token and curve
stellar contract upload \
  --wasm target/wasm32v1-none/release/token.wasm \
  --source deployer --network testnet

# Deploy the factory
stellar contract deploy \
  --wasm target/wasm32v1-none/release/factory.wasm \
  --source deployer --network testnet
```

Then initialize the factory with the token and curve WASM hashes, fee recipient, and graduation parameters. See [`scripts/`](./scripts) for helper scripts.

### Environment variables

Copy `.env.example` to `.env` in `frontend/` and `indexer/`:

```bash
# Network
STELLAR_NETWORK=testnet
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
NETWORK_PASSPHRASE="Test SDF Network ; September 2015"

# Deployed contracts
FACTORY_CONTRACT_ID=C...

# Indexer
DATABASE_URL=postgres://user:pass@localhost:5432/stellarfun
```

## Contract interface

Signatures are illustrative. Update them to match the code.

**Factory**

| Function | Description |
|---|---|
| `create_token(creator, name, symbol, metadata_uri)` | Deploys a token and curve, registers them, returns addresses. |
| `get_token(id)` | Returns registry info for a launched token. |
| `list_tokens(start, limit)` | Paginated registry listing. |

**Bonding curve**

| Function | Description |
|---|---|
| `buy(buyer, reserve_in, min_tokens_out)` | Buy tokens with reserve asset; reverts if output is below the slippage floor. |
| `sell(seller, tokens_in, min_reserve_out)` | Sell tokens back to the curve. |
| `quote_buy(reserve_in)` / `quote_sell(tokens_in)` | Read-only price quotes. |
| `graduate()` | Callable once the threshold is met; migrates liquidity to the AMM. |

**Events**: `create`, `buy`, `sell`, `graduate`. The indexer depends on these.

## Bonding curve math

stellarFun starts with a constant-product curve using virtual reserves, the same family of model popularized by pump.fun:

```
(virtual_reserve + real_reserve) × (virtual_tokens - tokens_sold) = k
```

Implementation notes:

- All math uses `i128` fixed-point integers. Soroban has no floating point.
- Rounding always favors the protocol, never the trader.
- Fees are taken on the reserve side and accounted for separately from curve reserves.
- Full derivations and parameter choices belong in [`docs/curve.md`](./docs).

## Frontend and indexer

**Frontend**: TypeScript with `@stellar/stellar-sdk` and Stellar Wallets Kit. Pages for creating a token, browsing launches, and trading with live quotes and slippage settings.

```bash
cd frontend
npm run dev
```

**Indexer**: polls Soroban RPC for contract events and stores them in Postgres. RPC retains only a limited window of event history, so the indexer must run continuously and track its last processed ledger.

```bash
cd indexer
npm run start
```

## Security considerations

- **Unaudited**: treat everything here as experimental until audited.
- **No mint authority**: token supply is fixed at creation. Verify this in tests.
- **State archival**: Soroban storage entries expire without TTL extensions. Contracts extend TTLs on normal use, and the indexer or a keeper should bump long-lived entries.
- **Resource limits**: deploy-and-initialize flows must fit within Soroban's per-transaction CPU, memory, and ledger-entry limits. Simulate before submitting.
- **Manipulation risks**: consider sandwich attacks, first-buyer advantages, and graduation-time price manipulation in the design.
- **Reentrancy and auth**: require authorization (`require_auth`) on every state-changing function and review cross-contract calls carefully.

To report a vulnerability, please email `security@your-domain.example` rather than opening a public issue.

## Roadmap

- [ ] SEP-41 token contract with locked supply
- [ ] Bonding curve contract with `buy`, `sell`, `quote`
- [ ] Factory contract with registry
- [ ] Testnet deployment and minimal UI
- [ ] Event indexer and price charts
- [ ] Graduation to Soroswap or Aquarius with LP lock
- [ ] Fuzzing and property tests
- [ ] External security audit
- [ ] Mainnet launch

## Contributing

Contributions are welcome.

1. Fork the repo and create a feature branch.
2. Make your changes with tests.
3. Run `cargo fmt`, `cargo clippy`, and `cargo test`.
4. Open a pull request describing what changed and why.

## License

Released under the [MIT License](./LICENSE). Replace with your preferred license.

## Disclaimer

stellarFun is experimental software provided as is, without warranty. Memecoins and bonding-curve tokens are highly speculative and can lose all value. This project is not financial advice.
