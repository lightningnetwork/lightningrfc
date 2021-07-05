# BOLT #12: Flexible Protocol for Lightning Payments

# Table of Contents

  * [Limitations of BOLT 11](#limitations-of-bolt-11)
  * [Payment Flow Scenarios](#payment-flow-scenarios)
  * [Encoding](#encoding)
  * [TLV Fields](#tlv-fields)
  * [Invoices](#invoices)
  * [Offers](#offers)
  * [Invoice Requests](#invoice-requests)
  * [Invoice Errors](#invoice-errors)

# Limitations of BOLT 11

The BOLT 11 invoice format has proven popular, but has several
limitations:

1. The entangling of bech32 encoding makes it awkward to send
   in other forms (i.e. inside the lightning network itself).
2. The signature applying to the entire invoice makes it impossible
   to prove an invoice without revealing its entirety.
3. Fields cannot generally be extracted for external use: the `h`
   field was a boutique extraction of the `d` field, only.
4. The lack of 'it's OK to be odd' rule makes backwards compatibility
   harder.
5. The 'human-readable' idea of separating amounts proved fraught:
   `p` was often mishandled, and amounts in pico-bitcoin are harder
   than the modern satoshi-based counting.
6. The bech32 encoding was found to have an issue with extensions,
   which means we want to replace or discard it anyway.
7. The `payment_secret` designed to prevent probing by other nodes in
   the path was only useful if the invoice remained private between the
   payer and payee.
8. Invoices must be given per-user, and are actively dangerous if two
   payment attempts are made for the same user.


# Payment Flow Scenarios

Here we use "user" as shorthand for the individual user's lightning
node, and "merchant" as the shorthand for the node of someone who is
selling or has sold something.

There are two basic payment flows supported by BOLT 12:

The general user-pays-merchant flow is:
1. A merchant publishes an *offer* ("send me money"), such as on a web page or a QR code.
2. Every user requests a unique *invoice* over the lightning network
   using an *invoice_request* message.
3. The merchant replies with the *invoice*.
4. The user makes a payment to the merchant indicated by the invoice.

The merchant-pays-user flow (e.g. ATM or refund):
1. The merchant provides a user-specific *offer* ("take my money") in a web page or QR code,
   with an amount (for a refund, also a reference to the to-be-refunded
   invoice).
2. A user sends an *invoice* for the amount in the *offer* (for a
   refund, a proof that they requested the original)
3. The merchant makes a payment to the user indicated by the invoice.

## Payment Proofs and Payer Proofs

Note that the normal lightning "proof of payment" can only demonstrate that an
invoice was paid (by showing the preimage of the `payment_hash`), not who paid
it.  The merchant can claim an invoice was paid, and once revealed, anyone can
claim they paid the invoice, too.[1]

1. Sharing the minimum information required to prove sections of the
   invoice in dispute (e.g. the description of the item, the payment
   hash, and the merchant's signature).
2. Contain a transferable proof that the user is the one who was
   responsible for paying the invoice in the first place.

Providing a key in *invoice_request* allows a user to prove that they were the one
to request the invoice.  In addition, the Merkle construction of the BOLT 12
invoice signature allows the user to selectively reveal fields of the invoice
in case of dispute.

# Encoding

Each of the forms documented here are in
[TLV](01-messaging.md#type-length-value-format) format.

The supported ASCII encoding is the human-readable prefix, followed by a
`1`, followed by a bech32-style data string of the TLVs in order,
optionally interspersed with `+` (for indicating additional data is to
come).

## Requirements

Readers of a bolt12 string:
- if it encounters a `+` followed zero or more whitespace characters between 
  two bech32 characters:
  - MUST remove the `+` and whitespace.

## Rationale

The use of bech32 is arbitrary, but already exists in the bitcoin
world.  We currently omit the six-character trailing checksum: QR
codes have their own checksums, bech32 doesn't protect against many
length differences, and bech32m is not yet widely supported.

The use of `+` (which is ignored) allows use over limited
text fields like Twitter:

```
lno1xxxxxxxx+

yyyyyyyyyyyy+

zzzzz
```

See [format-string-test.json](bolt12/format-string-test.json).

## Signature Calculation

All signatures are created as per
[BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki),
and tagged as recommended there.  Thus we define H(`tag`,`msg`) as
SHA256(SHA256(`tag`) || SHA256(`tag`) || `msg`), and SIG(`tag`,`msg`,`key`)
as the signature of H(`tag`,`msg`) using `key`.

Each form is signed using one or more TLV signature elements; TLV
types 240 through 1000 are considered signature elements.  For these
the tag is `lightning` | `messagename` | `fieldname`, and `msg` is the
Merkle-root; `lightning` is the literal 9-byte ASCII string,
`messagename` is the name of the TLV stream being signed (i.e. `offer`, `invoice_request` or `invoice`) and the `fieldname` is the TLV field containing the
signature (e.g. `signature` or `payer_signature`).

The formulation of the Merkle tree is similar to that proposed in
[BIP-taproot], with the each TLV leaf paired with a nonce leaf to avoid
revealing adjacent nodes in proofs (assuming there is a non-revealed TLV
which has enough entropy).

The Merkle tree's leaves are, in TLV-ascending order for each tlv:
1. The H(`LnLeaf`,tlv).
2. The H(`LnAll`|all-tlvs,tlv) where "all-tlvs" consists of all non-signature TLV entries appended in ascending order.

The Merkle tree inner nodes are H(`LnBranch`, lesser-SHA256|greater-SHA256);
this ordering means that proofs are more compact since left/right is
inherently determined.

If there are not exactly a power of 2 leaves, then the tree depth will
be uneven, with the deepest tree on the lowest-order leaves.

e.g. consider the encoding of an `offer` `signature` with TLVs TLV1, TLV2 and TLV3:

```
L1=H(`LnLeaf`,TLV1)
L1nonce=H(`LnAll`|TLV1|TLV2|TLV3,TLV1) 
L2=H(`LnLeaf`,TLV2)
L2nonce=H(`LnAll`|TLV1|TLV2|TLV3,TLV2) 
L3=H(`LnLeaf`,TLV3)
L3nonce=H(`LnAll`|TLV1|TLV2|TLV3,TLV3) 

Assume L1 < L1nonce, and L2 > L2nonce.

   L1    L1nonce                      L2   L2nonce               L3
     \   /                             \   /                     |
      v v                               v v                      v
L1A=H('LnBranch',L1|L1nonce) L2A=H('LnBranch',L2nonce|L2)       L3
                 
Assume L1A < L2A:

       L1A   L2A                                 L3
         \   /                                    |
          v v                                     v
  L1A2A=H('LnBranch',L1A|L2A)                    L3
  
Assume L1A2A > L3:

  L1A2A=H('LnBranch',L1A|L2A)          L3
                          \            /
                           v          v
                Root=H('LnBranch',L3|L1A2A)

Signature = SIG('lightningoffersignature', Root, nodekey)
```

# Offers

Offers are a precursor to an invoice: readers will either request an invoice
(or multiple) or send an invoice based on the offer.  An offer can be much longer-lived than a
particular invoice, so has some different characteristics; in particular it
can be recurring, and the amount can be in a non-lightning currency.  It's
also designed for compactness, to easily fit inside a QR code.

The human-readable prefix for offers is `lno`.

## TLV Fields for Offers

1. `tlv_stream`: `offer`
2. types:
    1. type: 2 (`chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 6 (`currency`)
    2. data:
        * [`...*utf8`:`iso4217`]
    1. type: 8 (`amount`)
    2. data:
        * [`tu64`:`amount`]
    1. type: 10 (`description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`absolute_expiry`)
    2. data:
        * [`tu64`:`seconds_from_epoch`]
    1. type: 16 (`paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 20 (`vendor`)
    2. data:
        * [`...*utf8`:`vendor`]
    1. type: 22 (`quantity_min`)
    2. data:
        * [`tu64`:`min`]
    1. type: 24 (`quantity_max`)
    2. data:
        * [`tu64`:`max`]
    1. type: 26 (`recurrence`)
    2. data:
        * [`byte`:`time_unit`]
        * [`tu32`:`period`]
    1. type: 64 (`recurrence_paywindow`)
    2. data:
        * [`u32`:`seconds_before`]
        * [`byte`:`proportional_amount`]
        * [`tu32`:`seconds_after`]
    1. type: 66 (`recurrence_limit`)
    2. data:
        * [`tu32`:`max_period`]
    1. type: 28 (`recurrence_base`)
    2. data:
        * [`byte`:`start_any_period`]
        * [`tu64`:`basetime`]
    1. type: 30 (`node_id`)
    2. data:
        * [`pubkey32`:`node_id`]
    1. type: 54 (`send_invoice`)
    1. type: 34 (`refund_for`)
    2. data:
        * [`sha256`:`refunded_payment_hash`]
    1. type: 240 (`signature`)
    2. data:
        * [`bip340sig`:`sig`]

1. subtype: `blinded_path`
2. data:
   * [`point`:`blinding`]
   * [`u16`:`num_hops`]
   * [`num_hops*onionmsg_path`:`path`]

## Recurrence

Some offers are *periodic*, such as a subscription service or monthly
dues, in that payment is expected to be repeated.  There are many
different flavors of repetition, consider:

* Payments due on the first of every month, for 6 months.
* Payments due on every Monday, 1pm Pacific Standard Time.
* Payments due once a year:
   * which must be made on January 1st, or
   * which are only valid if started January 1st 2021, or
   * which if paid after January 1st you (over) pay the full rate first year, or
   * which if paid after January 1st are paid pro-rata for the first year, or
   * which repeat from whenever you made the first payment

Thus, each payment has:
1. A `time_unit` defining 0 (seconds), 1 (days), 2 (months), 3 (years).
2. A `period`, defining how often (in `time_unit`) it has to be paid.
3. An optional `recurrence_limit` of total payments to be paid.
4. An optional `recurrence_base`:
   * `basetime`, defining when the first period starts
      in seconds since 1970-01-01 UTC.
   * `start_any_period` if non-zero, meaning you don't have to start
      paying at the period indicated by `basetime`, but can use
      `recurrence_start` to indicate what period you are starting at.
5. An optional `recurrence_paywindow`:
   * `seconds_before`, defining how many seconds prior to the start of
      the period a payment will be accepted.
   * `proportional_amount`, if set indicating that a payment made during
      the period itself will be charged proportionally to the remaining time
      in the period (e.g. 150 seconds into a 1500 second period gives a 10%
      discount).
   * `seconds_after`, defining how many seconds after the start of the period
      a payment will be accepted.
  If this field is missing, payment will be accepted during the prior period and
  the paid-for period.

Note that the `absolute_expiry` field already covers the case where an offer
is no longer valid after January 1st 2021.

## Offer Period Calculation

Each period has a zero-based index, and a start time and an end time.
Because the periods can be in non-seconds units, the duration of a
period can depend on when it starts.  The period with index N+1 begins
immediately following the end of period with index N.

- if an offer contains `recurrence_base`:
  - the start of period #0 is `basetime` seconds since 1970-01-01 UTC.
- otherwise:
  - the start of period #0 is the time of issuance of the first
    `invoice` for this particular offer and `payer_key`.

To calculate the start of period #`N` for `N` > 0:
- if `time_unit` is 0:
  - period `N` starts at period #0 start plus `period` multiplied by `N`,
    in seconds.
- otherwise, if `time_unit` is 1:
  - calculate the offset in seconds within the day of period #0 start.
  - add `period` multiplied by `N` days to get the day of the period start.
  - add the offset in seconds to get the period end in seconds.
- otherwise, if `time_unit` is 2:
  - calculate the offset in days within the month of period #0 start.
  - calculate the offset in seconds within the day of period #0 start.
  - add `period` multiplied by `N` months to get the month of the period start.
  - add the offset days to get the day of the period start.
    - if the day is not within the month, use the last day within the month.
  - add the offset seconds to get the period start in seconds.
- otherwise, if `time_unit` is 3:
  - calculate the offset in months within the year of period #0 start.
  - calculate the offset in days within the month of period #0 start.
  - calculate the offset in seconds within the day of period #0 start.
  - add `period` multiplied by `N` years to get the year of the period start.
  - add the offset months to get the month of the period start.
  - add the offset days to get the day of the period start.
    - if the day is not within the month, use the last day within the month.
  - add the offset seconds to get the period start in seconds.
- otherwise, the time is invalid.

Note that offset seconds can overflow only if the period start is in a
leap second; we ignore this!

See [offer-period-test.json](bolt12/offer-period-test.json).

## Authorization

Authorization is generally required for payments: without some
indication what someone intended to pay for and how much they intended
to pay, proof of payment is pointless.

Normally this is simple: get the user to authorize the exact amount
and description before paying an invoice.  With recurrence this
becomes more complex, as an implementation probably does not want to
prompt the user on every payment, but receive some initial
authorization to spend within a range (e.g. "Pay $5 AUD once a
week?").  In particular, the authorization may be in the user's native
currency, not the vendor's currency nor in Bitcoin.

For example, consider an offer with weekly recurrence (`time_unit`=1,
`period`=7), `amount` 500, `currency` `AUD` ($5 Australian dollars).
An implementation may present this to the user as USD $3.53 (max
$3.71), to allow up to 5% exchange slippage, and receive their
authorization.  As it received each invoice, it would convert the
`msat` into USD to check that it was below the maximum authorization
of USD$3.71.  If it was, it would simply pay the invoice without user
interaction.

On the other hand, if an invoice did exceed the authorization, it
would request re-authorization.  It could also indicate whether it was
due to AUD/USD changes (since the offer indicated that was the
currency it was using) or a disagreement on the bitcoin exchange rate.

Note that the problem is simpler for non-recurring offers, where
authorization may simply be delayed until the invoice is received and
the exact amount is known.

Also, the implementation of a trusted exchange rate service is left to
the reader.

## Requirements For Offers

A writer of an offer:
  - MUST set `node_id` to the public key of the node to request the invoice from.
  - MAY specify exactly one signature TLV: `signature`:
    - If so, it MUST set `sig` to the signature using `node_id` as described in [Signature Calculation](#signature-calculation).
  - MUST set `description` to a complete description of the purpose
    of the payment.
  - if the chain for the invoice is not solely bitcoin:
    - MUST specify `chains` the offer is valid for.
  - otherwise:
    - the bitcoin chain is implied as the first and only entry.
  - if a specific minimum `amount` is required for successful payment:
    - MUST specify `amount` to the amount expected (per item).
    - if the currency for `amount` is that of the first entry in `chains`:
      - MUST specify `amount` in multiples of the minimum lightning-payable unit
        (e.g. milli-satoshis for bitcoin).
    - otherwise:
      - MUST specify `iso4217` as an ISO 4712 three-letter code.
      - MUST specify `amount` in the currency unit adjusted by the ISO 4712
        exponent (e.g. USD cents).
  - if it supports offer features:
    - SHOULD set `features` to the bitmap of offer features.
  - if the offer expires:
    - MUST set `absolute_expiry` `seconds_from_epoch` to the number of seconds
      after midnight 1 January 1970, UTC that invoice_request should not be
      attempted.
  - if it is connected only by private channels:
    - MUST include `paths` containing one or more paths to the node from
      publicly reachable nodes.
  - otherwise:
    - MAY include `paths`.
  - if it includes `paths`:
    - SHOULD ignore any invoice_request which does not use the path.
  - if it sets `vendor`:
    - SHOULD set it to clearly identify the issuer of the invoice.
  - if it can supply more than one item for a single invoice
    - if the minimum quantity is more than 1:
      - MUST set that minimum in `quantity_min`
    - if the maximum quantity is known:
      - MUST set that maximum in `quantity_max`
    - if neither:
      - MUST set `quantity_min` to 1 to indicate `quantity` is supported.
  - MAY include `recurrence` to indicate offer should trigger time-spaced
    invoices.
  - if it includes `recurrence`:
      - MUST set `time_unit` to 0 (seconds), 1 (days), 2 (months), 3 (years).
      - MUST set `period` to how often (per `time-unit`) it wants to be paid.
      - if there is a maximum number of payments:
        - MUST include `recurrence_limit` with `max_period` set to the maximum number of payments
        - MUST NOT set `max_period` to 0.
      - otherwise:
        - MUST NOT include `recurrence_limit`.
      - if periods are always at specific time offsets:
        - MUST include `recurrence_base`
        - MUST set `basetime` to the initial period time in number of
          seconds after midnight 1 January 1970
        - if the first paid-for-period does not have to be the initial period:
          - MUST set `start_any_period` to 1.
        - otherwise:
          - MUST set `start_any_period` to 0.
      - otherwise:
        - MUST NOT include `recurrence_base`.
      - if payments will be accepted for the current or next period:
        - MAY include `recurrence_paywindow`
      - otherwise:
        - MUST include `recurrence_paywindow`
      - if it includes `recurrence_paywindow`:
        - MUST set `seconds_before` to the maximum number of seconds prior to
          a period for which it will accept payment or invoice_request for that period.
        - MUST set `seconds_after` to the maximum number of seconds into to a
          period for which it will accept payment or invoice_request for that period.
        - MAY NOT enforce this for the initial period for offers without `recurrence_base`
        - SHOULD NOT set `seconds_after` to greater than the maximum number of
          seconds in a period.
        - if it `amount` is specified and the node will proportionally reduce
          the amount charged for a period payed after the start of the period:
          - MUST set `proportional_amount` to 1 
        - otherwise:
          - MUST set `proportional_amount` to 0
  - otherwise:
    - MUST NOT include `recurrence_base`.
    - MUST NOT include `recurrence_paywindow`.
    - MUST NOT include `recurrence_limit`.
  - if `send_invoice` is present:
    - if the offer is for a partial or full refund for a previously-paid
      invoice:
      - SHOULD set `refunded_payment_hash` to the `payment_hash` of that
        invoice.
  - otherwise:
    - MUST NOT set `refunded_payment_hash`.

A reader of an offer:
  - if `features` contains unknown _odd_ bits that are non-zero:
    - MUST ignore the bit.
  - if `features` contains unknown _even_ bits that are non-zero:
    - MUST NOT respond to the offer.
    - SHOULD indicate the unknown bit to the user.
  - if `node_id` or `description` is not set:
    - MUST NOT respond to the offer.
  - if `description` is not present:
    - MUST NOT respond to the offer.
  - if `signature` is present, but is not a valid signature using `node_id` as described in [Signature Calculation](#signature-calculation):
    - MUST NOT respond to the offer.
  - SHOULD gain user consent for recurring payments.
  - SHOULD allow user to view and cancel recurring payments.
  - if it uses `amount` to provide the user with a cost estimate:
    - MUST warn user if amount of actual invoice differs significantly
        from that expectation.
  - SHOULD not respond to an offer if the current time is after
    `absolute_expiry`.
  - FIXME: more!

## Rationale

It's quite reasonable to set a `recurrence_paywindow` with seconds_after
equal to 0, but obviously this should not apply to the initial period if
there is no recurrence_base.

A signature is optional, because it makes for a longer string (potentially
limiting QR code use on low-end cameras); if the offer has an error, no
invoice will be given (or, for `send_invoice` offers, accepted), since
the `offer_id` already covers all the non-signature fields.

# Invoice Requests

Invoice Requests are a request for an invoice; the human-readable prefix for
invoices is `lnr`.

## TLV Fields for `invoice_request`

1. `tlv_stream`: `invoice_request`
2. types:
    1. type: 2 (`chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_id`)
    2. data:
        * [`sha256`:`offer_id`]
    1. type: 8 (`amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 12 (`features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 32 (`quantity`)
    2. data:
        * [`tu64`:`quantity`]
    1. type: 36 (`recurrence_counter`)
    2. data:
        * [`tu32`:`counter`]
    1. type: 68 (`recurrence_start`)
    2. data:
        * [`tu32`:`period_offset`]
    1. type: 38 (`payer_key`)
    2. data:
        * [`pubkey32`:`key`]
    1. type: 39 (`payer_note`)
    2. data:
        * [`...*utf8`:`note`]
    1. type: 50 (`payer_info`)
    2. data:
        * [`...*byte`:`blob`]
    1. type: 56 (`replace_invoice`)
    2. data:
        * [`sha256`:`payment_hash`]
    1. type: 240 (`payer_signature`)
    2. data:
        * [`bip340sig`:`sig`]

## Requirements for Invoice Requests

The writer of an invoice_request:
  - MUST set `payer_key` to a transient public key.
  - MUST remember the secret key corresponding to `payer_key`.
  - MUST set `offer_id` to the Merkle root of the offer as described in [Signature Calculation](#signature-calculation).
  - MUST NOT set or imply any `chain_hash` not set or implied by the offer.
  - MUST set `payer_signature` `sig` as detailed in [Signature Calculation](#signature-calculation) using the `payer_key`.
  - if the offer had a `quantity_min` or `quantity_max` field:
    - MUST set `quantity`
    - MUST set it within that (inclusive) range.
  - otherwise:
    - MUST NOT set `quantity`
  - if the offer did not specify `amount`:
    - MUST specify `amount`.`msat` in multiples of the minimum lightning-payable unit
      (e.g. milli-satoshis for bitcoin) for the first `chains` entry.
  - otherwise:
    - MAY omit `amount`.
    - if it sets `amount`:
      - MUST specify `amount`.`msat` as greater or equal to amount expected by the offer
        (before any proportional period amount).
  - if the sender has a previous unpaid invoice (for the same offer) which it wants to cancel:
    - MUST set `payer_key` to the same as the previous invoice.
    - MUST set `replace_invoice` to the `payment_hash` or the previous invoice.
  - if the offer contained `recurrence`:
    - for the initial request:
      - MUST use a unique `payer_key`.
      - MUST set `recurrence_counter` `counter` to 0.
    - for any successive requests:
      - MUST use the same `payer_key` as the initial request.
      - MUST set `recurrence_counter` `counter` to one greater than the highest-paid invoice.
    - if the offer contained `recurrence_base` with `start_any_period` non-zero:
      - MUST include `recurrence_start`
      - MUST set `period_offset` to the period the sender wants for the initial request
      - MUST set `period_offset` to the same value on all following requests.
    - otherwise:
      - MUST NOT include `recurrence_start`
    - MAY set `payer_info` to arbitrary data to be reflected into the invoice.
    - MAY set `payer_note` to arbitrary string to be reflected into the invoice.
    - if the offer contained `recurrence_limit`:
      - MUST NOT send an `invoice_request` for a period greater than `max_period`
    - SHOULD NOT send an `invoice_request` for a period which has
      already passed.
    - if the offer contains `recurrence_paywindow`:
      - if the offer has a `recurrence_basetime` or the `recurrence_counter` is non-zero:
        - SHOULD NOT send an `invoice_request` for a period prior to `seconds_before` seconds before that period start.
        - SHOULD NOT send an `invoice_request` for a period later than `seconds_after` seconds past that period start.
    - otherwise:
      - SHOULD NOT send an `invoice_request` with `recurrence_counter`
        is non-zero for a period whose immediate predecessor has not
        yet begun.
  - otherwise:
    - MUST NOT set `recurrence_counter`.
    - MUST NOT set `recurrence_start`

The reader of an invoice_request:
  - MUST fail the request if `payer_key` is not present.
  - MUST fail the request if `chains` does not include (or imply) a supported chain.
  - MUST fail the request if `features` contains unknown even bits.
  - MUST fail the request if `offer_id` is not present.
  - MUST fail the request if the `offer_id` does not refer to an unexpired offer.
  - MUST fail the request if there is no `payer_signature` field.
  - MUST fail the request if `payer_signature` is not correct.
  - if the offer had a `quantity_min` or `quantity_max` field:
    - MUST fail the request if there is no `quantity` field.
    - MUST fail the request if there is `quantity` is not within that (inclusive) range.
  - otherwise:
    - MUST fail the request if there is a `quantity` field.
  - if the offer included `amount`:
    - MUST calculate the *base invoice amount* using the offer `amount`:
      - if offer `currency` is not the invoice currency, convert to the
        invoice currency.
      - if request contains `quantity`, multiply by `quantity`.
    - if the request contains `amount`:
      - MUST fail the request if its `amount` is less than the *base invoice amount*.
      - MAY fail the request if its `amount` is much greater than the *base invoice amount*.
      - MUST use the request's `amount` as the *base invoice amount*.
  - otherwise:
    - MUST fail the request if it does not contain `amount`.
    - MUST use the request `amount` as the *base invoice amount*. (Note: invoice amount can be further modified by recurrence below)
  - if the offer has a `replace_invoice`:
    - if the `payment_hash` refers to an unpaid invoice for the same `offer_id` and `payer_key`:
      - MUST immediately expire/remove that unpaid invoice such that it cannot be paid in future.
    - otherwise:
      - MUST fail the request.
  - if the offer had a `recurrence`:
    - MUST fail the request if there is no `recurrence_counter` field.
    - if the offer had `recurrence_base` and `start_any_period` was 1:
      - MUST fail the request if there is no `recurrence_start` field.
      - MUST consider the period index for this request to be the
        `recurrence_start` field plus the `recurrence_counter` `counter`
        field.
    - otherwise:
      - MUST fail the request if there is a `recurrence_start` field.
      - MUST consider the period index for this request to be the
        `recurrence_counter` `counter` field.
    - if the offer has a `recurrence_limit`:
      - MUST fail the request if the period index is greater than `max_period`.
    - MUST calculate the period using the period index as detailed in [Period Calculation](#offer-period-calculation).
    - if `recurrence_counter` is non-zero:
      - MUST fail the request if the no invoice for the previous period
        has been paid.
      - if the offer had a `recurrence_paywindow`:
        - SHOULD fail the request if the current time is before the start of
          the period minus `seconds_before`.
        - SHOULD fail the request if the current time is equal to or after the
          start of the period plus `seconds_after`.
        - if `proportional_amount` is 1:
          - MUST adjust the *base invoice amount* proportional to time remaining in
            the period.
      - otherwise:
        - if `counter` is non-zero:
          - SHOULD fail the request if the current time is prior to the start
            of the previous period.
  - otherwise (the offer had no `recurrence`):
    - MUST fail the request if there is a `recurrence_counter` field.
    - MUST fail the request if there is a `recurrence_start` field.

## Rationale

We insist that recurring requests be in order (thus, if you pay an
invoice for #34 of a recurring offer, it implicitly commits to the
successful payment of #0 through #33).

The `recurrence_paywindow` constrains how far you can pay in advance
precisely, and if it isn't in the offer the defaults provide some
slack, without allowing commitments into the far future.

To avoid probing (should a payer_key become public in some way), we
require a signature; this ensures that no third party can determine
how many invoices have been paid already in the case of recurring
requests, and disallows replacement of old invoices by third parties.

`payer_info` might typically contain information about the derivation of the
`payer_key`.  This should not leak any information (such as using a simple
BIP-32 derivation path); a valid system might be for a node to maintain a base
payer key, and encode a 128-bit tweak here.  The payer_key would be derived by
tweaking the base key with SHA256(payer_base_pubkey || tweak).

`payer_note` allows you to compliment, taunt, or otherwise engrave
graffiti into the invoice for all to see.

Users can give a tip (or obscure the amount sent) by specifying an
`amount` in their invoice request, even though the offer specifies an
`amount`.  Obviously this will only be accepted by the recipient if
the invoice request amount exceeds the amount it's expecting (i.e. its
`amount` after any currency conversion, multiplied by `quantity` if
any).  Note that for recurring invoices with `proportional_amount`
set, the `amount` in the invoice request will be scaled by the time in
the period; the sender should not attempt to scale it.

`replace_invoice` allows the mutually-agreed removal of and old unpaid
invoice; this can be used in the case of stuck payments.  If
successful in replacing the stuck invoice, the sender may make a
second payment such that it can prove double-payment should the
receiver still accept the first, delayed payment.

# Invoices

Invoices are a request for payment, and when the payment is made they
it can be combined with the invoice to form a cryptographic receipt.

The human-readable prefix for invoices is `lni`.  It can be sent in
response to an `invoice_request` or an `offer` with `send_invoice`
using `onion_message` `invoice` field.

1. `tlv_stream`: `invoice`
2. types:
    1. type: 2 (`chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_id`)
    2. data:
        * [`sha256`:`offer_id`]
    1. type: 8 (`amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 10 (`description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 16 (`paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 18 (`blindedpay`)
    2. data:
        * [`...*blinded_payinfo`:`payinfo`]
    1. type: 19 (`blinded_capacities`)
    2. data:
        * [`...*u64`:`incoming_msat`]
    1. type: 20 (`vendor`)
    2. data:
        * [`...*utf8`:`vendor`]
    1. type: 30 (`node_id`)
    2. data:
        * [`pubkey32`:`node_id`]
    1. type: 32 (`quantity`)
    2. data:
        * [`tu64`:`quantity`]
    1. type: 34 (`refund_for`)
    2. data:
        * [`sha256`:`refunded_payment_hash`]
    1. type: 36 (`recurrence_counter`)
    2. data:
       * [`tu32`:`counter`]
    1. type: 54 (`send_invoice`)
    1. type: 68 (`recurrence_start`)
    2. data:
        * [`tu32`:`period_offset`]
    1. type: 64 (`recurrence_basetime`)
    2. data:
        * [`tu64`:`basetime`]
    1. type: 38 (`payer_key`)
    2. data:
        * [`pubkey32`:`key`]
    1. type: 39 (`payer_note`)
    2. data:
        * [`...*utf8`:`note`]
    1. type: 50 (`payer_info`)
    2. data:
        * [`...*byte`:`blob`]
    1. type: 40 (`timestamp`)
    2. data:
        * [`tu64`:`timestamp`]
    1. type: 42 (`payment_hash`)
    2. data:
        * [`sha256`:`payment_hash`]
    1. type: 44 (`relative_expiry`)
    2. data:
        * [`tu32`:`seconds_from_timestamp`]
    1. type: 46 (`cltv`)
    2. data:
        * [`tu32`:`min_final_cltv_expiry`]
    1. type: 48 (`fallbacks`)
    2. data:
        * [`u8`:`num`]
        * [`num*fallback_address`:`fallbacks`]
    1. type: 52 (`refund_signature`)
    2. data:
        * [`bip340sig`:`payer_signature`]
    1. type: 56 (`replace_invoice`)
    2. data:
        * [`sha256`:`payment_hash`]
    1. type: 240 (`signature`)
    2. data:
        * [`bip340sig`:`sig`]

1. subtype: `blinded_payinfo`
2. data:
   * [`u32`:`fee_base_msat`]
   * [`u32`:`fee_proportional_millionths`]
   * [`u16`:`cltv_expiry_delta`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]

1. subtype: `fallback_address`
2. data:
   * [`byte`:`version`]
   * [`u16`:`len`]
   * [`len*byte`:`address`]

## Requirements

A writer of an invoice:
  - MUST set `timestamp` to the number of seconds since Midnight 1
    January 1970, UTC.
  - MUST set `payment_hash` to the SHA2 256-bit hash of the
    `payment_preimage` that will be given in return for payment.
  - MUST set (or not set) `send_invoice` the same as the offer.
  - MUST set `offer_id` to the id of the offer.
  - MUST specify exactly one signature TLV: `signature`.
    - MUST set `sig` to the signature using `node_id` as described in [Signature Calculation](#signature-calculation).
  - if the chain for the invoice is not solely bitcoin:
    - MUST specify `chains` the invoice is valid for.
  - otherwise:
    - the bitcoin chain is implied as the first and only entry.
  - if it has bolt11 features:
    - MUST set `features` to the bitmap of features.
  - if the invoice corresponds to an offer with `recurrence`:
    - MUST set `recurrence_basetime` to the start of period #0 as calculated
      by [Period Calculation](#offer-period-calculation).
    - if it sets `relative_expiry`:
      - MUST NOT set `relative_expiry` `seconds_from_timestamp` more than the number of seconds after `timestamp` that payment for this period will be accepted.
  - otherwise:
    - MUST not set `recurrence_basetime`.
  - if the expiry for accepting payment is not 7200 seconds after `timestamp`:
    - MUST set `relative_expiry` `seconds_from_timestamp` to the number of
      seconds after `timestamp` that payment of this invoice should not be attempted.
  - if the `min_final_cltv_expiry` for the last HTLC in the route is not 18:
    - MUST set `min_final_cltv_expiry`.
  - if it accepts onchain payments:
    - MAY specify `fallbacks`
    - MUST specify `fallbacks` in order of most-preferred to least-preferred
      if it has a preference.
    - for the bitcoin chain, it MUST set each `fallback_address` with
      `version` as a valid witness version and `address` as a valid witness
      program
  - if it is connected only by private channels:
    - MUST include a `blinded_path` containing one or more paths to the node.
  - otherwise:
    - MAY include `blinded_path`.
  - if it includes `blinded_path`:
    - MUST specify `path` in order of most-preferred to least-preferred if
      it has a preference.
    - MUST include `blindedpay` with exactly one `payinfo` for
      each `onionmsg_path` in `blinded_path`, in order.
    - if it includes `blinded_capacities`:
      - MUST include exactly one `incoming_msat` (in millisatoshis) per `path`, reflecting the expected minimum amount that can be sent through the path.
    - SHOULD ignore any payment which does not use one of the paths.
  - otherwise:
    - MUST NOT include `blinded_payinfo`.
  - MUST set `vendor` exactly as the offer did.
  - MUST specify `amount`.`msat` in multiples of the minimum lightning-payable unit
    (e.g. milli-satoshis for bitcoin) for the first `chains` entry.
  - if responding to an `invoice_request`:
    - if for the same `offer_id`, `payer_key` and `recurrence_counter` (if any) as a previous `invoice_request`:
      - MAY simply reuse the previous invoice.
    - otherwise:
      - MUST NOT reuse a previous invoice.
    - MUST set `node_id` the same as the offer.
    - MUST set (or not set) `quantity` exactly as the invoice_request did.
    - MUST set (or not set) `recurrence_counter` exactly as the invoice_request did.
    - MUST set (or not set) `recurrence_start` exactly as the invoice_request did.
    - MUST set `payer_key` exactly as the invoice_request did.
    - MUST set (or not set) `payer_info` exactly as the invoice_request did.
    - MUST set (or not set) `payer_note` exactly as the invoice_request did,
      or MUST not set it.
    - MUST set (or not set) `replace_invoice` exactly as the invoice_request did.
    - MUST begin `description` with the `description` from the offer.
    - MAY append additional information to `description` (e.g. " +shipping").
    - if it does not set `amount` to the *base invoice amount* calculated from the invoice_request:
       - MUST append the reason to `description` (e.g. " 5% bulk discount").
    - MUST NOT set `refund_for`
    - MUST NOT set `refund_signature`
  - otherwise (responding to a `send_invoice` offer):
    - MUST set `node_id` to the id of the node to send payment to.
    - MUST set `description` the same as the offer.
    - if the offer had a `quantity_min` or `quantity_max` field:
      - MUST set `quantity`
      - MUST set it within that (inclusive) range.
    - otherwise:
      - MUST NOT set `quantity`
    - MUST set `payer_key` to the `node_id` of the offer.
    - MUST NOT set `payer_info`.
    - MUST set (or not set) `refund_for` exactly as the offer did.
    - if it sets `refund_for`:
      - MUST set `refund_signature` to the signature of the
        `refunded_payment_hash` using prefix `refund_signature` and the `payer_key` from the to-be-refunded invoice.
    - otherwise:
      - MUST NOT set `refund_signature`
    - FIXME: recurrence!

A reader of an invoice:
  - MUST reject the invoice if `signature` is not a valid signature using `node_id` as described in [Signature Calculation](#signature-calculation).
  - MUST reject the invoice if `msat` is not present.
  - MUST reject the invoice if `description` is not present.
  - MUST reject the invoice if `timestamp` is not present.
  - MUST reject the invoice if `payment_hash` is not present.
  - if `relative_expiry` is present:
    - MUST reject the invoice if the current time since 1970-01-01 UTC is greater than `timestamp` plus `seconds_from_timestamp`.
  - otherwise:
    - MUST reject the invoice if the current time since 1970-01-01 UTC is greater than `timestamp` plus 7200.
  - if `blinded_path` is present:
    - MUST reject the invoice if `blinded_payinfo` is not present.
    - MUST reject the invoice if `blinded_payinfo` does not contain exactly as many `payinfo` as total `onionmsg_path` in `blinded_path`.
  - SHOULD confirm authorization if `msat` is not within the amount range authorized.
  - if the invoice is a reply to an `invoice_request`:
     - MUST reject the invoice unless `offer_id` is equal to the id of the offer.
     - MUST reject the invoice unless `node_id` is equal to the offer.
     - MUST reject the invoice unless the following fields are equal or unset
       exactly as they are in the `invoice_request:`
       - `quantity`
       - `recurrence_counter`
       - `recurrence_start`
       - `payer_key`
       - `payer_info`
    - MUST reject the invoice if `payer_note` is set, and was unset or not equal to the field in the `invoice_request`.
    - SHOULD confirm authorization if the `description` does not exactly
      match the `offer`
      - MAY highlight if `description` has simply had a change appended.
    - SHOULD confirm authorization if `vendor` does not exactly
      match the `offer`.
  - otherwise if `offer_id` is set:
    - MUST reject the invoice if the `offer_id` does not refer an unexpired offer with `send_invoice`
    - MUST reject the invoice unless the following fields are equal or unset
      exactly as they are in the `offer`:
      - `refund_for`
      - `description`
      - `vendor`
    - if the offer had a `quantity_min` or `quantity_max` field:
      - MUST reject the invoice if there is no `quantity` field.
      - MUST reject the invoice if there is `quantity` is not within that (inclusive) range.
    - otherwise:
      - MUST reject the invoice if there is a `quantity` field.
  - if the offer contained `recurrence`:
    - MUST reject the invoice if `recurrence_basetime` is not set.
  - if the offer contained `refund_for`:
      - MUST reject the invoice if `payer_key` does not match the invoice whose `payment_hash` is equal to `refund_for` `refunded_payment_hash`
      - MUST reject the invoice if `refund_signature` is not set.
      - MUST reject the invoice if `refund_signature` is not a valid signature using `payer_key` as described in [Signature Calculation](#signature-calculation).
  - for the bitcoin chain, if the invoice specifies `fallbacks`:
    - MUST ignore any `fallback_address` for which `version` is greater than 16.
    - MUST ignore any `fallback_address` for which `address` is less than 2 or greater than 40 bytes.
    - MUST ignore any `fallback_address` for which `address` does not meet known requirements for the given `version`

## Rationale

Because the messaging layer is unreliable, it's quite possible to
receive multiple requests for the same offer.  As it's the caller's
responsibility not to reuse `payer_key` except for recurring invoices,
the writer doesn't have to check all the fields are duplicates before
simply returning a previous invoice.

The invoice duplicates fields rather than committing to the previous offer or
invoice_request.  This flattened format simplifies storage at some space cost, as
the payer need only remember the invoice for any refunds or proof.

The `recurrence_basetime` similarly enables calculation of the next period
without having to refer to the initial invoice (in the case where the
offer does not contain `recurrence_base`.

The reader of the invoice cannot trust the invoice correctly reflects the
offer and invoice_request fields, hence the requirements to check that they
are correct.

Note that the recipient of the invoice can determine the expected
amount from either the offer it received, or the invoice_request it
sent, so often already has authorization for the expected amount.

It's natural to set the `relative_expiry` of an invoice for a
recurring offer to the end of the payment window for the period, but
if that is a long time and the offer was in another currency, it's
common to cap this at some maximum duration.  For example, omitting it
implies the default of 7200 seconds, which is generally a sufficient
time for payment.

The invoice issuer is allows to ignore `payer_note` (it has an odd
number, so is optional), but if it does not, it must copy it exactly
as the invoice_request specified.

It's often useful to provide capacity hints, particularly where more
than one blinded path is included, for payers to use multi-part
payments.

# Invoice Errors

Informative errors can be returned in an onion message `invoice_error`
field (via the onion `reply_path`) for either `invoice_request` or
`invoice`.

## TLV Fields for `invoice_error`

1. `tlv_stream`: `invoice_error`
2. types:
    1. type: 1 (`erroneous_field`)
    2. data:
        * [`tu64`:`tlv_fieldnum`]
    1. type: 3 (`suggested_value`)
    2. data:
        * [`...*byte`:`value`]
    1. type: 5 (`error`)
    2. data:
        * [`...*utf8`:`msg`]

## Requirements

A writer of an invoice_error:
  - MUST set `error` to an explanatory string.
  - MAY set `erroneous_field` to a specific field number in the
    `invoice` or `invoice_request` which had a problem.
  - if it sets `erroneous_field`:
    - MAY set `suggested_value`.
    - if it sets `suggested_value`:
      - MUST set `suggested_value` to a valid field for that `tlv_fieldnum`.
  - otherwise:
    - MUST NOT set `suggested_value`.

A reader of an invoice_error:
   FIXME!

## Rationale

Usually an error message is sufficient for diagnostics, however there
is at least one case where it should be programatically parsable.  A
recurring offer which sets `send_invoice` can also specify a currency,
which opens the possibility for a disagreement on exchange rate.  In
this case, the `suggested_value` reflects its expected value, and the
sender can send a new invoice.

# FIXME: Possible future extensions:

1. The offer can require delivery info in the `invoice_request`.
2. An offer can be updated: the response to an `invoice_request` is another offer,
   perhaps with a signature from the original `node_id`
3. Any empty TLV fields can mean the value is supposed to be known by
   other means (i.e. transport-specific), but is still hashed for sig.
4. We could upgrade to allow multiple offers in one invoice_request and
   invoice, to make a shopping list.
7. All-zero offer_id == gratuitous payment.
8. Recurrent invoice requests?

[1] https://www.youtube.com/watch?v=4SYc_flMnMQ
