# ✅ Lumio Smart Contracts — Delta Audit (post-fix review)

**Scope (updated):**
- `LumioERC20Factory` / `CustomERC20`
- `NFTsCollectionFactory` / `NFTCollection`
- `LumioMarketplace`
- **New:** `DeployLumioTimelock` (OZ TimelockController helper), `DomainBoundNFT`, `VerifierOracle`, `LumioNFTStaking`

**Auditor:** Alexkat
**Date:** 2025-10-27  
**Audit type:** Follow-up (code deltas against previous report)

---

## 1) Executive summary

Great progress. You addressed all previously high-risk items and the one critical. ETH handling is now `call`-based, marketplace refunds overpayment, mint batching is capped, and a governance layer was added.

**Status vs. previous findings**

| Severity | Previous | Now | Notes |
|---|---:|---:|---|
| 🟥 Critical | 1 | **0** | Overpayment in `buyNFT` now refunded. |
| 🟧 High | 4 | **0** | `transfer`→`call`; refund DoS removed; allowlist enforced; mint cap added. |
| 🟨 Medium | 5 | **1** | Fee defaults still unrealistic (mitigated by custom setters + timelock). |
| 🟩 Low | 6 | **2** | Hardcoded factory address persists; some governance ergonomics. |
| 🟦 Info | 5 | **3** | Dual-governance clarity + docs suggested. |

**New contracts quick take**
- **DeployLumioTimelock:** solid helper for OZ `TimelockController` with parameter validation & single-use protection. 👍
- **DomainBoundNFT:** sound non-custodial bid/transfer flow with pull-payments; consider domain canonicalization. 👍
- **VerifierOracle:** simple, fits purpose; mind case/Unicode normalization. 👍
- **LumioNFTStaking (non-custodial):** straightforward staking registry; consider “cleanup” if NFT is transferred mid-stake. 👍

---

## 2) Resolved items (✅)

1. **C1 (critical)** — *Marketplace overpayment stuck* → **Fixed**  
   `buyNFT()` refunds `msg.value - listing.price`.

2. **H1 (high)** — *`transfer` 2300-gas risk* → **Fixed**  
   All ETH sends now use `call` (or equivalent), with success checks.

3. **H2 (high)** — *Refund DoS on factories* → **Fixed**  
   Safe, non-reverting refunds + events.

4. **H3 (high)** — *No collection policy on marketplace* → **Fixed**  
   `allowedCollections` + timelocked admin flow.

5. **H4 (high)** — *Unbounded batch mint gas risk* → **Fixed**  
   `MAX_MINT_PER_TX = 20` and single batched platform fee.

6. **M2 (med)** — *Overpayment in `NFTCollection.mint`* → **Addressed**  
   Optional refund with event; non-reverting.

7. **M3/M4 (med)** — *Pause policy clarity* → **Addressed**  
   `delistNFT` allowed under pause (documented), `endAuction` allowed even if paused.

8. **M5 (med)** — *No “dust” withdraw* → **Fixed**  
   Admin `withdrawDust()` with event.

9. **L1 (low)** — *ERC20 initial supply not scaled by decimals* → **Fixed**  
   Scales by `10**decimals`.

10. **L3 (low)** — *Event for mint price changes* → **Fixed**  
    `MintPriceUpdated` added.

---

## 3) New / remaining findings

### 🟨 Medium

**M1 — Fee defaults still unrealistic (economic risk, not technical)**  
- *Where:* Factories still declare giant placeholder constants, though you now have `custom*Fee` set via a **built-in timelock**.  
- *Risk:* If not updated post-deploy, UX/economics break.  
- *Recommendation:* Immediately propose & execute fee reductions via timelock after ownership transfer; also display current `custom*Fee` on the UI.

### 🟩 Low

**L4 — Hardcoded factory address in `LumioMarketplace`**  
- *Where:* `FACTORY_ADDRESS` is `constant`.  
- *Impact:* Makes migrations harder; you mitigated with allowlist governance, but the constant remains.  
- *Recommendation:* Make factory updatable via timelock (or drop the constant entirely and rely only on the allowlist).

**L5 — Non-custodial staking can “strand” stake records**  
- *Where:* `LumioNFTStaking`  
- *Scenario:* User stakes, then transfers the NFT away. The new owner can’t interact with the old stake record; the original staker can’t `unstake` (fails ownership check).  
- *Impact:* Only a stale record (no asset risk).  
- *Recommendation:* Add a `cleanupStake()` that the original staker (or anyone) can call to set `active=false` if the NFT is no longer owned by `staker`.

### 🟦 Informational

**I1 — Dual governance surfaces (OZ TimelockController + in-contract “mini-timelocks”)**  
- *Where:* `DeployLumioTimelock` (external OZ timelock) **and** proposal-based timelocks embedded in `NFTsCollectionFactory` and `LumioMarketplace`.  
- *Risk:* Operator confusion/ops errors.  
- *Recommendation:* Prefer a **single** governance path: transfer `owner` to OZ TimelockController and **remove/disable** in-contract timelock logic (or gate all proposal executions so only the OZ timelock can call them).

**I2 — Domain string canonicalization**  
- *Where:* `DomainBoundNFT` + `VerifierOracle`  
- *Risk:* `verifyDomain` strict string equality is sensitive to case/Unicode.  
- *Recommendation:* Normalize domains off-chain (lowercase, NFC) prior to writing on-chain; optionally store and compare `keccak256(abi.encodePacked(_lowercased))`.

**I3 — Marketplace allowlist bootstrap**  
- *Where:* `LumioMarketplace`  
- *Note:* You added an allowlist with timelock proposals. Ensure you **seed** it for all existing collections *before* enabling trading in prod (script + on-chain events).

---

## 4) Governance & ops checklist (go-live)

1. **Transfer ownerships to OZ TimelockController**
   - NFT factory, ERC20 factory, marketplace (and optionally staking & DomainBound).
2. **Seed marketplace allowlist**
   - Propose and execute `proposeCollectionAllowlistChange()` for all active collections.
3. **Set realistic fees**
   - Submit fee-change proposals (both factories & marketplace).
4. **Treasury sanity**
   - Propose/exe `proposeTreasuryChange()` to your multisig; verify on explorer.
5. **UI alignment**
   - Read `custom*Fee` & `marketplaceFee` on-chain; show exact price, fee, royalty, and refund behavior.
6. **Monitoring**
   - Alerts for: timelock queued/executed/cancelled ops, balances, and failed ETH sends.

---

## 5) Suggested micro-patches (optional)

- **Marketplace:** add `setFactory(address)` via timelock and drop the `constant` (or keep both: a mutable `factory` plus a read-only “legacy” constant).
- **Staking:** add `cleanupStake(address collection, uint256 tokenId)` (no ETH flow; emits event).
- **DomainBoundNFT:** add `normalizeDomain(string)` off-chain guideline (docs) and consider an internal helper that rejects mixed-case inputs.

---

## 6) Final verdict (follow-up)

Security posture is **significantly improved**. All prior critical/high issues are **resolved**. Remaining items are governance ergonomics and operational polish (fee defaults, factory constant, minor UX edge cases). With the governance flow consolidated under OZ TimelockController and the allowlist seeded, you’re good to proceed to broader testing and mainnet rollout.
