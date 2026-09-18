---
title: OpenSwap (BTG → USDC)
description: Swap native BTG for USDC (or any SPL pair) on Openverse via the CPMM pool using openverse-raydium-sdk-v2. Full copy-paste script: find the pool, quote with CurveCalculator.swap, build with raydium.cpmm.swap, simulate, and send. Covers baseIn semantics, WBTG wrapping, 9-decimal mints, the custom token/ATA programs, and the pitfalls that break the tradeV2 router on Openverse.
---

# OpenSwap (BTG → USDC)

## Contents

- [When to use this guidance](#when-to-use-this-guidance)
- [Key facts (read first)](#key-facts-read-first)
- [Program IDs & mints](#program-ids--mints)
- [Quick start: verified script](#quick-start-verified-script)
- [Step-by-step](#step-by-step)
- [Native BTG input (WBTG wrapping)](#native-btg-input-wbtg-wrapping)
- [Gotchas](#gotchas)
- [Verify settlement](#verify-settlement)
- [Safety checklist](#safety-checklist)
- [Resources](#resources)

## When to use this guidance

Use this guidance when the user asks about:

- Swapping BTG for USDC (or USDC → BTG, or any SPL↔SPL pair) on Openverse
- "Exchange", "swap", "convert", "buy USDC with BTG" on Raydium/OpenSwap
- Integrating a DEX swap into a dApp, script, or backend
- Deciding between the Raydium SDK and hand-rolled swap instructions

For a plain transfer of an existing token (no exchange-rate logic), use [payments.md](payments.md) instead. For reading a balance or looking up a pool, use [rpc-quick-lookups.md](rpc-quick-lookups.md).

## Key facts (read first)

These are the things that silently break swaps on Openverse if you don't know them:

- **Openverse mainnet only has CPMM pools.** `getProgramAccounts` on the AMM v4 (`675kPX9…`) and CLMM (`CAMMCzo5…`) programs returns **zero** pools. The BTG/USDC pair lives in a **CPMM** pool (`swapCpz48…` program). Ignore the AMM-v4 `routeSwap` path.
- **Both USDC and WBTG are 9 decimals.** USDC is *not* 6 decimals here. `0.01 BTG = 10_000_000` raw units, and `1 USDC = 1_000_000_000` raw units.
- **The token program is custom.** Openverse mints/accounts are owned by `Token9ADbPtdFC3PjxaohBLGw2pgZwofdcbj6Lyaw6c` (the SDK's `TOKEN_2022_PROGRAM_ID` from `open-token-web3`), and the ATA program is `AtokenhZ6AE34VMYRv1AqSv8q8QZJxxEaY1zKiXKwSWT` — **not** the standard `Tokenkeg…` / `ATokenGPvbd…`. Do not use `@solana/spl-token`'s `getAssociatedTokenAddressSync` to compute balances; it returns the wrong address.
- **`tradeV2` router quotes are unreliable here.** `getAllRouteComputeAmountOut` silently drops the direct CPMM pool (an `openTime` type bug) and can surface illiquid 2-hop routes with absurd "price impact". For a known pool, compute the quote yourself with `CurveCalculator.swap` (shown below).
- **`baseIn` means "input is the pool's mintA (base)".** For BTG → USDC, BTG is `mintB`, so `baseIn = false`.

## Program IDs & mints

| Item | Address |
|------|---------|
| CP-Swap / CPMM program (mainnet) | `swapCpz48CQA1zVD8xXr6e4RGCtZCc9pTDEjQ5cUukr` |
| Raydium Router (multi-hop only) | `routeUGWgWzqBWFcrCfv8tritsqukccJPu3q5GPP3xS` |
| Raydium AMM v4 (no pools on Openverse) | `675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8` |
| Raydium CLMM (no pools on Openverse) | `CAMMCzo5YL8w4VFF8KVHrK22GGUsp5VTaW7grrKgrWqK` |
| **Token program (custom)** | `Token9ADbPtdFC3PjxaohBLGw2pgZwofdcbj6Lyaw6c` |
| **ATA program (custom)** | `AtokenhZ6AE34VMYRv1AqSv8q8QZJxxEaY1zKiXKwSWT` |
| Wrapped BTG (native wrapper) | `B67JGY8hbUcNbpMufKJ4dF3egfbZuD4EkyffQ3cxZcUz` |
| USDC | `USDo1uHcFo9H6aHWcqCkhBiWiMhUqQJFienbKDBPEhN` |
| Example CPMM pool (WBTG/USDC) | `DGtDcYk2t4yBFVytm1NvkdMydSuRMJMCdzd1sGM6xCdf` |

In the SDK, `WSOLMint` and `NATIVE_MINT_2022` are both aliased to the WBTG address `B67JGY8…`, so "native BTG in" and "WBTG in" are the same thing.

## Quick start: verified script

This is the minimal, tested path (CommonJS — the SDK is a minified CJS bundle and its named exports don't survive ESM interop, so use `require`). It finds the pool, quotes, builds, simulates, and (with `--send`) broadcasts.

```bash
npm install openverse-raydium-sdk-v2 @solana/web3.js bn.js
```

```js
// swap.cjs
const { Raydium, CurveCalculator, TxVersion } = require('openverse-raydium-sdk-v2');
const { Connection, Keypair, PublicKey } = require('@solana/web3.js');
const BN = require('bn.js');
const fs = require('fs');

const OV_NODE = 'https://api.mainnet.openverse.network';
const INPUT_MINT = 'B67JGY8hbUcNbpMufKJ4dF3egfbZuD4EkyffQ3cxZcUz'; // WBTG (native BTG)
const OUTPUT_MINT = 'USDo1uHcFo9H6aHWcqCkhBiWiMhUqQJFienbKDBPEhN'; // USDC
const AMOUNT_BTG = 0.01;      // amount to swap (in BTG)
const SLIPPAGE = 0.01;        // 1%  (range 1 ~ 0.0001)
const SEND = process.argv.includes('--send');

const kp = Keypair.fromSecretKey(Uint8Array.from(JSON.parse(fs.readFileSync(process.argv[2] || 'keypair.json', 'utf8'))));
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function getPoolData(raydium, poolId) {
  for (let i = 0; i < 4; i++) {                       // intermittent RPC failures — retry
    try { return await raydium.cpmm.getPoolInfoFromRpc(poolId); }
    catch (e) { console.log('retry', i + 1, e.message); await sleep(2000); }
  }
  throw new Error('getPoolInfoFromRpc failed');
}

async function main() {
  const connection = new Connection(OV_NODE, 'confirmed');
  const raydium = await Raydium.load({ connection, owner: kp, disableLoadToken: true, disableFeatureCheck: true });

  // 1. Find the pool (or hardcode POOL_ID to skip the scan)
  const mintA = new PublicKey(INPUT_MINT), mintB = new PublicKey(OUTPUT_MINT);
  const pools = await raydium.cpmm.fetchAllPools();
  const found = pools.find(p =>
    (p.mintA.equals(mintA) && p.mintB.equals(mintB)) || (p.mintA.equals(mintB) && p.mintB.equals(mintA)));
  if (!found) throw new Error('no CPMM pool for pair');
  const poolId = found.poolId.toBase58();

  const { poolInfo, poolKeys, rpcData } = await getPoolData(raydium, poolId);

  // 2. Quote. baseIn = "input is the pool's mintA (base)". BTG is mintB => false.
  const baseIn = poolInfo.mintA.address === INPUT_MINT;
  const inputAmount = new BN(Math.round(AMOUNT_BTG * 1e9));
  const swapResult = CurveCalculator.swap(
    inputAmount,
    baseIn ? rpcData.baseReserve : rpcData.quoteReserve,
    baseIn ? rpcData.quoteReserve : rpcData.baseReserve,
    rpcData.configInfo.tradeFeeRate,
  );
  const amountOut = swapResult.destinationAmountSwapped;
  const minAmountOut = amountOut.mul(new BN(Math.round((1 - SLIPPAGE) * 10000))).div(new BN(10000));

  console.log('pool:', poolId);
  console.log('expected out:', (Number(amountOut) / 1e9).toFixed(9), 'USDC');
  console.log('min out (1%):', (Number(minAmountOut) / 1e9).toFixed(9), 'USDC');
  console.log('price:', (Number(amountOut) / 1e9 / AMOUNT_BTG).toFixed(6), 'USDC/BTG');

  // 3. Build + simulate
  const { transaction } = await raydium.cpmm.swap({
    poolInfo, poolKeys, inputAmount, swapResult, slippage: SLIPPAGE, baseIn, txVersion: TxVersion.LEGACY,
  });

  const { blockhash } = await connection.getLatestBlockhash('confirmed');
  transaction.recentBlockhash = blockhash;
  transaction.feePayer = kp.publicKey;
  const sim = await connection.simulateTransaction(transaction, [kp]); // web3.js >=1.96: signer array
  console.log('simulate err:', JSON.stringify(sim.value.err));

  if (!SEND) { console.log('dry-run. re-run with --send <keypair> to broadcast'); return; }

  // 4. Send
  const { txId } = await raydium.cpmm.swap({
    poolInfo, poolKeys, inputAmount, swapResult, slippage: SLIPPAGE, baseIn, txVersion: TxVersion.LEGACY,
  }).then((d) => d.execute({ sendAndConfirm: true }));
  console.log('txId:', txId);
}

main().catch((e) => { console.error(e); process.exit(1); });
```

## Step-by-step

### 1. Load the SDK

```js
const raydium = await Raydium.load({
  connection,                 // Connection to https://api.mainnet.openverse.network
  owner: kp,                  // Keypair (signs) or PublicKey (wallet-signing)
  disableLoadToken: true,     // skip the token-list API fetch (unused here)
  disableFeatureCheck: true,  // skip the availability/region check
});
```

`disableLoadToken: true` avoids the Raydium token-list API; `getPoolInfoFromRpc` fetches mint info (decimals, program) directly from the chain, so you never need the token list.

### 2. Find the pool

`raydium.cpmm.fetchAllPools()` returns every CPMM pool's `{ poolId, mintA, mintB, … }` (a single `getProgramAccounts`). Match `mintA`/`mintB` to your pair, then hardcode the `poolId` for future runs (pool addresses are stable).

### 3. Quote with `CurveCalculator.swap`

`CurveCalculator.swap(sourceAmount, swapSourceAmount, swapDestinationAmount, tradeFeeRate)` is a plain constant-product + fee calc, returning `{ sourceAmountSwapped, destinationAmountSwapped, tradeFee }`. Feed it the input-side reserve first, then the output-side reserve:

- `baseIn = true`  → input is `mintA`: `swap(baseReserve-reserve, quoteReserve-reserve)`
- `baseIn = false` → input is `mintB`: `swap(quoteReserve-reserve, baseReserve-reserve)`

`rpcData.baseReserve`/`quoteReserve`/`configInfo.tradeFeeRate` are already `BN`s; `tradeFeeRate = 1000` means 0.1% (denominator `1_000_000`).

### 4. Build with `raydium.cpmm.swap`

```js
await raydium.cpmm.swap({
  poolInfo, poolKeys,          // from getPoolInfoFromRpc
  inputAmount,                 // BN, raw input units
  swapResult,                  // from CurveCalculator.swap
  slippage: 0.01,              // 0.01 = 1%; the SDK bakes it into minAmountOut
  baseIn,
  txVersion: TxVersion.LEGACY,
});
```

This returns `{ transaction, signers, execute }`. It internally:
- wraps native BTG → WBTG (creates a seeded WBTG account + `InitializeAccount`),
- emits the CPMM `swapBaseInput` instruction,
- closes the WBTG account at the end.

`execute({ sendAndConfirm: true })` signs and broadcasts. To simulate first, set the blockhash and call `connection.simulateTransaction(transaction, [kp])` (note: web3.js ≥1.96 takes a **signer array**, not a config object; the older API `simulateTransaction(tx, { sigVerify:false })` throws `Invalid arguments`).

## Native BTG input (WBTG wrapping)

Native BTG is not an SPL token, so on-chain swaps move **wrapped BTG** (`B67JGY8…`). `raydium.cpmm.swap` handles the whole wrap/unwrap cycle automatically when the input mint is the WBTG address (which equals `WSOLMint` / `NATIVE_MINT_2022` in this fork). The compiled transaction is four instructions:

1. system `createAccountWithSeed` — fund a WBTG account with the input lamports,
2. token `InitializeAccount` — turn it into a WBTG token account (the Openverse token program credits native balance on init, so there is **no separate `syncNative` instruction**),
3. CPMM `swapBaseInput` (two inner `TransferChecked`: WBTG in, USDC out),
4. token `CloseAccount` — unwrap the remainder back to BTG.

Only add these steps yourself if you hand-roll the swap; the two most common hand-rolled bugs are forgetting the wrap and forgetting the final unwrap.

## Gotchas

- **`getAllRoute` needs `PublicKey`, not strings.** Passing `inputMint: 'B67…'` throws `Cannot read properties of undefined (reading 'negative')`. Pass `new PublicKey(...)`.
- **`getAllRouteComputeAmountOut` drops the direct CPMM pool.** It throws inside `poolReady` (an `openTime` string-vs-BN mismatch) and silently filters the direct route, leaving only 2-hop routes whose `priceImpact` looks absurd. Trust `CurveCalculator.swap` for a known pool instead.
- **`baseIn` is easy to invert.** `baseIn = poolInfo.mintA.address === INPUT_MINT`. Getting it wrong swaps the wrong direction (you'd buy WBTG with USDC).
- **Decimals are 9.** Both WBTG and USDC are 9-decimal. Using 6 for USDC under/over-scales amounts by 1000×.
- **Custom token/ATA programs.** `getTokenAccountsByOwner(owner, { programId: TOKEN_2022_PROGRAM_ID })` (from `open-token-web3`, = `Token9ADbPtd…`) is the only reliable way to read balances; `@solana/spl-token`'s ATA helpers compute the wrong address on Openverse.
- **Use CommonJS.** `import { Raydium } from 'openverse-raydium-sdk-v2'` under ESM often yields `Raydium` undefined because the package `main` is minified CJS without an `exports` map. `require` always works.

## Verify settlement

Don't trust the quote — read the chain after the tx confirms. `@solana/spl-token` ATA helpers are wrong here, so use the token-2022 program directly:

```js
const { TOKEN_2022_PROGRAM_ID } = require('open-token-web3');
const accs = await connection.getTokenAccountsByOwner(owner, { programId: TOKEN_2022_PROGRAM_ID });
for (const a of accs.value) {
  const info = await connection.getParsedAccountInfo(a.pubkey);
  const p = info.value.data.parsed.info;
  if (p.mint === 'USDo1uHcFo9H6aHWcqCkhBiWiMhUqQJFienbKDBPEhN')
    console.log('USDC balance:', p.tokenAmount.uiAmountString);
}
```

Or inspect the finalized tx directly: `getTransaction(txId, { maxSupportedTransactionVersion: 0 })` → `meta.pre/postTokenBalances` gives the exact per-token deltas, and `meta.pre/postBalances[0]` gives the fee-payer native (BTG) delta.

## Safety checklist

- **Show the quote before signing.** Always display amount-in, expected amount-out, `minAmountOut`, slippage %, and fee payer — per the transaction-review guardrail in the main SKILL.
- **Simulate before sending** and surface the result (`err: null` + the swap program logs ending in `success`).
- **Never store or request keys** — use wallet-standard signing (`walletSigner`) for UI flows.
- **Default to devnet/localnet** unless the user explicitly requests mainnet.
- **Validate the pool** is the intended pair (both mints match) before swapping — prevents wrong-pool / fake-token phishing.
- **Confirm settlement on-chain** by reading the USDC ATA after the tx, not from the quote alone.

## Resources

- [openverse-raydium-sdk-v2](https://github.com/openlab-openos/openverse-raydium-sdk-v2) — this fork's source; `src/raydium/cpmm/cpmm.ts` (`swap`, `getPoolInfoFromRpc`, `fetchAllPools`) and `src/raydium/cpmm/curve/calculator.ts` (`CurveCalculator.swap`) are the authoritative references.
