# NUT-CTF-creator-fee Test Vectors

These test vectors provide reference data for implementing the creator fee mechanism for conditional tokens. All values use the same condition setup as [NUT-CTF-split-merge test vectors](CTF-split-merge-tests.md).

## Fee Calculation

### Test 1: Basic creator fee

```shell
# Creator fee = ceil(sum_input_amount * creator_fee_ppk / 1000)
sum_input_amount:   100
creator_fee_ppk:    10
creator_fee:        ceil(100 * 10 / 1000) = ceil(1.0) = 1
```

### Test 2: Fee with rounding up

```shell
sum_input_amount:   101
creator_fee_ppk:    10
creator_fee:        ceil(101 * 10 / 1000) = ceil(1.01) = 2
```

### Test 3: Small amount with rounding

```shell
sum_input_amount:   1
creator_fee_ppk:    10
creator_fee:        ceil(1 * 10 / 1000) = ceil(0.01) = 1
```

### Test 4: Zero fee when unclaimed

```shell
# No creator has claimed for this condition
creator_fee_claimed: false
creator_fee:         0
# Swap equation: sum(outputs) = sum(inputs) - fees(inputs)
```

## Swap with Creator Fee

### Test 5: NUT-03 swap with creator fee deduction

```shell
# Setup: Alice swaps YES conditional tokens
creator_fee_ppk:    10
input_fee_ppk:      100    # per conditional keyset
num_input_proofs:   3
sum_input_amount:   100
# All inputs use YES conditional keyset

# NUT-02 fee (per-proof)
nut02_fee:          ceil((3 * 100) / 1000) = ceil(0.3) = 1

# Creator fee (per-amount)
creator_fee:        ceil(100 * 10 / 1000) = ceil(1.0) = 1

# Swap outputs
total_deduction:    1 + 1 = 2
output_amount:      100 - 2 = 98

# Request (NUT-03 swap)
swap_request:       {
  "inputs": [<3 YES conditional proofs totaling 100 sats>],
  "outputs": [<blinded messages totaling 98 sats, same YES keyset>]
}

# Result: PASS
```

### Test 6: Split does NOT deduct creator fee

```shell
# Even with a claimed creator fee, split only deducts NUT-02 fee
creator_fee_ppk:     10
creator_fee_claimed: true
sum_input_amount:    100
nut02_fee:           1
creator_fee:         0    # NOT applied on split
per_outcome:         100 - 1 = 99

# Result: PASS — creator fee applies to swaps only
```

### Test 7: Merge does not deduct creator fee

```shell
# Merge operation — creator_fee_ppk is ignored
creator_fee_ppk:     10
per_outcome_amount:  98
num_input_proofs:    6   # 3 YES + 3 NO
input_fee_ppk:       100

# Only NUT-02 fee applies
nut02_fee:           ceil((6 * 100) / 1000) = ceil(0.6) = 1
merge_output:        98 - 1 = 97

# Result: PASS — creator fee applies to swaps only
```

### Test 7a: Swap of regular tokens has no creator fee

```shell
# NUT-03 swap of regular (non-conditional) tokens
# Even though creator_fee_ppk is set, it does not apply
creator_fee_ppk:     10
input_keyset_type:   regular
sum_input_amount:    100
num_input_proofs:    3
input_fee_ppk:       100

nut02_fee:           ceil((3 * 100) / 1000) = ceil(0.3) = 1
creator_fee:         0
output_amount:       100 - 1 = 99

# Standard NUT-03 equation: sum(outputs) = sum(inputs) - fees(inputs)
# Result: PASS
```

### Test 7b: Swap of fee claim tokens has no creator fee

```shell
# NUT-03 swap within fee claim keyset
# Creator fee does not apply to fee claim token swaps
creator_fee_ppk:     10
input_keyset_type:   creator_fee
sum_input_amount:    10000
num_input_proofs:    2
input_fee_ppk:       0

creator_fee:         0
output_amount:       10000

# Result: PASS
```

## Creator Fee via Condition Registration

### Test 8: Register condition with creator fee outputs

```shell
# POST /v1/conditions — new condition with creator_fee_outputs
# creator_fee_total_supply = 10000
request:             {
  "threshold": 1,
  "tags": [["description", "Will BTC reach $100k?"]],
  "announcements": ["fdd824fd..."],
  "creator_fee_outputs": [
    {"amount": 8192, "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_1>"},
    {"amount": 1024, "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_2>"},
    {"amount": 512,  "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_3>"},
    {"amount": 256,  "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_4>"},
    {"amount": 16,   "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_5>"}
  ]
}
# sum(creator_fee_outputs) = 8192 + 1024 + 512 + 256 + 16 = 10000

response:            {
  "condition_id": "<condition_id_hex>",
  "creator_fee_signatures": [
    {"amount": 8192, "id": "<fee_claim_keyset_id>", "C_": "<blind_sig_1>"},
    {"amount": 1024, "id": "<fee_claim_keyset_id>", "C_": "<blind_sig_2>"},
    {"amount": 512,  "id": "<fee_claim_keyset_id>", "C_": "<blind_sig_3>"},
    {"amount": 256,  "id": "<fee_claim_keyset_id>", "C_": "<blind_sig_4>"},
    {"amount": 16,   "id": "<fee_claim_keyset_id>", "C_": "<blind_sig_5>"}
  ]
}

# Result: 200 OK — condition created, fee claim tokens issued
```

### Test 9: Register existing condition with creator fee (already registered)

```shell
# POST /v1/conditions — condition already exists and was already registered with creator_fee_outputs
request:             {
  "threshold": 1,
  "tags": [["description", "Will BTC reach $100k?"]],
  "announcements": ["fdd824fd..."],
  "creator_fee_outputs": [...]
}

# Result: error 13050 — Creator fee already claimed
```

### Test 10: Register condition without creator fee outputs

```shell
# POST /v1/conditions — no creator_fee_outputs field
request:             {
  "threshold": 1,
  "tags": [["description", "Will BTC reach $100k?"]],
  "announcements": ["fdd824fd..."]
}

response:            {
  "condition_id": "<condition_id_hex>"
}

# Result: 200 OK — condition created, no creator assigned
```

### Test 11: Late registration with creator fee outputs

```shell
# POST /v1/conditions — condition already exists but was registered without creator_fee_outputs
request:             {
  "threshold": 1,
  "tags": [["description", "Will BTC reach $100k?"]],
  "announcements": ["fdd824fd..."],
  "creator_fee_outputs": [
    {"amount": 8192, "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_1>"},
    {"amount": 1024, "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_2>"},
    {"amount": 512,  "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_3>"},
    {"amount": 256,  "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_4>"},
    {"amount": 16,   "id": "<fee_claim_keyset_id>", "B_": "<blinded_point_5>"}
  ]
}
# sum(creator_fee_outputs) = 10000

response:            {
  "condition_id": "<condition_id_hex>",
  "creator_fee_signatures": [...]
}

# Result: 200 OK — existing condition_id returned, fee claim tokens issued
```

### Test 12: Amount mismatch

```shell
# POST /v1/conditions — creator_fee_outputs sum to 5000, but total_supply = 10000
request:             {
  "threshold": 1,
  "tags": [["description", "Test"]],
  "announcements": ["fdd824fd..."],
  "creator_fee_outputs": [
    {"amount": 4096, "id": "<fee_claim_keyset_id>", "B_": "<blinded_point>"},
    {"amount": 512,  "id": "<fee_claim_keyset_id>", "B_": "<blinded_point>"},
    {"amount": 256,  "id": "<fee_claim_keyset_id>", "B_": "<blinded_point>"},
    {"amount": 128,  "id": "<fee_claim_keyset_id>", "B_": "<blinded_point>"},
    {"amount": 8,    "id": "<fee_claim_keyset_id>", "B_": "<blinded_point>"}
  ]
}
# sum(creator_fee_outputs) = 5000 != 10000

# Result: error 13052 — Fee claim total supply mismatch
```

## Redeem Creator Fee

### Test 15: Proportional redemption (50% share)

```shell
# Setup
total_supply:        10000
accumulated_fees:    200
holder_amount:       5000
input_fee_ppk:       0     # for simplicity

# Payout = floor(holder_amount * accumulated_fees / total_supply)
payout:              floor(5000 * 200 / 10000) = floor(100.0) = 100

# POST /v1/redeem_creator_fee
request:             {
  "inputs": [<fee claim proofs totaling 5000>],
  "outputs": [<blinded messages totaling 100 sats, regular keyset>]
}

# Result: 200 OK
```

### Test 16: Full supply redemption

```shell
total_supply:        10000
accumulated_fees:    200
holder_amount:       10000
input_fee_ppk:       0

payout:              floor(10000 * 200 / 10000) = 200

# Result: 200 OK — holder gets all accumulated fees
```

### Test 17: Rounding behavior (dust to mint)

```shell
total_supply:        10000
accumulated_fees:    100
holder_amount:       3333
input_fee_ppk:       0

payout:              floor(3333 * 100 / 10000) = floor(33.33) = 33

# Rounding dust: 100 - floor(3333*100/10000) - floor(6667*100/10000)
#              = 100 - 33 - 66 = 1 sat (goes to mint)
```

### Test 18: Redemption with NUT-02 fee on inputs

```shell
total_supply:        10000
accumulated_fees:    200
holder_amount:       5000
num_input_proofs:    3
input_fee_ppk:       100   # fee claim keyset has its own input_fee_ppk

payout:              floor(5000 * 200 / 10000) = 100
nut02_fee:           ceil((3 * 100) / 1000) = 1
output_amount:       100 - 1 = 99

# sum(outputs) = payout - fees(inputs) = 100 - 1 = 99
# Result: 200 OK
```

### Test 19: Redemption before attestation

```shell
# Condition status: pending (no attestation yet)

# POST /v1/redeem_creator_fee with valid fee claim proofs

# Result: error 13053 — Condition not in vesting period
```

### Test 20: Redemption after vesting expires

```shell
# Condition status: attested
# event_maturity_epoch + vesting_period has passed

# POST /v1/redeem_creator_fee with valid fee claim proofs

# Result: error 13053 — Condition not in vesting period
```

### Test 21: Redemption with zero accumulated fees

```shell
# No swaps have occurred for this condition
accumulated_fees:    0

# POST /v1/redeem_creator_fee with valid fee claim proofs

# Result: error 13054 — No accumulated fees
```

## Fee Claim Token Trading

### Test 22: NUT-03 swap within fee claim keyset

```shell
# Standard NUT-03 swap
# Input: 10000 fee claim tokens (single keyset)
# Output: 5000 + 5000 fee claim tokens (same keyset)

# Result: PASS — allowed, enables splitting shares
```

### Test 23: Cross-keyset swap rejected

```shell
# NUT-03 swap attempt
# Input: fee claim keyset proofs
# Output: regular keyset blinded messages

# Result: MUST reject — fee claim keysets cannot swap with regular keysets
# Use POST /v1/redeem_creator_fee instead
```

## Keyset Discovery

### Test 24: Conditional keysets include kind and accumulated fee

```shell
# GET /v1/conditional_keysets
# After 3 swaps of conditional tokens totaling 300 sats with creator_fee_ppk = 10

response:            {
  "keysets": [
    {
      "id": "<yes_keyset_id>",
      "unit": "sat",
      "active": true,
      "input_fee_ppk": 100,
      "condition_id": "<condition_id_hex>",
      "outcome_collection": "YES",
      "outcome_collection_id": "<oc_id_hex>",
      "kind": "conditional"
    },
    {
      "id": "<no_keyset_id>",
      "unit": "sat",
      "active": true,
      "input_fee_ppk": 100,
      "condition_id": "<condition_id_hex>",
      "outcome_collection": "NO",
      "outcome_collection_id": "<oc_id_hex>",
      "kind": "conditional"
    },
    {
      "id": "<fee_claim_keyset_id>",
      "unit": "sat",
      "active": true,
      "input_fee_ppk": 0,
      "condition_id": "<condition_id_hex>",
      "kind": "market_creator_fee",
      "total_supply": 10000,
      "accumulated_fee": 3
    }
  ]
}

# accumulated_fee = ceil(100*10/1000) + ceil(100*10/1000) + ceil(100*10/1000) = 3
# Holder of 5000 tokens: current_value = floor(5000 * 3 / 10000) = 1 sat
# Result: 200 OK
```

## Volume Info

### Test 25: Volume info response

```shell
# GET /v1/conditions/{condition_id}/volume
# After 3 swaps of conditional tokens totaling 300 sats with creator_fee_ppk = 10

response:            {
  "condition_id": "<condition_id_hex>",
  "total_issued": {
    "YES": 297,
    "NO": 297
  },
  "accumulated_creator_fee": 3,
  "creator_fee_claimed": true
}

# accumulated_creator_fee = ceil(100*10/1000) + ceil(100*10/1000) + ceil(100*10/1000) = 3
# Result: 200 OK
```

[CTF-creator-fee]: ../CTF-creator-fee.md
[CTF]: ../CTF.md
[CTF-split-merge]: ../CTF-split-merge.md
