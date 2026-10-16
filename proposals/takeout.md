# Takeout: Pickup Orders on the Private Commerce Envelope

Status: experimental proposal. This page is not current normative Open Markets text.

## Problem

A café takes an order for pickup. To make it work at the counter, the two sides need to say a few things the base messages of the [private commerce envelope](https://github.com/OpenMarketsFoundation/specification/pull/12) do not carry:

- the customer may ask for a pickup time;
- the café answers with when the order will be ready and a short code the customer says at the counter;
- when the café turns an order down because something ran out, it says which items.

The envelope keeps its base messages thin on purpose and gives implementers one place to add what a kind of business needs: a named entry in `extensions`. This proposal defines that entry for takeout. It is one name, `openmarkets:takeout`, with a small content on each of the three messages it appears on.

Synvya runs takeout on this design for restaurants today. The example below is a real order.

## Non-goals

- Delivery, table service, or anything other than the customer collecting the order.
- Payment. The envelope's payment messages cover it; see *Paying at the counter* for the one rule this proposal adds.
- Scheduling rules: how far ahead a pickup time may be, in what steps. A merchant decides those and declines what it cannot honour.

## Terms

- **Customer**: the party that sends the order; the author of the `order` rumor.
- **Merchant**: the party that answers; the recipient of the `order`.
- **Pickup option**: the merchant's `kind:30406` shipping option with `service` = `pickup`, which the order's delivery selection names with method `pickup`.

## The `openmarkets:takeout` entry

The entry appears inside `extensions` on three message types. A message that carries it lists it in `requires`, because a reader that cannot understand the entry cannot act on the order correctly: it would not know when the food is ready or what code to give.

| Message | Author | Fields in `openmarkets:takeout` |
|---|---|---|
| `order` | customer | `pickup_at` (optional) |
| `order_accepted` | merchant | `ready_at`, `tzid`, `code` |
| `order_cancelled` | merchant | `unavailable` (optional) |

Every time is a whole number of seconds since the Unix epoch. `tzid` is an IANA timezone name, so a client can show the time as the merchant's wall clock.

### On the order

```jsonc
"extensions": { "openmarkets:takeout": { "pickup_at": 1791640800 } },
"requires": ["openmarkets:takeout"]
```

`pickup_at` is when the customer would like to collect. When it is absent, the customer wants the order as soon as possible. An order for pickup carries one delivery selection with `method` = `pickup` naming the merchant's pickup option; the entry adds only the time.

A merchant that cannot honour the time (too soon, too far ahead, outside its hours) declines the order; it does not silently pick another time.

### On the acceptance

```jsonc
"extensions": { "openmarkets:takeout": { "ready_at": 1791641700, "tzid": "America/Los_Angeles", "code": "JQE7" } },
"requires": ["openmarkets:takeout"]
```

`ready_at` is when the order will be ready to collect. `code` is a short string, letters and digits, that the customer says at the counter so staff can match them to the bag; it is not a secret and proves nothing. A client shows the code large and the time in the merchant's zone.

### On a decline

```jsonc
"reason": "out_of_stock",
"refund_required": false,
"extensions": { "openmarkets:takeout": { "unavailable": ["b2…", "c3…"] } },
"requires": ["openmarkets:takeout"]
```

`unavailable` lists the `line_id`s of the lines the merchant could not fill. A client uses it to let the customer fix the order and send it again, instead of guessing. It is present only when the decline is for stock; a decline for any other reason carries no entry.

## Paying at the counter

The envelope's rule is that a merchant that wants payment sends a `payment_request`. For takeout paid at collection, the merchant sends none. A client treats an accepted order with no payment request as "pay when you collect". No field says this; the absence does. A merchant that wants prepayment uses the payment messages unchanged.

## Advertising support

A merchant that accepts takeout orders lists the entry in its capability object, so a client knows it may send `pickup_at` and will understand the answer:

```json
"versions": [{ "version": "1", "receive": ["order", "order_cancel_requested"], "extensions": ["openmarkets:takeout"] }]
```

## Worked example

A customer orders a 12 oz latte with oat milk and two pumps of vanilla, as soon as possible, from a café that closes at 22:00. It is 23:13, so the café, which accepts orders while closed for its next opening, answers with a time tomorrow morning.

```jsonc
// order (customer → café), content
{
  "buyer": "<customer>", "merchant": "<café>",
  "items": [
    { "line_id": "a1…", "listing": { "coordinate": "30402:<café>:latte-12-oz",   "event_id": "…" }, "quantity": "1", "unit": "item", "amount": { "value": "4.50", "unit": "USD" } },
    { "line_id": "b2…", "listing": { "coordinate": "30402:<café>:oat-milk",      "event_id": "…" }, "quantity": "1", "unit": "item", "amount": { "value": "0.60", "unit": "USD" } },
    { "line_id": "c3…", "listing": { "coordinate": "30402:<café>:syrup-vanilla", "event_id": "…" }, "quantity": "2", "unit": "item", "amount": { "value": "1.00", "unit": "USD" } }
  ],
  "total": { "value": "6.10", "unit": "USD" },
  "deliveries": [{ "delivery_id": "d4…", "line_ids": ["a1…", "b2…", "c3…"], "method": "pickup", "shipping_option": { "coordinate": "30406:<café>:pickup", "event_id": "…" } }],
  "extensions": { "openmarkets:add_ons": { "b2…": "a1…", "c3…": "a1…" } },
  "requires": ["openmarkets:add_ons"]
}

// order_accepted (café → customer), content
{
  "order_operation": "<order rumor id>",
  "extensions": { "openmarkets:takeout": { "ready_at": 1791640500, "tzid": "America/Los_Angeles", "code": "JQE7" } },
  "requires": ["openmarkets:takeout"]
}
```

The customer's screen shows "Ready at 6:15 AM tomorrow", the code JQE7, and "pay at the counter", because no payment request followed.

## Compatibility

- The entry is an `extensions` entry under the envelope's own rules; nothing in the base messages changes.
- A merchant that does not list `openmarkets:takeout` in its capabilities is not asked for a pickup time, and a client never sends the entry to it.
- A client that does not know the entry refuses an acceptance that requires it, which is the envelope's rule for a required extension; it is better than showing an accepted order with no time and no code.

## Open questions

- Whether `code` should be optional for merchants that match orders by name alone.
- Whether a merchant should be able to answer a `pickup_at` it cannot honour with a counter-proposal, instead of a decline. This proposal keeps accept or decline only.
