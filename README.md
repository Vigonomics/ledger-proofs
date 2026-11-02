# ledger-proofs

Public, append-only timestamps for the Vigonomics pick record (https://vigonomics.com/record). Every hour the picks
recorded in the last closed UTC hour are sealed into one Merkle root, and once a day the day is stamped with
OpenTimestamps (anchored in Bitcoin).

* `roots/main/YYYY/MM/DD/YYYY-MM-DDTHHZ.json`: one file per UTC hour with picks: the hour's row hashes in order,
  their RFC 6962 SHA-256 Merkle root, and the chain head before and after the hour.
* `daily/main/YYYY/YYYY-MM-DD.json`: the day's hour files (with their SHA-256), a Merkle root over the hourly roots
  and `day_root` (SHA-256 of the day's row hashes joined by newlines).
* `daily/main/YYYY/YYYY-MM-DD.json.ots`: an [OpenTimestamps](https://opentimestamps.org) proof of that manifest,
  upgraded to a Bitcoin attestation once confirmed (usually within hours).
* `heartbeat/main.json` and `heartbeat/days/main/YYYY/YYYY-MM-DD.jsonl`: written every hour, with or without picks,
  so a quiet hour can be told from a missed one.

Each pick is chained to the one before it: `content_hash = sha256(sealed text)` and
`row_hash = sha256(prev_hash + ":" + content_hash)` (the first pick links to 64 zeros). Files here are never edited
or deleted (an `.ots` proof is upgraded in place once; the heartbeat is rewritten). Git history shows when each file
was published.

## Verify one pick

1. **Recompute its hashes.** On https://vigonomics.com/record, open the pick's *Verify* and then *How to verify*: it
   shows the sealed text, `content_hash`, `prev_hash` and `row_hash`, and recomputes them in your browser. By hand:
   `printf '%s' "$SEALED_TEXT" | sha256sum` must equal `content_hash`, and
   `printf '%s' "$PREV_HASH:$CONTENT_HASH" | sha256sum` must equal `row_hash`.
2. **Find it in the hour file.** The pick's hourly proof link (https://vigonomics.com/proof) shows the hour file for
   the UTC hour it was recorded in: `row_hash` must be in `row_hashes`, and the RFC 6962 Merkle root of that list
   must equal `merkle_root`.
3. **Check the timestamp.** The day's `daily/main/YYYY/YYYY-MM-DD.json` must list that hour file with the SHA-256 of
   its bytes. Then, with a Bitcoin node, `ots verify daily/main/YYYY/YYYY-MM-DD.json.ots` must print
   `Success! Bitcoin block <height> attests existence as of <time>`. Without a node,
   `ots --no-bitcoin verify daily/main/YYYY/YYYY-MM-DD.json.ots` prints the block height and Merkle root to compare
   on any block explorer; https://opentimestamps.org does the same in a browser.

This proves every row hash in the manifest existed at that block's time: a pick recorded before its game started
can't have been written later.

## Strategy Board picks

Strategy Board picks have their own chain, anchored the same way: `roots/board/...`, `daily/board/...` (+ `.ots`),
`heartbeat/board.json` and `heartbeat/days/board/...`, with `"chain": "board"`. A Board pick's time is when it was
sealed. Board anchoring started on 19 Oct 2026: days before the first Board publish carry a timestamp from that
first publish, not from the day itself. To check a Board pick, use steps 2 and 3 with `roots/board/` and
`daily/board/`.
