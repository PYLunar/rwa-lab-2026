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

> Your answer:
A comparable accounting relation is 'vault.reserveBalance() + custodian.realHoldings() == vault.totalClaimValue()'.
This equation asserts that reported assets equal the value of recorded claims. The chain can check the calculation, but the current contracts do not guarantee that the relationship always holds.
More importantly, the equation no longer proves that the underlying Treasury bills actually exist. A permissioned address can increase realHoldings() without buying securities, while the reporter can change NAV without independently verifying their value. Matching numbers can therefore create the appearance of backing even when the reports are false.
It also cannot establish legal ownership, asset segregation, freedom from liens, or immediate liquidity. Those properties require custody records, independent reconciliation, audits and enforceable legal arrangements.


<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:
A legal structure is part of the technical design because a token needs an enforceable connection to assets outside the blockchain. Smart contracts can record balances and control transfers, but they cannot determine legal title, creditor priority, or ownership during insolvency. The SPV separates the fund’s assets from the issuer’s business, while legal documents define whose benefit those assets serve and how token holders can enforce their claims.
If the custodian goes bankrupt, the token entitles its holder to the rights established by the fund documents and applicable law, such as a beneficial interest in the SPV’s assets or a right to distributions. It does not automatically give direct ownership of a particular Treasury bill or guarantee immediate payment

<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:
Deliberate manipulation or bribery: The reporter could inflate NAV to create excessive redemption payouts. Require agreement between independent reporting parties, supported by custody reconciliation. Financial bonds and legal accountability can discourage misconduct, although they cannot replace verification.
Key compromise: An attacker could publish false prices using a stolen reporting key. Use hardware-protected signing, multisignature approval and a tested emergency revocation process. Reporting permissions should remain separate from administrative and reserve-management powers.
Incorrect calculations or inputs: An honest reporter might submit the wrong decimal scale, an incorrect valuation or corrupted source data. Apply explicit unit checks, independent calculation checks and limits on unusual NAV changes. Exceptional movements should require review, since legitimate market shocks can also exceed normal limits.

<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:
A frozen NAV may have been correct when published, but it becomes an exploitable trading price after the underlying value changes. An informed investor can benefit without stealing the reporter’s key or submitting false information.
Suppose the displayed NAV remains $1 while the assets are now worth $1.10 per share. A whitelisted investor pays 1,000 USDC and receives 1,000 shares. At the current valuation, that payment should buy only about 909.09 shares. The extra shares dilute existing holders. After the feed updates, the investor can request redemption at the higher NAV, subject to settlement availability and costs.
The opposite opportunity appears when the true value falls. If shares are worth $0.90 but the displayed NAV remains $1, an existing holder can enqueue 1,000 shares and lock a 1,000 USDC payout instead of $900. Correcting NAV afterwards does not change the locked ticket. If settlement is funded, the fund pays the outdated amount, shifting losses to remaining holders.
<br><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:
If NAV rises after enqueueing, the redeemer receives the earlier, fixed amount and misses the subsequent gain. For example, 1,000 shares submitted at a NAV of $1 create a 1,000 USDC ticket. If NAV reaches $1.10 before settlement, the payout remains 1,000 USDC. Assuming the increase reflects real asset gains, the additional value remains in the fund for the holders who stayed. The redeemer loses potential upside, rather than necessarily losing principal.
If NAV falls, the allocation reverses: the redeemer keeps the higher locked payout, while remaining holders bear more of the loss. This can encourage early exits, particularly when the published NAV is stale.
Locking the price can be reasonable if enqueueing clearly marks the end of market exposure. However, the redeemer still bears settlement delay and default risk, so fixing the price does not eliminate all uncertainty.

I would price requests in batches at the next independently verified dealing NAV, using published cutoffs. Once priced, each request would become a fixed cash obligation. All investors in the same batch should receive consistent treatment, with clear rules for transaction costs and delays. Using whatever NAV happens to exist at an operator-chosen settlement time could create opportunities for favouritism.

<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:
Taking the question’s comparison, the 2008 redemption freeze protected immediate liquidity and reduced pressure for forced asset sales, but left investors unable to exit. USDC’s continued redemption channel supported confidence and arbitrage toward its peg, provided redemptions could actually be honoured. Keeping redemption open can nevertheless accelerate withdrawals when holders fear losses.
For a tokenized T-Bill fund, closing the channel could mean suspending new requests or delaying settlement while securities are sold and cash arrives. These actions should be distinguished. Existing tickets, escrowed shares, locked payouts and FIFO priority should remain recorded. The fund should explain the restriction, review it regularly and publish reopening conditions. A closure should not silently erase obligations or allow favoured investors to jump the queue.
The lab’s `pause()` is too broad for a liquidity squeeze. It blocks tBILL transfers, minting and the burns needed for settlement, disrupting operations even when the problem is simply delayed cash. Claiming USDC already settled in the queue can still work because it does not change tBILL balances.
I would use separate controls for subscriptions, new redemption requests, ordinary transfers and settlement. Existing requests should settle in order as cash becomes available, while reserved funds for settled claims should generally remain withdrawable. Emergency intervention should have clear authority, audit records, governance oversight and review deadlines. A cash shortage needs controlled scheduling and liquidity management, rather than an indefinite freeze of every token operation.

<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:
For a regulated fund, eligibility should follow legal requirements and published investment rules. Users who have not completed KYC should wait for approval, and checking both endpoints prevents approved accounts from passing shares to ineligible recipients.
However, an incomplete application should not be treated like a confirmed sanctions violation. Users need clear requirements, consistent decisions, and a review process for mistakes where legally permitted.
I would give COMPLIANCE_ROLE to a dedicated compliance multisignature, separate from minting and NAV reporting. Role assignments should require independent approval. Routine whitelist updates should follow documented procedures. Changes to eligibility policy should require governance approval and legal review. Emergency restrictions should be recorded and reviewed promptly.
Personal identity documents should remain off-chain. The fund should also explain how existing holders can recover or redeem their claims after losing eligibility, subject to applicable law. Removing an address should restrict its operations without automatically destroying its underlying economic rights.

<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:
Freezing and burning can enforce sanctions, court orders and stolen-asset recovery, but also enable censorship, mistakes and insider abuse.
Here, freezing also blocks burning because both pass through `_update`. Lawful seizure therefore needs a narrowly defined recovery exception.
I would give `COMPLIANCE_ROLE` to a compliance multisignature and routine `MINTER_ROLE` only to audited Vault and Queue contracts. A separate governance multisignature with a timelock should manage role grants. Exceptional seizures should require a separate recovery multisignature, documented legal authority and public audit records. Mistakes should have a clear review and remediation process.
<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | test_Ex2_SharesExistWhileTheCustodianHoldsNothing|
| a | `realHoldings()` moves with no cash moving | test_Ex2_RealHoldingsIsJustANumber |
| b | The claim value floats while the share count stays | test_Ex3_TheClaimFloats_TheShareCountDoesNot|
| b | One `attest` re-prices every holder at once | test_Ex3_OneCallMovesTheWholeBook|
| d | A non-whitelisted address cannot receive shares, even by paying | test_Ex4_NoKyc_NoShares_EvenIfYouPay|
| d | A whitelisted holder cannot send shares to a non-whitelisted address | test_Ex4_TransferChecksBothEndpoints|
| d | Removing an address freezes that holder — and blocks a burn | test_Ex4_FreezeBeatsConfiscation |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:
I would examine a whitelisted investor who also controls REPORTER_ROLE, through key theft or bribery. The attacker first subscribes at a reasonable NAV, then publishes an inflated NAV and enqueues their shares. Because the queue locks its payout when the request is created, correcting the NAV later would not automatically reduce that ticket’s promised amount.
If the Vault has sufficient cash and an authorised settler processes the queue, the attacker can claim an excessive USDC payment. The extra cash comes from the fund’s reserves, reducing what remains available to legitimate holders. If cash is insufficient, the ticket may instead block later requests under FIFO rules.
This breaks plank (b), attestation, and exploits its connection to plank (c), redemption. The queue’s two invariants can still hold because shares and settled cash are accounted for correctly; the economic input was dishonest. I chose this scenario because it turns one trusted operational key into a direct claim on pooled reserves. Independent NAV confirmation and review of unusual price changes should occur before a manipulated price can create redeemable claims.