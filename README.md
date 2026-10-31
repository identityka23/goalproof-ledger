# GoalProof public ledger

Every GoalProof prediction is sealed **before kick-off** and opened **after kick-off**, so anyone can check
that nothing was added, removed or edited afterwards. This repository is written only by the daily
GoalProof job. Nothing here is betting advice: football predictions for information only, 18+.
No ads, no betting links.

Browser check: **https://goalproof.app/verify/** (pick a day, press Verify).
The same repository also serves the GoalProof website (`index.html`, `record/`, `day/`, …).

## What is published, and when

| Path | When | Contents |
|---|---|---|
| `muhur/{id}.txt` | before the first match of the day | The seal text: rule version, ledger head, candidate-set digest and the hash of every published pick. No match names, no picks. |
| `muhur/{id}.txt.ots` | same time | [OpenTimestamps](https://opentimestamps.org) proof of the seal text, anchored in Bitcoin a few hours later. |
| `muhur/{id}.json` | same time | The same facts in JSON, plus `digest` = SHA-256 of the seal text and `opens_after`. |
| `acik/{id}.jsonl` | after the last match of that seal has kicked off (next daily run) | The picks, one per line, exactly as hashed. Free and VIP picks alike. |
| `aday/{id}.json` | same time | The candidate set: every eligible match evaluated that day, with probabilities and market-average prices. |
| `sonuclar.jsonl` | daily | Result ledger (written only after a match ends). |
| `gecmis/tahminler.jsonl` | daily | **Pre-timestamp history:** records from 28 Sep 2026 until the first seal, published as they are in the chained ledger. They were not publicly sealed before kick-off. Any later record that missed its seal would also appear here. |
| `index.json` | daily | List of seals with their status. |

`{id}` is the day (`2026-10-29`); a second run on the same day gets `2026-10-29-2`.

## How to verify (method)

All hashes are SHA-256, written as lowercase hex. All files are UTF-8 with `\n` line endings.

1. **Seal digest.** `digest = SHA-256(bytes of muhur/{id}.txt)`. It must match `digest` in
   `muhur/{id}.json` and the digest inside `muhur/{id}.txt.ots` (check the `.ots` at opentimestamps.org:
   drop both the `.txt` and the `.ots` file).
2. **Seal text format** (one field per line, final newline included):
   ```
   GoalProof daily seal v1
   id: <id>
   sealed_at: <UTC time, ISO 8601>
   rule: <rule version>
   ledger_head: <hash of the last ledger record at sealing time>
   fixtures: <eligible matches considered>
   candidates: <matches with enough prices> <SHA-256 of aday/{id}.json>
   picks: <n>
   <pick hash 1>
   ...
   <pick hash n>
   ```
3. **Picks.** For each line of `acik/{id}.jsonl`, `SHA-256(line bytes without the trailing \n)` must equal
   the pick hash in the same position of the seal text. Change any character of a line and its hash changes.
   Each line is the ledger record without its `hash` field, serialised as JSON with keys sorted
   (Python: `json.dumps(record, sort_keys=True, ensure_ascii=False)`, default separators `", "` and `": "`).
4. **Chain.** Each record's `onceki_hash` is the hash of the record before it in the private ledger, so
   inside one seal line *k* has `onceki_hash` = hash of line *k−1*, and the last pick hash equals `ledger_head`.
5. **Candidate set.** `SHA-256(bytes of aday/{id}.json)` must equal the digest on the `candidates:` line.
   This shows which matches were available that day, so picks cannot be chosen after the fact.

Shell example (Linux/macOS):

```sh
sha256sum muhur/2026-10-29.txt                       # = "digest" in muhur/2026-10-29.json
sha256sum aday/2026-10-29.json                       # = digest on the "candidates:" line
while IFS= read -r line; do printf '%s' "$line" | sha256sum; done < acik/2026-10-29.jsonl
tail -n +9 muhur/2026-10-29.txt                      # the sealed pick hashes, same order
```

## Field names

Ledger fields are in Turkish. The main ones: `fixture_id`, `lig` / `lig_adi` (league code / name),
`ev` / `dep` (home / away), `baslama` (kick-off), `liste` (`ucretsiz` = free, `vip`), `pazar` (market),
`olasilik` (probability of the pick), `olasiliklar` (all market probabilities), `ort_oran`
(market-average price), `bahisci_sayisi` (number of prices averaged), `kural` (rule version),
`bant` / `basamak` (price band / widening step), `katman` (match pool tier), `kayit_zamani`
(record time, UTC), `onceki_hash` (previous record), `hash`.
In `sonuclar.jsonl`: `tahmin_hash` (the pick), `skor_90dk` (90-minute score), `sonuc`
(`tuttu` = won, `yatti` = lost, `iptal` = void).

Probabilities are the market consensus: the average of several bookmakers' prices with the margin
removed. We do not claim to beat the market; the long-run expected return at market prices is negative.
