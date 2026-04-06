# NUT-CTF-creator-fee: Creator Fee for Conditional Tokens

`optional`

`depends on: NUT-CTF`

---

This NUT defines a creator fee mechanism for conditional tokens ([NUT-CTF][CTF]). The first person to register a condition receives tradeable **fee claim tokens** — divisible ecash shares of a fee pool that accumulates from every [NUT-03][03] swap of conditional tokens on that condition. Fee claim token holders redeem their shares for regular ecash during the [vesting period][CTF].

The creator fee incentivizes market creation and rewards markets that attract trading volume. The fee rate is set globally by the mint to prevent liquidity fragmentation across duplicate conditions.

Caution: Applications must verify that the mint supports NUT-CTF and NUT-CTF-creator-fee via the [info][06] endpoint.

## Overview

```
Register Condition → (Users Swap) → Fees Accumulate → Redeem Fee Tokens
        │                  │                │                  │
        ▼                  ▼                ▼                  ▼
  First caller gets    creator_fee       Per-condition      Holders swap
  total_supply fee     deducted from     fee pool grows     fee claim tokens
  claim tokens         each swap                            for regular ecash
                                                            during vesting
```

1. **Register**: First caller to register a condition via [NUT-CTF][CTF] with `creator_fee_outputs` receives fee claim tokens (total amount = `creator_fee_total_supply`)
2. **Accumulate**: Each [NUT-03][03] swap of conditional tokens deducts a creator fee into the per-condition fee pool
3. **Redeem**: After oracle attestation, during the [vesting period][CTF], fee claim token holders redeem for regular ecash proportional to their share
4. **Expire**: Unclaimed fees after vesting are forfeit to the mint

## Creator Fee Mechanism

When swapping conditional tokens via [NUT-03][03], the total fee is the sum of two components:

1. **NUT-02 fee** ([NUT-02][02]): Per-proof fee based on `input_fee_ppk` of each input keyset
2. **Creator fee**: Per-amount fee based on `creator_fee_ppk` set by the mint

Both use parts per thousand (ppk) and ceiling division:

```
nut02_fee    = ceil(sum(input_fee_ppk for each proof) / 1000)
creator_fee  = ceil(sum_input_amount * creator_fee_ppk / 1000)
total_fee    = nut02_fee + creator_fee
```

The swap equation becomes:

```
sum(outputs) = sum(inputs) - total_fee
```

**Key properties:**

- Applied only on [NUT-03][03] swaps where all inputs use a conditional keyset whose condition has a registered creator
- NOT applied on split, merge, redemption (`POST /v1/redeem_outcome`), or swaps of regular or fee claim keysets
- If no one registered the condition with `creator_fee_outputs`, `creator_fee` is `0` and only the NUT-02 fee applies
- The fee is accumulated atomically as part of the swap operation into a per-condition fee pool tracked by the mint

**Example** (`Alice` swaps 100 sats of YES conditional tokens, 3 input proofs with `input_fee_ppk = 100`, `creator_fee_ppk = 10`):

- NUT-02 fee: `ceil((3 * 100) / 1000)` = `ceil(0.3)` = 1 sat
- Creator fee: `ceil(100 * 10 / 1000)` = `ceil(1.0)` = 1 sat
- Total fee: `1 + 1` = 2 sats
- Output amount: `100 - 2 = 98` sats

## Fee Claim Keyset

Each condition registered with `creator_fee_outputs` gets a unique fee claim keyset.

### Keyset ID Derivation

Fee claim keyset IDs extend [NUT-02 V2 derivation][02] by appending creator fee data:

```
<NUT-02 V2 base preimage> + "|creator_fee:" + condition_id_hex
```

In Python:

```python
keyset_id_bytes += f"|creator_fee:{condition_id}".encode("utf-8")
```

Where `condition_id` is a 64-character hex string. The version byte remains `01`.

### Properties

- **Unit**: Matches the collateral unit (e.g., `"sat"`)
- **Active flag**: `true` during condition lifetime + vesting period, `false` after vesting expires
- **Discovery**: Via `GET /v1/conditional_keysets` ([NUT-CTF][CTF]) with `"kind": "market_creator_fee"`. See [Conditional Keyset Discovery](#conditional-keyset-discovery-extension).
- **NUT-03 swap**: Allowed within the same fee claim keyset (enables trading and splitting)
- **Cross-keyset swap**: MUST be rejected — fee claim keysets cannot swap with regular or conditional keysets

### Conditional Keyset Discovery Extension

Fee claim keysets appear in the `GET /v1/conditional_keysets` response ([NUT-CTF][CTF]). This NUT adds the `kind` field to all keysets and additional fields for fee claim keysets:

```json
{
  "keysets": [
    {
      "id": <hex_str>,
      "unit": <str>,
      "active": <bool>,
      "input_fee_ppk": <int>,
      "condition_id": <hex_str>,
      "kind": "conditional",
      ...
    },
    {
      "id": <hex_str>,
      "unit": <str>,
      "active": <bool>,
      "input_fee_ppk": <int>,
      "condition_id": <hex_str>,
      "kind": "market_creator_fee",
      "total_supply": <int>,
      "accumulated_fee": <int>,
      ...
    }
  ]
}
```

- `kind`: `"conditional"` for outcome collection keysets, `"market_creator_fee"` for fee claim keysets
- `total_supply`: Fixed number of fee claim tokens issued (only for `kind: "market_creator_fee"`)
- `accumulated_fee`: Total creator fees collected for this condition so far, in collateral unit (only for `kind: "market_creator_fee"`)

Wallets can calculate the current value of their fee claim tokens:

```
current_value = floor(holder_amount * accumulated_fee / total_supply)
```

## Condition Registration Extension

Creator fee claim tokens are issued as part of condition registration ([NUT-CTF][CTF]). The first caller to register a new condition with `creator_fee_outputs` receives fee claim tokens. See [NUT-CTF: Register Condition][CTF] for the extended request and response format.

**Request extension** — optional field on `POST /v1/conditions`:

```json
{
  "creator_fee_outputs": <Array[BlindedMessage]>
}
```

- `creator_fee_outputs`: `BlindedMessage` objects using the fee claim keyset ID. `sum(outputs)` MUST equal `creator_fee_total_supply` (error 13052 if mismatch).

**Response extension** — present when fee claim tokens are issued:

```json
{
  "condition_id": <hex_str>,
  "creator_fee_signatures": <Array[BlindSignature]>
}
```

### Mint Behavior

When processing `POST /v1/conditions` with `creator_fee_outputs`:

1. Condition registration proceeds as normal ([NUT-CTF][CTF])
2. If condition already exists (idempotent): `creator_fee_outputs` is ignored, no signatures returned
3. If condition is new and no `creator_fee_outputs` provided: no creator is assigned
4. If condition is new and `creator_fee_outputs` provided:
   - Checks creator fee support (error 13051 if not supported)
   - Verifies `sum(creator_fee_outputs)` equals `creator_fee_total_supply` (error 13052 if mismatch)
   - Creates fee claim keyset with [derivation formula](#keyset-id-derivation)
   - Signs blinded messages
   - Returns `creator_fee_signatures` alongside `condition_id`
5. If condition already exists but was registered without `creator_fee_outputs`: the caller MAY register as creator by re-submitting with `creator_fee_outputs` and identical condition config. The mint returns the existing `condition_id` plus `creator_fee_signatures`.
6. If condition already exists and was already registered with `creator_fee_outputs`: error 13050

## Consequence for NUT-03

The mint identifies the condition from the input proofs' keyset IDs (each conditional keyset is bound to a `condition_id` per [NUT-CTF][CTF]). When the condition has a registered creator, the mint MUST deduct the creator fee in addition to the [NUT-02][02] fee on every [NUT-03][03] swap. See [Creator Fee Mechanism](#creator-fee-mechanism) for the formula.

## Redeem Creator Fee

```http
POST https://mint.host:3338/v1/redeem_creator_fee
```

Fee claim token holders redeem their shares for regular ecash. Payout is proportional to the share of total supply held.

**Request** of `Alice`:

```json
{
  "inputs": <Array[Proof]>,
  "outputs": <Array[BlindedMessage]>
}
```

- `inputs`: `Proof` objects from a **single fee claim keyset**
- `outputs`: `BlindedMessage` objects using a **regular keyset** (same unit)

```bash
curl -X POST https://mint.host:3338/v1/redeem_creator_fee \
  -H "Content-Type: application/json" \
  -d '{"inputs":[...],"outputs":[...]}'
```

**Response** of `Bob`:

```json
{
  "signatures": <Array[BlindSignature]>
}
```

### Payout Calculation

```
payout = floor(holder_amount * accumulated_fees / total_supply)
```

Where:

- `holder_amount`: Sum of input proof amounts
- `accumulated_fees`: Total creator fees collected for this condition (available as `accumulated_fee` in [keyset discovery](#conditional-keyset-discovery-extension))
- `total_supply`: Fixed total supply from [Mint Info Setting](#mint-info-setting)

Output requirement: `sum(outputs) = payout - fees(inputs)` per [NUT-02][02].

**Example** (`total_supply = 10000`, `accumulated_fees = 200` sats):

- `Alice` holds 5000 fee claim tokens: payout = `floor(5000 * 200 / 10000)` = 100 sats
- `Carol` holds 3333 fee claim tokens: payout = `floor(3333 * 200 / 10000)` = 66 sats

Rounding dust (from `floor`) goes to the mint.

### Redemption Verification

`Bob`:

1. All inputs MUST use the same fee claim keyset (derive `condition_id` from keyset)
2. All outputs MUST use a regular keyset (same unit)
3. Condition MUST be in vesting period — attested and within `vesting_period` after `event_maturity_epoch` (error 13053)
4. Accumulated fees MUST be > 0 (error 13054)
5. Compute payout using proportional formula
6. Verify output amount equals `payout - fees(inputs)`
7. Sign blinded messages, mark input proofs as spent
8. Deduct payout from accumulated fee pool

### Timing Constraints

- Redemption is available after oracle attestation and during the [vesting period][CTF]
- After vesting period expires, unclaimed fees are forfeit to the mint
- The fee claim keyset is deactivated (`active = false`) after vesting expires

## Volume Info

```http
GET https://mint.host:3338/v1/conditions/{condition_id}/volume
```

Optional endpoint that reports the total tokens issued per outcome collection and accumulated creator fees.

**Response** of `Bob`:

```json
{
  "condition_id": <hex_str>,
  "total_issued": {
    "<outcome_collection_1>": <int>,
    "<outcome_collection_2>": <int>,
    ...
  },
  "accumulated_creator_fee": <int>,
  "creator_fee_claimed": <bool>
}
```

- `total_issued`: Total tokens issued per outcome collection across all splits (in collateral unit)
- `accumulated_creator_fee`: Total creator fees collected for this condition across all swaps (in collateral unit)
- `creator_fee_claimed`: Whether the condition was registered with `creator_fee_outputs`

```bash
curl -X GET https://mint.host:3338/v1/conditions/a1b2c3d4.../volume
```

Mints MAY use [NUT-24][24] to require payment for access to this endpoint.

## Security Considerations

- **First-come-first-served**: Fee claim tokens are issued to the first caller who registers a condition with `creator_fee_outputs`. The mint MUST ensure atomicity to prevent race conditions.
- **Tradability risk**: Fee claim tokens are bearer instruments. Losing them means losing the right to accumulated fees. Wallets SHOULD warn users.
- **Fee pool accuracy**: The mint MUST accurately track accumulated fees across all swap operations. Fee accumulation MUST be part of the atomic swap transaction.
- **Redemption atomicity**: Redemption MUST be atomic — proofs spent and outputs signed in a single transaction.
- **NUT-03 enforcement**: Fee claim tokens can be freely swapped within the same fee claim keyset via [NUT-03][03]. The mint MUST reject swaps between fee claim keysets and any other keyset type (regular, conditional, or other fee claim keysets).
- **Unclaimed fee forfeiture**: After vesting expires, the mint keeps unclaimed fees. This incentivizes timely redemption.

## Error Codes

| Code  | Description                     |
| ----- | ------------------------------- |
| 13050 | Creator fee already claimed     |
| 13051 | Creator fee not supported       |
| 13052 | Fee claim total supply mismatch |
| 13053 | Condition not in vesting period |
| 13054 | No accumulated fees             |

## Mint Info Setting

The [NUT-06][06] `MintMethodSetting`:

```json
{
  "CTF-creator-fee": {
    "supported": true,
    "creator_fee_ppk": <int>,
    "creator_fee_total_supply": <int>
  }
}
```

- `supported`: Boolean indicating NUT-CTF-creator-fee support
- `creator_fee_ppk`: Parts per thousand fee on conditional-token swap input amount (integer). E.g., `10` = 1%.
- `creator_fee_total_supply`: Fixed number of fee claim tokens issued to the creator per condition (integer)

[00]: 00.md
[01]: 01.md
[02]: 02.md
[03]: 03.md
[04]: 04.md
[05]: 05.md
[06]: 06.md
[07]: 07.md
[08]: 08.md
[09]: 09.md
[10]: 10.md
[11]: 11.md
[12]: 12.md
[14]: 14.md
[24]: 24.md
[CTF]: CTF.md
[CTF-numeric]: CTF-numeric.md
