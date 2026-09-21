# Soroban Smart Contracts — Stellargent

[![Smart Contracts CI](https://github.com/StellarXagent/StellarXagent-contracts-v2/actions/workflows/smart-contracts-ci.yml/badge.svg)](https://github.com/StellarXagent/StellarXagent-contracts-v2/actions/workflows/smart-contracts-ci.yml)

Two interconnected Soroban smart contracts deployed on **Testnet**:

- **MyToken** (`my-token`): SEP-0041 fungible token with owner-governed minting, burning via sell, and pausable functionality.
- **PromptMarketplace** (`prompt-marketplace`): A marketplace where admins register prompts, users purchase them (burning tokens), and admins can re-mint.

---

## Deployments on Testnet

| Contract | ID | Wasm Hash | Size |
|---|---|---|---|
| **MyToken** | `CCHAUOEVX6TQD56VFZY2GI3N3HNF5W6QSRRKAGKDXV6S53T4BKD5PYQD` | `6613593d...` | 9.7 KB |
| **PromptMarketplace** | `CA6RRLV4IBLKRRLPUDCVXZFDKRE77YHBNJFSXEFFLXV6EUAVVVS6HJUQ` | `f31af684...` | 5.6 KB |

> ⚠️ **These instances predate the `set_marketplace` architecture.** They still expose `mint_forwarded` / `sell_forwarded` without a trusted caller. Do not use them to measure economics or as a reference for the current auth model. The token and marketplace must be re-deployed and bound using `set_marketplace` exactly once.

**Current Architecture**: The marketplace stores the token ID in its `__constructor`. The token stores a single Wasm marketplace address via `set_marketplace`, executable only once by the owner; a standard account (`G...`) or the token itself will be rejected. `buy_prompt` and `buy_private_prompt` call `my_token::sell_forwarded` via `env.invoke_contract`; `remint` calls `my_token::mint_forwarded`. Before each sub-invocation, the marketplace calls `authorize_as_current_contract`, and the token rejects any caller that is not the bound marketplace.

---

## Tests

### Unit tests (58)

```bash
# All tests (58)
SOROBAN_SDK_BUILD_SYSTEM_SUPPORTS_SPEC_SHAKING_V2=1 cargo test --workspace --target aarch64-apple-darwin

# Per package
SOROBAN_SDK_BUILD_SYSTEM_SUPPORTS_SPEC_SHAKING_V2=1 cargo test -p my-token --target aarch64-apple-darwin
SOROBAN_SDK_BUILD_SYSTEM_SUPPORTS_SPEC_SHAKING_V2=1 cargo test -p prompt-marketplace --target aarch64-apple-darwin
```

> On Linux/CI, use `--target x86_64-unknown-linux-gnu`, and on Windows, use `--target x86_64-pc-windows-msvc`, instead of `aarch64-apple-darwin`. The default workspace target (`.cargo/config.toml`) is `wasm32v1-none`, which does not support `cargo test` — you must always explicitly pass `--target` to run tests.

### Cross-contract authorization in Soroban v25

Soroban v25's mock auth cannot satisfy a SECOND `require_auth()` for the SAME address within a single invocation tree. If the root invocation calls `buyer.require_auth()` and then a sub-invocation (marketplace → token via `invoke_contract`) also calls `require_auth()` for `buyer`, the host rejects it with `Error(Auth, ExistingValue)`. Mock auth cannot represent "this address already authorized higher up in the tree".

**Mitigation adopted in this code:** `sell_forwarded` and `mint_forwarded` (`contracts/tokens/src/contract.rs`) only mutate balances if the Wasm marketplace bound via `set_marketplace` authorizes the call. The marketplace handles this authorization with `authorize_as_current_contract` right before invoking the token. A direct external call, a call without a configured marketplace, a call from a different marketplace, or a binding to an account (`G...`) will fail before altering the balance or supply.

`scripts/integration-test.sh` exercises the entire end-to-end flow against a real testnet using genuine signatures (Soroban CLI). This is the only place where a genuine nested `require_auth()` (if re-introduced by mistake) would be caught.

### Integration test (Testnet)

Tests the complete flow against a real testnet: mint → register → buy (cross-contract) → verify → remint → verify.

```bash
# Requires freshly deployed and bound contracts. The Testnet IDs in the table above DO NOT work for this, as they do not implement get_marketplace.
TOKEN="<MY_TOKEN_ID>" \
MKT="<MARKETPLACE_ID>" \
bash scripts/integration-test.sh
```

### Token: 21 tests

| Test | Coverage |
|---|---|
| `test_token_mint_and_balance` | Mint via storage API + balance + total_supply |
| `test_token_metadata` | name, symbol, decimals |
| `test_mint_multiple_same_user` | Cumulative mint to the same address |
| `test_mint_to_different_users` | Mint to multiple users, supply tracking |
| `test_mint_overflow_panics` | i128::MAX + 1 must panic |
| `test_zero_balance_default` | Default balance is 0 |
| `test_set_marketplace_stores_address` | Owner configures the trusted marketplace |
| `test_get_marketplace_fails_before_binding` | Reading marketplace fails if not yet configured |
| `test_set_marketplace_requires_owner` | Non-owner cannot configure the marketplace |
| `test_set_marketplace_cannot_retarget` | The marketplace cannot be silently retargeted |
| `test_set_marketplace_rejects_account_address` | An account (`G...`) cannot be the marketplace |
| `test_set_marketplace_rejects_token_self` | The token cannot be bound to itself |
| `test_sell_forwarded_fails_without_marketplace_binding` | Burn forwarded fails if no marketplace is configured |
| `test_sell_forwarded_fails_from_direct_external_call` | Direct external call cannot burn tokens |
| `test_sell_forwarded_rejects_holder_auth_without_marketplace_caller` | Holder auth does not substitute the marketplace caller |
| `test_sell_forwarded_updates_balance` | Happy path: marketplace auth burns tokens and emits `SellEvent` |
| `test_mint_forwarded_fails_without_marketplace_binding` | Mint forwarded fails if no marketplace is configured |
| `test_mint_forwarded_fails_from_direct_external_call` | Direct external call cannot mint tokens |
| `test_mint_forwarded_rejects_owner_auth_without_marketplace_caller` | Owner auth does not substitute the marketplace caller |
| `test_mint_forwarded_updates_balance` | Happy path: marketplace auth mints tokens and emits `MintEvent` |
| `test_mint_forwarded_rejects_non_positive_amount` | Amount 0 panics in the forwarded path |

### Marketplace: 36 tests

| Test | Coverage |
|---|---|
| `test_register_and_query_prompt` | Happy path: register → get_price / get_owner |
| `test_duplicate_registration_panics` | Same ID cannot be registered twice |
| `test_update_price` | Admin changes price |
| `test_remove_prompt` | Admin removes prompt (idempotent) |
| `test_multiple_prompts_independent` | Distinct prompts do not interfere |
| `test_get_price_unregistered_panics` | Query price of non-existent prompt |
| `test_non_admin_cannot_register` | Non-admin cannot register |
| `test_non_admin_cannot_update_price` | Non-admin cannot change price |
| `test_non_admin_cannot_remove` | Non-admin cannot remove |
| `test_register_zero_price_panics` | Price 0 is invalid |
| `test_update_price_zero_panics` | Updating to price 0 is invalid |
| `test_register_max_price` | i128::MAX works as price |
| `test_update_unregistered_prompt_panics` | Update price of non-existent prompt |
| `test_register_after_remove` | Re-register same ID post-removal |
| `test_has_access_unregistered` | has_access without purchase returns false |
| `test_token_mint_and_balance` | Mint via storage in token context |
| `test_buy_prompt_cross_contract` | E2E: register → buy_prompt (real cross-contract) → burned balance |
| `test_has_access_after_buy` | has_access is false before buying and true after `buy_prompt` |
| `test_buy_prompt_emits_event` | `buy_prompt` emits `PromptPurchased` with correct buyer/prompt_id/price |
| `test_remint_cross_contract` | E2E: `remint` (real cross-contract) → minted balance |
| `test_remint_emits_event` | `remint` emits `TokensReminted` with correct admin/to/amount |
| `test_unbound_marketplace_cannot_burn_forwarded_tokens` | An unbound marketplace cannot burn balances |
| `test_unbound_marketplace_cannot_mint_forwarded_tokens` | An unbound marketplace cannot mint balances |

---

## Build WASM

```bash
SOROBAN_SDK_BUILD_SYSTEM_SUPPORTS_SPEC_SHAKING_V2=1 stellar contract build --package my-token
SOROBAN_SDK_BUILD_SYSTEM_SUPPORTS_SPEC_SHAKING_V2=1 stellar contract build --package prompt-marketplace
```

Requires the `wasm32v1-none` target and the `SOROBAN_SDK_BUILD_SYSTEM_SUPPORTS_SPEC_SHAKING_V2=1` environment variable for Spec Shaking v2. Alternatively, for mainnet, use `cargo build --target wasm32v1-none --release`.

---

## Deploy

```bash
# Token
stellar contract deploy \
  --wasm target/wasm32v1-none/release/my_token.wasm \
  --source default \
  --network testnet \
  --alias my_token \
  -- \
  --owner "$(stellar keys address default)" \
  --name "PromptToken" \
  --symbol "PRMPT" \
  --decimals 7

# Marketplace (use the ID of the newly deployed token)
stellar contract deploy \
  --wasm target/wasm32v1-none/release/prompt_marketplace.wasm \
  --source default \
  --network testnet \
  --alias prompt_marketplace \
  -- \
  --admin "$(stellar keys address default)" \
  --token "$TOKEN_ID"

# Bind token -> marketplace once (use real deployment IDs, not the table ones)
stellar contract invoke \
  --id "$TOKEN_ID" \
  --source default \
  --network testnet \
  --send=yes \
  -- \
  set_marketplace \
  --marketplace "$MARKETPLACE_ID"
```

## Deploy on Mainnet

> ⚠️ **SECURITY WARNING — READ BEFORE EXECUTING**
>
> Mainnet involves real value and irreversible transactions. Before deploying:
>
> 1. **Do not hardcode or `source` private keys.** Use a secrets manager (1Password CLI, AWS Secrets Manager, HashiCorp Vault, GitHub encrypted secrets, etc.) to inject `MAINNET_DEPLOYER_SOURCE` and `MAINNET_ADMIN_SOURCE` at runtime.
> 2. **Verify the code and WASM before deploying.** Recompile from a trusted source, calculate the SHA-256 hash, and compare it with the artifact you are about to upload.
> 3. **Prefer multisig accounts or hardware wallets** for the admin account (`MAINNET_ADMIN_ADDR`). The deployer and admin can be different accounts.
> 4. **Check constructor parameters.** An error in `owner`/`admin` or in the marketplace's `token` can leave the contracts uncontrollable.
> 5. **Have sufficient XLM** in the deployer account to cover rent/storage for both contracts on mainnet.

### Prerequisites

- Stellar CLI configured for mainnet.
- Access to a mainnet RPC endpoint.
- Account with real XLM to pay for fees and rent.
- Separate admin account (recommended) with secure key pairs.
- Build target installed: `rustup target add wasm32v1-none`.
- Environment variable for Spec Shaking V2: `export SOROBAN_SDK_BUILD_SYSTEM_SUPPORTS_SPEC_SHAKING_V2=1`

### Mainnet Scripts

#### `scripts/deploy-mainnet.sh`

Builds release, calculates hashes, deploys, initializes, and validates both contracts. Emits a JSON summary in `deploy-artifacts/`.

```bash
MAINNET_DEPLOYER_SOURCE="deployer" \
MAINNET_ADMIN_SOURCE="admin" \
MAINNET_ADMIN_ADDR="G..." \
bash scripts/deploy-mainnet.sh
```

#### `scripts/verify-mainnet.sh`

Verifies a pair of already deployed contracts without re-deploying. Useful for CI or periodic validations.

```bash
MAINNET_TOKEN_ID="C..." \
MAINNET_MARKETPLACE_ID="C..." \
MAINNET_ADMIN_ADDR="G..." \
bash scripts/verify-mainnet.sh
```

### Mainnet Release Gate & Security

Before executing `deploy-mainnet.sh` with real funds, this release is subject to security checks defined in our internal documentation:
- [`docs/security/MAINNET_RELEASE_CHECKLIST.md`](docs/security/MAINNET_RELEASE_CHECKLIST.md) - Master checklist for deployment.
- [`docs/security/MAINNET_CUSTODY_POLICY.md`](docs/security/MAINNET_CUSTODY_POLICY.md) - Deployer/admin custody policy.
- [`docs/security/AUDIT_PROCESS.md`](docs/security/AUDIT_PROCESS.md) - Independent security review process.
- [`docs/operations/MONITORING_AND_INCIDENT_RESPONSE.md`](docs/operations/MONITORING_AND_INCIDENT_RESPONSE.md) - Monitoring and rollback runbook.

### Invocation Examples

*(Old Testnet instances — useful only for metadata reading, not for the current purchasing flow)*

```bash
# Read token name
stellar contract invoke --source default --network testnet \
  --id CCHAUOEVX6TQD56VFZY2GI3N3HNF5W6QSRRKAGKDXV6S53T4BKD5PYQD \
  -- name

# Register prompt
stellar contract invoke --source default --network testnet \
  --id CA6RRLV4IBLKRRLPUDCVXZFDKRE77YHBNJFSXEFFLXV6EUAVVVS6HJUQ \
  --send=yes \
  -- register_prompt --prompt_id "alpha" --price 500 \
  --owner "$(stellar keys address default)"

# Query price
stellar contract invoke --source default --network testnet \
  --id CA6RRLV4IBLKRRLPUDCVXZFDKRE77YHBNJFSXEFFLXV6EUAVVVS6HJUQ \
  -- get_price --prompt_id "alpha"
```

---

## Project Structure

```
contracts/
├── tokens/                      # MyToken (fungible token)
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs               # module exports, public re-export
│       ├── contract.rs          # #[contract] MyToken — public API + traits
│       ├── events.rs            # #[contractevent] structs (MintEvent, SellEvent, etc.)
│       ├── core/
│       │   └── token.rs         # TokenManager — business logic
│       ├── storage/
│       │   └── types.rs         # DataKey, contracttype structs
│       └── tests.rs             # 21 tests
└── marketplace/                 # PromptMarketplace
    ├── Cargo.toml
    └── src/
        ├── lib.rs               # module exports
        ├── contract.rs          # #[contract] PromptMarketplace — public API
        ├── storage/
        │   └── types.rs         # DataKey, Prompt struct, errors
        └── tests.rs             # 36 tests
Cargo.toml                       # workspace: tokens + marketplace
```

---

## Stack

- **Soroban SDK**: `25.3.0`
- **OZ Stellar Contracts**: `v0.7.1` (fungible token, ownable, pausable)
- **Stellar CLI**: `26.1.0`
- **Target**: `wasm32v1-none` (release), `aarch64-apple-darwin` (tests)
