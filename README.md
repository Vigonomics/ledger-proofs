# ledger-proofs

Public, append-only anchors for the Vigonomics pick ledger (main chain).

* `roots/main/YYYY/MM/DD/YYYY-MM-DDTHHZ.json`: one file per UTC hour with picks: the hour's row hashes in sequence
  order, their RFC 6962 SHA-256 Merkle root, and the chain head before and after the hour.
* `daily/main/YYYY/YYYY-MM-DD.json`: the day's hourly files (with their SHA-256), a Merkle root over the hourly roots
  and `day_root` (SHA-256 of the day's row hashes joined by newlines, as in the ledger's daily digest).
* `daily/main/YYYY/YYYY-MM-DD.json.ots`: an [OpenTimestamps](https://opentimestamps.org) proof of that manifest,
  upgraded to a Bitcoin attestation once confirmed (`ots verify <file>.ots`).

How a pick is committed: `content_hash = sha256(hash_input)`, `row_hash = sha256(prev_hash + ":" + content_hash)`.
Every row hash, including picks whose game hasn't started yet, is in the public feed
`POST /rest/v1/rpc/ledger_chain_hashes` (the pick itself stays hidden until kick-off). Once a pick is public, recompute
its row hash, find it in the hour file for its `created_at`, and check that the hour's Merkle root is in a stamped
manifest. Files are never edited or deleted; Git history shows when each was published.
