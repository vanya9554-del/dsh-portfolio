# Multisig setup on Base: runbook

Practical runbook for creating a Safe multisig on Base, adding signers, and running a first
test transfer. Written to be executed by anyone with the signer wallets — every step has a
verification, and every verification is something you can hand to an auditor.

> **Deliverable contract:** on a paid order this runbook is executed end-to-end and returned
> with screenshots + transaction hashes for creation, each signer confirmation, the test
> transfer, and its execution.

---

## 0. Before you start

| Item | Why it matters |
|---|---|
| Network | **Base mainnet** (chainId `8453`) or **Base Sepolia** (`84532`) for a dry run |
| Signer wallets | Each signer must control a wallet that can sign on Base; hardware wallets supported |
| Threshold | `m-of-n` — how many confirmations a transaction needs (common: 2-of-3, 3-of-5) |
| Owner funding | The Safe address needs ETH on Base for gas when it sends transactions |
| Gas budget | Base fees are cents; keep ~$1 of ETH on the Safe for the test and follow-ups |

**Do not reuse an EOA that already holds funds as a signer wallet** unless you accept that the
key is now an operational key. Prefer dedicated signer keys.

---

## 1. Create the Safe

1. Go to `https://app.safe.global/` and connect the wallet that will be the first signer.
2. Select **Base** as the network (the app shows the chain next to the account menu).
3. **Create new Safe account** → name it (`e.g. "Team Treasury"`).
4. Add the **signer addresses** (n) and set the **threshold** (m).
5. Review and submit the deployment transaction from the first signer.
6. Wait for confirmation; copy the **Safe address**.

**Verify:**

```bash
# The proxy should be deployed and hold the singleton
cast code <SAFE_ADDRESS> --rpc-url https://mainnet.base.org   # non-empty 0x...
```

Record:

- Safe address: `0x…`
- Deploy tx hash: `0x…`
- Signers: `0x…, 0x…, 0x…`
- Threshold: `m` of `n`

---

## 2. Fund the Safe for the test

1. Send a small amount of ETH (or the asset being tested) **to the Safe address**.
2. Wait for at least one block.

**Verify:** `GET https://api.etherscan.io/api?module=account&action=balance&address=<SAFE>` returns a
non-zero balance (or check the address on Basescan).

---

## 3. Run the test transfer

1. In the Safe app open the Safe → **New transaction** → **Send tokens**.
2. Recipient: a wallet **you control** (never a third party for the first test).
3. Amount: a trivial amount (e.g. `0.0001 ETH` or `0.01 USDC`).
4. Create the transaction → it becomes a **pending multisig transaction**.

**Verify the proposal:** the transaction appears in the queue with status
`1 of m confirmations`.

---

## 4. Confirm from each signer

1. Each signer opens the queue and **confirms** the transaction.
2. When confirmations reach `m`, the transaction becomes executable.
3. Any signer (or the executor) **executes** it.

**Verify:**

| Check | Where |
|---|---|
| Pending tx hash | Safe queue → transaction details |
| Confirmation count | `m of m` shown on the tx |
| Execution tx hash | After execute → transaction details → "Transaction hash" |
| Balance moved | Basescan shows the transfer out of the Safe |
| Nonce advanced | Safe queue shows the next nonce |

---

## 5. Evidence package (what to hand over)

- [ ] Safe address + network
- [ ] Creation tx hash
- [ ] Signer list and threshold
- [ ] Test transfer transaction (proposal) hash
- [ ] Each confirmation (screenshot or signature)
- [ ] Execution tx hash
- [ ] Before/after balance screenshots
- [ ] A short note on how to add/remove a signer and change the threshold later

---

## 6. Operating notes

**Adding a signer.** Safe → Settings → Owners → Add owner → submit as a Safe transaction; it needs
the current threshold's confirmations. The threshold does not change automatically.

**Changing the threshold.** Settings → Threshold → set the new `m` → submit as a Safe transaction.
Keep `m ≤ n` at all times.

**Removing a signer.** Same flow; a removed signer cannot confirm new transactions but can still
confirm the removal transaction if it was pending.

**Lost key.** If the number of remaining keys drops below `m`, the Safe is frozen — that is the
security property, not a bug. Choose `m` with one key of redundancy.

**Spending policy.** For treasury use, keep a low-value operational Safe for day-to-day payouts
and a high-threshold main Safe that only moves funds into the operational one.

---

## 7. Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| "Insufficient funds for gas" on execute | Safe has no ETH for execution | Fund the Safe address |
| Transaction stuck "0 of m" | Signers are offline / using wrong chain | Check each signer is on Base |
| "Safe already deployed" | Re-using the same address | Use the create flow, not connect |
| Cannot execute | Confirmations < threshold | Confirm from remaining signers |
| Wrong network | Wallet connected to Ethereum mainnet | Switch wallet + Safe app to Base |

---

## 8. Quick reference

```bash
# Base mainnet
RPC=https://mainnet.base.org
# Safe singleton / proxy factory (verify against Safe docs for the current deployment)
# The Safe app handles deployment; use `cast` only for verification:

cast code <SAFE> --rpc-url $RPC
cast call <SAFE> "getThreshold()(uint256)" --rpc-url $RPC
cast call <SAFE> "getOwners()(address[])" --rpc-url $RPC
```

`getThreshold()` and `getOwners()` are the two on-chain checks an auditor will ask for first.
