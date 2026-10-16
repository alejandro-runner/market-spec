# Product Add-ons and Variation Details

Status: experimental proposal. This page is not current normative Open Markets text.

## Problem

A café sells a latte in two sizes. A customer can choose the milk, add syrups, and add extra shots. Each of those changes the price, and the order has to say exactly what was chosen so the café can make it and charge for it.

The specification already covers the sizes. A product can be published as a `variable` parent with one `variation` listing per size, and each size has its own price and stock. But it does not answer three questions, and this proposal answers them:

- The parent must carry a price. Which price?
- How does a size listing say "I am the 16 oz"?
- Can a customer order the parent itself, or only a size?

The specification has nothing for the milk, the syrups or the shots. Making every combination a separate size does not work: two sizes, four milks and a handful of syrups is dozens of listings for one latte. And an order line can only name a product and a quantity, so there is no way to say "oat milk, on the first latte, not the second".

This proposal adds two things.

The first is a rule a collection can carry, the **`select` tag**, that says how many of its members a customer may pick: exactly one milk, up to three syrups. The specification has no way to say that today.

The second is the **add-on**: an ordinary product that is sold only together with another product. Add-ons are grouped into **add-on groups**, which are collections marked as such and carrying a `select` rule, and a product says which groups it accepts. An order line that names an add-on also says which line it belongs to.

Everything added here is optional. A reader that knows nothing about this proposal still sees valid listings and a valid order with the right total. Synvya runs this design in production for restaurant takeout; the example below is a real coffee menu.

## Non-goals

- An add-on whose price depends on the size of the product it goes with. An add-on has one price.
- An add-on on an add-on.
- One product that is both sold on its own and used as an add-on. Fries as a side dish and fries added to a burger are two products.
- A second variation axis on one product. If a drink comes in sizes and also hot or iced, the sizes are the variations and "hot or iced" is an add-on group where the customer picks exactly one.

## Terms

- **Product**: a `kind:30402` listing.
- **Parent**: a product with `type` = `variable`. It describes the product and holds what its variations share. It is never sold.
- **Variation**: a product with `type` = `variation` and an `a` tag that names its parent. It is one sellable form of the parent, for example one size.
- **Add-on**: a product sold only together with another product: a syrup, a milk, an extra shot, a side.
- **Add-on group**: a `kind:30405` collection marked `["type", "add-on"]`, whose members are add-ons, with a `select` rule for how many may be picked.
- **Sellable product**: anything an order line may name: a simple product, a variation, or an add-on.

## Variations: three things the current text leaves unsaid

The specification requires a `price` on every product and says a variation must carry an `a` tag pointing at its parent. It stops there. The three readings below fill the gaps so that two implementations behave the same way.

### The parent's price is its lowest price

A parent's `price` is the lowest price among its variations, in the same currency. A reader that only knows one price per product shows it as a starting price: "from $4.50". When a merchant reprices a variation and that changes which one is cheapest, the merchant republishes the parent too.

### A variation says what it is with a `spec` tag

A variation carries one `spec` tag with the axis it varies on and its value:

```jsonc
["spec", "size", "16 oz"]
```

The axis is free text; `size` is the usual one. A reader lists a parent's variations by price, lowest first, and then by this value.

The variation's `title` is a complete name on its own, such as `Latte — 16 oz`, so a reader that ignores `spec` still shows something sensible. A reader never takes the label apart from the title; it reads `spec`.

### A parent is never sold, and each variation has its own stock

An order line never names a parent. If a merchant receives one that does, the merchant declines the order. A parent has no `stock` tag. Each variation carries its own stock, and a reader never adds variation stock up into a parent total or subtracts from one. The open [event-commerce proposal](https://github.com/OpenMarketsFoundation/specification/pull/15) says the same for the listings it assigns to events; this proposal says it for every variable product.

## Choosing from a collection: the `select` tag

A collection can say how many of its members a customer may pick. That is a new idea for the specification: today a collection only groups products, it never limits a choice. The rule is one tag:

```jsonc
["select", "<min>", "<max>"]
```

Read it as "pick at least `min` and at most `max` from this list". `1`, `1` means the customer must pick exactly one. `0`, `3` means the customer may pick none, or up to three. `min` and `max` are whole numbers, `min` is not above `max`, and `max` is at least `1`.

The numbers are for one unit of the product the choice goes with. "Up to three" on a syrups group means up to three pumps on that one latte, in any mix of flavours; a second latte on the same order gets its own three.

| Group | `select` | What the customer sees |
|---|---|---|
| Milk | `1`, `1` | Pick one; a milk is required |
| Syrups | `0`, `3` | Optional; up to three pumps in any mix |
| Extra shots | `0`, `4` | Optional; up to four of the one add-on |
| Tea blend | `1`, `1` | Pick one; every blend is free |

This proposal uses `select` on add-on groups only. The tag is defined on its own so that a later proposal can put the same rule on another kind of collection without redefining it.

## Add-ons

### The add-on product

An add-on is an ordinary simple product. Its price is what it adds to the order, and a free choice has a price of `0`. It references the same shipping options as the products it goes with, so a merchant's fulfilment check works line by line without special cases. Its `a` tag names the add-on group it belongs to.

```jsonc
{
  "kind": 30402,
  "content": "**Vanilla syrup**",
  "tags": [
    ["d", "syrup-vanilla"],
    ["title", "Vanilla syrup"],
    ["type", "simple", "physical"],
    ["price", "0.50", "USD"],
    ["shipping_option", "30406:<merchant>:pickup"],
    ["a", "30405:<merchant>:syrups"]
  ]
}
```

### The add-on group

An add-on group is a list of choices that go with a drink or a dish: the milks a latte can have, the syrups that can be added to any coffee, the blends a tea comes in. It is a `kind:30405` collection with two extra tags:

- `["type", "add-on"]` says this collection is an add-on group, not a menu or a section. Every product it lists is an add-on.
- `["select", "<min>", "<max>"]` says how many of them a customer may pick, as defined above. The tag is required on an add-on group.

The add-ons are listed in the order the merchant wants them shown. When the group requires a choice, the first add-on in that order that is not sold out is the one preselected for the customer.

A group has no `parent` tag, and it never lists a `variable` or `variation` product.

```jsonc
{
  "kind": 30405,
  "content": "",
  "tags": [
    ["d", "syrups"],
    ["title", "Syrups"],
    ["type", "add-on"],
    ["select", "0", "3"],
    ["a", "30402:<merchant>:syrup-vanilla"],
    ["a", "30402:<merchant>:syrup-caramel"]
  ]
}
```

### A product says which groups it accepts

A product lists each add-on group it accepts with an `add_on` tag, one per group:

```jsonc
["add_on", "30405:<merchant>:milk"],
["add_on", "30405:<merchant>:syrups"]
```

This is the same pattern the specification already uses for shipping: a product points at a collection to take on what that collection defines, and nothing applies to a product unless the product points at it. One group can serve any number of products, so "any coffee can have a syrup" is one syrups group referenced by every coffee.

A parent and each of its variations carry the same `add_on` tags, so every listing a customer might order is complete on its own. If a product points at a group that cannot be found, a reader ignores that tag, and a merchant republishing the product keeps it.

### What a reader shows

A reader looks at each product and decides what it is:

| Listing | How a reader can tell | Where it appears | Can it be ordered |
|---|---|---|---|
| Simple product | `type` is `simple`, and its collection is not marked `add-on` | On the menu | Yes |
| Parent | `type` is `variable` | On the menu, with its variations beneath it | No |
| Variation | `type` is `variation`, and its `a` tag names a parent that exists | Under its parent | Yes |
| Add-on | Its collection is marked `["type", "add-on"]` | As a choice on a product that accepts its group | Only together with a product |

A variation whose parent cannot be found, and an add-on whose group cannot be found, are neither shown nor sold.

## Order lines

An order is a list of lines, and each line names a sellable product and a quantity. This proposal adds one optional value to a line that names an add-on: the position of the line it belongs to, counting from zero.

In plain terms: the latte is line 0, so the oat milk line says "0". A second drink is line 3, so its extra shot says "3". Without this, an order with two drinks cannot say which one gets the oat milk.

### On the current `kind:16` order

The `item` tag gains an optional fourth value:

```jsonc
["item", "30402:<merchant>:latte-16-oz",   "1"],
["item", "30402:<merchant>:oat-milk",      "1", "0"],
["item", "30402:<merchant>:syrup-vanilla", "2", "0"],
["item", "30402:<merchant>:americano",     "1"],
["item", "30402:<merchant>:extra-shot",    "1", "3"]
```

That order is one 16 oz latte with oat milk and two pumps of vanilla, and one americano with an extra shot. `amount` is still the sum of every line.

### On the private commerce envelope

The [private structured commerce messages](https://github.com/OpenMarketsFoundation/specification/pull/12) proposal leaves the shape of an order line open. Where a line is written as `[coord, price, quantity, total]`, the link is an optional fifth value:

```jsonc
"order": [
  ["30402:<merchant>:latte-16-oz",   "5.00", 1, "5.00"],
  ["30402:<merchant>:oat-milk",      "0.60", 1, "0.60", 0],
  ["30402:<merchant>:syrup-vanilla", "0.50", 2, "1.00", 0]
]
```

### What makes an order well formed

A merchant checks these before accepting:

1. Every line names a product that can be sold. A line that names a parent is refused.
2. A line that names an add-on has a link, and a line with a link names an add-on.
3. The link points at a product line, never at another add-on line.
4. The product on that line accepts the add-on's group.
5. The add-on's quantity is a whole multiple of the product line's quantity. Two lattes with one vanilla each is a latte line of quantity 2 and a vanilla line of quantity 2. Two lattes with different add-ons are two latte lines.
6. For each group the product accepts, the number of add-ons picked from that group for one product stays within the group's `select` range.

A merchant that finds a line breaking one of these declines the order. Every line is priced at the merchant's own price for that product; nothing is computed from another line.

A reader that knows nothing about the link still sees every line as a product with a quantity, and the total is the same. The only thing it cannot tell is which product each add-on goes with.

## Stock

Each sellable product has its own stock, whether it is a simple product, a variation or an add-on. Running out of oat milk is `stock` = `0` on the oat milk product, and the merchant declines any line that names it. No product's stock is shared with or subtracted from another's.

## Worked example

A latte in two sizes, with a required milk and optional syrups.

```jsonc
// The parent
{ "kind": 30402, "content": "**Latte**\n\nEspresso with steamed milk.", "tags": [
  ["d", "latte"], ["title", "Latte"], ["type", "variable", "physical"],
  ["price", "4.50", "USD"],
  ["shipping_option", "30406:<merchant>:pickup"],
  ["add_on", "30405:<merchant>:milk"],
  ["add_on", "30405:<merchant>:syrups"],
  ["a", "30405:<merchant>:coffee"] ] }

// The 12 oz size
{ "kind": 30402, "content": "**Latte — 12 oz**\n\nEspresso with steamed milk.", "tags": [
  ["d", "latte-12-oz"], ["title", "Latte — 12 oz"], ["type", "variation", "physical"],
  ["price", "4.50", "USD"], ["spec", "size", "12 oz"], ["stock", "20"],
  ["shipping_option", "30406:<merchant>:pickup"],
  ["add_on", "30405:<merchant>:milk"],
  ["add_on", "30405:<merchant>:syrups"],
  ["a", "30402:<merchant>:latte"] ] }

// The 16 oz size
{ "kind": 30402, "content": "**Latte — 16 oz**\n\nEspresso with steamed milk.", "tags": [
  ["d", "latte-16-oz"], ["title", "Latte — 16 oz"], ["type", "variation", "physical"],
  ["price", "5.00", "USD"], ["spec", "size", "16 oz"], ["stock", "12"],
  ["shipping_option", "30406:<merchant>:pickup"],
  ["add_on", "30405:<merchant>:milk"],
  ["add_on", "30405:<merchant>:syrups"],
  ["a", "30402:<merchant>:latte"] ] }

// A group where the customer must pick one
{ "kind": 30405, "content": "", "tags": [
  ["d", "milk"], ["title", "Milk"], ["type", "add-on"], ["select", "1", "1"],
  ["a", "30402:<merchant>:whole-milk"],
  ["a", "30402:<merchant>:oat-milk"] ] }

// The free default in that group
{ "kind": 30402, "content": "**Whole milk**", "tags": [
  ["d", "whole-milk"], ["title", "Whole milk"], ["type", "simple", "physical"],
  ["price", "0", "USD"],
  ["shipping_option", "30406:<merchant>:pickup"],
  ["a", "30405:<merchant>:milk"] ] }
```

The "Coffee" section lists the latte only. A customer sees one latte, "from $4.50", with its two sizes, a milk to pick with whole milk preselected, and up to three syrups.

## Compatibility

- Every tag this proposal adds is optional, and every listing is still valid without it.
- A reader that ignores `type` and `select` on a collection sees an ordinary collection of cheap products. It never offers them with a drink, and nothing breaks.
- A reader that ignores the link on an order line reads a valid order with the right total.
- The three readings under *Variations* add no tag. They only fix a meaning where the text was silent.

## Open questions

- Whether `select` should also carry a limit per add-on (four shots in total, but at most two of one kind). None of the cases above needs it.
- Whether `select` should be allowed on collections that are not add-on groups, for example a bundle where the customer picks any three for one price. This proposal leaves the tag free for that but does not define it.
- Whether one product may be both sold on its own and used as an add-on. This proposal says no, because then two collections own it and can disagree.
- Whether an add-on's price may depend on the size of the product it goes with. This proposal says no; a merchant that needs it publishes two add-ons.
