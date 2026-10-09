---
layout: default
title: "Repairing a broken chain: replacing one participant in a multi-party match without starting over"
---

# Repairing a broken chain: replacing one participant in a multi-party match without starting over

Say you run a matching system where companies buy from each other in loops. Company A buys a tool from B, B buys from C, C buys from A. Each company gets a supplier it needs and a customer who pays. Everybody has to accept before anything is charged, and each has a limited window to decide.

Now A and B accept. C declines.

The lazy answer is to throw the whole match away and start over. That wastes the two acceptances you already have, and it means A and B have to go through the whole decision again with different partners. The better answer is to **repair** the chain: find a company C′ that can take C's place, so A and B's decisions still stand.

This post walks through how to find that replacement, with TypeScript.

## What a slot actually requires

In a loop, each participant fills a slot defined by its two neighbours. For C in `A → B → C → A` (arrow meaning "buys from"):

- B buys from C, so the replacement must **sell something B needs**.
- C buys from A, so the replacement must **need something A sells**.

Those two constraints define the slot. Anything else (size, region, language, budget) is a filter on top.

```ts
type CompanyId = string;

interface Company {
  id: CompanyId;
  verified: boolean;
  sells: Set<string>;      // category ids
  needs: Set<string>;
  region: string;
}

interface Slot {
  supplierTo: CompanyId;   // the neighbour who will buy from the replacement (B)
  customerOf: CompanyId;   // the neighbour the replacement will buy from (A)
  needCategory: string;    // what B needs
  offerCategory: string;   // what A sells
}
```

Note that the slot stores the specific categories that were matched, not just the neighbours. B may need several things; the slot is about the one thing C was going to deliver.

## Candidate search

A first pass finds every company that satisfies both sides of the slot. With a category index (category → companies) it is a set intersection, not a scan.

```ts
interface Index {
  sellersOf: Map<string, Set<CompanyId>>;
  buyersOf: Map<string, Set<CompanyId>>;
}

function candidates(slot: Slot, idx: Index, exclude: Set<CompanyId>): CompanyId[] {
  const canSupply = idx.sellersOf.get(slot.needCategory) ?? new Set();
  const wantsOffer = idx.buyersOf.get(slot.offerCategory) ?? new Set();
  const [small, large] = canSupply.size < wantsOffer.size
    ? [canSupply, wantsOffer] : [wantsOffer, canSupply];
  const out: CompanyId[] = [];
  for (const id of small) {
    if (large.has(id) && !exclude.has(id)) out.push(id);
  }
  return out;
}
```

`exclude` holds everyone already in the chain plus everyone who declined this slot before. Without it, the algorithm happily proposes C again, or proposes A as its own supplier.

Iterating over the smaller set is a small thing that matters once categories get popular: "CRM" may have hundreds of sellers while "needs our niche analytics tool" has five buyers.

## Ranking: the replacement must fit both neighbours

Every candidate satisfies the hard constraints. Ranking decides which one to ask first. A candidate is a good stand-in if it would have been a good match in the first place, so the ranking reuses the same fit signals as the original matching, scored against **both** neighbours.

```ts
function rank(slot: Slot, ids: CompanyId[], get: (id: CompanyId) => Company,
              fit: (buyer: Company, seller: Company, category: string) => number) {
  const b = get(slot.supplierTo);
  const a = get(slot.customerOf);
  return ids
    .map(id => {
      const c = get(id);
      const asSeller = fit(b, c, slot.needCategory);   // would B want to buy from c?
      const asBuyer  = fit(c, a, slot.offerCategory);  // would c want to buy from A?
      return { id, score: Math.min(asSeller, asBuyer) };
    })
    .filter(x => x.score > 0)
    .sort((x, y) => y.score - x.score);
}
```

Why `min` and not the sum? Because a chain is only as strong as its weakest link. A candidate who is a perfect supplier for B but a lukewarm buyer of A's product will probably decline, and then you have lost another decision window. `min` prefers balanced candidates.

## Asking one at a time, with a budget

Once you have a ranked list, the tempting move is to ask the top five candidates at once and take the first yes. That creates a new problem: four companies get an offer that disappears as soon as someone else accepts. People notice when they are being used as a backup.

So the repair asks one candidate at a time, each with its own decision window, and stops after a fixed number of attempts:

```ts
async function repair(slot: Slot, ranked: { id: CompanyId }[],
                      ask: (id: CompanyId) => Promise<'accept' | 'decline' | 'timeout'>,
                      maxAttempts = 3): Promise<CompanyId | null> {
  for (const { id } of ranked.slice(0, maxAttempts)) {
    const answer = await ask(id);
    if (answer === 'accept') return id;
  }
  return null;
}
```

In a real system `ask` is not an `await` that blocks for 24 hours; it is a state transition and a timer, driven by events. But the logic is the same, and writing it as a loop first made the rules much easier to review.

## When repair fails

If no stand-in accepts within the attempt budget, the chain is dissolved and nobody is charged. A and B are told plainly what happened and go back into the matching pool. Their earlier acceptances are not reused for a different chain without asking again, because they agreed to a specific supplier and customer, not to "whatever comes next".

That rule is easy to get wrong in the name of efficiency. Consent in a multi-party deal is attached to the concrete partners, not to the abstract slot.

## A note on fairness and speed

Repair keeps the two companies who said yes from being punished for the one who said no. It also means the decliner is not under pressure: saying no is cheap for them, because the system can usually recover. That matters. A system where declining blows up a deal for everyone else creates social pressure to accept, and pressure is how reciprocal buying goes wrong.

![Example of a match between companies](https://files.catbox.moe/16oktt.png)

I worked through this repair problem while thinking about the matching design of [ProductBay (productbay.io)](https://productbay.io), a network where verified companies buy each other's software and services in pairs and chains (pre-launch). If you have worked on kidney-exchange-style or other multi-party matching, I would like to know how you handled the "ask one or ask many" question.
