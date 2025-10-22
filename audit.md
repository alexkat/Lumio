# 🧾 Smart Contract Audit Report — Lumio

**Scope:**  
- `LumioERC20Factory` / `CustomERC20`  
- `NFTsCollectionFactory` / `NFTCollection`  
- `LumioMarketplace`

**Auditor:** Alexkat  
**Date:** 2025-10-22  
**Audit Type:** Preliminary  
**OpenZeppelin:** v5-style modules in use (`ERC20{Burnable,Pausable,Capped}`, `ERC721`, `ERC2981`, `Ownable`, `ReentrancyGuard`, `Pausable`)

---

## 🧭 Executive Summary

**Strengths**
- Correct OZ mixin usage with proper multiple inheritance override (`_update`) in ERC20.
- ERC20 safety toggles (mint/pause/cap) are feature-flagged.
- Marketplace uses `ReentrancyGuard` and splits payouts (fee/royalty/seller).
- Auctions implement a safe refund pattern via `pendingWithdrawals` (pull model).

**Key Risks (at a glance)**
- **ETH transfers via `transfer`** in several places → DoS/compatibility risk (2300 gas stipend).
- **No change returned to buyers** in `buyNFT` (overpayment gets stuck in contract).
- **No allowlist for collections** on marketplace (any ERC721 can be listed despite `FACTORY_ADDRESS` being present).
- **Potential DoS on refunds** in factories (refund via `transfer` can revert the whole call).
- **No upper bound on `amount` in NFT mint** → gas exhaustion / griefing.
- **Fee constants in ETH** look like placeholders (1000/500/50 ETH) — high risk of broken economics if deployed as-is.

**Preliminary Verdict:** **Changes required** (most importantly: replace `transfer` with safe `call`, handle overpayments, and enforce a collection policy on the marketplace).

| Severity         | Issues |
|------------------|------:|
| 🟥 Critical      | 1 |
| 🟧 High          | 4 |
| 🟨 Medium        | 5 |
| 🟩 Low           | 6 |
| 🟦 Informational | 5 |

---

## 📂 Scope

| Contract | Purpose |
|---|---|
| `LumioERC20Factory`, `CustomERC20` | Deploy configurable ERC20 (custom decimals, mint/burn/pause flags, capped supply) |
| `NFTsCollectionFactory`, `NFTCollection` | Deploy ERC721 collections with ERC2981 royalties, mint pricing, and platform fee |
| `LumioMarketplace` | Escrow listings/auctions, payouts to seller/treasury/royalties, private sales |

---

## 🧩 Findings

### 🟥 Critical

**C1 — Overpayment in `LumioMarketplace.buyNFT` is not refunded and becomes permanently stuck**  
- **Where:** `buyNFT()`  
- **What:** `require(msg.value >= listing.price)` allows overpaying, but no change is returned. After fee/royalty/seller payouts, any remainder stays in the contract. There is **no** generic withdrawal function for this remainder (buyer, seller, or admin).  
- **Risk:** user funds stuck, reputational and legal risk.  
- **Fix:**  
  1) Enforce **exact** payment (`msg.value == listing.price`), or  
  2) Refund `msg.value - listing.price` to the buyer (`_safeTransferETH(msg.sender, excess)`), and  
  3) (Optionally) add a restricted admin-withdraw for residual dust with strict invariants and events.

---

### 🟧 High

**H1 — Using `transfer` for ETH sends (DoS/compatibility)**  
- **Where:**  
  - `LumioERC20Factory.createToken` — refunding overpayment  
  - `LumioERC20Factory.withdrawTreasury`  
  - `NFTsCollectionFactory.createCollection` — refunding overpayment  
  - `NFTsCollectionFactory.withdrawTreasury`  
  - `NFTCollection.mint` — paying collection owner  
- **What:** `transfer` uses a 2300 gas stipend and fails with contracts having non-trivial `receive()/fallback()`. This is fragile given EVM/EIP changes.  
- **Fix:** replace with low-level `call{value: amount}("")` (or OZ `Address.sendValue`) and handle `success`. For **refunds**, avoid reverting the whole flow on refund failure: either enforce exact payment, or accrue refundable balances and let users withdraw.

**H2 — DoS risk on overpayment refunds in factories**  
- **Where:** `LumioERC20Factory.createToken`, `NFTsCollectionFactory.createCollection`  
- **What:** If `msg.value > fee`, refund via `transfer` can revert (e.g., non-payable EOA/contract, stipend issues) → entire deployment reverts even if token/collection was created.  
- **Fix:** enforce exact payment; or process refunds via `call` and on failure **do not revert** deployment (accumulate and allow later withdrawal).

**H3 — No explicit allowlist policy for marketplace collections**  
- **Where:** `LumioMarketplace`  
- **What:** Contract defines `FACTORY_ADDRESS` and uses a factory interface, but does **not** verify that `collection` is produced by that factory. Any ERC721 can be listed (including malicious ones).  
- **Fix:**  
  - Add `mapping(address => bool) allowedCollection`, plus admin moderation, **or**  
  - Verify membership in `factory.getDeployedCollections()` (cache result in mapping for gas).

**H4 — Gas exhaustion risk in `NFTCollection.mint` for large `amount`**  
- **Where:** `NFTCollection.mint` (two `for` loops over `amount`)  
- **What:** Paying platform fee and `_safeMint` inside loops without an upper bound can make transactions fail for big `amount`.  
- **Fix:** introduce a **hard cap** on `amount` (e.g., `<= 20`) or a batched minting strategy appropriate for gas profiles.

---

### 🟨 Medium

**M1 — Economic constants appear unrealistically large**  
- **Where:** `NFTsCollectionFactory`: `DEPLOYMENT_FEE = 1000 ether`, `COLLECTION_FEE = 500 ether`, `NFT_FEE = 50 ether`; `LumioERC20Factory`: `DEPLOYMENT_FEE = 5000 ether`  
- **What:** Likely placeholders for testing; if deployed, they break UX/economics.  
- **Fix:** make them configurable (`set*Fee()`), governed by multisig/timelock; or compute from basis points. Frontend should also constrain values.

**M2 — No refund of overpayment in `NFTCollection.mint`**  
- **Where:** `NFTCollection.mint`  
- **What:** Requires `msg.value >= totalUserCost` and keeps any excess as owner revenue. Might be intentional, but commonly UX expects refunds.  
- **Fix:** enforce exact payment or refund excess.

**M3 — `delistNFT` not gated by `whenNotPaused`**  
- **Where:** `LumioMarketplace.delistNFT`  
- **What:** Delist works even when paused. That may be OK (allow safe exits during incidents), but should be intentional and documented.  
- **Fix:** document incident policy; if needed, keep delist allowed while prohibiting list/buy during pause.

**M4 — `endAuction` is blocked by pause**  
- **Where:** `LumioMarketplace.endAuction` is `whenNotPaused`  
- **What:** During global pause you cannot finalize auctions (even expired), potentially freezing funds/NFTs.  
- **Fix:** allow `endAuction`/`cancelAuction` when paused (fail-safe), while disallowing new listings/bids.

**M5 — No admin withdraw for “stuck dust” on marketplace**  
- **Where:** `LumioMarketplace`  
- **What:** Contract may accumulate dust (e.g., overpayments, rounding). There is no controlled admin withdrawal.  
- **Fix:** add a restricted admin-withdraw with events and strict invariants (e.g., cannot touch escrowed balances).

---

### 🟩 Low

**L1 — ERC20 `_initialSupply` not scaled by `decimals()`**  
- **Where:** `CustomERC20` constructor  
- **What:** `_mint(_owner, _initialSupply)` assumes smallest units are provided. Common source of UX mistakes.  
- **Fix:** either document clearly or accept human-readable amount and scale by `10 ** _decimals`.

**L2 — Flags `mintable/burnable/pausable` immutable**  
- **What:** Might be intentional. If flexibility is required, add controlled toggles with care.

**L3 — Missing events for all admin changes**  
- **Where:** Some are covered (`BaseURIUpdated`, `WhitelistStateUpdated`, `MintStateUpdated`), but e.g., `setMintPrice` emits no event.  
- **Fix:** emit events for all economy-relevant changes.

**L4 — Hardcoded `FACTORY_ADDRESS`**  
- **Where:** `LumioMarketplace`  
- **What:** Upgrading/migrating factory would require a new marketplace deployment.  
- **Fix:** make factory address configurable (guarded by timelock/multisig), or support multiple factories via allowlist.

**L5 — Inconsistent use of `nonReentrant`**  
- **What:** Not critical, but standardize for clarity (apply to external functions that move ETH/tokens).

**L6 — `_formatPrice` rounding is purely cosmetic**  
- **What:** Non-security; prefer handling formatting on the frontend.

---

### 🟦 Informational

**I1 — No timelock/multisig on sensitive admin operations**  
- **Fix:** before production, move owner/admin to a **multisig** (e.g., Gravity Safe) and consider a timelock for fee/treasury/allowlist changes.

**I2 — Document ERC20 decimals semantics**  
- Clarify that `_initialSupply` and `_maxSupply` are in smallest units.

**I3 — Royalty policy mutability**  
- `NFTCollection` sets default ERC2981 royalty, but has no mutation pathway (could be intended). If mutability is desired, add functions + events + timelock.

**I4 — Whitelist mint policy**  
- Works as intended; consider batch updates and aggregate eventing for large lists.

**I5 — Incident runbook**  
- Specify what’s allowed on pause (delist/finalize auctions), owner handover steps, and procedures to resolve stuck states.

---

## 🔒 Recommendations (summary)

1) **Replace all `transfer` with `call`** (or OZ `Address.sendValue`), especially for refunds and payouts.  
2) **Handle overpayments**:
   - In `buyNFT()` enforce exact payment **or** refund change.  
   - In `NFTCollection.mint()` do the same (or at least document intent).  
3) **Collection policy on marketplace**: add allowlist or verify against factory deployments (cache in mapping).  
4) **Cap `amount`** in NFT `mint()` to a reasonable upper bound.  
5) **Pause policy**: allow `endAuction`/`cancelAuction` during pause; disallow list/bid/buy.  
6) **Multisig + timelock** for sensitive ops (fees, treasury, allowlist).  
7) **Emit events** for all economy-impacting parameters (`setMintPrice`, fees, treasury).  
8) **Clarify ERC20 supply/decimals UX** in docs and/or code.

---

## 📊 Summary Table

| Severity | Issues | Fixed | Remaining |
|---|---:|---:|---:|
| Critical | 1 | 0 | 1 |
| High | 4 | 0 | 4 |
| Medium | 5 | 0 | 5 |
| Low | 6 | 0 | 6 |
| Informational | 5 | 0 | 5 |

---

## ✅ Final Assessment (Preliminary)

Overall architecture aligns with OZ best practices, but it needs improvements around ETH transfer safety (`transfer` → `call`), overpayment handling, and explicit marketplace collection controls. After code changes, run:
- Static analysis (Slither/Solhint),
- Fuzz tests for auctions/listings (Foundry),
- Negative scenarios (overpayments, pause flows, cancellations),
- Integration tests with ERC2981 royalties.  
