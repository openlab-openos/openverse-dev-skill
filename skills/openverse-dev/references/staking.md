---
title: Native BTG Staking (delegate to a validator)
description: End-to-end playbook for staking (delegating) native BTG to a validator on Openverse using @solana/kit + @solana-program/stake. Covers finding a validator's vote account by name, creating + initializing + delegating a stake account in one transaction, the mandatory sysvar accounts the Codama client omits, rent exemption, and verification/unstake.
---

# Native BTG Staking

## When to use this guidance

Use when the user asks to "stake", "delegate", "delegate stake", "stake btg to a validator", or "stake X BTG to validator Y" — i.e. lock native BTG with a validator to earn staking rewards. This is the classic Stake Program flow (not liquid staking / stake pools).

Do **not** use for: liquid staking tokens, stake pools, Marinade/Jito-style products, or governance voting.

## Program IDs & sysvar addresses

| Constant | Address |
|---|---|
| Stake program | `Stake11111111111111111111111111111111111111` |
| System program | `11111111111111111111111111111111` |
| Vote program (validator) | `Vote111111111111111111111111111111111111111` |
| Config program (validator names + stake config) | `Config1111111111111111111111111111111111111` |
| Rent sysvar | `SysvarRent111111111111111111111111111111111` |
| Clock sysvar | `SysvarC1ock11111111111111111111111111111111` |
| Stake history sysvar | `SysvarStakeHistory1111111111111111111111111` |
| Stake config account | `StakeConfig11111111111111111111111111111111` |

Use `SYSVAR_RENT_ADDRESS`, `SYSVAR_CLOCK_ADDRESS`, `SYSVAR_STAKE_HISTORY_ADDRESS` from `@solana/sysvars` instead of hand-typing them (they are 43 chars, easy to mistype).

## Step 1 — Resolve a validator name to a vote account

`getVoteAccounts` returns only pubkeys (no human names). Validator display names (e.g. "C01 An anonymous National-Level Fund") live in the **Config program**, one `validatorInfo` account per validator. Each has a `configData.name` and a `keys[]` array whose `signer: true` entry is the validator **identity** (node pubkey).

```bash
# List all validator info accounts (jsonParsed)
curl -s https://api.mainnet.openverse.network -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getProgramAccounts","params":["Config1111111111111111111111111111111111111",{"encoding":"jsonParsed"}]}'
```

Then match the name → identity pubkey, and map identity → vote pubkey via `getVoteAccounts` (`current[].nodePubkey` → `votePubkey`). Verify the vote account is owned by the Vote program before delegating:

```bash
curl -s https://api.mainnet.openverse.network -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getAccountInfo","params":["<VOTE_PUBKEY>",{"encoding":"base64"}]}'
# expect: "owner":"Vote111111111111111111111111111111111111111"
```

If you already have the vote pubkey directly, skip this step.

## Step 2 — Numbers you need

- **1 BTG = 1_000_000_000 lamports.** Stake `0.01 BTG` = `10_000_000n` lamports.
- **Stake account size = 200 bytes** (`StakeStateV2::size_of()`).
- **Rent-exempt minimum (200 B)** = `getMinimumBalanceForRentExemption(200n)` → `2_282_880` lamports on mainnet (note: `getMinimumBalanceForRentExemption` returns the value directly, NOT `{ value }`).
- **Total to fund the stake account** = `stake_amount + rent` (e.g. `10_000_000n + 2_282_880n = 12_282_880n`). You fund rent + stake together in the single `createAccount`.
- **Minimum delegation**: the runtime enforces a minimum via `get_minimum_delegation()`; a too-small amount fails `DelegateStake` with `InvalidArgument` (which maps to `StakeError::InsufficientDelegation`). Simulate first (Step 4) to confirm the amount passes.

## Step 3 — Build the transaction (create + initialize + delegate in ONE tx)

The flow is exactly three instructions:

1. **System `createAccount`** — new 200-byte account owned by the Stake program, funded with `stake + rent`.
2. **Stake `initialize`** — set `authorized.staker` and `authorized.withdrawer` to the wallet (lockup defaults: `unixTimestamp: 0, epoch: 0, custodian: System program`).
3. **Stake `delegateStake`** — point the stake at the validator's vote account, authority = wallet.

### CRITICAL gotcha — the Codama client omits sysvar accounts

`@solana-program/stake`'s generated `getInitializeInstruction` / `getDelegateStakeInstruction` omit the sysvar accounts the on-chain program actually requires. Without them simulation fails with `NotEnoughAccountKeys` / `InvalidArgument`. You must splice them in manually:

```ts
import {
  createKeyPairSignerFromBytes, createPrivateKeyFromBytes, getPublicKeyFromPrivateKey,
  address, createSolanaRpc, createSolanaRpcSubscriptions, sendAndConfirmTransactionFactory,
  pipe, createTransactionMessage, setTransactionMessageFeePayerSigner,
  setTransactionMessageLifetimeUsingBlockhash, appendTransactionMessageInstructions,
  signTransactionMessageWithSigners, assertIsTransactionWithBlockhashLifetime,
  getSignatureFromTransaction,
} from '@solana/kit';
import { STAKE_PROGRAM_ADDRESS, getInitializeInstruction, getDelegateStakeInstruction } from '@solana-program/stake';
import { getCreateAccountInstruction } from '@solana-program/system';
import { SYSVAR_RENT_ADDRESS, SYSVAR_CLOCK_ADDRESS, SYSVAR_STAKE_HISTORY_ADDRESS } from '@solana/sysvars';

const RPC = 'https://api.mainnet.openverse.network';
const VOTE_ACCOUNT = address('<VOTE_PUBKEY>');
const STAKE_AMOUNT_LAMPORTS = 10_000_000n;      // 0.01 BTG
const STAKE_CONFIG = address('StakeConfig11111111111111111111111111111111');

const rpc = createSolanaRpc(RPC);
const rpcSubscriptions = createSolanaRpcSubscriptions(RPC.replace(/^https/, 'wss'));

const wallet = /* KeyPairSigner loaded from keypair file — see Step 3a */;
const stakeAccount = /* KeyPairSigner for the NEW stake account — see Step 3a */;

const minRent = await rpc.getMinimumBalanceForRentExemption(200n).send(); // bigint, not { value }
const total = STAKE_AMOUNT_LAMPORTS + minRent;

// 1) create stake account
const createIx = getCreateAccountInstruction({
  payer: wallet,
  newAccount: stakeAccount,
  lamports: total,
  space: 200,
  programAddress: STAKE_PROGRAM_ADDRESS,
});

// 2) initialize — MUST append Rent sysvar (2 accounts total: [stake, rent])
const initIx = getInitializeInstruction({
  stake: stakeAccount.address,
  arg0: { staker: wallet.address, withdrawer: wallet.address },
  arg1: { unixTimestamp: 0, epoch: 0, custodian: address('11111111111111111111111111111111') },
});
const initIxWithRent = { ...initIx, accounts: [...initIx.accounts, { address: SYSVAR_RENT_ADDRESS, role: 0 }] };

// 3) delegate — MUST insert Clock, StakeHistory, StakeConfig BEFORE the authority
//    (6 accounts total: [stake, vote, clock, stake_history, stake_config, authority])
const delegateIx = getDelegateStakeInstruction({
  stake: stakeAccount.address,
  vote: VOTE_ACCOUNT,
  stakeAuthority: wallet,
});
const delegateIxWithSysvars = {
  ...delegateIx,
  accounts: [
    ...delegateIx.accounts.slice(0, 2),
    { address: SYSVAR_CLOCK_ADDRESS, role: 0 },
    { address: SYSVAR_STAKE_HISTORY_ADDRESS, role: 0 },
    { address: STAKE_CONFIG, role: 0 },
    ...delegateIx.accounts.slice(2),
  ],
};

const { value: latestBlockhash } = await rpc.getLatestBlockhash().send();
const message = pipe(
  createTransactionMessage({ version: 0 }),
  (m) => setTransactionMessageFeePayerSigner(wallet, m),
  (m) => setTransactionMessageLifetimeUsingBlockhash(latestBlockhash, m),
  (m) => appendTransactionMessageInstructions([createIx, initIxWithRent, delegateIxWithSysvars], m),
);
const signed = await signTransactionMessageWithSigners(message);
assertIsTransactionWithBlockhashLifetime(signed);
const sendAndConfirm = sendAndConfirmTransactionFactory({ rpc, rpcSubscriptions });
await sendAndConfirm(signed, { commitment: 'confirmed' });
```

The account-order reference (from the Stake program source) for `DelegateStake`:
`0.[WRITE] stake  1.[] vote  2.[] Clock  3.[] StakeHistory  4.[] StakeConfig  5.[SIGNER] authority`.
`Initialize`: `0.[WRITE] stake  1.[] Rent`.

### Step 3a — Create and persist the stake account keypair

`generateKeyPairSigner()` gives you a non-extractable `CryptoKey`, so you **cannot** serialize it back to a 64-byte JSON file later. To persist the stake account's keypair (needed for future deactivate/withdraw), generate raw bytes yourself:

```ts
import { createPrivateKeyFromBytes, getPublicKeyFromPrivateKey, createKeyPairSignerFromBytes } from '@solana/kit';
import { webcrypto } from 'node:crypto';

// Generate fresh 64-byte keypair (32 secret + 32 public) and persist it
const secret = new Uint8Array(32);
webcrypto.getRandomValues(secret);
const priv = await createPrivateKeyFromBytes(secret, true);
const pub = await getPublicKeyFromPrivateKey(priv, true);
const pubBytes = new Uint8Array(await webcrypto.subtle.exportKey('raw', pub));
const full = new Uint8Array(64);
full.set(secret, 0);
full.set(pubBytes, 32);
writeFileSync('stake-keypair.json', JSON.stringify(Array.from(full)));
const stakeAccount = await createKeyPairSignerFromBytes(full, false);

// Reload later
const stakeAccount = await createKeyPairSignerFromBytes(
  new Uint8Array(JSON.parse(readFileSync('stake-keypair.json', 'utf8'))), false
);
```

Alternatively, reuse `createKeyPairSignerFromBytes(new Uint8Array(raw))` for the wallet too — the keypair JSON file is a plain 64-element array (32 secret + 32 public).

## Step 4 — Simulate first (required)

`rpc.simulateTransaction(base64WireTx, { encoding: 'base64', commitment: 'confirmed' })`. Expect `err: null` and three `success` lines (System, Stake initialize, Stake delegate). Use `getBase64EncodedWireTransaction(signed)` to get the wire encoding. Any `NotEnoughAccountKeys` / `InvalidArgument` → missing/misordered sysvar accounts (Step 3).

Per the guardrails: **show the user the transaction summary (validator, amount, fee payer, cluster) and get explicit approval before sending on mainnet.**

## Step 5 — Verify

```bash
curl -s https://api.mainnet.openverse.network -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getAccountInfo","params":["<STAKE_ACCOUNT>",{"encoding":"jsonParsed"}]}'
# expect data.parsed.type == "delegated", stake.delegation.voter == <VOTE_PUBKEY>, stake.delegation.stake == amount
```

Stake becomes effective (`activationEpoch`) after the current epoch; rewards settle automatically in subsequent epochs.

## Unstake / deactivate / withdraw

- **Deactivate** (`getDeactivateInstruction`): stake account `[WRITE]`, Clock sysvar, authority `[SIGNER]` (3 accounts — again the Codama client may omit Clock; add it). Then wait for cooldown.
- **Withdraw** (`getWithdrawInstruction`): stake account `[WRITE]`, recipient `[WRITE]`, Clock, StakeHistory, withdraw authority `[SIGNER]` (+ optional custodian).

Follow the same sysvar-splicing pattern as Step 3 for any instruction the generated client under-specifies.

## Safety / risk notes

- **Staking is NOT reversible instantly.** Deactivating enters a cooldown (warmup/cooldown rate, default 0.25/epoch); withdrawing active stake is rejected.
- **Slashing risk**: the delegated BTG can be slashed if the validator double-signs (5%) or is offline (small). Pick a trustworthy validator.
- The stake account keypair (stake authority lives in it too, via the wallet) must be kept safe to deactivate/withdraw later. The **wallet** (staker/withdrawer authority) is what actually authorizes unstake.
- Never hand-roll `/ 1e9` for BTG; use `lamports(...)` / the SDK fixed-point helpers.
