# fintech-collections-callback

A runnable phone-call workflow app that places **consent-gated, compliant
payment-reminder calls** to overdue fintech accounts using
[CALL-E](https://www.heycall-e.com), and returns a **structured collections
outcome** per account (promise-to-pay date, dispute, or callback request).

It is built for the exact place voice agents earn their keep in lending and BPO
operations: first-party soft collections and appointment/payment reminders —
where *who you may call, when, and what you may say* is as important as the call
itself. An operator (or an agent administering this workflow) hands the app a
batch of accounts; the app decides which ones are eligible to dial right now,
places those calls under a spend cap, and writes an auditable report.

## What it does

1. Loads a batch of overdue accounts (sample fixtures included).
2. Runs a **pre-dial gate** on each account:
   - explicit, timestamped **consent** to be contacted;
   - valid **E.164** phone number;
   - valid **IANA timezone**, used to enforce **quiet hours** (no calls before
     08:00 or at/after 21:00 local time);
   - a per-run **spend cap** on the number of calls.
3. For each allowed account, calls CALL-E with a task whose compliance rules are
   fixed in code (identify as automated, right-party only, no threats, never
   collect payment on the call), and a `recipientResultSchema` so the result
   comes back structured.
4. Writes a JSON report of every decision and outcome.

CALL-E is imported and invoked at runtime in
[`src/calle.ts`](src/calle.ts); orchestration and the gate live in
[`src/client.ts`](src/client.ts) and [`src/gate.ts`](src/gate.ts).

## Setup

Requires Node.js 20+.

```bash
npm install
```

## Credentials

Only **live** runs need credentials. Copy `.env.example` to `.env` (or export
the variables) and set:

- `CALLE_API_KEY` — your key from the CALL-E dashboard. **Required for `--live`.**
- `CALLE_BASE_URL` — optional; defaults to `https://api.heycall-e.com`.

The key is read from the environment only, never written to disk or logs. A
dry-run needs no key.

## Dry-run / preview behavior (default)

The app is **dry-run by default and places no calls**. It runs the full gate and
prints exactly which accounts *would* be dialed and which are blocked and why —
counting eligible accounts against the same spend cap a live run would use.

```bash
npm run dry-run
# or override the cap:
npm run dev -- --max-calls=3
```

`npm test` runs the gate unit tests with no credentials, network, or calls.

## Going live

```bash
# with CALLE_API_KEY set in your environment or .env
npm run live
npm run dev -- --live --max-calls=5
```

Each eligible account triggers one real outbound call via
`client.calls.createAndWait(...)`, guarded by a deterministic idempotency key
(`collections_<accountId>_<dueDate>`) so a re-run never double-dials.

## Side effects

- **Live mode places real phone calls** that cost money (CALL-E bills per
  billable call) and are subject to telecom regulations in the recipient's
  jurisdiction. You are responsible for having a lawful basis and consent to
  call each recipient.
- Dry-run mode has no external side effects.
- The only local side effect is a report written under `runs/` (see below).

## Cancellation and rollback

- Press **Ctrl-C** to cancel a run: no new call is started, the in-progress
  report is still written, and remaining accounts are left untouched.
- Because the idempotency key is derived from the account and due date, safely
  **re-running the batch will not re-dial** accounts already handled for that
  billing cycle — the effective rollback for "I ran it twice."
- A call already answered cannot be un-placed; the report records it so
  operators can reconcile.

## Where results are stored

Every run writes a timestamped JSON report to `runs/<ISO-timestamp>.json`
containing the mode, spend cap, per-account decisions, structured outcomes, and
estimated cost. The `runs/` directory is git-ignored.

## Project layout

```
fintech-collections-callback/
├── src/
│   ├── client.ts     # entry point: batch loop, spend cap, cancellation, report
│   ├── gate.ts       # consent + E.164 + IANA timezone + quiet-hours checks
│   ├── calle.ts      # the only CALL-E SDK integration (task + structured result)
│   ├── fixtures.ts   # sample accounts (fictional numbers; some intentionally blocked)
│   ├── types.ts      # domain types
│   ├── gate.test.ts  # gate unit tests (no network)
│   └── calle.test.ts # result-mapping unit tests (no network)
├── .env.example
├── package.json
└── tsconfig.json
```

## Scripts

| Script | Purpose |
| --- | --- |
| `npm run dry-run` | Full gate, no calls (default) |
| `npm run live` | Place real calls (needs `CALLE_API_KEY`) |
| `npm run dev -- <args>` | Run directly with custom flags (`--live`, `--max-calls=N`) |
| `npm run build` | Compile to `dist/` |
| `npm start` | Run the compiled build |
| `npm run typecheck` | `tsc --noEmit` |
| `npm test` | Unit tests — gate + result mapping (no network) |

## License

MIT
