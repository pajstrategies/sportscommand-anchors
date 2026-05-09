# SportsCommand Prediction Anchor Log

**Public, append-only timestamp anchor for SportsCommand predictions.**

Every 30 minutes, a cron on the SportsCommand production server commits the SHA-256 hashes of all newly-created prediction rows to this repository. GitHub's commit timestamps serve as **external, third-party witnesses** that those predictions existed in our database at the time of commit — before the games they refer to had started.

This is a non-invasive observer. Nothing in the SportsCommand prediction pipeline is changed by it; the anchoring runs in parallel and is read-only against the predictions tables.

---

## Why this exists

A buyer or auditor reviewing SportsCommand's historical performance might reasonably ask: *"How do I know your timestamps weren't backdated?"* The answer this repository provides is: **you don't have to trust SportsCommand's timestamps. You can trust GitHub's.**

For any prediction in SportsCommand's audit export (`predictions.csv`), a verifier can:

1. Compute the prediction's canonical hash (formula below) using the columns already in the audit CSV.
2. Search this repository's commit history for that hash.
3. Read the commit timestamp — set by GitHub at push time, outside SportsCommand's control.
4. Confirm `commit_timestamp < commence_time` — proving the prediction existed before the game.

A backdated prediction would either be missing from this log entirely or appear in a commit dated *after* the game it predicts. Either is a smoking gun.

---

## Hash formula

For each prediction row, the canonical hash is:

```
SHA-256(
  prediction_id || "|" ||
  created_at_utc_iso || "|" ||
  commence_time_utc_iso || "|" ||
  sport || "|" ||
  bet_type || "|" ||
  selected_team || "|" ||
  line || "|" ||
  odds || "|" ||
  confidence_score
)
```

All fields are rendered as their PostgreSQL string representations. `created_at_utc_iso` and `commence_time_utc_iso` are formatted as `YYYY-MM-DDTHH:MM:SSZ`. Numeric fields use Postgres default text representation. NULL `selected_team` is rendered as the empty string.

The canonical hash function is implemented in `scripts/sportscommand-anchor-predictions.sh` in the main SportsCommand repository for full reproducibility.

---

## Repository layout

```
anchors/
  YYYY-MM-DD.log    Append-only hash records, one row per prediction.
                    Format: <ts_utc>|<prediction_id>|<sha256>|<sport>|<bet_type>|<commence_time_utc>
.last_anchor        Marker file: most recent anchored created_at value.
```

Each commit message follows the pattern:
`anchor batch <UTC timestamp> — <N> new predictions hashed`

---

## Verification example (for a buyer)

Suppose `predictions.csv` contains a row with `id = 0fa1b2c3-...` for a game on 2026-05-12 at 02:00:00 UTC. To verify the prediction existed before the game:

```bash
git clone https://github.com/pajstrategies/sportscommand-anchors
cd sportscommand-anchors
grep '0fa1b2c3-' anchors/*.log
# Expected output:
# 2026-05-11T20:30:14Z|0fa1b2c3-...|<sha256>|basketball_nba|spread|2026-05-12T02:00:00Z

# Now check WHEN that line was committed:
git log --format='%aI %s' -- anchors/2026-05-11.log | head
# Expected: a commit dated 2026-05-11T20:30:14Z or shortly after.
```

If the commit timestamp is before `commence_time`, the prediction is verified as having existed before the game. If the prediction ID is missing from this log entirely, that's evidence it was created after-the-fact.

---

## What this DOES and DOES NOT prove

**Does prove:**
- A row with this exact `(id, created_at, commence_time, sport, bet_type, selected_team, line, odds, confidence_score)` existed in SportsCommand's database at the GitHub commit timestamp shown for that hash.
- Therefore that prediction existed before any game whose `commence_time` is later than the commit timestamp.

**Does NOT prove (out of scope for anchoring):**
- That the AI's prediction turned out correct.
- That the outcome the prediction received later was graded honestly. (For that, the audit's `actual_result.home_score / away_score` columns can be verified against any independent box-score source — BDL, ESPN, the league's official records.)
- That every prediction SportsCommand made appears in this log. Possible to omit a prediction. Defense-in-depth requires also looking at SportsCommand's application logs and ops Telegram archive (the daily 8 AM ET slate post is itself another timestamp witness, this one provided by Telegram).

---

## Note on prediction deletions / regenerations

A small number of prediction IDs in this anchor log will not exist in the current `predictions` or `predictions_ml` tables when a buyer queries them. This is by design and is documented here for transparency.

**Why deletions happen:** SportsCommand's `auto_predict` job continuously re-evaluates upcoming games. If odds shift past a staleness threshold OR if player props become available after an initial prediction, the original prediction row is deleted and a new one is generated with the updated context. This happens *before* the game starts; both the deleted row and the replacement row are pre-game predictions.

**What this means for anchor verification:**

- The anchor log is **append-only**. Every hash committed remains forever. Modifying history would defeat the integrity proof.
- A prediction that was deleted+regenerated will show TWO entries in the log — the original (now orphaned) and the replacement.
- The replacement's hash will reproduce against the current row in `predictions_combined`. The original's hash will not — it references a row that no longer exists.

**Why this is not a tamper signal:**

- An orphaned hash whose commit timestamp is *before* `commence_time` proves a prediction was made before the game; whether or not the row still exists doesn't change that proof.
- A backdating attacker would need to forge GitHub commit timestamps, not just delete rows from a database.
- The presence of orphan hashes in the log is itself evidence that SportsCommand makes pre-game predictions and updates them — exactly the opposite signature of post-hoc fabrication.

**Audit reconciliation:** the SportsCommand weekly audit tarball includes a `predictions.csv` with every currently-live prediction. Buyers who want to verify "every live prediction has an anchor entry" can compute hashes from `predictions.csv` and grep the anchor log; missing entries would be the actual concerning signal. Buyers who want to verify "the anchor log only references live predictions" should expect a small number of mismatches representing regen activity (currently ~10 of ~2,000 entries, ≈ 0.5%).

---

## License

Public domain (CC0). This log is intentionally a public record. Verify, fork, mirror, archive at will.
