# Plan: Polymarket structural-arbitrage scanner

Goal: find groups of logically linked Polymarket markets whose *executable* prices
contradict each other, so that a portfolio of purchases pays more than it costs
in every possible outcome. The work is about logical relationships between
markets, not about forecasting.

Constraints carried through every step:

- Read-only. Public Gamma and CLOB endpoints only; no wallet, no keys, no orders.
- Executable prices only: best ask to buy, best bid to sell, walked through
  depth. Midpoints, `outcomePrices` and last-trade prices are never used for a
  test (they may be shown for context).
- Every snapshot is timestamped; the books behind the top candidates are
  re-fetched immediately before the report is finalised.
- No opportunity is reported without its order-book levels and the full
  resolution text of every leg.

---

## 1. Data acquisition

| Source | Endpoint | Use |
|---|---|---|
| Gamma | `GET /events/keyset?active=true&closed=false&limit=…&after_cursor=…` (fallback: `GET /events?…&offset=…`) | All active events with nested markets: grouping metadata, negRisk IDs, resolution text, end dates, fee schedule, min order size, tick size |
| Gamma | `GET /markets?…` (only to back-fill markets whose nested copy lacks fields) | Same fields at market level |
| CLOB | `POST /books` (batches of token IDs; fallback `GET /book?token_id=`) | Full bid/ask ladders, `min_order_size`, `tick_size`, `neg_risk`, book timestamp, hash |
| CLOB | `GET /markets/{condition_id}` (candidates only) | Second source for `fees_enabled` / fee schedule before a candidate is reported |

Every raw response is written to `data/raw/<run-timestamp>/` with a manifest
(URL, parameters, HTTP status, UTC fetch time). The scan can be re-run offline
from that cache.

Book handling. Both tokens (YES and NO) of every binary market are fetched. A
"sell YES" is executed as "buy NO" (the CLOB matches a NO buy against YES bids
by minting), so every leg is modelled as a **purchase of a token at its ask
ladder**. The two token books are not merged, because they may be mirrors of
one order set; merging could double-count depth. Using one book per leg can
only understate liquidity.

## 2. Normalised model

- `Market`: question, outcome labels, YES/NO token IDs, condition ID, event ID
  and slug, `groupItemTitle`, negRisk flags (`negRisk`, `negRiskMarketID`,
  augmented flag, Other/placeholder status), end date, description (resolution
  text), resolution source, fee schedule, minimum order size, tick size,
  `acceptingOrders`.
- `Instrument` (one per token): a payoff function over atomic outcomes
  (YES pays `1[ω ∈ S]`, NO pays `1 − 1[ω ∈ S]`) and an ask ladder.
- `Group`: a set of markets plus a declared outcome space and the payoff
  encoding of each market in that space, a relation type, a match confidence
  and match notes.

## 3. Execution-cost model

1. **Taker fee** (Polymarket fee structure V2, effective 2026-03-30 and revised
   July 2026): `fee = C · rate · (p·(1−p))^exponent` USDC for `C` shares at
   price `p`. On buys the fee is collected in shares, so the net shares per gross
   share are `1 − f(p)/p` and the cost per net share is `p / (1 − f(p)/p)`.
   The schedule is read from each market's `feeSchedule` (`rate`, `exponent`,
   `takerOnly`, `rebateRate`) and `feesEnabled`. If fees are enabled but no
   schedule is present, the category table is used; if the category is unknown,
   the upper envelope of all category curves is used (conservative). Maker
   rebates are ignored, since every leg is a taker fill.
2. **Minimum order size**: each leg must be either 0 or at least the book's
   `min_order_size` shares, and at least the configurable minimum notional
   (default $1 for marketable orders). Enforced with a MILP when the LP solution
   violates it.
3. **Fixed costs**: on-chain transactions (negRisk conversion, redemption of
   each winning leg). Polymarket's relayer pays gas for proxy wallets, but the
   default charges a configurable $0.01 per transaction. Deposit cost defaults
   to $0 (capital assumed already on Polygon) and is configurable. Profit is
   reported before and after fixed costs.
4. **Tick and rounding**: fees round to 4 decimals; a configurable safety
   margin (default 0.1¢ per share) is added to every effective price.

## 4. Consistency engine

Three layers, from cheapest to most general. All three share the same
instrument and cost model.

### 4.1 Closed-form checks

| Check | Condition for arbitrage | Payoff in every outcome |
|---|---|---|
| YES basket on an exhaustive, mutually exclusive set of `n` | `Σ askYES_i < 1` | exactly 1 |
| NO basket on a mutually exclusive set (exhaustiveness not needed) | `Σ askNO_i < n − 1` | `n − 1` if one occurs, `n` if none |
| Implication `A ⇒ B` | `askNO_A + askYES_B < 1` (i.e. `bidA > askB`) | ≥ 1 |
| Equivalence `A ⇔ B` (duplicates) | implication test in both directions | ≥ 1 |
| Monotone ladder `A_1 ⇒ A_2 ⇒ … ⇒ A_k` | implication test on **every** ordered pair, not only neighbours | ≥ 1 |

Size: the legs are walked level by level together; the marginal cost of the
next share of the basket is the sum of the legs' current ask levels (fee
adjusted). The profit-maximising size is where that marginal cost reaches the
guaranteed payout; the break-even size (where cumulative profit returns to zero)
is also reported.

### 4.2 Explicit-atom LP (small outcome spaces)

1. **Atoms.** Enumerate mutually exclusive, exhaustive atomic outcomes `Ω`.
   Examples: each candidate of a negRisk event plus "none of the listed";
   intervals between all cut-points of a numeric quantity (see §5.4); the four
   cells of Senate × House control.
2. **Encoding.** Each market is a subset `S ⊆ Ω`; its YES token pays
   `1[ω ∈ S]`, its NO token pays `1[ω ∉ S]`.
3. **Feasibility (primal).** Find `q ≥ 0`, `Σq = 1`, such that for every token
   `E_q[payoff] ≤ ask`. Because NO is bought at `askNO ≈ 1 − bidYES`, this is
   the same as requiring `bidYES ≤ q(S) ≤ askYES` for every market. Implemented
   as `min s  s.t.  A q − s·1 ≤ c,  Σq = 1,  q ≥ 0`. If `s* ≤ 0`, a consistent
   distribution exists and there is no arbitrage among those markets.
4. **Dual → portfolio.** The dual of that LP is
   `max λ − cᵀx  s.t.  Σ_i x_i A_iω ≥ λ ∀ω,  Σx = 1,  x ≥ 0`. Its solution `x` is
   a portfolio (shares per token, normalised to one share in total) whose
   worst-case payoff `λ` exceeds its cost by `s*`. The dual values are read from
   the solver (HiGHS marginals) and cross-checked by solving the dual directly.
5. **Sizing with depth.** Each ladder level `k` of token `i` becomes a variable
   `0 ≤ x_ik ≤ size_ik` at its fee-adjusted price. Solve
   `max z − Σ price_ik x_ik  s.t.  z ≤ Σ_ik A_iω x_ik  ∀ω`. This yields the
   dollar-maximising portfolio across the whole depth, and the payoff table
   `payoff(ω) − cost` for every atom.

### 4.3 Sum-structured LP (large outcome spaces)

Aggregate markets (seat counts, control, Electoral College totals, number of
Fed cuts) depend on many component races. Enumerating `2^35` Senate outcomes is
infeasible, and assuming the races are independent is not allowed. Instead:

- **State.** Component `j` (a race, a meeting) takes an outcome `o_j` from a
  finite set, contributing an integer `w_j(o_j)` to a total
  `S = base + Σ_j w_j(o_j)`. An optional finite context variable `c` (for
  example which party controls the House) joins the state.
- **Payoffs.** Component markets pay `g(o_j)`; aggregate markets pay `f(S, c)`
  (bracket indicators, thresholds, control rules including tie-breaks).
  Components that have no market stay in the model as free variables, so the
  check never assumes an outcome for an unpriced race.
- **Cutting planes.** Solve the sizing LP of §4.2 over a working set of states.
  Then find the true worst-case state of the current portfolio by dynamic
  programming over the partial sum (`O(#components × range of S)` per context
  value). If that state pays less than `z`, add it and re-solve. Stop when no
  state violates `z`. The result is exact. It is verified in tests against
  brute-force enumeration.
- The primal reading is a column-generation search for a joint distribution
  over states. It is consistent with all prices if and only if the cutting-plane
  optimum is ≤ 0.

## 5. Relationship catalogue

Each relationship lists how it is **detected**, how it is **encoded** as a
test, and the main **resolution caveats** that verification must clear.

### 5.1 Exhaustive, mutually exclusive sets (R1)

- **Detection.** (a) Events with `negRisk = true`, grouped by
  `negRiskMarketID`. (b) Non-negRisk events whose markets share a "Will ___
  win/be …?" template with a single-winner title (`winner`, `who will`,
  `which`). These are lower confidence and go to the review log.
- **Tests.** YES basket (`Σ askYES < 1`), NO basket (`Σ askNO < n − 1`), and the
  explicit-atom LP over `{each outcome} ∪ {none listed}`, which also finds mixed
  YES/NO portfolios that neither closed form sees.
- **Coverage ("Other").** Detect an explicit Other / "Field" / "Someone else"
  outcome; detect augmented negRisk (`negRiskAugmented`, placeholder markets
  such as "Person A", Other). Rules applied:
  - The YES basket is a **true** arbitrage only when it holds every market of
    the negRisk set, including Other and placeholders. Polymarket narrows
    "Other" each time a placeholder is clarified, so a basket of named outcomes
    plus Other can lose if a placeholder is later named and wins. Without
    placeholders the result is a near-arbitrage.
  - Without an Other outcome, the YES basket is a near-arbitrage that depends on
    the listed outcomes covering every possibility (and on how "no winner",
    cancellation, or a 50-50 resolution is treated).
  - The NO basket needs only mutual exclusivity. In a negRisk event it can be
    converted at once through the NegRiskAdapter (`n` NOs → `n − 1` USDC), so
    the capital lock-up is close to zero. The adapter's conversion fee
    (`feeBips`) is checked.
- **Exclusivity sanity checks.** Flag events whose markets are nested rather
  than exclusive (date ladders "by March / by June", threshold ladders
  "above 100k / above 110k"), so that they go to §5.3/§5.4 instead.

### 5.2 Aggregate vs. component markets (R2)

| Aggregate | Components | Encoding |
|---|---|---|
| Senate seat-count brackets, "party X controls the Senate", balance-of-power combos | the 35 individual 2026 Senate races (33 Class II seats plus the Ohio and Florida specials) | §4.3 with `S` = party seats, `base` = seats not up in 2026 (from a configuration file that must be checked before use) |
| House control / seat brackets | the district markets that exist; districts without a market are free variables | §4.3, `base` = 0, 435 components |
| Governorship counts / control | the individual 2026 governor races | §4.3 |
| Electoral College totals, "party wins presidency" | state and district (ME, NE) winner markets, weighted by electoral votes | §4.3 with `w_j` = electoral votes |
| Popular-vote margin brackets | popular-vote winner markets | §4.2 on a 1-D margin quantity: "D wins popular vote" = margin > 0, with an atom at exactly 0 for a tie |
| Number of Fed cuts in a year / year-end target range | per-meeting decision markets (outcomes "no change", "25 bp cut", "50+ bp cut", "hike") | §4.3 with `w_j` in 25 bp steps and "50+" expanded into 50, 75, 100 bp sub-outcomes |
| Party wins the presidency | candidate-level winner markets, partitioned by the party nominee events | §4.2 with an atom per candidate plus "unlisted" atoms per party |

Race markets are matched by state, office, year and party (or by candidate when
the market lists a party tag such as "(D)"). Unmatched or ambiguous race
markets go to the review log.

### 5.3 Implication and nesting (R3)

`A ⇒ B` requires `P(A) ≤ P(B)`; the arbitrage is to buy NO on `A` and YES on `B`.

| Pair | Detection |
|---|---|
| X wins presidency ⇒ X wins a party nomination | candidate label appears in both the winner event and a nominee event; near-arbitrage at best, because X could win without that party's nomination |
| Happens by `T1` ⇒ happens by `T2 > T1` | same normalised question with the date phrase removed, within or across events; parsed deadlines |
| Price above $100k ⇒ above $90k (same asset, time, source) | parsed threshold and comparator; same subject key and same resolution timestamp |
| Wins by more than 5 ⇒ wins | same race and party; parsed margin threshold |
| Champion ⇒ conference/league champion ⇒ reaches final | team label in both negRisk events; title pattern pairs (Super Bowl/AFC/NFC, NBA Finals/East/West, Stanley Cup, World Series/pennant, etc.) |
| Touches $X during a period ⇒ ... | "hit ↑X" ladders are monotone in X; "dip ↓X" ladders are monotone in the opposite direction |

Every ladder is tested on all ordered pairs and, when it shares a quantity with
bracket markets, through the interval LP of §5.4.

### 5.4 Bracket and threshold consistency (R4)

- **Quantity key.** (subject, metric, observation time, resolution source),
  for example (BTC, Binance BTC/USDT 1-minute close, 2026-10-31 12:00 ET). Built
  from the question, `groupItemTitle` and the resolution text.
- **Atoms.** Sort every cut-point `c_1 < … < c_m` used by any market of the key.
  The atoms are the open intervals `(−∞, c_1), (c_1, c_2), …, (c_m, ∞)` plus the
  points `{c_i}`, so `>` and `≥` are encoded exactly and boundary values are
  never assumed away.
- **Encoding.** "Above X" = atoms with value `> X` (or `≥ X` per the rules);
  bracket "X–Y" = the atoms inside it with the stated endpoint inclusion;
  "by date T" = event time `≤ T` with "never" as the final atom.
- **Coverage.** Applies to crypto prices, CPI, Fed rate, GDP, unemployment,
  vote shares and margins, counts (tweets, seats, cuts) and dates.
- Discrete quantities (CPI published to 0.1) make some open-interval atoms
  impossible. They stay in the model by default (conservative). Any opportunity
  that disappears when they are dropped is a near-arbitrage at most.

### 5.5 Control-definition logic (R5)

- 2026 Senate: the Vice President (JD Vance, R) breaks a 50–50 tie, so
  "Republicans control" = `R seats ≥ 50`, "Democrats control" = `D seats ≥ 51`.
  Independents count with the party they caucus with only if the market's rules
  say so. An independent who wins and does not commit to a caucus (e.g. a
  Nebraska independent) is modelled as a third outcome, not as a Democrat.
- House: control = 218 seats. Vacancies and the timing of the count follow the
  market text.
- The resolution date matters: "control on January 3" vs. "after the election"
  vs. "majority leader" wording are different propositions; differences are
  recorded and downgrade the classification.

### 5.6 Duplicates and near-duplicates (R6)

- **Detection.** Normalise questions (case, punctuation, party and state
  synonyms, number formats such as `$100k` = `$100,000`, dates); bucket by
  normalised text and by (entities, threshold, date) signature; then use token
  Jaccard similarity ≥ 0.8 within a bucket.
- **Test.** Equivalence = implication both ways. Before a candidate is reported,
  the resolution texts are diffed. Any difference in source, deadline, time zone
  or edge-case clause makes it a near-arbitrage or a false positive.

### 5.7 Other relationships (record as discovered)

- **Joint/combination markets vs. marginals**: balance-of-power cells (Senate ×
  House) vs. each chamber's control market (§4.2 over 4 atoms).
- **Point-in-time close vs. touch markets**: "BTC closes above X at time t"
  implies "BTC touches X during a period containing t" when both use the same
  exchange and candle data. Recorded as near-arbitrage because the sources often
  differ.
- **Up/Down markets vs. price brackets**: once the opening price is known,
  "Up" is an interval of the closing price. These pairs are only matched
  manually because the candle definitions often differ.
- **50-50 resolutions**: markets whose rules allow a 50-50 payout (cancelled
  games, ambiguous outcomes) break basket payoffs. They are flagged during
  verification.
- **negRisk conversion**: used both as a lock-up reducer (NO baskets) and as an
  extra relation, `NO_i ≡ Σ_{j≠i} YES_j` in exhaustive sets, which the
  explicit-atom LP already includes.

## 6. Grouping and matching

1. Structural groups: event ID, event slug, `negRiskMarketID`.
2. Parsed attributes per market: US state, district, office, year, party,
   candidate, numeric threshold with comparator and unit, bracket bounds,
   deadline date, asset/subject, observation time.
3. Cross-event keys: (subject, metric, time) for numeric families; normalised
   question with the date removed for deadline families; label sets for nested
   negRisk events; race keys for elections.
4. **Confidence**: `high` (structural link, e.g. same negRisk set), `medium`
   (parsed keys match exactly and the texts agree on source/time), `low` (fuzzy
   match). Low-confidence groups are still tested, but every group that could
   not be matched with confidence is written to `review_queue.jsonl` and
   summarised in the report.

## 7. Verification protocol

For every candidate:

1. Print the full resolution text (`description`), resolution source, end date
   and event description of every leg.
2. Automatic flags: different resolution sources across legs; different end
   dates or time zones; "Other"/placeholder/augmented outcomes; 50-50 clauses;
   tie rules; "postponed/cancelled" clauses; deadlines in different time zones;
   threshold inclusivity (`above` vs `at or above`); markets not
   `acceptingOrders`; books older than the refresh window.
3. Manual reading against those flags, then classification:
   - **true arbitrage**: profitable in every outcome under every reading of
     the rules;
   - **near-arbitrage**: profitable only under a stated interpretation;
   - **false positive**: with the reason (usually different resolution
     criteria, stale or one-sided books, or a leg that cannot meet the minimum
     size).

## 8. Capital lock-up

Report the latest end date among the legs, the days locked, simple return
`r`, simple annualised `r · 365/d`, and compounded `(1+r)^(365/d) − 1`.
Conversions that release capital at once (negRisk NO baskets) are reported
separately. End dates are trading end dates; resolution can come later
(elections need certification), so a configurable delay is added.

## 9. Ranking and report

`REPORT.md` ranks the opportunities by net profit (USD) after costs and by
executable size, and lists for each one: the legs (market, token, side, levels
walked, average and worst price, shares, cost), the worst-case payoff table
over the atoms, fees and fixed costs, lock-up and annualised return, snapshot
timestamps (initial and refresh), resolution-text excerpts, verification notes
and the classification. A closing section lists every relationship type that was
tested and showed no mispricing, with the number of groups tested.

## 10. Tests

Synthetic markets with known answers:

- fee formula against Polymarket's published examples;
- ladder walking, profit-maximising and break-even sizes;
- YES-basket, NO-basket, implication, ladder and duplicate checks, each with an
  arbitrage case and a consistent case;
- explicit-atom LP: infeasible primal, dual certificate equal to the direct
  portfolio LP, payoff table ≥ profit in every atom, consistent prices give no
  arbitrage, mixed portfolios that the closed forms miss;
- sum-structured LP: cutting-plane optimum equal to brute-force enumeration on
  small instances; seat-count and control mispricings found; free (unpriced)
  components handled;
- text parsing (thresholds, brackets, dates, states, parties) and grouping;
- an end-to-end run on fixture API responses, through to the report.

## 11. Known limitations

- Polymarket API schemas change. The parser accepts the known field variants
  and logs unknown shapes rather than guessing.
- The match heuristics favour precision; anything uncertain goes to the review
  queue instead of being reported.
- Static configuration (Senate seats not up, electoral votes, VP party) must be
  checked against current facts before each run.
