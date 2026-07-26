# Design calls behind the prompt sequence

Three judgement calls shape the whole sequence. None of them is forced by the BrightFlag API; each
was chosen, and each is reversible by a reviewed contract change rather than a configuration tweak.
They are recorded here so a reader can disagree with them deliberately.

The BrightFlag operations and field names cited below come from the external OpenAPI document at
`https://app.brightflag.com/v3/api-docs/external`, read on 2026-07-26.

---

## 1. Approved for payment requires batch release *and* an allow-listed status

**The call.** An invoice is reported as approved for payment only when both hold: its `invoiceID`
appears in an accounts-payable batch returned by `getAPBatches` or `getAPBatchesByDateRange`, and
its `InvoiceSummaryAPI.invoiceStatus` is in a reviewed, tenant-configured allow-list. The allow-list
ships empty and startup fails until an integrator configures it.

**Why.** `getInvoiceSummaryList` accepts an `invoiceStatus` filter, so the obvious implementation is
a single call filtered by a status that sounds approving. The OpenAPI document defines no
enumeration for that field, and BrightFlag status vocabularies are configured per customer. An agent
implementing the obvious version has to invent the string — and a wrong guess produces a tool that
silently reports the wrong set of invoices to whoever is about to pay them.

The accounts-payable batch is the artefact BrightFlag creates when invoices are released to accounts
payable: the endpoint describes them as "invoices batched together ready for retrieval and
processing in the accounts payable system." That is a released-to-finance event, not an opinion
about a status label. `getBatchInvoices` returns only `invoiceID` values, so the summary read is
still needed for number, date, vendor, currency, and amounts — which makes a two-source join
unavoidable anyway. Requiring both signals costs nothing extra and removes the guess.

This is the direct analogue of the rule in `SAPProxyMCPServerBuilder` that an invoice counts as paid
only on clearing evidence, never on posting status.

**Cost.** Stricter than the API requires. A tenant that does not use the accounts-payable batch feed
will get an empty result from a correctly working server. That failure is loud rather than silent,
which is the point, but it will need explaining during a first integration.

**What makes it visible.** Batched invoices whose status is outside the allow-list are excluded
*and* counted, with the observed status reported. A misconfigured allow-list therefore shows up as a
non-zero excluded count rather than as a short list nobody questions.

**Rejected alternatives.** Filter on a hard-coded status string; ship a default allow-list
containing plausible values; match statuses case-insensitively or by substring on "approv". Each
trades a visible configuration step for an invisible wrong answer.

**Changing it** means changing the definition in Prompt 4, the ontology vocabulary in Prompt 7, and
the documentation in Prompt 9, under `narrative-required`.

---

## 2. The payment write is gated hard, and deliberately cannot retry

**The call.** `brightflag_mark_invoice_paid` accepts only a server-issued plan token plus explicit
confirmation. `paymentStatus` is fixed to `PAID`. Exactly one POST is issued per confirmed payment,
with no automatic retry on timeout, 409, or 5xx. An ambiguous outcome is returned as ambiguous,
naming the `paymentRef`, and requires reconciliation in BrightFlag before a fresh plan.

**Why.** Three properties of `POST /api/v1/invoicePayment/invoice-payment-status` drive this. It
accepts one invoice per call, so there is no bulk path to reason about. It is not documented as
idempotent and carries no client-supplied idempotency key, so a retried submission may or may not
produce a second payment assertion. And it echoes the submitted `InvoicePaymentStatus` on success,
which gives the server something concrete to verify rather than trusting a 200.

Ordinary resilience engineering — retry the 5xx, retry the timeout — is the wrong instinct against a
non-idempotent financial write. A timeout after the server has received the request is
indistinguishable from a timeout before it. Retrying resolves that ambiguity by guessing, in the
direction that risks a duplicate payment record in a system finance teams reconcile against real
money. Refusing to retry leaves the ambiguity where a human can settle it.

Fixing `paymentStatus` to `PAID` excludes `PARTIALLY PAID`, `POSTED`, and `VOID`. Partial payment
needs an amount model this version does not have; posting is a different accounting event; and
`VOID` is not a payment at all, so reaching it through a tool called "mark invoice paid" would be a
misfeature.

**Cost.** An operator has to reconcile manually after an ambiguous submission, and a caller cannot
express a partial payment at all. Both are real limitations, deliberately taken in the first
version.

**Rejected alternatives.** Retry idempotent-looking failures with backoff; derive an idempotency key
from `paymentRef` and hope BrightFlag deduplicates on it; expose `paymentStatus` as a caller
argument validated against the four documented values. The third is the most tempting and the worst:
it turns one reviewed capability into four unreviewed ones.

---

## 3. Build order is read, then write, then schema

**The call.** The prompt sequence implements the approved-for-payment read (Prompts 4–5) before the
payment write (Prompt 6), and generates the ontology schema last (Prompt 7). The `README.md`
capability list keeps the order the work was requested in; the prompts do not.

**Why.** The payment tool's plan step re-verifies that an invoice is currently approved for payment
using the Prompt 4 rule and a fresh read. Building the write first would mean either stubbing that
check or writing it twice, and a stub on the authorisation path of a financial mutation is exactly
the kind of temporary code that survives to production.

The schema comes last because it is generated from the settled contracts and reviewed OpenAPI
snapshot. Generating it earlier guarantees churn: every later stage that adds a field or renames an
entity invalidates it, and a schema that is regenerated five times teaches nothing about
determinism. Generating it once, against finished contracts, with a build-time drift gate, does.

Ordering also matters pedagogically. A learner reaches the only irreversible operation in the
project after having already seen the evidence rule, the fake server, the audit record, and the
authorization model — not before.

**Cost.** The capability the reader asked for first is built third. Anyone skimming the prompt list
expecting it to match the README's capability order will be briefly confused, which is why both
documents say so explicitly.
