# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?


>
> An equivalent accounting invariant would be:
>
> `custodian.realHoldings() + vault.reserveBalance() == vault.totalClaimValue()`
> 
> This equality asserts that the values reported to the contracts are internally consistent. The left side represents *the reported off-chain T-Bill holdings* plus *USDC that is still in the on-chain vault*. The right side represents *the current USDC value promised to all token holders*. **If they are equal, the system's accounting says that reported assets cover token-holder claims.**
>
> However, this is weaker than Lab 1's invariant. In Lab 1, `balanceOf(vault)` was an ERC-20 balance maintained by another smart contract, so the chain could independently read and enforce it. Here, `realHoldings()` returns `treasury`, which an address with `CUSTODIAN_ROLE` changes by calling `recordPurchase()` or `recordSale()`. It does not inspect a bank account, brokerage statement, or actual T-Bill. Likewise, `navPerShare()` depends on an attested number.
>
> Therefore, the invariant proves consistency between on-chain reports, not real-world solvency. A dishonest or mistaken custodian could report enough holdings to make the equality pass while the SPV owns nothing. The check remains useful for detecting coding and bookkeeping mismatches, but audits, legal records, and independent attestations are still needed to prove real backing.

<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?


> A legal structure is part of the technical design because the system operates across two environments. Solidity controls the on-chain shares, while custody law and contracts control the real T-Bills. The SPV legally connects them by owning the T-Bills separately and defining token holders’ claims on those assets. Without this connection, the blockchain could correctly show that a person owns tokens, but the person might have no enforceable right to the money or T-Bills represented by them. 
> 
> Therefore, the system is not complete merely because its smart contracts work correctly. Its legal documents must implement the same ownership, custody, and redemption rules that the code assumes.
>
> If the SPV's custodian goes bankrupt, the token does not automatically give its holder possession of a specific T-Bill or an on-chain right to withdraw one. It represents a claim under the fund's legal documents. Whether holders recover their money depends on whether the securities were properly segregated, accurately recorded, and legally protected from the custodian's creditors. Recovery may require an administrator, court process, or transfer to a replacement custodian, so it may be delayed even when the assets still exist.
>
> Therefore, “bankruptcy-remote” reduces risk but does not mean “bankruptcy-proof.” The Solidity token records the claim; the legal structure determines whether that claim can be enforced against the off-chain assets.

<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.


> 1. **Issue:** An attacker could use the reporter key to submit an extreme NAV. 
>   **Mitigation:** Require a multisignature or threshold signature so one stolen key is insufficient.
> 2. **Issue:** The reporter might enter the wrong units, decimals, or reporting period.
> **Mitigation:** Validate reasonable bounds and maximum percentage changes, then require a second reviewer for unusual values.
> 3. **Issue:** The reporter could deliberately overstate NAV so an account receives too many assets, or understate it so another account buys cheaply. 
> **Mitigation:** Use an independent regulated administrator, contractual liability, insurance, and a financial bond that can be forfeited after proven misconduct.
> 4. **Issue:** A lost key, outage, or organization failure can leave the last NAV frozen. 
> **Mitigation:** Enforce an `updatedAt` freshness limit and provide a controlled emergency replacement process.
> 5. **Issue:** An honest reporter may publish incorrect custodian or market data.
>    **Mitigation:** Compare multiple independent data sources and reconcile reports with custodian statements.
>
> These protections should be combined. The contract should also emit events, keep previous reports, and allow monitoring software to alert users. Most importantly, subscription and redemption should stop when a report is missing or fails validation, rather than silently trusting the last number forever.

<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.


> A frozen NAV can be accurate when published and still become unfair later. Every holder’s claim is backed by the same shared pool of fund assets. However, 
> - if we suppose the reported NAV is higher than the actual one, it may result in an overpayment for the earlier redeemer, so that the actual NAV may fall further. Therefore, the loss is left for slower holders. This resembles a race: informed users exit before the contract catches up.
> - If we suppose the stale NAV is lower than the actual one, it may result in an over-submission. Investers can be minted extra tBill shares, and the existing holders' value per share falls. This is dilution: the new subscriber gains value at the expense of existing holders.
>
>
> This differs from a wrong-NAV attack because the old report may have been completely honest. The
> problem is missing time information in the decision. A production contract should read `updatedAt`
> and reject subscriptions and redemption requests after a maximum age. It could also use shorter
> reporting intervals, pause only the affected entry or exit operation, and apply a later verified NAV
> to queued redemptions.

<br><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?


> Assume a holder enqueues 100 shares at a NAV of 1.00, so the queue promises 100 USDC. Before  settlement, NAV rises to 1.10. The holder still receives only 100 USDC even though those shares are now worth 110 USDC. Therefore, the redeemer loses the 10-USDC increase. The holders who stayed gain because the fund retires the redeemer's 100 shares while paying less than their value, leaving more value per remaining share.
>
> If NAV falls instead, the result reverses: the redeemer keeps the higher locked payout, and the remaining holders absorb the loss. The current design therefore transfers all price risk during the waiting period to the people who remain in the fund. Locking the price gives the redeemer certainty, but it can encourage a run when informed users expect NAV to fall.
>
> I would normally calculate the final payout using the verified NAV at settlement, because T+1 means the underlying assets are sold later. At enqueue time, the contract could record shares and an estimated payout, not a guaranteed amount. The user interface should clearly display this risk.
>
> Another design is to use the actual sale proceeds, subtract trading fees, and allocate the result pro rata to the shares in the settlement batch.

<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?


> The two crises show different redemption mechanisms. In 2008, redemptions were eventually suspended so the portfolio could be liquidated in an orderly way; investors had legal fund claims but could not exit immediately. In the 2023 event, on-chain transfers continued, while direct banking redemption process was limited by the weekend and banking hours; normal redemption resumed when banks reopened and the reserve risk was resolved.
>
> For a tokenized T-Bill fund, “close the redemption channel” should mean rejecting new redemption requests. Existing queue entries must not disappear. They should remain auditable, preserve a clearly defined priority, and either settle under the published rules or be cancelled through a documented emergency process that returns escrowed shares.
>
> The lab's `pause()` is too broad for a liquidity squeeze. Because `_update()` guards every balance change, pausing stops transfers, subscription minting, queue escrow transfers, and settlement burns. It can prevent the queue from making progress even when cash arrives, and it unnecessarily blocks ordinary transfers between compliant holders.
>
> I would use separate controls: `pauseSubscriptions`, `pauseNewRedemptions`, `pauseTransfers`, and possibly `pauseSettlement`. Existing claims should remain claimable. Each switch should have its own role, reason event, time limit, and review process. This allows the fund to stop a run at the entrance while continuing an orderly FIFO settlement of requests already accepted.

<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> For a regulated fund, the whitelist should enforce the law without turning ordinary operations into arbitrary censorship. A reasonable line is that an address may receive or hold shares only after its controller has completed required KYC, sanctions screening, and investor-eligibility checks. Both minting and transfers should check the receiver, because otherwise a verified buyer could transfer shares to an unverified address immediately after purchase. Checking the sender also prevents a frozen or sanctioned holder from moving assets away.
>
> However, “not yet verified” and “legally frozen” are different states. A simple Boolean whitelist combines them and loses useful information. I would use explicit statuses such as `Pending`, `Approved`, `Frozen`, and `Rejected`. The transfer rule can then explain why an operation failed and permit carefully designed exceptions, such as moving a frozen balance to a recovery or seizure address under legal authority.
>
> Every change should emit an event with a case or reason code, and off-chain records should preserve the supporting evidence. The contract administrator should control role assignments, not individual whitelist decisions. This separation limits damage if one key is stolen and creates accountability without removing the controls a regulated security needs.

<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.


> 1. **For the powers:** A regulated security may have to comply with sanctions, court orders, stolen key recovery, inheritance, or correction of an illegal issuance. Freezing stops immediate movement; an authorized burn or forced transfer can then enforce the final legal decision. Without these functions, the issuer may be unable to meet obligations that exist outside Solidity.
> 2. **Against the powers:** Holders do not have censorship-resistant ownership if the issuer can erase balances. A stolen or malicious issuer key could target innocent users, hide insolvency, or apply rules inconsistently. The lab also demonstrates a design bug: removing Alice from the whitelist makes `_update()` reject Alice as the burn source, so the freeze accidentally disables seizure.
>
> I would represent approval and freezing separately. Normal transfers would require approved sender and receiver statuses, while an exceptional `forcedBurn` or `forcedTransfer` would use a narrow path that bypasses only the relevant transfer check. It should still verify a case identifier, amount, destination, and authorized role.
>
> The keys should also be separated. A limited compliance key may place a short emergency freeze. A multisignature containing compliance, legal, and an independent trustee should authorize forced burns or transfers, preferably after a delay that permits review. The contract administrator should only grant and revoke roles, and should be held by a separate multisignature or timelock. Events, public policies, periodic audits, per-action limits, and an appeal process provide additional checks. This preserves necessary legal control while making secret or accidental confiscation much harder.

<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name (Click it to direct to the function) |
|---|---|---|
| a | Shares exist while the custodian holds nothing | [`test_Ex2_SharesExistWhileTheCustodianHoldsNothing`](test/exercises/01_BridgeTasks.t.sol#L74) |
| a | `realHoldings()` moves with no cash moving | [`test_Ex2_RealHoldingsIsJustANumber`](test/exercises/01_BridgeTasks.t.sol#L87) |
| b | The claim value floats while the share count stays | [`test_Ex3_TheClaimFloats_TheShareCountDoesNot`](test/exercises/01_BridgeTasks.t.sol#L105) |
| b | One `attest` re-prices every holder at once | [`test_Ex3_OneCallMovesTheWholeBook`](test/exercises/01_BridgeTasks.t.sol#L127) |
| d | A non-whitelisted address cannot receive shares, even by paying | [`test_Ex4_NoKyc_NoShares_EvenIfYouPay`](test/exercises/01_BridgeTasks.t.sol#L149) |
| d | A whitelisted holder cannot send shares to a non-whitelisted address | [`test_Ex4_TransferChecksBothEndpoints`](test/exercises/01_BridgeTasks.t.sol#L165) |
| d | Removing an address freezes that holder — and blocks a burn | [`test_Ex4_FreezeBeatsConfiscation`](test/exercises/01_BridgeTasks.t.sol#L181) |


Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:


> The scenario I consider most likely to be exploited is **trading against a stale NAV**, which breaks plank (b), attestation.
>
> 1. The reporter publishes a correct NAV of 1.00 on Friday.
> 2. During the weekend, market news lowers the value of the underlying portfolio to about 0.95, but no new attestation arrives.
> 3. A well-informed holder sees the news before ordinary users and submits a redemption request at the old 1.00 NAV.
> 4. The queue locks the larger payout. When settlement occurs, the fund pays the difference from assets that economically belong to the remaining holders.
>
> This attack is likely because it only requires information to travel faster through financial markets than through the reporting process. Professional traders are specifically equipped to detect and automate this kind of price difference. The same weakness can be exploited in reverse by subscribing at a stale low NAV after the real value rises.

