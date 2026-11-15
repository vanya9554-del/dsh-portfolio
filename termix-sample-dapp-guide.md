# Dapp integration guide with error-handling notes

A practical, copy-paste-oriented guide for integrating a dapp front-end with an EVM wallet
and a smart contract. Written to be framework-agnostic; examples use `viem` + `wagmi`, which
are the current defaults for EVM dapps, but every pattern maps 1:1 to `ethers` or a wallet SDK.

---

## 1. Connection flow (happy path)

1. **Detect a provider** — `window.ethereum` (injected) or WalletConnect. Never assume the
   injected provider is MetaMask; check `provider.isMetaMask`, `isRabby`, `isCoinbaseWallet`.
2. **Request accounts** with `eth_requestAccounts`. This is the only call that may open a
   wallet popup, so call it from a user gesture (click), never on page load.
3. **Read the chain** with `eth_chainId` and compare with the chain your contract lives on.
4. **Wire the account and chain listeners** before the first transaction:
   - `accountsChanged` → re-read balances and permissions, reset any signed-in state.
   - `chainChanged` → re-create the client, do not keep stale chain state.
   - `disconnect` → clear the session and show the connect button again.
5. **Sign a message** (`personal_sign`) if you need authentication; verify it server-side with
   `ethers.verifyMessage` or `viem.verifyMessage`. Never trust the address the client sends.
6. **Simulate** the transaction (`eth_call` / `simulateContract`) before sending it; this catches
   most reverts before the user pays gas.
7. **Send** with `eth_sendTransaction`, then wait for the receipt and watch for `status === 0`.

---

## 2. Error-handling matrix

| Failure | What the user sees | How to detect | Recommended handling |
|---|---|---|---|
| User rejects the request | Wallet popup closes, no error toast | `code 4001` (`UserRejectedRequest`) | Silent for `eth_requestAccounts` on first load; friendly toast for a click-driven action |
| User rejects the transaction | "Transaction rejected" | `code 4001` on `eth_sendTransaction` | Keep the form state, let them retry without re-typing |
| Wallet not installed | `window.ethereum === undefined` | Check on mount | Show an "Install a wallet" panel with the wallet's store link |
| No accounts authorized | Empty `eth_accounts` result | `eth_accounts` returns `[]` | Prompt to connect; do not show "not connected" as an error |
| Wrong chain | Transactions fail or the app looks read-only | `chainId !== EXPECTED` | Offer `wallet_switchEthereumChain`; if it fails with `4902`, call `wallet_addEthereumChain` first |
| Insufficient funds for gas | Wallet shows a warning and refuses | Pre-check `eth_getBalance` vs. estimated fee | Show the shortfall in the native token, with a faucet/bridge link |
| `eth_estimateGas` revert | "Cannot estimate gas" | `eth_estimateGas` throws before the wallet opens | Decode the revert reason, show a domain-level message ("Not enough allowance") |
| Transaction reverted on-chain | Toast after confirmation | Receipt `status === 0` | Show a block explorer link and the decoded revert reason; never mark the flow "done" |
| RPC timeout / 5xx | Spinner forever | Request throws or exceeds your timeout | Retry with exponential backoff (max 3), then fall back to a secondary RPC |
| Nonce too low | Second transaction fails right after a fast one | `nonce too low` error from RPC | Re-read `eth_getTransactionCount(address, "pending")` and retry once |
| Replacement/underpriced | Stuck transaction | `replacement transaction underpriced`, or pending > 2 min | Show a "speed up" action; re-send with the same nonce and +20% `maxFeePerGas` |
| User switches account mid-flow | Transaction from an unexpected address | `accountsChanged` fired | Abort the in-flight flow and ask the user to confirm from the new account |
| Rate-limited RPC (429) | Intermittent read failures | HTTP 429 | Rotate to a secondary RPC endpoint; cache reads for a few seconds |

---

## 3. Client code: correct connection + typed errors

```ts
import { createWalletClient, custom, createPublicClient, http } from 'viem';
import { base } from 'viem/chains';

const EXPECTED_CHAIN_ID = base.id;

type WalletError = { code?: number; message: string };

export async function connectWallet() {
  if (!window.ethereum) throw new Error('NO_WALLET');

  const client = createWalletClient({ chain: base, transport: custom(window.ethereum) });
  const [address] = await client.requestAddresses();

  // chain might be wrong right after connecting
  await ensureChain(client);
  return { client, address };
}

export async function ensureChain(client: ReturnType<typeof createWalletClient>) {
  const current = await client.getChainId();
  if (current === EXPECTED_CHAIN_ID) return;
  try {
    await client.switchChain({ id: EXPECTED_CHAIN_ID });
  } catch (err) {
    const e = err as WalletError;
    if (e.code === 4902) {
      await client.addChain({ chain: base });
      return;
    }
    throw err;
  }
}

export function explainWalletError(err: unknown): string {
  const e = err as WalletError;
  if (e?.code === 4001) return 'Request cancelled in the wallet.';
  if (e?.code === -32002) return 'A wallet request is already open — finish it there.';
  if (/insufficient funds/i.test(e?.message ?? '')) return 'Not enough ETH on Base to pay for gas.';
  if (/user rejected/i.test(e?.message ?? '')) return 'Transaction cancelled.';
  return 'Something went wrong. Please try again.';
}
```

---

## 4. Transaction wrapper with simulation and retry

```ts
export async function sendSafely({ client, publicClient, to, data, value = 0n }) {
  // 1. simulate first — most reverts are caught here for free
  try {
    await publicClient.simulateContract({
      account: client.account, address: to, abi, functionName, args,
      value,
    });
  } catch (simErr) {
    throw new Error(`WOULD_REVERT: ${decodeReason(simErr)}`);
  }

  // 2. send
  const hash = await client.sendTransaction({
    account: client.account,
    to,
    data,
    value,
    // let the wallet estimate fees; only override for "speed up"
  });

  // 3. wait and check status explicitly
  const receipt = await publicClient.waitForTransactionReceipt({
    hash,
    confirmations: 1,
    timeout: 120_000,
  });
  if (receipt.status !== 'success') throw new Error('TX_REVERTED');
  return { hash, receipt };
}
```

**Retry policy:**

| Layer | Retries | Backoff | Notes |
|---|---|---|---|
| RPC reads | 3 | 300 ms → 1 s | Rotate endpoint on 429/5xx |
| `eth_estimateGas` | 1 | none | A revert is deterministic; retrying wastes time |
| Transaction send | 1 | none | Never blindly re-send: it can double-pay. Use nonce replacement |
| Receipt wait | 2 | 2 s | After timeout, keep the hash and show "pending" |

---

## 5. Error decode helper

```ts
const KNOWN_ERRORS: Record<string, string> = {
  '0x08c379a0': 'revert',
  '0x4e487b71': 'panic',
};

export function decodeReason(err: any): string {
  const data: string | undefined = err?.data?.data ?? err?.data;
  if (!data || data === '0x') return err?.shortMessage ?? err?.message ?? 'Unknown revert';
  const selector = data.slice(0, 10);
  if (selector === '0x08c379a0') {
    // Error(string) — decode the ABI-encoded string after the selector
    return decodeAbiString(data.slice(10));
  }
  return `Custom error ${selector}${KNOWN_ERRORS[selector] ? ` (${KNOWN_ERRORS[selector]})` : ''}`;
}
```

---

## 6. Checklist before shipping

- [ ] Connect button only calls `eth_requestAccounts` on click.
- [ ] Chain check on connect **and** on `chainChanged`.
- [ ] `accountsChanged` resets signed-in state.
- [ ] Every write goes through `simulateContract` first.
- [ ] `receipt.status` is checked; a revert is never reported as success.
- [ ] Every error mapped to one human sentence; no raw stack traces in UI.
- [ ] Secondary RPC configured with a fallback.
- [ ] Explorer link shown for every confirmed transaction.
- [ ] Wallet-not-installed state has an install path.
- [ ] No private keys or seed phrases ever touch the front-end code.
