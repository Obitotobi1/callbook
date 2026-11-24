# CallBook

A public track record for predictions. Write a prediction, the web page that will settle it, and exactly what counts as a **hit** and a **miss**. The conditions are locked on-chain, so nobody can move the goalposts later. When the time comes, **anyone** can ask GenLayer validators to read the page and settle the call. Every wallet builds a **public hit rate** that other contracts can read.

Anyone can say "I called it". CallBook makes that a fact many independent validators agreed on, with the exact words from the page as proof.

- **Live app:** https://obitotobi1.github.io/callbook/
- **Contract (GenLayer Studio):** `0xf144E1D206ccD6bB644C8Da39e886a5892ae119c` (v1.1; the earlier instance `0xbFc888C376977a77a9A84cba04Ab12B092FC8AF7` is v1.0)
- **Contract source:** [`callbook.py`](./callbook.py)
- **Tests:** [`tests/test_callbook.py`](./tests/test_callbook.py)

## How it works

```
make_call(statement, url, hit_when, miss_when) --> OPEN --resolve--> HIT   (final)
                                                     |   \--------> MISS  (final)
                                                     +-- page shows neither / unreachable: stays OPEN
```

1. `make_call` stores the prediction (8 to 200 characters), a public https page, and two fixed conditions: what the page must show for a HIT and what it must show for a MISS.
2. `resolve(id)` (**anyone**): validators fetch the page (`gl.nondet.web.render`), a model reads it against the two conditions, and validators must agree on the verdict (`gl.eq_principle.prompt_comparative`, the `verdict` field must be identical).
3. A HIT or MISS is only kept if the model's **quote really appears on the page**. A made-up or unmatched quote turns the verdict into OPEN. A settled call is final.
4. If the page shows neither outcome yet, or cannot be read, the call stays OPEN and can be tried again. Each wallet gets 3 tries per call and there is no shared cap, so nobody can use up a call's chance to settle: another wallet can always try.
5. `get_record(author)` returns a wallet's hits, misses, open calls and hit rate in percent.

## What it protects against

| Risk | How CallBook handles it |
|---|---|
| Moving the goalposts after the event | Hit and miss conditions are fixed when the call is made. |
| Author picking a convenient moment to settle | Anyone can settle, not only the author. |
| A model inventing proof | A HIT or MISS needs a quote that appears word for word on the page, checked in code. |
| Prompt injection from the page or the text fields | The statement, both conditions and the page are each wrapped in their own tag with `<` and `>` escaped, and the prompt says never to follow instructions inside them. |
| A flaky page deciding a call | An unreadable or empty page never settles anything. |
| Validators fetching internal addresses | Only `https` links to public hosts. `localhost`, private and link-local IPs, `.local`/`.internal`, credentials and IPv6 literals are refused. |
| Spam | 30 calls per wallet, no duplicate call on the same page, 3 settle tries per wallet per call (no shared cap), bounded text and page size. |

## Read API (composable)

- `get_call(call_id)` returns the call as JSON, including the verdict quote
- `get_calls_by_author(author)` returns a wallet's calls
- `get_record(author)` returns hits, misses, open calls and `hit_rate_percent` (`-1` while nothing is settled)
- `get_recent(limit)` returns the newest calls from everyone
- `is_hit(call_id)` is `true` only for a settled HIT

## Try it alone in two minutes

The app creates two demo accounts in your browser (no wallet, free on Studio).

1. Open the app, go to **New call**. Use a page that already shows the outcome, for example a project's release page: write the prediction, the page link, a hit condition and a miss condition.
2. Open **My calls**, select the call and click **Settle now**. Wait for consensus (a minute or two). It becomes **Hit** or **Miss**, with the quote validators found.
3. Make a second call on a page that shows neither condition: it stays **Open**, and you can try again later.
4. Switch to **Account B**, open **All calls** and settle someone else's call: anyone can.

Every write shows the transaction lifecycle: signed, submitted, accepted by consensus, finalized. Failed, undecided and timed-out transactions are shown under their real names and never as a success.

## Tests

`tests/test_callbook.py` runs the contract's real logic against a small stand-in for the `genlayer` library, with the web fetch replaced by a dictionary of pages and the model by a scripted reply. No node is needed:

```
python3 tests/test_callbook.py
```

It covers input and link refusal, duplicates and caps, HIT and MISS with a real quote, made-up quotes and garbage output staying open, unreachable pages and retries, the per-wallet try limit, 12 failed attempts by different wallets followed by a successful settlement, prompt isolation, track records and the feed (15 tests).

## v1.1 (milestone): settlement can no longer be used up

A steward review of the contract found that v1.0 counted every `resolve` attempt, including early or unreadable ones, against a global cap of 8, so eight such attempts made a call impossible to settle later. This version of the app runs against the fixed contract (v1.1):

- **Contract:** no shared cap. Each wallet may try a call 3 times, recorded per wallet in the call (`tries`). A different wallet can always try again. HIT and MISS stay final and still need a quote found word for word on the page.
- **App:** the call detail shows "your tries: X of 3" for the selected account. After 3 tries the Settle button is replaced by a hint to switch account. The create form explains the rule. The saved contract address key was renamed, so older browsers pick up the new contract instead of the stored old one.
- **Verified in the app:** the same strict outcome handling as before (a receipt without status, an unchanged record, or validator disagreement are never shown as success), plus a test that an account is stopped at 3 tries.
- **Live check of the contract on Studio:** one call stayed OPEN after an inconclusive attempt (tries recorded as 1), and another settled HIT with a quote from the page.

## Limits

- The contract has no clock, so a call can be settled as soon as the page shows an outcome. Choose a page that only shows the outcome after the event.
- It only sees what validators fetch. Pages that need a login or build their text with scripts are not supported.
- The hit rate counts settled calls only and does not weigh how bold a call was.
- Studio test network only. No payments or enforcement.

## Run the app locally

It is a single static file. Serve the folder (for example `python3 -m http.server`) and open `http://localhost:8000`. The `genlayer-js` library loads from a CDN at runtime, and demo keys are kept in your browser's local storage for the Studio test network only.
