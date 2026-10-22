# ledger-proofs

Public, append-only anchors for the Vigonomics pick ledger (main chain). Every hour the publisher seals the picks
posted in the last closed UTC hour, and once a day it stamps the day with OpenTimestamps (anchored in Bitcoin).

* `roots/main/YYYY/MM/DD/YYYY-MM-DDTHHZ.json`: one file per UTC hour with picks: the hour's row hashes in sequence
  order, their RFC 6962 SHA-256 Merkle root, and the chain head before and after the hour.
* `daily/main/YYYY/YYYY-MM-DD.json`: the day's hourly files (with their SHA-256), a Merkle root over the hourly roots
  and `day_root` (SHA-256 of the day's row hashes joined by newlines, as in the ledger's daily digest).
* `daily/main/YYYY/YYYY-MM-DD.json.ots`: an [OpenTimestamps](https://opentimestamps.org) proof of that manifest,
  upgraded to a Bitcoin attestation once confirmed (usually within hours; see step 5 below to verify it).
* `heartbeat/main.json`: rewritten by every hourly run (at :07 UTC), with or without picks, so a quiet hour can be
  told from a stopped publisher. Fields: `format` (`vig-anchor-heartbeat-v1`), `chain`, `hour` (the last closed UTC
  hour), `n_picks_hour` (picks created in that hour, 0 allowed), `chain_head_seq` + `chain_head` (the newest row hash
  created before the hour ended; 0 + 64 zeros for an empty chain), `published_at`, `publisher`. Older than about
  3 hours = the publisher is not running.
* `heartbeat/days/main/YYYY/YYYY-MM-DD.jsonl`: the same record, one JSON line per hourly run, appended through the
  day; a missing hour is a run that didn't happen.

How a pick is committed: `content_hash = sha256(hash_input)`, `row_hash = sha256(prev_hash + ":" + content_hash)`
(seq 1 links to 64 zeros). Every row hash is public as soon as the pick is posted, through the hash feed
`POST /rest/v1/rpc/ledger_chain_hashes`, including picks whose game hasn't started yet. The pick itself (teams,
market, price) becomes public only once it is **settled** (graded final, or voided): until then only its hash is
out. Subscribers see picks earlier, so public and subscriber record numbers can differ; both are the same chain.
Root, manifest and proof files are never edited or deleted (the `.ots` proof is upgraded in place once); the
heartbeat file is the only file rewritten, and the day logs only grow. Git history shows when each was published.

## Verify one pick

You need the pick's chain `seq`, the public API URL and anon key (the same ones vigonomics.com uses), `sha256sum`
and, for the last step, the [OpenTimestamps client](https://github.com/opentimestamps/opentimestamps-client).

1. **Get the pick and its hashes.**
   `GET /rest/v1/record_picks?chain=eq.main&seq=eq.<seq>&select=seq,created_at,content_hash,prev_hash,row_hash,hash_input`
   (settled picks only). For a vig-pick-v2 row `hash_input` is the exact sealed text; for an early vig-pick-v1 row it
   is withheld (it names the strategy) and you check steps 3 to 5 only.
2. **Recompute the content hash.** `printf '%s' "$HASH_INPUT" | sha256sum` must equal `content_hash`, and the
   fields in `hash_input` (teams, market, line, price, book, start time, `created_at`) must match what the record shows.
3. **Recompute the row hash.** `printf '%s' "$PREV_HASH:$CONTENT_HASH" | sha256sum` must equal `row_hash`, and
   `prev_hash` must be the `row_hash` of seq - 1 (`ledger_chain_hashes` with `p_after_seq = seq - 2`).
4. **Find it in the hour file.** Open `roots/main/YYYY/MM/DD/YYYY-MM-DDTHHZ.json` for the UTC hour of `created_at`:
   `row_hash` must be in `row_hashes`, and the RFC 6962 Merkle root of that list must equal `merkle_root`.
5. **Check the timestamp.** The day's `daily/main/YYYY/YYYY-MM-DD.json` must list that hour file with the SHA-256
   of its bytes (`sha256sum` the hour file). Then check the manifest's proof, either way:
   * with a Bitcoin node: `ots verify daily/main/YYYY/YYYY-MM-DD.json.ots` must print `Success! Bitcoin block
     <height> attests existence as of <time>`. Without a node the client only prints `Could not connect to Bitcoin
     node` (and still exits 0), which is not a verification;
   * without a node: `ots --no-bitcoin verify daily/main/YYYY/YYYY-MM-DD.json.ots` checks the proof is for this
     manifest (`File does not match original!` otherwise) and prints `check that Bitcoin block <height> has merkleroot
     <hex>`; look that block up on any explorer (e.g. https://blockstream.info/api/block-height/<height>, then
     `/api/block/<hash>`) and compare `merkle_root`. The web verifier at https://opentimestamps.org does the same.
   This proves the manifest, so every row hash in it, existed at that block's time: a pick posted before its game
   started can't have been written later.

Or let the ledger do all of it: `python -m vig_ledger verify --anchors` (in the Vigonomics repo, `ledger/`)
recomputes every hash, root, manifest and heartbeat field from the public feed and this repo.
