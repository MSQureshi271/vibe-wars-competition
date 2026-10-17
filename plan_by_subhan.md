# Vibe Wars Competition Plan

> **Status:** Pre-starter-kit strategy and implementation plan
>
> **Source:** `vibe-wars-study-guide.pdf`
>
> **Important:** The starter repository, `docs/RULES.md`,
> `docs/BOT_SDK_CONTRACT.md`, and the real engine are authoritative when they
> arrive. This document deliberately separates confirmed rules from design
> decisions that must be checked against the SDK.

## 1. Objective

We will submit one reliable Python file, `my_bot.py`, containing a class named
`MyBot`. The bot must survive unseen maps and opponents, make a legal and
deterministic decision within 200 ms, and finish near the top on **every map**.

The tournament rewards series placement rather than raw map score:

| Map finish | Series points |
|---|---:|
| 1st | 3 |
| 2nd | 2 |
| 3rd | 1 |
| 4th | 0 |

Therefore our primary objective is not “maximize kills at any cost.” It is:

1. Never crash, time out, or return an illegal action.
2. Establish a dependable source of points on nearly every map.
3. Convert favorable situations into kills, tags, cargo banks, and Core uplinks.
4. Avoid high-variance fights when a safe second-place result is more valuable.
5. Exploit overtime, role bonuses, marked targets, and predictable event timing.

The intended bot is an **adaptive hybrid**: it normally farms safe cargo and
opportunistic Core value, but temporarily becomes an aggressive combatant when
the view provides a high-confidence kill, a valuable marked target, or a late
game scoring opportunity.

## 2. Confirmed competition model

### 2.1 Match and visibility

- Four bots play simultaneously in a walled 12 x 12 arena.
- A map lasts 40 turns; turns 32–39 are Overtime.
- All bots choose from the same turn-start state, then actions resolve
  together. We cannot react to an action until the next turn.
- Normal vision reaches 3 cells and is blocked by walls. A `SCOPE` upgrade,
  `SCAN`, radar sweep, and radar item can extend or reveal visibility.
- The complete wall layout, Core, uplink pads, crates, smoke, airstrike
  markers, and marked bots are always visible as specified by the view.
- We see our own score and rank, but not rival scores.
- Starting corners rotate across the series, so the strategy must be coordinate
  independent and must not memorize one spawn corner.
- Wall layouts change between maps. We must use the current view and the SDK's
  geometry helpers rather than hard-coded routes.

### 2.2 Baseline bot resources

- Health: 70.
- Energy: 50 maximum.
- Energy recovery: +5 each turn, or +10 in Overtime, capped at 50.
- Cargo capacity: 4 resource units before upgrades and role effects.
- Normal respawn: 3 turns after death, at our base, with full health and energy.
- Spawn protection: 2 turns after respawn; taking an offensive or collection
  action ends it early.
- A death loses carried cargo, held item, and kill streak, but not banked score.
- There is no permanent elimination during a map.

### 2.3 Actions and costs

Every action returned by `decide(view)` will be selected only after checking
its cost and all target, cooldown, visibility, wall, and board constraints.

| Action | Cost | Planned use |
|---|---:|---|
| Move one of 8 directions | 1 | Path following, dodging, retreating, positioning |
| Attack adjacent bot | 5 | High-confidence melee finish or close-range trade |
| Shoot up to 4 cells in a straight line | 6 | Safe damage through a current clear lane |
| Collect object under us | 2 | Resource, crate, tag, or Core according to SDK object format |
| Defend | 2 | Survive an imminent blast or likely incoming damage |
| Scan | 8 | Reveal extra cells next turn when a search is worthwhile |
| Ability | 10 | Mobility, defense, displacement, or area damage |
| Use held item | 4 | Healing, damage, information, smoke, or legendary effect |

Walking into a wall or bot is legal but wastes energy and position. Walking off
the board is illegal. A same-cell movement collision means neither bot moves.
Diagonal movement and line of sight obey corner-blocking rules, so geometry
must be delegated to `vibe_sdk.geometry` where available.

### 2.4 Scoring priorities

The score sources have different reliability and risk:

- **Cargo:** collect resources, carry them home, and stand on our base to bank
  them. Cargo in the hold at match end is worth zero.
- **Core:** collect it after turn 8, carry it onto any uplink pad, and protect
  the planted Core for 3 turns for +60. An enemy can defuse it by remaining
  within 1 cell for 2 consecutive turns; all eligible enemies share +40.
- **Kills:** a kill is +25 immediately and +15 when its tag is collected, with
  additional bonuses for First Blood, crown kills, shutdowns, streaks, revenge,
  nemesis, clutch health, assists, blast kills, shove kills, and tag denial.
- **Overtime:** points from kills, bonuses, banked cargo, uplinks, and defuses
  double from turn 32 onward. Energy recovery also doubles.

The scoring model creates a key rule: **do not die while carrying unbanked
value unless the alternative is an even larger guaranteed objective.**

## 3. Strategic thesis

### 3.1 Why a hybrid bot is the safest baseline

The hidden opponents may be hunters, cargo farmers, Core specialists, or broken

- A pure hunter can spend the whole map chasing targets and finish last if
  opponents avoid fights.
- A pure hauler can be intercepted and may never convert its cargo.
- A pure Core runner exposes itself to three enemies and can lose a map on one
  failed escort or failed defuse.

We will instead maintain a short-horizon state machine:

1. **Recover / survive** when low health, carrying valuable cargo, threatened,
   or freshly respawned.
2. **Secure immediate objective** when standing on a collectable or base.
3. **Exploit a high-confidence attack** when the target is adjacent, visible,
   in a legal firing lane, or predictably vulnerable to an item.
4. **Contest the Core** when the route and local threat make capture or defuse
   favorable.
5. **Farm cargo** when no high-value tactical opportunity is present.
6. **Position for Overtime** before turn 32, then favor actions with immediate
   doubled value and safe conversion.

The bot will not attempt expensive long-horizon planning every turn. A fast
priority policy plus a small amount of memory and bounded pathfinding is more
likely to pass conformance and remain within the time budget.

### 3.2 Series-aware risk policy

The bot can see its current rank and score but not rival scores. We will use
rank as a noisy signal, not as exact information:

- When currently 1st or 2nd, prefer banked cargo, survival, safe tags, and
  low-risk Core denial over speculative fights.
- When currently 3rd or 4th, accept more combat and Core risk, especially in
  Overtime, because a safe low score may not change placement.
- A visible crown is an attractive target, but only if the attack is legal and
  the escape/retreat path is not obviously fatal.
- We will not assume rank changes only after our own score updates; evaluate it
  fresh from every view.

## 4. Bot behavior by match phase

The phase thresholds are defaults. We will tune them from real engine logs.

### Phase A — Opening: turns 0–7

Goals: leave the spawn pocket efficiently, collect a nearby safe resource or
crate, and reach a useful central approach without entering a blind fight.

Rules:

1. Identify our base and current location from the view; do not infer a corner
   from fixed coordinates.
2. Prefer the highest-value visible resource whose route to our base is short and
   does not pass through a visible enemy's attack range.
3. If a crate is on the route and the item slot is empty, collect it; otherwise
   do not detour merely for a random item.
4. Do not spend the opening on `SCAN` unless the map geometry or a known event
   makes an early reveal materially useful.
5. Keep enough energy for an emergency `DEFEND`, attack, or mobility action.
6. Move using a bounded wall-aware path. If the first step is blocked, select a
   legal alternative rather than repeatedly trying the same cell.

### Phase B — Core arrival: turns 8–15

The Core first appears on turn 8 and is always visible. It is the most important
decision point in the map.

1. Estimate a route to the Core and then to the nearest safe uplink pad.
2. Contest it when the route is short, our health is adequate, and visible
   threats do not make immediate death likely.
3. If another bot carries the Core, prioritize an intercept or legal ranged
   attack, especially when the carrier is marked, isolated, low health, or near
   a predictable choke point.
4. If we carry the Core, minimize detours. Use mobility or defense only when it
   increases the probability of reaching a pad and surviving the countdown.
5. If an enemy plants the Core, decide whether to defuse, attack the planter,
   or continue cargo based on distance, health, remaining countdown, and number
   of enemies already near the Core.
6. Do not let a Core chase cause us to lose a full cargo hold or walk into an
   unavoidable multi-bot crossfire.

### Phase C — Midgame: turns 16–31

Goals: bank repeatedly, collect high-value tags when safe, exploit events/items,
and keep the Core in the decision loop.

1. A full hold sends the bot home unless a visible kill, tag, or Core opportunity
   has a substantially higher expected value.
2. A partially full hold may be banked before a dangerous route or before
   Overtime; do not let cargo remain unbanked just because capacity is not full.
3. A resource collected near an enemy gets the 1.3 hot-pickup multiplier, but a
   hot pickup is worth taking only if we can escape and bank it.
4. Keep a recent-memory record of visible enemies, their last known locations,
   marks, health, and whether we damaged them. Treat stale locations as
   hypotheses, not current facts.
5. During `BLOOD_MOON`, raise the value of a safe attack and lower the threshold
   for `DEFEND`, retreat, and Medpack use. Blasts are not boosted, so do not
   overvalue bombs/airstrikes solely because of the event.
6. During `RESOURCE_RUSH`, prioritize the nearest safe high-value resource and
   expect more traffic in the inner arena.
7. Rotate between objective routes when a previous route is repeatedly blocked;
   never assume an opponent will move out of the way.

### Phase D — Overtime setup: turns 28–31

Overtime begins on turn 32. Before it starts:

1. Bank cargo rather than entering Overtime with a full hold far from base.
2. Heal if the item is available and health is low enough for the heal to matter.
3. Preserve energy for the doubled-reward window; do not spend all energy on
   low-probability attacks at turn 31.
4. Move toward a safe resource, tag, Core route, or defensive position that can
   score immediately after turn 32.
5. If we are ahead by rank, prepare a low-risk conversion route. If behind, set
   up an aggressive but legal Core or kill attempt.

### Phase E — Overtime: turns 32–39

Every scoring action is more valuable, but death still costs time and carried
value. The ordering changes:

1. Take a guaranteed or near-guaranteed doubled score now: bank, collect a safe
   tag, complete a viable uplink, or complete a defuse.
2. Finish a high-confidence kill if its attack can land this turn and the escape
   risk is acceptable.
3. Use items and abilities aggressively when they produce a score before the
   match ends. A delayed blast that lands after turn 39 is useless.
4. Stop collecting cargo that cannot reach base before the final turn.
5. At the last one or two turns, prefer base/score conversion or a legal
   high-probability attack over travel toward a distant objective.

## 5. Action priority policy

`decide(view)` will evaluate the following top-to-bottom ladder. Every helper
will return either a fully validated action or `None`; the final fallback will
always be a legal `DEFEND` or legal one-step `MOVE`.

### Priority 0 — State and safety gate

Before strategic decisions:

- Read `self`, turn number, board dimensions, walls, energy, health, score,
  rank, cooldowns, current item, cargo, Core state, crates, resources,
  markers, and visible enemies using defensive `.get(...)` access where the
  contract allows it.
- Detect whether we are dead, respawning, spawn-protected, carrying Core, or
  carrying cargo.
- Recompute available legal directions from the current map; never reuse a
  path step without checking the current view.
- Estimate imminent hazards from bomb, airstrike, orbital, and smoke markers.
- If a scheduled blast will hit our current cell and movement cannot safely
  escape, use the action that protects us on the correct preceding turn
  (`DEFEND`, `SHIELD`, or `OVERSHIELD` behavior as applicable).

### Priority 1 — Immediate collection and banking

If standing on a collectable object and the action is legal:

- Collect the Core if we are in the approved Core mode.
- Otherwise collect a resource if capacity remains and the resource is worth
  taking.
- Collect a kill tag when it confirms our own kill or denies a rival's tag, with
  the distance and danger threshold checked.
- Collect a crate only when the slot policy says the item is valuable.

If standing on our base with bankable cargo, bank through the engine's normal
base behavior (usually by taking no special collect action; this must be
verified in `BOT_SDK_CONTRACT.md`). If the engine requires an explicit action,
use that action exactly as documented.

### Priority 2 — Escape, heal, and defense

Use a Medpack when health is low, the heal is legal, and keeping the item is
less valuable than surviving. Use smoke or mobility to break a known threat
when it leads to a safe route. Defend when:

- a blast is due and we cannot move clear,
- multiple visible attacks could land this turn,
- our health is low and an offensive action is not a high-confidence kill, or
- we are protecting a planted Core and defense meaningfully increases its
  survival probability.

Do not defend by default. It costs energy and produces no score.

### Priority 3 — Guaranteed or high-confidence kill

For every visible enemy, calculate:

- Chebyshev distance for adjacency/range.
- Current and expected line of sight through the current walls.
- Whether the target is already damaged or marked (`CROWN`, `CORE`, `CARGO`,
  `PLANTER`, `ON_FIRE`, or `LOUD`).
- Whether we have enough energy and whether the action lands before/after
  movement in the relevant way.
- Whether the target can be killed by one action or a deterministic sequence.
- Whether the action leaves us exposed to a likely counterattack.

Attack adjacent targets when it is legal and the target is a credible kill or a
valuable carrier. Shoot only when the current geometry gives a clear legal line
and the target is within four cells. Prefer a kill on a crown/Core carrier or a
target worth a revenge/nemesis/shutdown bonus, but do not chase an invulnerable
or protected target.

### Priority 4 — Core objective

Core behavior is a small state machine:

`AVAILABLE -> APPROACHING -> CARRYING -> PLANTING -> DEFENDING` or
`AVAILABLE -> CONTESTING -> DEFUSING`.

We will keep the chosen objective in `self` only as a hint and reconcile it with
the current view every turn. The Core is always visible, so stale state must
never override the current Core status.

### Priority 5 — Valuable item use

Use items only when the target and timing are valid:

- **Medpack:** health threshold and no better immediate use.
- **Smoke:** escape or approach, especially with cargo/Core; remember marks
  remain visible through smoke.
- **Radar:** information is most valuable before a planned Core run, hunt, or
  Overtime push; do not use it when the match is nearly over.
- **Railgun:** fire only along a currently legal line; prefer multiple enemies,
  a carrier, or a finish.
- **Airstrike:** aim at an enemy cluster or predictable future cell, accounting
  for the two-turn delay and visible marker.
- **Orbital:** use only if the delayed radius-2 blast has a high-confidence
  target or denies a decisive Core objective.
- **Overshield:** use before a likely lethal hit, not after damage is already
  resolved.
- **Reaper:** use before a sequence of likely kills; do not consume it for a
  speculative single attack.

If an item is not present in `view["item_pool"]` or the current held item is not
the expected type, the branch must be skipped safely.

### Priority 6 — Cargo route

When no tactical event dominates:

1. If capacity is full, path to base.
2. If Overtime is near and the route home is short, bank before turn 32.
3. Choose the best visible resource using value, path length, danger, hot-pickup
   opportunity, and whether the resource can still be banked before the match
   ends.
4. If no resource is visible, move to a search waypoint that preserves a route
   home and avoids known enemy locations.
5. If the hold has valuable cargo and a threat becomes visible, abort the target
   route and retreat.

### Priority 7 — Positioning and fallback

If no objective exists, move toward a safe, useful waypoint: a central approach,
nearby cover, a resource cluster, or a route to base. If all strategic moves are
blocked or unsafe, use a legal `DEFEND`; do not return `None`, an unknown action,
or an unchecked move.

## 6. Navigation and geometry plan

### 6.1 Use the engine's geometry

The guide explicitly names `vibe_sdk.geometry.line_clear`, `shot_path`, and
`step_open`. Once the SDK arrives, inspect their signatures and use them rather
than reimplementing wall corner rules. The code must tolerate any small naming
or argument differences by following the contract, not by guessing.

### 6.2 Bounded pathfinding

The arena is only 144 cells, but the 200 ms budget is shared with all decision
logic. We will use a bounded BFS/A* implementation with:

- Current walls as obstacles.
- Board edge checks.
- Current visible bot positions as temporary obstacles when appropriate.
- Eight-direction steps and the engine's diagonal corner rule.
- A deterministic neighbor ordering.
- A maximum expansion limit; if exceeded, fall back to a safe legal step.

The pathfinder will produce a first step, not a long-lived commitment. Replan
after every turn because all bots move simultaneously and the board state can
change.

### 6.3 Deterministic tie-breaking

All candidate choices will use explicit stable ordering:

1. Higher expected score.
2. Lower immediate death risk.
3. Shorter path length.
4. Greater distance from threats.
5. Fixed object priority or lexicographic ID/coordinate order.

No set iteration, hash-dependent ordering, current time, network input, file
reads, or unseeded randomness may influence a decision.

## 7. Combat design

### 7.1 Target selection

For visible enemies, calculate a deterministic target score. The score should
increase for:

- an enemy that can be killed this turn;
- a Core carrier or live planter;
- the crown holder;
- a high cargo carrier;
- a target with a kill streak (shutdown bonus);
- our revenge or previous-map nemesis;
- a target already damaged by us, enabling an assist if someone else finishes;
- a target that is isolated and cannot immediately counterattack.

It should decrease for:

- spawn protection;
- poor line of sight or a route through a hazard;
- a target near multiple enemies;
- low energy after the attack;
- carrying our own valuable cargo/Core with no escape;
- an attack that is legal but cannot realistically finish or gain objective
  value.

### 7.2 Damage and kill confirmation

The bot must not assume exact damage when the rules give ranges. It will use
conservative bounds for decisions. For example, a target at 20 health is not
necessarily killed by an attack whose damage range is 16–24 unless the lower
bound is sufficient.

After our own kill, prioritize collecting the tag when it is safe. The tag lasts
6 turns, and collecting it gives the confirmation points. If another bot is
likely to take it first, tag denial may be worthwhile, but not if it costs a
death or loses a more valuable bank.

### 7.3 Fighting during Blood Moon

Blood Moon increases attack, shoot, railgun, and shove-slam damage by 1.5 but
does not increase blast damage. We will:

- use a lower threshold for a clean ranged finish;
- avoid opening a melee trade when the enemy's boosted retaliation kills us;
- prefer railgun/attack/shoot only when the current line and health checks are
  complete;
- use blasts for area control rather than assuming their damage is boosted.

## 8. Core strategy in detail

### 8.1 Capture decision

Attempt a Core pickup when all of the following are true, unless we are in an
emergency comeback mode:

- the Core is available and collectable;
- we can reach it with a bounded, legal path;
- we have enough energy to collect and still move/defend;
- our health and visible threat count support the trip;
- the selected uplink pad has a reasonable route and the countdown can be
  defended or made irrelevant by distance.

The exact thresholds will be tuned from simulations rather than hard-coded from
the guide alone.

### 8.2 Choosing a pad

Score pads by route length, wall cover, distance from visible enemies, distance
from our base/cargo route, and the number of adjacent escape cells. Prefer a pad
that makes an enemy defuse difficult. Never assume the nearest pad is safest.

### 8.3 Planted Core defense

When we plant:

- mark the expected countdown in memory but always read the actual view;
- attack a nearby defuser or planter threat if the action is legal;
- use shield/defend, smoke, or mobility when it increases survival;
- remain near enough to contest a defuse without standing in a visible blast;
- if death is unavoidable, prioritize completing the countdown rather than
  abandoning the planted objective for a small cargo pickup.

### 8.4 Defusing an enemy Core

To defuse, we need to stay within 1 cell for 2 consecutive turns. The bot will
remember the number of consecutive qualifying turns only as a cross-check; the
view's planted-Core state is authoritative. Defuse if:

- we can reach the area before the uplink completes;
- the enemy cannot immediately kill or displace us;
- the +40 doubled Overtime value justifies the risk; or
- we are behind and need a swing objective.

Warden is a possible role for a Core-defense/defuse plan, but the role cannot
pick up the Core. We will compare it experimentally against Runner and neutral
play before locking the policy.

## 9. Role, ability, and shop policy

### 9.1 Initial default configuration to test

The first implementation should test these candidates rather than commit to one
without measurements:

- **Ability candidate A — DASH:** reliable movement through open cells and
  useful for reaching pads or escaping; requires both traversed cells to be
  free.
- **Ability candidate B — BLINK:** strong escape/route correction over walls
  and bots; landing cell must be free.
- **Ability candidate C — SHIELD:** simple defensive value but only for one
  turn and no movement.

The initial hybrid candidate is **DASH** because it improves both cargo routes
and Core conversion, but BLINK may outperform it on dense maps. We will make
the final choice after checking actual cooldown fields and running map seeds.

Bomb and Shove are higher-variance candidates: Bomb damages us and is visible,
while Shove needs a very specific nearby geometry. They should only be selected
if simulations demonstrate consistent value.

### 9.2 Role candidates

Roles change the optimization target and have serious penalties:

- **Hunter:** +10 per kill and streak bonuses, but reduced cargo banking and a
  larger bounty on our death. Use only if combat testing shows reliable kills.
- **Hauler:** +1 cargo space and strong multi-unit banking, but kills pay 10 less.
  Good default if safe resources are abundant and opponents fight elsewhere.
- **Runner:** uplink bonuses and faster ability recharge, but 5 less maximum
  health. Strong if Core access is reliable.
- **Warden:** defuse and Core-defense bonuses, but cannot pick up the Core,
  banks at 0.9x, and has 1 less cargo space. Strong counter-role if Core
  contesting is common.
- **None:** no bonus or penalty; use as a robust fallback if the shop/role
  contract is ambiguous or the other roles' penalties dominate.

The likely default is **Hauler + mobility ability** for a safe baseline, with
**Runner** when the bot's Core conversion rate is high. The map-by-map buy
function will choose based on `shop["credits"]`, map number, past placements,
and our own validated series policy. It cannot react to rival identities because
the shop does not expose them.

### 9.3 Credit spending

Credits start at 4, increase after each map, carry across maps, and cap at 10.
The bot receives +3 plus a placement bonus after each map and up to +2 for
consecutive non-wins. We will spend credits deliberately rather than always
emptying the wallet:

1. Preserve at least enough credits for a useful upgrade next map unless an
   immediate map has a clear high-value purchase.
2. Buy only items whose effect the decision policy actually uses.
3. Avoid duplicate items and more than one `KIT_` item.
4. Return a valid dictionary; if uncertain, return `None` and use the class
   defaults rather than risking an invalid buy.

### 9.4 Upgrade priorities to benchmark

Candidate order for the hybrid bot:

1. `ARMOR` when frequent deaths occur or before an aggressive round.
2. `POCKETS` for a cargo-first build; verify interaction with Hauler/Warden
   capacity.
3. `OVERCLOCK` for DASH/BLINK/SHIELD value across 40 turns.
4. `QUICK_RESPAWN` if deaths are frequent and recovery is a material source of
   lost placement.
5. `SCOPE` if logs show many missed resources or Core/enemy sightings.
6. One kit only: `KIT_MEDPACK` for survival, `KIT_SMOKE` for safe conversion,
   `KIT_RADAR` for planned hunts, or `KIT_RAILGUN` for a combat build.

Purchases will be selected by a deterministic affordability check using the
actual `shop["prices"]`, not copied constants alone.


The starter kit permits only one submitted Python file. We will keep the code
modular inside `my_bot.py` with small helpers and a clear top-level policy.

### 10.1 Proposed structure

```text
my_bot.py
├── imports from vibe_sdk and Python standard library only
├── constants and deterministic direction order
├── class MyBot(Bot)
│   ├── ABILITY / optional default ROLE
│   ├── __init__      transient match memory
│   ├── buy(shop)     deterministic role and item purchase
│   ├── decide(view)  state read + priority ladder
│   ├── _parse_state(view)
│   ├── _legal_action checks
│   ├── _path_step(...) and geometry wrappers
│   ├── _choose_target(...)
│   ├── _choose_resource(...)
│   ├── _choose_pad(...)
│   ├── _choose_move(...)
│   ├── _combat_action(...)
│   ├── _core_action(...)
│   ├── _item_action(...)
│   └── _fallback_action(...)
```

The exact helper names are not important. The important separation is:

- strategic scoring chooses a *candidate*;
- legality helpers verify it against the current view;
- only validated candidates can reach the return statement.

### 10.2 Memory to retain

Use `self` for small, useful, deterministic memory:

- current phase and objective hint;
- last known safe target/route;
- recent enemy locations and last-seen turn;
- damage dealt to targets in recent turns for assist decisions;
- own last killer and per-map nemesis information if exposed;
- whether we planted/are defusing the Core and the observed countdown;
- blocked destination counters so the bot changes route;
- one-turn hazard expectations for delayed explosions.

Never let memory override facts in the current view. Memory is especially
important for fog of war, but stale memory must decay and be treated as
uncertain.

### 10.3 Defensive view handling

The guide warns that lists may be empty and fields may not be present in every
situation. The implementation will:

- use the exact field names from `BOT_SDK_CONTRACT.md`;
- handle empty visible-enemy/resource/crate lists;
- handle absent optional items, markers, cooldowns, and events;
- avoid indexing `[0]` until non-empty checks pass;
- avoid assuming all item types occur in a match;
- avoid assuming a visible enemy remains at the same location next turn;
- treat an unknown optional field conservatively rather than inventing a legal
  action from it.

### 10.4 Action validation checklist

Before returning any action, validate:

- energy is sufficient;
- the action is supported by the current ability/item/role;
- target ID exists in the relevant current list;
- target distance and line of sight are valid;
- movement destination is in bounds;
- movement is open under current walls and diagonal rules;
- ability cooldown is ready;
- item coordinates are in bounds where required;
- collection object is actually under our current cell;
- a delayed action is still useful before turn 40;
- no branch can return a raw string, `None`, malformed object, or exception.

The fallback must be tested in views with zero enemies, zero resources, no item,
low energy, low health, wall-adjacent position, spawn protection, and late turn.

## 11. Testing and measurement plan

We will not judge a strategy from one exciting match. Every meaningful change
must go through the following loop:

```text
1. Change one policy or helper.
2. Run the fast validity check.
3. Run a one-match log and inspect decisions.
4. Run many deterministic seeds and record summary metrics.
5. Run opponent and chaos-bot matches.
6. Run the full conformance test.
7. Keep the change only if it improves the target metric without introducing
   legality, timing, or determinism regressions.
```

### 11.1 Commands after the starter kit arrives

Run these from the kit root, using the repository's exact Python command if it
differs:

```text
python check.py
python play.py
python play.py --seed 42
python play.py --seeds 20
python play.py --vs sparring:ChaosShove
python conformance.py
```

The guide says the default `play.py` setup uses three copies of our bot. That is
useful for repeatability but not sufficient. We must test against every provided
sparring bot, plus small locally written baselines such as:

- a cargo farmer;
- a Core runner;
- a melee hunter;
- a ranged shooter;
- a defensive bot that refuses combat;
- a deliberately chaotic bot that moves, shoves, and uses items.

### 11.2 Metrics to record

For each seed and opponent set, record:

- map placement and rank points;
- final score and score components;
- kills, deaths, assists, tags confirmed, tags denied;
- cargo collected, hot pickups, cargo banked, and cargo lost on death/end;
- Core captures, plants, uplinks, defuse attempts, and successful defuses;
- health at death, average health, and time spent respawning;
- action failures, blocked moves, illegal-action count, and timeout count;
- average and worst `decide()`/`buy()` duration if the harness exposes it;
- item and ability use rate and resulting score;
- placement variance across maps rather than just mean placement.

The most important aggregate metric is **percentage of maps finishing 1st or
2nd**, followed by average rank points and then average score. A change that
raises score but increases fourth-place finishes is probably bad for the
tournament objective.

### 11.3 Scenario tests

Once the engine/view format is known, construct focused tests or fixtures for:

1. Empty fog-of-war lists.
2. One visible adjacent enemy with enough/insufficient energy.
3. A target at exactly each attack/shoot boundary.
4. Every board edge and each diagonal corner-blocking case.
5. Two bots attempting the same destination.
6. A wall between shooter and target.
7. Low energy values from 0 through each action threshold.
8. Ability ready, cooling down, and upgraded cooldown.
9. Current cell containing resource, crate, tag, Core, or nothing.
10. Full cargo, empty cargo, and cargo near match end.
11. Core on ground, carried, planted, being defused, uplinked, and respawning.
12. Delayed bomb, airstrike, and orbital with one turn remaining.
13. Blood Moon and Resource Rush.
14. Legendary warning and legendary item use.
15. Respawn protection and forced early cancellation of protection.
16. Overtime at turns 31, 32, 38, and 39.
17. Rank 1 through 4 and missing optional fields.
18. A bot with a stale remembered enemy location.
19. Shop with 0, 1, 2, 3, 4, 9, and 10 credits.
20. Invalid/unknown shop data to verify safe `None` behavior.

### 11.4 Determinism and timing

Run the same seed repeatedly and compare complete logs or action sequences. The
same input must produce the same decision. Benchmark enough turns to catch a
slow worst case, especially when the bot is near several objects and pathfinding
is active. Keep operations bounded and avoid repeated full-board scans.

Forbidden or risky behavior in the submitted file includes:

- `time.time()` or any wall-clock decision;
- unseeded `random`;
- internet, sockets, subprocesses, or AI calls;
- reading files or relying on another local file;
- imports outside `vibe_sdk` and the Python standard library;
- `eval`, `exec`, and blocked modules listed by the guide;
- mutable global state that leaks between bot instances unless the contract
  explicitly permits it.

### 11.5 Regression discipline

After every passing milestone, save an untouched backup outside the submitted
file, for example `my_bot_backup_passed.py` if the kit permits local backups.
Before submission:

- restore the best measured version, not necessarily the newest version;
- inspect the final file manually;
- run `python check.py`;
- run `python conformance.py` early enough to fix failures;
- rerun several seeds after the final change;
- submit exactly `my_bot.py`, not a backup, generated file, or directory.

## 12. Development schedule once the repository is provided

The competition provides roughly 75 minutes, so the plan must front-load
reliability and avoid ambitious rewrites near the deadline.

### Before the event

- Install and verify Python 3.9+.
- Have the editor/AI tool logged in and ready.
- Practice opening a terminal in a folder.
- Read this plan together and agree that one teammate owns edits while the
  other owns tests/log review.
- Prepare a short prompt for the AI that includes legality, determinism,
  timeout, and one-file constraints.
- Prepare a decision table for role, ability, and shop candidates.

### 8:45–9:00 — Arrival

- Report and write down the team ID.
- Confirm the submission email, team name spelling, and deadline procedure if
  organizers provide them.

### 9:00–9:15 — Kit verification

- Download/unzip or clone the starter kit.
- Run `python check.py` and `python play.py` before edits.
- Read `docs/RULES.md`, `docs/BOT_SDK_CONTRACT.md`, `vibe_sdk`, and relevant
  engine geometry code.
- Resolve every discrepancy between this guide and the engine in favor of the
  engine.

### 9:15–9:30 — Safe skeleton

- Preserve the legal starter structure and class name `MyBot`.
- Implement defensive view parsing, deterministic movement, and a legal
  fallback first.
- Run `check.py` immediately.
- Implement the first hybrid policy with no exotic optimization.

### 9:30–9:50 — Core gameplay

- Add wall-aware bounded pathfinding.
- Add cargo collection/banking and Core state handling.
- Add conservative combat and tag collection.
- Test one seed, then 20 seeds.
- Fix observed stuck routes and illegal-action risks before adding items.

### 9:50–10:05 — Items, abilities, and shop

- Add one selected ability and the highest-confidence item behaviors.
- Implement `buy(shop)` with actual prices and safe affordability checks.
- Test each optional branch against views where the field is absent or the item
  is unavailable.

### 10:05–10:15 — Admission safety

- Run `python conformance.py`.
- If it fails, prioritize the failure over score improvements.
- Save the last passing copy.
- Run sparring bots and inspect the full log for repeated blocked movement,
  stuck objectives, and deaths while carrying value.

### 10:15–10:25 — Final controlled tuning

- Make only one or two evidence-based policy changes.
- Compare before/after on fixed seeds.
- Re-run conformance after every change.
- If a change is ambiguous, keep the last passing measured version.

### 10:25–10:30 — Freeze and submit

- Stop editing before the deadline buffer.
- Run the final conformance test one last time.
- Attach exactly `my_bot.py`.
- Include team name and team ID.
- Confirm the email was sent before 10:30.

## 13. AI-assisted build protocol

The AI is a coding assistant, not a substitute for engine verification. Use
small, testable prompts and always paste complete command output when debugging.

### Initial repository-reading prompt

```text
Read docs/RULES.md, docs/BOT_SDK_CONTRACT.md, the starter my_bot.py, and the
vibe_sdk geometry/action definitions. Do not edit yet. Summarize the exact view
schema, action constructors, cooldown fields, collection semantics, and every
condition that makes an action illegal. Identify anything that differs from our
pre-starter-kit plan.
```

### Implementation prompt

```text
Edit only my_bot.py. Keep class MyBot. Implement the agreed hybrid policy.
Use only vibe_sdk and the Python standard library. Every decide(view) and
buy(shop) call must be deterministic and comfortably under 200 ms. Treat every
view list as possibly empty. Validate energy, target/range, board edges, walls,
diagonal geometry, item availability, and ability cooldown before returning an
action. Every branch must return one legal Action. Do not add files, I/O,
internet, time-based code, or unseeded randomness. Run python check.py and report
the full output.
```

### Debug prompt

```text
Here is the exact command and full output: [paste]. Here is the relevant turn
log: [paste]. Explain the smallest cause, change only my_bot.py, preserve the
strategy and all legality checks, then run the relevant check again. Do not claim
success without showing the command output.
```

The teammate who is not typing should verify that the AI did not:

- invent view keys or action constructors;
- use a fixed map or fixed spawn corner;
- assume an item exists;
- return an unchecked movement direction;
- remove a legality guard to make a test pass;
- add prohibited imports or file dependencies;
- optimize for self-play only;
- introduce nondeterminism.

## 14. Decision table for first implementation

This is the initial policy specification to hand to the AI after the real SDK is
understood. It is intentionally simpler than the final tuned implementation.

1. Parse current state and refresh memory from the view.
2. If respawning or dead, return the contract's safe legal no-op/defense/move
   action; do not assume a normal board position.
3. If a lethal or damaging delayed marker threatens our cell, escape or defend
   using a validated action.
4. If on a valid collectable and collection improves the current objective,
   collect it.
5. If a valid Medpack/Smoke/Shield/Overcast defensive use prevents likely death,
   use it.
6. If a high-value enemy can be safely attacked or killed this turn, attack it;
   prefer Core carriers, crown holders, streak holders, and revenge/nemesis
   targets using deterministic tie breaks.
7. If carrying the Core, move toward the best validated uplink pad and defend or
   use mobility when appropriate.
8. If an enemy planted the Core and we can plausibly defuse it, contest it for
   the required consecutive turns; otherwise attack the planter or continue a
   scoring route.
9. If carrying cargo, bank it when full, near Overtime, near match end, or under
   threat.
10. If a visible safe resource is reachable and can be banked, path to it.
11. If a useful crate or legendary crate is reachable and the item policy wants
    it, path to it.
12. Use radar/scan only when enough future turns remain to exploit the reveal.
13. Move toward a deterministic safe waypoint.
14. Return a legal defensive fallback.

## 15. Risk controls and failure handling

### Illegal-action prevention

One illegal action rejects the bot during admission. This is a higher priority
than scoring. We will make legality checks centralized and boring, with no
strategic branch allowed to bypass them.

### Three-failure rule

In live matches, a crash, timeout, or bad action loses that turn; three failures
in a row switch the bot off for the rest of the map. Our design therefore treats
exceptions and slow paths as catastrophic even if they are rare.

### No assumption of observability

The bot cannot see rival scores, hidden enemies, or future actions. It must never
pretend to know them. Marked bots are globally visible, but an unmarked rival can
disappear behind walls or into fog immediately.

### No fixed-map exploit

All coordinates, walls, resources, pads, crates, and starting corners must be
read from the live view. Map themes affect presentation only; geometry does not
have a stable pattern we should rely on.

### No end-of-match waste

The bot must account for action timing. It should not launch a two-turn strike,
travel toward a distant base, or collect a resource on turn 39 if the result
cannot score before the map ends.

## 15.1 Canonical rules reference for implementation

This compact reference keeps the guide's important numbers visible while the
strategy is being translated into code. The engine remains authoritative if any
number or semantic detail differs.

### Exact combat and scoring values

| Event | Guide value | Implementation implication |
|---|---:|---|
| Attack damage | 16–24 | Use 16 as the conservative guaranteed damage |
| Shoot damage | 10–14 | Use 10 as the conservative guaranteed damage |
| Railgun | 25 to every bot on its clear line | Never assume it stops at the first bot |
| Bomb | 45 center, 25 one cell away | Damages the owner; one live bomb at a time |
| Airstrike | 40 center, 25 adjacent; lands in 2 turns | Plan against its visible marker |
| Orbital | 60 within radius 2, 30 at edge; lands in 2 turns | Legendary delayed area threat |
| Shove slam | 15 | Happens when the pushed cell is blocked by wall, bot, or edge |
| Kill | +25 now, +15 on killer tag collection | A kill is not fully converted until its tag is collected |
| Repeat kill | +15 if victim was killed by us within 8 turns | Track recent kills per map |
| First Blood | +20 | Do not throw away opening survival solely for this bonus |
| Crown kill | +30, plus +15 per extra consecutive map win by victim | Marked crown is a high-value but dangerous target |
| Shutdown | +10/+20/+30/+40 for victim streak 2/3/4/5+ | Prefer stopping an active streak when safe |
| On Fire | +10 for killing while our streak is 3+ | Adds value to maintaining a streak without over-chasing |
| Multi-kill | +20 for each additional same-turn kill | Delayed area damage can create this, but is high variance |
| Revenge | +15 against the bot that last killed us | Retain the killer identity if the view/contract exposes it |
| Nemesis | +15 against the bot that killed us most last map | Use only with reliable cross-map memory from the SDK |
| Clutch | +15 for a kill at 15 health or less | Do not intentionally enter lethal situations for it |
| Assist | +10 when we damaged the victim in the last 3 turns | Keep bounded recent damage memory |
| Blast | +10 when our bomb, airstrike, or orbital gets the kill | Explosions can score without direct attack damage |
| Shove kill | +10 shared when a shoved bot dies to a blast within 2 turns | Only attempt with a validated displacement setup |
| Tag confirmed | +15 for collecting our own kill tag | Usually worth a safe short detour |
| Tag denied | +10 for collecting someone else's tag | Deny only when risk is acceptable |
| Core uplink | +60 after planted Core survives 3 turns | Planter may score even if killed afterward |
| Core defuse | +40 shared by every eligible enemy next to Core | Requires 2 consecutive turns within 1 cell |

Points in the listed scoring table, including role bonuses, double in Overtime.
The guide specifically states that a bot killed by its own bomb gives nobody
points. Shared kills award the kill to every bot that dealt damage on the killing
turn. Tags disappear after 6 turns. A 3-kill streak grants a free three-turn
radar sweep and a 5-kill streak grants an airstrike in the item slot.

### Abilities and item effects

| Choice | Exact effect and key guard |
|---|---|
| DASH | Move 2 cells in a straight one of 8 directions; both cells must be free |
| BLINK | Teleport 3 cells in a straight line over walls/bots; landing cell must be free |
| SHIELD | Reduce incoming damage by 70% for this turn only |
| BOMB | Drop on own cell; explodes at start of turn 3 turns later; hurts owner |
| SHOVE | Push adjacent enemy one straight cell; blocked push causes a 15-damage slam |
| RAILGUN | Hits every bot to the first wall/edge along a line for 25; loud |
| AIRSTRIKE | Aim at any cell, including fog; 2-turn delay; 40 center/25 adjacent |
| MEDPACK | Heal 40, never above maximum health |
| SMOKE | 3 x 3 cloud on own cell for 3 turns; marks remain visible through it |
| RADAR | Reveal every enemy for 3 turns |
| ORBITAL | Radius 2; 2-turn delay; 60 center/30 edge |
| OVERSHIELD | For 3 turns, the next lethal hit leaves us at 1 health |
| REAPER | For 5 turns, each kill's kill points are doubled |

Bombs, airstrikes, and orbitals explode at the **start** of a turn before
decisions. The defensive action protecting against a blast due in one turn must
therefore be chosen on the preceding turn. Blasts can set off other bombs in
their area. Each normal match uses only four of the five normal item types, and
the exact pool is available in `view["item_pool"]`.

### Role and shop reference

| Role | Benefit | Cost |
|---|---|---|
| Hunter | +10 per kill; +5 per kill for each kill already on streak | Cargo banks at 0.85x; killer gets +25 |
| Hauler | +1 cargo space; 3-at-once bank x1.25, 4+ bank x1.5 | Kills pay 10 less |
| Runner | +10 per uplink, +10 more for each earlier uplink; ability recharges 2 turns faster | 5 less maximum health |
| Warden | +20 per defuse; +20 for killing Core carrier, live planter, or bot within 2 cells of planted Core | Cannot pick up Core; banks at 0.9x; 1 less cargo space |

Role bonuses double in Overtime. Shop upgrades are bought at most once per map:
`ARMOR` costs 2 (+15 max health), `POCKETS` 2 (+1 cargo space), `SCOPE` 2
(+1 vision), `QUICK_RESPAWN` 3 (one turn sooner), `OVERCLOCK` 3 (ability
recharges 2 turns faster, never below 3), `KIT_RAILGUN` 3, `KIT_MEDPACK` 2,
`KIT_SMOKE` 2, and `KIT_RADAR` 2. At most one `KIT_` item can be purchased.

### Tournament and tie-break reference

The full-field plan is Round 1: 5 maps, top 1 advances; Round 2: 5 maps, top 2
advance; Semifinal: 7 maps, top 2 advance; Grand Final: 10 maps, one group of
four. Organizers may change this for the actual field size.

Map rank ties use, in order, more kills, fewer deaths, more cargo banked, then
more resources collected. Series rank-point ties use head-to-head finishes,
then map wins, total score, Core captures, and banked cargo value. If a rule
separates only some tied teams, apply the tie procedure again to the remaining
teams.

## 16. Open questions and ambiguities to resolve at repository arrival

The following items are intentionally left open because the PDF does not provide
the exact SDK schema or engine implementation. We will answer them by reading
`docs/BOT_SDK_CONTRACT.md`, `docs/RULES.md`, and the engine, then update this
plan or encode the result in `my_bot.py`.

### View schema

- What is the exact key for the current turn, base location, cargo contents,
  held item, health maximum, respawn state, and ability cooldown?
- Is the Core represented as one object, a status dictionary, or separate
  ground/carried/planted fields?
- How are resources represented: value, ID, x/y, hot-pickup status, and whether
  multiple resources can occupy one cell?
- How are crates, tags, smoke, bombs, airstrikes, orbitals, and legendary
  warnings represented, including their countdown fields?
- Are all marked-bot reasons exposed, and is a bot's current health always
  visible when it is visible through fog or a global mark?
- Is `rank` available while dead/respawning and at the start/end of a turn?
- Which lists can be absent versus present-but-empty?

### Action and resolution semantics

- What exact constructor and argument names are required for each Action?
- What is the contract's legal no-op or safe fallback when a bot is dead or
  respawning?
- Is standing on base enough to bank cargo, or is banking an explicit action?
- Does `Action.collect(id)` accept resource, crate, tag, and Core IDs in one
  namespace or different namespaces?
- How does the engine expose current ability readiness and cooldown duration?
- What are the exact legal conditions for `scan`, `defend`, `shove`, and each
  item use?
- Do taunts affect object construction or conformance in any way?
- Does the action object support chaining `.taunt(...)` in the installed SDK?

### Geometry and movement

- What are the exact signatures and return values of `step_open`, `line_clear`,
  and `shot_path`?
- Do those helpers include bots as obstacles or only walls?
- How does the engine resolve a diagonal move when one or both adjacent cells
  are occupied by walls or bots?
- Are temporary effects such as smoke and blast markers obstacles for movement?
- Does `BLINK` validate only the landing cell, and how does it handle board
  boundaries and a wall at the landing cell?

### Timing and scoring

- At what exact point in a turn do collection, bank, damage, tags, bombs,
  airstrikes, orbitals, Core countdowns, and rank updates resolve?
- Does a Core planted on turn `t` count down on turns `t+1`, `t+2`, `t+3`, or
  another convention?
- When exactly is the two-turn defuse streak reset?
- Are all role bonuses and all tag/kill bonuses doubled in Overtime, including
  bonuses that are awarded after a delayed action?
- How are damage ranges sampled, and is the sampling deterministic under a map
  seed?
- How are ties among equal-score bots broken inside the engine when selecting
  the crown?
- How are shared kills and assists exposed to a bot, if at all?

### Roles and shop

- Is the class-level `ROLE` overridden by a returned shop role, and what does
  returning `None` preserve exactly?
- What are the precise keys and permitted values in `shop`?
- Does the shop reject a purchase atomically or partially apply valid items?
- How do role capacity modifiers combine with `POCKETS`, and what does “banking
  3 units at once” mean when hot pickup produces fractional values?
- How are prior placements represented and does the current map number start at
  zero or one?
- Are role/upgrade effects visible in `view["self"]` after buying?

### Tournament and submission

- How many teams and groups will actually participate?
- What is the real round format if the field is not the full-field format in the
  guide?
- What exact submission email address, filename requirements, and team-name
  formatting will organizers require?
- Will the organizers run the submitted file in the same Python minor version
  and SDK build as the starter kit?
- Are local backup files ignored, or must the submission directory contain only
  `my_bot.py`?

## 17. Definition of done

The plan is complete enough to implement when:

- the exact SDK view and action contract have been read;
- every open question that affects legality has an engine-backed answer;
- a first hybrid bot passes `check.py` and `conformance.py`;
- repeated fixed-seed runs are deterministic;
- the bot stays comfortably below 200 ms in worst-case decisions;
- it handles sparse views, changing walls, all corners, Core states, hazards,
  items, Overtime, shop states, and respawn;
- it has been tested against self-play and chaos/sparring opponents;
- the final version is selected by map placement/rank points, not anecdotal
  score;
- the last passing `my_bot.py` is backed up and submitted before the deadline.

The guiding principle for the final implementation is simple: **a reliable
second-place-capable bot with selective high-value aggression beats a brilliant
but fragile bot that occasionally forfeits turns or dies carrying the whole
map's score.**
