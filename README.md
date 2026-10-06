# Agent Dispute Court

A reusable on-chain arbitration contract for [GenLayer](https://genlayer.com). Two parties record an agreement, either side can file a dispute, and GenLayer validators read both statements plus any linked evidence and record a ruling through Optimistic Democracy consensus.

One deployment serves any number of agreements and disputes. There is no need to redeploy per case.

- **Contract (GenLayer Studio):** [`0x1D450d694E6eD8B20366fd4dF334A7dDc1b76Ab6`](https://explorer-studio.genlayer.com/address/0x1D450d694E6eD8B20366fd4dF334A7dDc1b76Ab6)
- **Frontend:** [`docs/index.html`](docs/index.html), a single static file that talks to the deployed contract
- **Live demo:** https://isnoop4.github.io/Agent-dispute-court/

## How it works

1. **Create an agreement.** Party A calls `create_agreement(party_b, terms)` and receives an `agreement_id`.
2. **File a dispute.** Either party calls `file_dispute(agreement_id, claim, evidence_url)` and receives a `dispute_id`. The evidence link is optional.
3. **Respond.** The other party calls `submit_response(dispute_id, response, evidence_url)`.
4. **Rule.** Either party calls `resolve_dispute(dispute_id)`. The leader fetches the evidence URLs and asks an LLM for a verdict. Validators repeat the process independently and must reach the same verdict for it to be accepted.

Each dispute moves through three states:

```
AWAITING_RESPONSE  ->  READY_FOR_RULING  ->  RESOLVED
   (file_dispute)      (submit_response)    (resolve_dispute)
```

### Verdicts

| Verdict | Meaning |
|---|---|
| `CLAIMANT_FAVORED` | Terms and evidence support the party who filed |
| `RESPONDENT_FAVORED` | Terms and evidence support the responding party |
| `SPLIT` | Both sides share responsibility |
| `INSUFFICIENT_EVIDENCE` | Not enough information to rule fairly |

The stored verdict is JSON: `{"verdict": "...", "confidence": "HIGH|MEDIUM|LOW", "reason": "..."}`.

## Contract interface

### Write methods

| Method | Who can call | Notes |
|---|---|---|
| `create_agreement(party_b: str, terms: str) -> int` | Anyone (becomes party A) | `party_b` must differ from the caller, terms must not be empty |
| `file_dispute(agreement_id: int, claim: str, evidence_url: str) -> int` | Party A or B | The caller becomes the claimant |
| `submit_response(dispute_id: int, response: str, evidence_url: str)` | The respondent only | Dispute must be `AWAITING_RESPONSE` |
| `resolve_dispute(dispute_id: int) -> str` | Party A or B | Dispute must be `READY_FOR_RULING` |

### View methods

| Method | Returns |
|---|---|
| `get_counts()` | `{agreements, disputes}` |
| `get_agreement(agreement_id)` | `{party_a, party_b, terms, dispute_count}` |
| `get_dispute(dispute_id)` | Full case file: parties, claim, response, evidence links, status, verdict |
| `get_status(dispute_id)` | Current state string |
| `get_verdict(dispute_id)` | Verdict JSON string |
| `get_winner(dispute_id)` | Address of the winning party, or `SPLIT` / `INSUFFICIENT_EVIDENCE` / `PENDING` |

## Design notes

- **Registry pattern.** State is held in `TreeMap`s keyed by agreement and dispute ids, so one contract handles many cases.
- **State changes only after consensus.** Nothing is written inside `leader_fn` or `validator_fn`. The verdict and status are stored after `run_nondet_unsafe` returns.
- **Evidence is fetched on-chain.** Validators fetch the linked URLs themselves with `gl.nondet.web.render`, rather than trusting text pasted by a party.
- **Prompt-injection hardening.** Terms, statements, and fetched evidence are wrapped in tags and the prompt tells the model to treat them as data, not instructions.
- **Verdict-level equivalence.** Validators agree when their verdict label matches. The free-text reason may differ between validators.
- **Ruling access.** Only the two parties of an agreement can request a ruling, which prevents outsiders from spamming `resolve_dispute`.

## Limitations

- The contract records a ruling but does not hold or move funds. Enforcement (escrow, payouts) would be a separate layer that reads `get_winner`.
- Evidence is read at ruling time. If a linked page changes or goes offline between filing and ruling, the result can differ.
- Rulings come from LLM judgment and are not legal advice.
- Validators compare verdict labels exactly, so ambiguous cases may need extra validator rounds before consensus.

## Run the frontend

The frontend is one static file with no build step. It loads [`genlayer-js`](https://github.com/genlayerlabs/genlayer-js) from `esm.sh` and connects to GenLayer Studio.

```bash
# any static server works, for example:
cd docs && python3 -m http.server 8000
# then open http://localhost:8000
```

On first load it creates a local account and stores its key in the browser's `localStorage`. Use it only for testing on Studio. You can also connect a browser wallet.

To test a full case you need two addresses: act as party A, copy party B's address into the agreement form, then use **Switch to a new local account** (or the wallet) to act as party B for the response step.

> Do not use the local account for anything of value. Its private key sits in the browser.

## Deploy to GitHub Pages

1. Keep the frontend at `docs/index.html`.
2. In the repo go to **Settings → Pages**.
3. Under **Build and deployment**, choose **Deploy from a branch**, select `main` and the `/docs` folder, then save.
4. After a minute the site is live at https://isnoop4.github.io/Agent-dispute-court/.

## Test the contract

In GenLayer Studio, with two accounts:

1. `create_agreement` as A with B's address and some terms.
2. `file_dispute` as A with `agreement_id = 0`.
3. `submit_response` as B with `dispute_id = 0`.
4. `resolve_dispute` with `dispute_id = 0`.
5. Check `get_status`, `get_verdict`, `get_winner`, and `get_counts`.
6. To confirm reusability, create a second agreement and dispute and check that dispute 0 is unchanged.

## Repository layout

```
.
├── contracts/
│   └── AgentDisputeCourtV4.py   # the Intelligent Contract
├── docs/
│   └── index.html               # frontend (served by GitHub Pages)
└── README.md
```

## License

MIT
