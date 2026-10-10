---
layout: default
title: "Interactive: find the loops in a toy supplier graph"
---

# Interactive: find the loops in a toy supplier graph

## Intro (md cell)

This notebook is a small playground for one idea: when companies buy from each other, the interesting matches are **loops**. A loop is a set of companies where each buys from the next and the last buys from the first. Every company in the loop gets a supplier it needs and a customer who pays.

Below is a toy list of eight fictional companies, each with one thing it sells and a few things it needs. Fork the notebook and edit the list, or select rows to restrict it; the notebook recomputes every pair and chain up to the length you choose.

## Cell: the companies (editable)

```js
viewof companies = Inputs.table(
  [
    {name: "Alder Studio",   sells: "design",      needs: "accounting, hosting"},
    {name: "Birch Books",    sells: "accounting",  needs: "recruiting"},
    {name: "Cedar Hire",     sells: "recruiting",  needs: "design, legal"},
    {name: "Dune Hosting",   sells: "hosting",     needs: "support desk"},
    {name: "Elm Support",    sells: "support desk",needs: "design"},
    {name: "Fir Legal",      sells: "legal",       needs: "accounting"},
    {name: "Gale Analytics", sells: "analytics",   needs: "hosting, legal"},
    {name: "Holly CRM",      sells: "crm",         needs: "analytics"}
  ]
)
```

(With no rows selected, the table's value is all rows; select rows to restrict the graph to them. To change needs, fork and edit the literal; the rest of the notebook reacts.)

## Cell: maximum loop length

```js
viewof maxLen = Inputs.range([2, 5], {step: 1, value: 3, label: "Max companies per loop"})
```

## Cell: build the graph

```js
graph = {
  const rows = companies.length ? companies : [];
  const sellersOf = new Map();
  for (const c of rows) {
    const k = c.sells.trim();
    if (!sellersOf.has(k)) sellersOf.set(k, []);
    sellersOf.get(k).push(c.name);
  }
  const adj = new Map(rows.map(c => [c.name, new Set()]));
  for (const c of rows) {
    for (const need of c.needs.split(",").map(s => s.trim()).filter(Boolean)) {
      for (const seller of sellersOf.get(need) ?? []) {
        if (seller !== c.name) adj.get(c.name).add(seller); // c buys from seller
      }
    }
  }
  return adj;
}
```

## Cell: find loops

```js
loops = {
  const names = [...graph.keys()].sort();
  const out = [];
  for (const start of names) {
    const path = [start];
    const seen = new Set([start]);
    (function dfs(node) {
      for (const next of graph.get(node)) {
        if (next === start && path.length >= 2) {
          // report each loop once: only when start is its smallest name
          if (path.every(p => p >= start)) out.push([...path]);
        } else if (!seen.has(next) && next > start && path.length < maxLen) {
          seen.add(next); path.push(next);
          dfs(next);
          path.pop(); seen.delete(next);
        }
      }
    })(start);
  }
  return out.map(p => ({
    size: p.length,
    loop: [...p, p[0]].join(" → "),
    reads: p.map((x, i) => `${x} buys from ${p[(i + 1) % p.length]}`).join("; ")
  }));
}
```

The trick `next > start` makes every loop appear exactly once, starting from its alphabetically smallest member, instead of once per rotation.

## Cell: results

```js
Inputs.table(loops, {columns: ["size", "loop", "reads"], width: {loop: 360}})
```

## Cell: who is left out

```js
leftOut = [...graph.keys()].filter(n => !loops.some(l => l.loop.includes(n)))
```

## What to try (md cell)

1. **Set the maximum to 2.** With the default data there are no pairs at all: no two companies need each other's products.
2. **Set it to 3.** Three loops appear: Alder Studio → Birch Books → Cedar Hire (design studio buys accounting, accounting company buys recruiting, recruiting startup buys design), Alder Studio → Dune Hosting → Elm Support, and Birch Books → Cedar Hire → Fir Legal. Six of eight companies now have a match, without a single pair.
3. **Set it to 4 or 5.** Nothing new appears. Longer loops are not the bottleneck here; demand is.
4. **Look at "left out".** Gale Analytics and Holly CRM stay unmatched at any length: nobody needs CRM, and the only company that needs analytics is Holly. Add "crm" to Alder Studio's needs, set the maximum to 5, and both join a five-company loop through Dune Hosting and Elm Support.

## Why this matters (md cell)

The toy shows three things that hold in real matching systems too:

- **Pairs are the exception.** Most matches only exist as loops of three or more.
- **Short loops are better.** A three-company loop is easy to explain and to agree on. Every extra company adds one more decision that can fail.
- **Coverage depends on demand, not on size.** A company is matched only if someone needs what it sells. Growing the network helps only where it adds demand.

Real systems add what the toy ignores: verified companies, fit and price, independent acceptance by every participant, a trial before payment, and rules for what happens when one company leaves a loop.

## Context (md cell)

I built this playground while thinking about [ProductBay (productbay.io)](https://productbay.io), a matching network where companies buy each other's software and services in pairs and chains (pre-launch). All companies above are fictional. Fork it and try your own industry.

[Back to all notes](./)
