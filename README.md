# CallBook

A public track record for predictions. Write a prediction, the web page that will settle it, and exactly what counts as a **hit** and a **miss**. The conditions are locked on-chain, so nobody can move the goalposts later. When the time comes, **anyone** can ask GenLayer validators to read the page and settle the call. Every wallet builds a **public hit rate** that other contracts can read.

Anyone can say "I called it". CallBook makes that a fact many independent validators agreed on, with the exact words from the page as proof.

- **Live app:** https://obitotobi1.github.io/callbook/
- **Contract (GenLayer Studio):** `0xbFc888C376977a77a9A84cba04Ab12B092FC8AF7`
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
4. If the page shows neither outcome yet, or cannot be read, the call stays OPEN and can be tried again (up to 8 attempts).
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
| Spam | 30 calls per wallet, no duplicate call on the same page, 8 settle attempts per call, bounded text and page size. |

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

It covers input and link refusal, duplicates and caps, HIT and MISS with a real quote, made-up quotes and garbage output staying open, unreachable pages and retries, the resolve limit, prompt isolation, track records and the feed (13 tests).

## Limits

- The contract has no clock, so a call can be settled as soon as the page shows an outcome. Choose a page that only shows the outcome after the event.
- It only sees what validators fetch. Pages that need a login or build their text with scripts are not supported.
- The hit rate counts settled calls only and does not weigh how bold a call was.
- Studio test network only. No payments or enforcement.

## Run the app locally

It is a single static file. Serve the folder (for example `python3 -m http.server`) and open `http://localhost:8000`. The `genlayer-js` library loads from a CDN at runtime, and demo keys are kept in your browser's local storage for the Studio test network only.
