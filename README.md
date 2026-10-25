# Solana Token Staking Smart Contract

A **Solana token staking smart contract** built with [Anchor](https://www.anchor-lang.com/) that lets users stake SPL tokens and earn rewards over time. The program supports admin configuration, reward deposits, proportional reward distribution, and optional pausing—suitable for token staking dApps on Solana devnet and mainnet.

---

## Table of Contents

- [Overview](#overview)
- [Project Architecture](#project-architecture)
- [Prerequisites](#prerequisites)
- [How to Run This Project](#how-to-run-this-project)
- [Features](#features)
- [Staking & Rewards System](#staking--rewards-system)
- [CLI Reference](#cli-reference)
- [License](#license)

---

## Overview

This repository contains a **Solana token staking program** (smart contract) that:

- Lets an **admin** configure the staking pool (token mint, claim period, reward multiplier, etc.) and deposit reward tokens.
- Lets **users** stake tokens, unstake (withdraw principal + rewards), or claim rewards only.
- Distributes rewards **proportionally** by share of total staked amount over configurable time periods.
- Supports **two-step authority transfer** and **pause/unpause** for safety.

Built with **Rust** and **Anchor 0.30.1**, with a **TypeScript CLI** for testing and interaction.

---

## Project Architecture

### High-Level Layout

```
Solana-Token-Staking-Program/
├── programs/
│   └── tapestry-explorer-statking-contract/   # Anchor program (Rust)
│       └── src/
│           ├── lib.rs                         # Program entry, instruction routing
│           ├── state/                         # On-chain account types
│           │   ├── mod.rs
│           │   ├── config.rs                  # Global config (authority, token mint, rewards)
│           │   └── user.rs                    # Per-user staking state
│           ├── instructions/
│           │   ├── mod.rs
│           │   ├── admin/                     # Admin-only instructions
│           │   │   ├── configure.rs           # Set global config
│           │   │   ├── deposit_fund.rs        # Deposit reward tokens
│           │   │   ├── nominate_authority.rs  # Propose new admin
│           │   │   ├── accept_authority.rs    # Accept admin role
│           │   │   └── pause.rs               # Pause/unpause contract
│           │   └── user/                      # User instructions
│           │       ├── stake.rs               # Stake tokens
│           │       ├── unstake.rs             # Unstake + receive rewards
│           │       └── claim_reward.rs        # Claim rewards only
│           ├── errors.rs                      # Custom error codes
│           ├── constants.rs                   # Seeds (CONFIG, USERINFO)
│           └── utils.rs                       # Math/helpers
├── lib/                                       # Shared TS helpers for CLI
│   ├── scripts.ts                             # Transaction builders (config, stake, unstake, etc.)
│   ├── util.ts                                # Execution helpers
│   └── constant.ts                            # SEED_*, TOKEN_ADDRESS, etc.
├── cli/
│   ├── command.ts                             # CLI entry (commander)
│   └── scripts.ts                             # CLI actions (config, deposit, stake, unstake, claim, pause, getuserinfo)
├── target/                                    # Build output (IDL, .so)
├── keys/                                      # Wallet keypairs (e.g. admin.json)
├── Anchor.toml                                # Anchor workspace & provider config
├── package.json                               # Node deps & scripts
└── Cargo.toml                                 # Rust workspace
```

### On-Chain Accounts

| Account   | Seeds              | Description |
|----------|--------------------|-------------|
| **Config** | `["config"]`     | Global state: authority, pending_authority, token_mint_config, claim_period, total_rate, total_stakers, last_reward_time, reward_multiplier, deposit_time, total_deposit, purchase_amt, is_stop, initialized. |
| **User**  | `["userinfo", user_pubkey]` | Per-user state: user pubkey, deposit (staked amount), debt (claimed rewards), last_update, initialized. |

### Instruction Flow

- **Admin:** `configure` → (optional) `nominate_authority` / `accept_authority` → `deposit_fund` to fund rewards; `pause` to stop/resume.
- **User:** `stake(amount)` → later `unstake()` (principal + rewards) or `claim_reward()` (rewards only).

Rewards are computed from `total_rate`, `reward_multiplier`, `claim_period`, and user’s share of `total_deposit` (see [Staking & Rewards System](#staking--rewards-system)).

---

## Prerequisites

Before building or running the Solana token staking smart contract:

1. **Rust** (stable + nightly used by Anchor)
2. **Solana CLI** (e.g. 1.18.x)
3. **Anchor CLI 0.30.1**

Installation guide: [Solana docs – Installation](https://solana.com/docs/intro/installation).

Install Anchor 0.30.1:

```bash
avm install 0.30.1
avm use 0.30.1
anchor --version   # should show 0.30.1
```

4. **Node.js** and **Yarn** (for CLI and tests):

```bash
node --version
yarn --version
```

5. **Wallet keypair** for deploy/admin (e.g. `./keys/admin.json`). Set path in `Anchor.toml` under `[provider]` and in CLI (see below).

---

## How to Run This Project

### 1. Clone and install dependencies

```bash
git clone <repo-url>
cd Solana-Token-Staking-Program
yarn install
```

### 2. Build the program

Use the same Rust nightly as in the project:

```bash
RUSTUP_TOOLCHAIN="nightly-2024-11-19" anchor build
```

Artifacts are in `target/deploy/` (e.g. `tapestry_explorer_statking_contract.so` and the IDL).

### 3. Configure cluster and wallet

- **Cluster:** In `Anchor.toml`, set `[provider]` to your target cluster and RPC:

  ```toml
  [provider]
  cluster = "https://api.devnet.solana.com"   # or your RPC URL
  wallet = "./keys/admin.json"
  ```

- **CLI:** Default RPC and keypair are in `cli/command.ts`. Override per run with `-r` and `-k` (see [CLI Reference](#cli-reference)).

### 4. Deploy to devnet (or another cluster)

Ensure Solana CLI is pointed at the same cluster:

```bash
solana config set --url devnet
solana airdrop 2   # if needed
anchor deploy
```

Program ID is in `Anchor.toml` under `[programs.devnet]` and in `programs/.../src/lib.rs` (`declare_id!(...)`). After changing program ID, run `anchor build` again.

### 5. Configure the staking program (admin)

Set global config (token mint, claim period, reward multiplier, purchase amount, etc.):

```bash
yarn script config
```

Use your admin keypair (e.g. `./keys/admin.json`). Ensure `lib/constant.ts` has the correct `TOKEN_ADDRESS` (staking/reward token mint) for your deployment.

### 6. Deposit reward tokens (admin)

Fund the program’s reward vault so users can earn rewards:

```bash
yarn script deposit
```

Admin’s token account for the configured mint must hold enough tokens; the program transfers them into the protocol’s vault.

### 7. Stake, unstake, claim (users)

- **Stake** (amount in token base units, e.g. lamports for 6 decimals):

  ```bash
  yarn script stake -a <TOKEN_AMOUNT>
  ```

- **Unstake** (withdraw staked tokens + accrued rewards):

  ```bash
  yarn script unstake
  ```

- **Claim** (rewards only):

  ```bash
  yarn script claim
  ```

Use the keypair set via `-k` (or default in `cli/command.ts`).

### 8. Optional: pause and user info

- **Pause / unpause** (admin):

  ```bash
  yarn script pause -s 1   # pause
  yarn script pause -s 0   # unpause
  ```

- **Inspect user state:**

  ```bash
  yarn script getuserinfo
  ```

### 9. Run tests (if present)

```bash
yarn test
```

(Uses test script from `package.json`; add tests under `tests/` if needed.)

---

## Features

- **Initialize / configure program** – Admin sets token mint, claim period, reward multiplier, and other parameters.
- **Deposit funds** – Admin deposits reward tokens into the program vault.
- **Stake tokens** – Users stake SPL tokens and start earning rewards proportionally.
- **Unstake tokens** – Users withdraw staked tokens plus accrued rewards in one tx.
- **Claim rewards** – Users can claim only rewards without unstaking.
- **Authority management** – Two-step transfer: `nominate_authority` → `accept_authority`.
- **Pause** – Admin can pause staking/unstaking/rewards when needed.

---

## Staking & Rewards System

- **`purchase_amt`** is the total reward token allocation for a single **reward period** (e.g. 2 weeks, depending on `claim_period` and config).
- Rewards are distributed **proportionally** to each user’s share of **total staked amount**.
- **Example:**
  1. User A stakes 100 tokens → 100% of rewards go to A while they are the only staker.
  2. User B stakes 400 tokens → Total pool = 500. B has 80%, A has 20%; rewards split 80% / 20%.
  3. On **unstake**, the user receives their **original staked tokens** plus **accrued reward tokens**.

Reward math uses `total_rate`, `reward_multiplier`, and `claim_period`; see `state/config.rs` and `lib/scripts.ts` (e.g. `calcReward`) for details.

---

## CLI Reference

CLI is built with `commander` and run via:

```bash
yarn script <command> [options]
```

**Common options (all commands):**

| Option | Short | Description | Default |
|--------|--------|-------------|---------|
| `--env` | `-e` | Cluster name (e.g. mainnet-beta, devnet) | `devnet` |
| `--rpc` | `-r` | RPC URL | `https://api.devnet.solana.com` |
| `--keypair` | `-k` | Path to keypair JSON | See `cli/command.ts` |

**Commands:**

| Command | Description |
|---------|-------------|
| `config` | (Admin) Initialize/update global program config. |
| `deposit` | (Admin) Deposit reward tokens into the vault. |
| `stake` | (User) Stake tokens. Requires `-a` / `--amount`. |
| `unstake` | (User) Unstake and receive principal + rewards. |
| `claim` | (User) Claim rewards only. |
| `pause` | (Admin) Pause (e.g. `-s 1`) or unpause (`-s 0`). |
| `getuserinfo` | Print current user staking state and reward info. |

**Example:**

```bash
yarn script stake -a 1000000000 -e devnet -k ./keys/user.json
```

---

## Contract

[Telegram](https://t.me/fiv3dev)
