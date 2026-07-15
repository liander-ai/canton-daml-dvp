# canton-daml-dvp

**Atomic Delivery-versus-Payment (DvP) settlement for a tokenized real-world asset, written in [Daml](https://www.digitalasset.com/developers) for the [Canton Network](https://www.canton.network/).**

This is the institutional settlement problem at the heart of RWA tokenization: a
security-token leg and a cash leg must change hands **atomically** — both transfer
in a single transaction, or neither does. Any gap between the two legs is
*principal risk* (one party pays or delivers and the other defaults). Canton's
underlying model — Daml's multi-party authorization — expresses true DvP natively,
which is why regulated finance is building on it.

Same core protocol I've implemented on two other chains, now in a third paradigm:

| Chain | Paradigm | Repo |
|-------|----------|------|
| Solana | Rust / Anchor | `anchor-staking-rewards` |
| EVM | Solidity / Foundry | [`evm-staking-vault`](https://github.com/liander-ai/evm-staking-vault) |
| **Canton** | **Daml** | **this repo** |

## The model (`daml/Rwa.daml`)

- **`Iou`** — a cash token, signed by its issuer (a central bank / stablecoin issuer), owned by a holder.
- **`Asset`** — a tokenized RWA unit (e.g. a bond), signed by its registrar/issuer, owned by a holder.
- Both use the canonical Daml **propose → accept** transfer: the owner initiates a transfer contract that preserves the issuer's signature, and the new owner accepts it, so the resulting token stays issuer-authorized.
- **`Trade`** — the atomic DvP. The seller has initiated an `AssetTransfer` to the buyer; the buyer has initiated an `IouTransfer` to the seller. `Trade_Settle` exercises **both legs inside one transaction**, combining the authority of both counterparties (and both issuers, carried through the transfer contracts). It commits or aborts as a whole — no principal risk.

## Why this shows Canton's strength

An EVM/Solidity atomic swap needs a custom escrow contract holding both assets and
trusting its own code. In Daml, atomicity and authorization are properties of the
**ledger model**: a transaction is only valid if every required party's authority is
present, and it is all-or-nothing by construction. `Trade_Settle` is a few lines and
cannot half-execute.

## Tests (`daml/Test.daml`)

Run on an in-memory ledger via Daml.Script — **no node, no Docker required**:

- `testAtomicDvp` — bond + cash settle atomically; ownership swaps in one transaction.
- `testOnlyBuyerCanSettle` — a third party (`Mallory`) cannot settle someone else's trade (`submitMustFail`).
- `testRejectReturnsAsset` — a rejected transfer returns the asset to its original owner.

## Run it

```bash
# install the Daml SDK (bring your own JDK 11+)
curl -sSL https://get.daml.com/ | sh -s 2.9.0
export PATH="$HOME/.daml/bin:$PATH"

# compile + run all Daml.Script tests
daml test
```

CI (`.github/workflows/ci.yml`) installs the SDK and runs `daml test` on every push.

## License

MIT — see [LICENSE](LICENSE).
