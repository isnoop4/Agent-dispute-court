# Agent Dispute Court

A reusable on-chain arbitration primitive for [GenLayer](https://www.genlayer.com/). Two parties record an agreement, either side can file a dispute with a claim and evidence links, the other side responds, and GenLayer validators independently fetch the evidence and reach consensus on a verdict.

One deployed contract serves unlimited agreements and disputes. Verdicts use a fixed vocabulary so downstream contracts (for example an escrow layer reading `get_winner()`) can act on them programmatically.

## Deployed contract

GenLayer Studio:
[`0xf23a7B406C7d3d6E3Ed04988F86B83bb583158a4`](https://explorer-studio.genlayer.com/address/0xf23a7B406C7d3d6E3Ed04988F86B83bb583158a4)

## Verdicts

`CLAIMANT_FAVORED`, `RESPONDENT_FAVORED`, `SPLIT`, `INSUFFICIENT_EVIDENCE`

## Dispute lifecycle

```
FILED ──> AWAITING_RESPONSE ──respond──────────────> READY_FOR_RULING ──resolve──> RESOLVED ──appeal──> READY_FOR_RULING ──resolve──> FINAL
               │                                            ▲
               ├──withdraw (claimant)──> WITHDRAWN          │
               └──window expired (claimant)─────────────────┘
```

| Step | Method | Who |
| --- | --- | --- |
| Create agreement | `create_agreement(party_b, terms, criteria, response_window_seconds)` | Party A |
| File dispute | `file_dispute(agreement_id, claim, evidence_urls)` | Either party |
| Respond | `submit_response(dispute_id, response, evidence_urls)` | Respondent |
| Withdraw | `withdraw_dispute(dispute_id)` | Claimant, before a response |
| Expire window | `expire_response_window(dispute_id)` | Claimant, after the window |
| Rule | `resolve_dispute(dispute_id)` | Either party |
| Appeal (once) | `appeal_dispute(dispute_id, reason)` | Either party |

Safeguards:

- **No stuck disputes.** The agreement sets a response window (1 hour to 30 days, default 7 days). If the respondent stays silent, the claimant can move the case to ruling. Silence alone is not treated as proof; the arbiter is told so explicitly.
- **Withdrawal.** The claimant can withdraw a dispute that has not been answered.
- **One appeal.** A resolved dispute can be appealed once with a stated reason. The second ruling is `FINAL`, and the previous ruling is shown to the arbiter.
- **Audit log.** Every state change is appended to an on-chain event log returned by `get_dispute`.

## Trust model

Also available on-chain through `get_trust_model()`.

- **Who decides:** the verdict comes from validator consensus on an LLM ruling, not from a single judge or the contract owner. Each validator re-runs the ruling independently and must agree on the verdict label.
- **What the arbiter may use:** only the agreement terms, the optional agreement-specific `criteria`, both statements, and the fetched evidence pages. No outside knowledge.
- **What is untrusted:** all statements and evidence text are treated as data. Instructions embedded in them (prompt injection) are ignored.
- **Agreement-specific rules:** parties can write `criteria` when creating the agreement (for example "delivery counts as late after 48 hours"), so rulings are not based on a generic standard.
- **Confidence gate:** a `LOW` confidence ruling never picks a winner. It is downgraded to `INSUFFICIENT_EVIDENCE`, and the response records that it was downgraded.
- **Access control:** only parties to the agreement can file, rule, or appeal. The claimant alone can withdraw or expire the window; only the respondent can answer.

## Evidence handling

- Up to 3 `https` URLs per side, passed as a space or comma separated string.
- Each page is fetched by the validators at ruling time and truncated to 4000 characters.
- If a page cannot be read it is marked `UNAVAILABLE`, is not counted for either side, and the ruling still completes.
- The verdict stores a report for every evidence URL (`OK` or `UNAVAILABLE`) with a short content digest, so a ruling can be audited later.

## Known limitations

- Web evidence can change after it is submitted. The stored digest makes a ruling auditable but does not pin the content. Parties who need permanence should link to archived or content-addressed copies.
- The response window relies on the transaction timestamp exposed by the runtime. If it is unavailable, `expire_response_window` fails with a clear error instead of guessing.
- Rulings are LLM judgments reached by consensus. They are not legal judgments.
- The contract records verdicts only. It does not hold or move funds; an escrow contract is expected to read `get_winner()`.

## Read methods

`get_agreement`, `get_dispute`, `get_status`, `get_verdict`, `get_winner`, `get_counts`, `get_trust_model`

## Quick test in GenLayer Studio

1. Deploy `AgentDisputeCourtV5.py`.
2. `create_agreement`: party B address, terms, criteria (can be empty), window `3600`.
3. `file_dispute`: agreement `0`, a claim, and one `https` evidence URL.
4. From party B's account, `submit_response`; or from party A, `withdraw_dispute`.
5. `resolve_dispute` with dispute `0`, then `get_dispute` to see the verdict, evidence report and log.
6. `appeal_dispute` once, then `resolve_dispute` again for the `FINAL` ruling.

## Files

- `contracts/AgentDisputeCourtV5.py`: the Intelligent Contract
- `docs/index.html`: GitHub Pages frontend
