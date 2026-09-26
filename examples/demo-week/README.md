# Demo week

The sample used in the Splashdown launch video. Four Razorpay settlement batches and
five lines of an SBI statement (four credits, one rent debit). All values are made up.

```bash
python -m splashdown reconcile \
    --settlements examples/demo-week/settlements.csv --vendor razorpay_settlement_csv \
    --bank sbi --statement examples/demo-week/statement.csv \
    --as-of 2026-10-12 --out-dir out
```

Expected: 3 matched (the gateway UTR never equals the bank UTR, so they pair on date and
net amount), 1 settlement the bank never received, 1 credit with no settlement behind it,
and the rent debit set aside as money out rather than flagged.
