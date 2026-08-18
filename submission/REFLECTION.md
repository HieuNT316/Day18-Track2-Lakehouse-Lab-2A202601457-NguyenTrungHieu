<!--
DRAFT — two things you must do before submitting:

1. Check this against the actual "Top 5 Lakehouse Anti-Patterns" slide. That list
   is not in the repo, so the anti-pattern named below is the one THIS LAB
   measured most directly, not necessarily the slide's exact wording.
2. Replace the two [EDIT] placeholders with your team's real situation. The
   question asks about YOUR team's data — that cannot be answered from lab
   output, and a grader can tell.

Body is 189 words (limit 200). Delete this comment block before submitting.
-->

# Reflection

The anti-pattern our data is most at risk of is **running a lakehouse with no
maintenance jobs** — treating the table as a file dump and assuming the engine
cleans up.

NB6 made the cost concrete. 100,000 rows arrived as 200 streaming commits,
average file size **51.5 KB** against a production target of 128–512 MB. A full
scan cost **10M GETs/day ($4.00)** versus **200K ($0.08)** after compaction — a
50× request bill for identical bytes.

The invisible half worries me more. The table reported 100,000 rows while **15**
parquet files sat on disk and only **10** appeared in the log: five files we pay
for and cannot see. `VACUUM` never reclaims them, because a file a crashed job
left behind was never tombstoned. Iceberg was worse — `expire_snapshots` cut 20
snapshots to 3, deleted **0** avro files, and metadata *grew*.

Our ingest is [EDIT: append-heavy streaming / hourly batch / CDC] and we have
[EDIT: no scheduled OPTIMIZE, no orphan scan], so both apply directly. The fix:
schedule Jobs 1–5 as pairs — expiry always chained to an orphan sweep — and
alert on on-disk versus log file count.
