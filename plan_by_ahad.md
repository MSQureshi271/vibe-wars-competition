# Vibe Wars competition plan

Prepared 14 October 2026 for the competition on Sunday, 18 October 2026, at the Civil AV Auditorium, NED University. Event times below use Pakistan time (Asia/Karachi).

## 1. Objective, evidence, and working approach

Build one admissible, deterministic Python bot that maximizes our probability of advancing through each round and winning the final. We will develop and compare several policy variants, but submit exactly one `my_bot.py` containing `MyBot` and all its helpers.

This plan is based on a complete reading of all 27 pages of [vibe-wars-study-guide.pdf](./vibe-wars-study-guide.pdf), including its tables and visual examples. Page references below refer to PDF pages. The starter repository has not been supplied. No engine behavior, SDK signature beyond the examples, resource value, or performance result has been invented or experimentally verified.

**Evidence labels:** “Guide rule” means stated in the PDF. “Proposal” means our implementation or strategic choice. “Derived” means a calculation from stated rules. All unresolved mechanics, permissions, and questions are collected in section 20 at the end. Before implementation, reconcile the plan with `docs/RULES.md`, `docs/BOT_SDK_CONTRACT.md`, and the engine: the guide explicitly says the engine decides when they disagree (p. 25).

Our order of priorities is:

1. Pass admission and never lose a turn to an avoidable crash, timeout, or illegal action.
2. Convert movement into banked points or completed objectives instead of wandering or chasing.
3. Earn strong placements across varied maps and opponents, with few catastrophic maps.
4. Exploit tactical opportunities, overtime, and role bonuses without making the baseline fragile.
5. Preserve a tested submission and submit before the deadline.

There is no evidence yet that any one role or ability dominates. Our initial implementation candidate is a **cargo-led hybrid with opportunistic tags, Core play, and selective combat**, compared against a Runner/Core variant and a Warden/denial variant. A neutral-role version is the control. The final choice comes from paired, varied-opponent tests, not a theoretical claim that one build guarantees victory.

The AI tools are development assistants. The tournament bot itself is ordinary bounded Python logic: no live LLM, service calls, external data, or human intervention during matches.

## 2. What actually determines winning

### 2.1 Map and series objectives

**Guide rules (pp. 9–11, 17–18):** Each map lasts 40 turns. Highest score wins that map. Map ties break by more kills, fewer deaths, more cargo banked, then more resources collected.

A series awards **3 / 2 / 1 / 0 rank points** for first / second / third / fourth. Raw score matters only later in series tie-breaking. Optimize placement and advancement, not a spectacular score in one favorable matchup.

| Round | Maps per series | Advancement under the announced full-field format |
|---|---:|---|
| Round 1 | 5 | Top one of four |
| Round 2 | 5 | Top two of four |
| Semifinal | 7 | Top two of four |
| Grand final | 10 | Highest-ranked bot becomes champion |

The format may change with turnout. Groups are drawn in advance, and starting corners rotate across a series. Each group uses the same three opponents across its maps.

Series ties break by: (1) finishes above the other tied teams, (2) map wins, (3) total score, (4) Core captures, and (5) banked cargo value. If only some tied teams separate, the remaining tied subset restarts at head-to-head. Evaluation must reproduce that rule rather than use an ordinary flat sort.

**Strategic consequences:**

- Track mean rank points, first-place rate, fourth-place rate, and actual series advancement separately.
- Round 1 requires outright series leadership. Consistent second places are useful but not sufficient against a consistently stronger rival.
- Top-two rounds reward avoiding collapses, but the bot cannot safely act on a supposed qualifying margin unless that information is actually exposed.
- Hidden enemy scores prevent exact “we are 20 points ahead” decisions. Use our score, rank, observed threats, and obtainable points; rank alone does not reveal the gap.
- Never intentionally throw a map for catch-up credits. Any benefit arrives later and sacrifices the primary objective now.

### 2.2 Evaluation objective

For development, rank candidates by this ordered set of criteria:

1. Admission, determinism, and runtime pass.
2. Advancement rate in realistic grouped series, when the harness supports it.
3. Mean rank points over paired seed/corner/opponent scenarios.
4. Low fourth-place rate and acceptable results against each opponent family.
5. Map wins, score, and scoring-source efficiency as diagnostic tie-breakers.

For runtime decisions, use expected obtainable points and risk as a tractable proxy for placement. Do not attempt to reconstruct hidden scores as facts. Increase tolerance for uncertain opportunities late when trailing; protect deliverable points when leading, without becoming inactive.

## 3. Non-negotiable submission and execution contract

**Guide rules (pp. 1–3, 18–24, 26):**

| Requirement | Implementation and release consequence |
|---|---|
| Python 3.9 or newer; kit needs no `pip install` | Use Python 3.9-compatible syntax and only allowed dependencies. |
| One file, `my_bot.py`; class named `MyBot` | Put every runtime helper and policy inside that file. No companion modules or model files. |
| Import only `vibe_sdk` and permitted standard-library functionality | Inspect the checker’s actual restrictions before choosing imports. “Standard library” is not blanket permission. |
| Both `decide(view)` and `buy(shop)` must finish within 200 ms | Use fixed work limits and aim for tens of milliseconds on the tournament machine. |
| Deterministic behavior | Stable iteration and tie-breaking; no wall-clock dependence, unseeded randomness, file reads, or network access. |
| Example blocked modules/calls: `os`, `sys`, `subprocess`, `socket`, `pickle`, `threading`, `eval`, `exec`, `open` | Keep these out of the submission. Read the full actual allow/block rules. |
| One bot object persists through a match | Store compact match memory on `self`; reset match-dependent data correctly. |
| One illegal action in admission rejects the bot | Every candidate action must pass a legality gate before return. |
| Runtime crash, timeout, or bad action loses a turn; three consecutive failures disable the bot for the map | Reliability has direct strategic value, not just administrative value. |
| Bad shopping choices buy nothing rather than producing an illegal map action | Validate purchases anyway; lost upgrades can decide a series. |
| Only `my_bot.py` is edited/submitted as the competition entry | Read engine and SDK code; never change them to make tests pass. |
| Code freezes at 10:30 AM | Send one submission email before then, attaching the exact tested file and including team name and team ID. |
| Results remain tentative until code review | Do not exploit hidden engine state, bypass restrictions, or depend on local-only facilities. |

**Proposal:** use deterministic rules without randomness initially. If exploration needs variety, use an explicit fixed ordering transformed by our corner or stable public input. A seeded generator is permitted by the guide, but it adds state and replay concerns without obvious value to the first version.

Profiling and result analysis happen outside the submitted bot using allowed development tooling. The bot itself does not read a clock to decide when to stop searching.

## 4. Complete mechanics reference for implementation

This section preserves the guide’s operative rules so that no scoring system is overlooked. Exact engine semantics will be bound during the repository audit.

### 4.1 Board, visibility, movement, and survival

**Guide rules (pp. 4–8):**

- Four bots; 12 × 12 walled arena; 40 turns. Each bot starts in a corner with its own banking base. Base pockets have two exits and block distant shots into spawn.
- Walls have fourfold rotational symmetry. Resources and crates arrive in fair groups of four. Symmetry helps normalize strategy but does not reveal hidden live entities.
- Movement is one cell in any of eight compass directions. Diagonal movement costs one action. Chebyshev distance is a lower bound before considering walls.
- Walls block movement, shots, and sight. No diagonal passage between two walls touching at a corner, including shots and sight lines.
- Walking into a wall or bot is legal but fails to move and still costs energy. Walking outside the board is illegal. If two bots attempt the same destination, neither moves.
- All bots decide from the same pre-action picture; actions resolve together. No strategy may assume an opponent’s current cell is its final cell.
- Base health is 70, energy capacity 50, regeneration +5 per turn, or +10 in overtime. Base vision is 3 cells with wall occlusion.
- Walls, Core, uplink pads, crates, smoke, airstrike markers, and marked bots are globally visible. Ordinary nearby visibility is limited by fog. Only our score and rank are exposed, never exact rival scores.
- Death preserves banked score, drops carried resources as one pile and a held item as a crate, resets kill streak, and respawns the bot at its base after 3 turns with full health and energy.
- Spawn protection lasts 2 turns; attacking, shooting, collecting, using an ability, or using an item ends it early. Movement and defending are not listed as protection-breaking actions.

| Mark | Trigger |
|---|---|
| `CROWN` | Highest score; chosen at the start of each turn |
| `CORE` | Carrying the Core |
| `PLANTER` | Our planted Core is counting down |
| `CARGO` | Carrying cargo value at least 60 |
| `ON_FIRE` | Killing streak at least 3 |
| `LOUD` | Attacked, shot, or fired a big weapon last turn |

Marks reveal bots globally, including through walls and smoke; they do not make a shot pass through a wall.

### 4.2 Actions

**Guide rules (pp. 7–8, 12):** Examples show syntax only; import/helper signatures must come from the kit.

| Action example | Energy | Effect and timing |
|---|---:|---|
| `Action.move("EAST")` | 1 | Move one cell. Respect bounds and geometry. |
| `Action.attack(bot_id)` | 5 | Adjacent target including diagonals; 16–24 damage. Hits even if target moves away that turn. |
| `Action.shoot("NORTH")` | 6 | Up to 4 cells; first bot on line; 10–14 damage. Resolves after movement; walls stop it. |
| `Action.collect(id)` | 2 | Collect resource, crate, or Core on our current cell. |
| `Action.defend()` | 2 | Halve incoming damage for the applicable turn. |
| `Action.scan()` | 8 | Extend vision by 3 cells on the next turn. |
| `Action.ability(...)` | 10 | Use equipped ability with its requirements and cooldown. |
| `Action.use_item(...)` | 4 | Use held item with type-specific arguments. |
| `Action.shove(bot_id)` | Ability cost: 10 | SHOVE uses this special action form; verify its cooldown handling in SDK. |

Insufficient energy, off-board moves, invalid ranges/targets, and abilities on cooldown are examples of illegal actions. A merely ineffective move must still be avoided when a productive legal alternative exists.

### 4.3 Scoring, tags, and combat bonuses

**Guide rules (pp. 9–11):** From turn 32, all listed point awards double, including role bonuses, banking, uplinks, and defuses. Energy regeneration also doubles. This does not mean damage doubles.

| Event | Normal-time award |
|---|---:|
| Ordinary kill | 25 immediately + 15 in its tag = 40 total if confirmed |
| Re-kill a bot we killed within the last 8 turns | 15, all in the tag |
| First Blood | +20 |
| Kill crown holder | +30; +15 per extra consecutive map win by that holder |
| Shutdown victim streak 2 / 3 / 4 / at least 5 | +10 / +20 / +30 / +40 |
| Kill while our streak is at least 3 | +10 |
| Each extra kill in the same turn | +20 |
| Revenge on bot that last killed us | +15 |
| Kill nemesis, the bot that killed us most last map | +15 |
| Kill while at at most 15 health | +15 |
| Assist: hurt victim in last 3 turns, another bot kills it | +10 |
| Our bomb, airstrike, or orbital makes kill | +10 |
| A bot we shoved dies to a blast within 2 turns | +10 and shared kill |
| Collect tag from our own kill | +15; this is the deferred component above, not a second extra +15 |
| Collect someone else’s tag | +10 and deny its killer the deferred award |
| Bank cargo | Cargo value, with applicable modifiers |
| Completed Core uplink | +60 |
| Completed Core defuse | Guide lists +40 shared by adjacent enemies; exact distribution is deferred to section 20 |

Bonuses can stack. Every score award appears with a reason in the match log. Use these logs to reconcile our expectations.

- Tags are collected automatically by standing on their cell at end of turn; no collect action is needed. They expire after 6 turns.
- Anyone other than the killer, including the respawned victim, can deny a tag.
- Every bot that damages a victim on the killing turn gets a shared kill.
- A 3-kill streak grants a free 3-turn radar sweep; a 5-kill streak grants an airstrike in the item slot.
- A bot killed by its own bomb awards nobody any points.

### 4.4 Cargo and Core

**Guide rules (p. 11):**

- Resources must be collected, transported, and banked by standing on our own base. Pickup is not scoring.
- Base cargo capacity is 4 resource units. A full hold must be banked before collecting more resources.
- Collecting while another bot is within 2 cells loads resource value at ×1.3 (“hot pickup”). A spilled pile never receives this bonus again.
- Unbanked cargo at match end scores zero. Four resources exist initially; most arrive in later supply waves.
- The Core first spawns on turn 8 on one of four central cells. Stand on it and collect `"CORE"`.
- Carrying it marks us. Death drops it. Entering any of four side uplink pads plants it automatically at end of turn.
- Surviving its 3-turn countdown pays the planter 60, even if the planter has died.
- An enemy staying within 1 cell at the end of 2 consecutive turns defuses it, preventing the uplink. Adjacent enemies receive the defuse award as specified by the engine.
- After an uplink or defuse, the Core returns to the centre 4 turns later. Its position and status are always public.

### 4.5 Abilities

**Guide rules (p. 12):** Select one `ABILITY` for the match, or `None`. All cost 10 energy and normally recharge over 6 turns.

| Ability | Effect | Strategy constraint |
|---|---|---|
| `DASH` | Move 2 cells in one of 8 directions | Both cells must be free of walls and bots. |
| `BLINK` | Teleport 3 cells in one of 8 directions, over walls and bots | Landing must be free; check board bounds. |
| `SHIELD` | Reduce incoming damage by 70% | One-turn protection; timing matters. |
| `BOMB` | Drop at our cell; detonates 3 turns later for 45 centre / 25 one cell away | Hurts us too; one live bomb at a time; visible nearby. |
| `SHOVE` | Push adjacent enemy one cell away; blocked push slams for 15 | Can disrupt defuses or combine with blasts. |

**Critical timing:** bombs, airstrikes, and orbitals detonate at the start of a turn, before decisions. To reduce an imminent blast, defend or shield on the preceding turn, when its countdown shows `turns_until_detonation: 1`. Blasts trigger bombs in their area. Treat chain reactions as immediate additional danger.

### 4.6 Items, events, presentation

**Guide rules (pp. 12–14):** Crates appear in groups of four and are globally visible. Their item is selected when opened. Lower rank increases the chance of railgun/airstrike but guarantees nothing. One item can be held; another crate replaces it. Rivals can see that we hold an item but not its type. Each match includes 4 of the 5 ordinary types, listed in `view["item_pool"]`.

| Item | Effect |
|---|---|
| Railgun | After movement, 25 damage to every bot along a direction until wall or edge; loud. |
| Airstrike | Target any cell, including fog; lands 2 turns later, 40 centre / 25 one cell away; public marker; self-damage possible. |
| Medpack | Heal 40, capped at maximum health. |
| Smoke | 3 × 3 cloud at our cell for 3 turns; hides bots from observers more than 1 cell away; marks remain visible. |
| Radar | Reveal all enemies for 3 turns. |
| Legendary Orbital | Lands 2 turns later, radius 2, 60 centre / 30 edge; public marker. |
| Legendary Overshield | For 3 turns, a hit that would kill leaves us at 1 health instead, as described; exact consumption semantics belong in section 20. |
| Legendary Reaper | For 5 turns, doubles kill points from our kills; verify scope of “kill points.” |

One legendary crate drops near the centre per match, announced 2 turns earlier in `view["legendary_incoming"]`. Every match also has one `BLOOD_MOON` lasting 6 turns (attack, shoot, railgun, shove-slam damage ×1.5; blasts unchanged), and one instantaneous `RESOURCE_RUSH` adding four persistent inner-arena resources.

Eight themes—Scrap Street, Glitch Garden, Rooftop Rumble, Vibe Dome, Candy Carnival, Haunted Arcade, Underwater Disco, Dreamscape Dojo—and team characters affect appearance only. Do not branch strategy on them.

Taunts change no gameplay. Allowed: `GG`, `NICE`, `OOPS`, `TOO_SLOW`, `WATCH_THIS`, `RIP`, `BOOM`, `HELLO`, `THANKS`, `CATCH_ME`. Omit them until all substantive work is complete.

## 5. Roles, shopping, and series economy

### 5.1 Role reference and candidate builds

**Guide rules (pp. 15–16):** Choose a role secretly before each map, or `None` for no bonus/penalty. Roles are fixed during that map. Role awards also double in overtime.

| Role | Benefits | Costs | Proposed use |
|---|---|---|---|
| `None` | No specialization penalty | No role bonus | Control variant; fallback when specialization has not proved useful. |
| `HUNTER` | +10 per kill; Rampage +5 for each kill already on our streak | Cargo banking ×0.85; killer gets +25 for killing us | Only promote if combat tests demonstrate sustainable kills and survival. |
| `HAULER` | +1 cargo capacity; bank 3 units at ×1.25, 4+ at ×1.5 | Kills pay 10 less | First cargo-led candidate; retain opportunistic Core, tags, and safe kills. |
| `RUNNER` | +10 per uplink, plus +10 per earlier uplink that map; ability recharge 2 turns faster | Maximum health −5 | Core/mobility candidate with cargo fallback. |
| `WARDEN` | +20 per defuse; Guard +20 for killing a carrier, live planter, or bot within 2 cells of a planted Core | Cannot pick up Core; cargo banking ×0.9; capacity −1 | Objective-denial candidate, especially against Core-heavy opponents. |

**Derived examples, before engine rounding/stacking details:**

- Four resources of value `v` banked by a Hauler pay `6v` normally or `12v` in overtime. This does not prove Hauler wins: pickup turns, route length, availability, and survival determine throughput.
- Hauler already holds 5 units; `POCKETS` raises that to 6. The 1.5 multiplier begins at 4 units, so waiting for 6 is not automatically optimal.
- Runner’s first successful uplink pays 70 normally; its second pays 80, before overtime. Runner + Overclock implies the minimum 3-turn cooldown under the stated reductions and floor.
- A lone Warden defuse has a nominal 40 + 20 = 60 award before overtime, if the full base award applies to the sole defuser. Multi-defuser distribution must use the engine.
- Neutral capacity 4, Warden capacity 3, and Warden + Pockets capacity 4 follow from the published modifiers.

### 5.2 Shop reference

| Purchase | Credits | Effect |
|---|---:|---|
| `ARMOR` | 2 | +15 maximum health |
| `POCKETS` | 2 | +1 resource capacity |
| `SCOPE` | 2 | +1 vision range |
| `QUICK_RESPAWN` | 3 | Respawn 1 turn sooner |
| `OVERCLOCK` | 3 | Ability cooldown −2, floor 3 turns |
| `KIT_RAILGUN` | 3 | Start with railgun |
| `KIT_MEDPACK` | 2 | Start with medpack |
| `KIT_SMOKE` | 2 | Start with smoke |
| `KIT_RADAR` | 2 | Start with radar |

Each purchase at most once per map; at most one `KIT_` total. Shop input includes our credits, prices, role options, map number, and our past placements, not rival information. `buy(self, shop)` returns a role/items mapping; `None` preserves the class `ROLE` and buys nothing. Unknown names, duplicates, excess spending, multiple kits, crashes, or timeouts buy nothing.

### 5.3 Purchase policy proposal

Implement a small ordered list of prevalidated bundles or a bounded enumeration of subsets, using current shop prices. Reject invalid bundles before scoring them. Do not use board topology, current enemy styles, or rival purchases in `buy()` unless the contract actually supplies that information.

Starting four-credit bundles to compare:

- **Cargo baseline:** Hauler + Armor + Pockets. Compare against Hauler + Armor + Medpack and Hauler + Armor with two credits saved. Pockets must earn its place through fewer trips or extra realized banking; Hauler already has five slots.
- **Core mobility:** Runner + Overclock with one credit saved, compared with Runner + Armor + Medpack. This tests movement frequency against survival.
- **Denial:** Warden + Armor + Medpack, compared with Warden + Armor + Pockets.
- **Neutral control:** no role + Armor + Medpack.

Higher-budget priorities depend on measured bottlenecks: Armor for lethal-damage thresholds, Overclock for repeatedly useful mobility, Pockets for actual capacity pressure, and a kit that our item logic uses well. Scope must create better decisions; Quick Respawn must recover enough productive turns to beat survival investment. Do not buy a kit merely to replace it immediately at a nearby crate.

No default instruction to buy every affordable item. Consider saving for a useful next-map bundle, but spend before the 10-credit cap wastes income. If the final map is known from valid input, leftover credits have no future value.

### 5.4 Credits and persistence

**Guide rules:** Start each series with 4 credits. After a map gain 3 plus placement bonus 0/1/2/3 for first/second/third/fourth, plus 1 per consecutive map without a win up to 2. Unspent credits carry, capped at 10.

Use supplied credit totals as truth. A local evaluator may reproduce:

`next_credits = min(10, unspent + 3 + placement_bonus + min(nonwinning_streak, 2))`

Bind the streak update order to engine behavior. Across maps, the prior winner’s crown and increased consecutive-win bounty, nemesis, and credits carry. Health, energy, scores, board state, equipment effects, and match objectives restart according to the new-map setup. Do not assume our own per-match memory persists across maps; use supplied shop history.

Adapt role selection only with legitimate available evidence. Past placements can justify switching among pretested bundles, but they do not identify which rival uses which strategy. Never infer opponent roles as certain from an unavailable field.

## 6. Strategic design: choose tasks by realized value

### 6.1 Separate strategic tasks from tactical actions

The bot maintains one current task, such as `BANK`, `COLLECT_RESOURCE`, `CONFIRM_TAG`, `FETCH_CORE`, `DELIVER_CORE`, `DEFUSE`, `DEFEND_PLANT`, `GET_CRATE`, `FIGHT`, or `EXPLORE`. Each turn it:

1. Refreshes known state and invalidates stale plans.
2. Handles urgent hazards and immediate scoring/survival opportunities.
3. Scores a small set of feasible tasks.
4. Chooses a legal next action that advances the best task.
5. Revalidates that action against the current view before returning it.

This avoids a brittle unconditional ladder such as “always chase Core before banking” or “always heal before taking a guaranteed winning action.” Urgent rules can override tasks, but ordinary choices compete on expected value.

### 6.2 Proposed utility model

For each task, estimate:

`utility = expected_own_points / max(1, committed_turns) + denial_weight * expected_denied_points / max(1, committed_turns) - risk_penalty - switching_penalty`

Where:

- `expected_own_points` includes completion probability, actual scoring time, role effects, and points already carried but not yet banked.
- `committed_turns` includes movement, collection/use actions, countdown or required residence, and necessary return travel. Count resource pickup as an action and tags/banking/planting as automatic only where rules confirm.
- `expected_denied_points` is a tactical estimate for preventing an observable rival award, not an invented rival score.
- `risk_penalty` covers likely lost cargo, item value, streak, death/respawn downtime, contested movement, and losing the current opportunity.
- `switching_penalty` prevents flip-flopping between similar destinations; urgent danger or disappearing targets overrides it.

Use a few interpretable constants, not a huge collection of tuned weights. Keep estimates conservative under fog. First implement explicit rules with measurable thresholds; add utility comparisons only where two otherwise sensible rules conflict.

Never treat previously scored points as lost when dying. The cost of death is future opportunity and current carried value, plus enemy awards. A planted Core can finish after our death; a suicide bomb gives no kill reward to anyone for our own death.

### 6.3 Initial task ordering

This is a proposal to implement and then test, not a rigid substitute for the utility model:

1. Use the contract-defined path if dead/inactive and decision calls still occur.
2. Escape a known imminent lethal blast, or apply correctly timed mitigation if escape is worse or impossible.
3. Evaluate guaranteed legal survival/finish opportunities: heal, kill an adjacent vulnerable enemy, complete an urgent defuse, or deliver valuable cargo.
4. Preserve an almost-complete defuse or protect a valuable live plant when the expected reward exceeds alternatives.
5. Bank when capacity, danger, overtime timing, or match horizon warrants it.
6. Take high-value nearby tags, favorable shared kills, or efficient resource pickups.
7. Pursue feasible Core, crate, and longer resource tasks.
8. Explore a useful unobserved region; take a legal defensive fallback if no productive action exists.

No branch may bypass legality. A “safe fallback” is safe only if its energy and state preconditions hold.

## 7. Navigation, fog, and memory

### 7.1 Board graph and routes

- Build a static walkability graph from the current map’s walls using the SDK’s geometry helpers. There are at most 144 cells and 8 movement neighbors per cell.
- Use BFS for uniform-cost movement. A single traversal is `O(V + E)` over at most roughly 1,152 directed neighbor candidates; cache reusable adjacency and one distance field per relevant source/target set.
- Use BFS distances for actual detours; Chebyshev distance only for cheap lower bounds. Never repeatedly walk directly toward an objective through a wall, as the guide’s example does.
- Treat currently visible occupied cells and likely contested destinations as dynamic risk/blockage. Keep static connectivity separate so a temporarily occupied corridor is not permanently unreachable.
- Include ability edges only when their full legality is known. A time/cooldown-aware planner can improve mobility estimates later; the first version evaluates a few legal Dash/Blink destinations and their remaining route distance.
- Avoid trying an unready ability in an estimated route. Energy, cooldown, landing occupancy, and geometry must be valid on the actual action turn.
- Replan after failed movement. Distinguish “intended destination still blocked” from “task remains worth pursuing”; do not repeat the same failed move indefinitely.
- If two nearly equal routes exist, prefer the one with less blast exposure, fewer visible enemy contacts, and fewer shared target cells.

### 7.2 Partial observation

Maintain last-seen enemy positions with turn stamps, last-seen resources with confirmation status, and recent failed destinations. Current observations always override memory.

- Never convert “not visible now” into “gone” unless we can currently observe its old cell and the contract makes absence meaningful.
- Stale enemy positions expand into reachable regions rather than becoming exact targets. Start with a small confidence/risk adjustment; do not build a large probabilistic engine under the contest clock.
- Current globally marked positions are reliable observations for location, but do not reveal hidden item type, role, score, or all other state.
- Use known geometry and symmetry for route preferences; do not manufacture mirrored resources that have not been observed or guaranteed by a verified spawn rule.
- Exploration should move toward reachable unseen cells with potential resources or useful sight of objectives. Do not repeatedly scan instead of moving.
- Scan/Radar must improve a pending choice enough to justify the action and energy. Globally visible Core/crates often already provide useful destinations.

### 7.3 Memory and plan stability

Keep `self` data compact: map identity/wall signature, static adjacency, current task/target, previous position and action, failed-move count, observation timestamps, recent target choices, and only scoring history that the view does not already provide.

Clear tactical commitments after death, a target disappearing, objective state changing, or map reset. Preserve reusable map geometry through a respawn only when the map is unchanged. Use turn progression/map identity from the actual contract rather than assuming `__init__` and `buy()` always share one object.

Use stable tie-breaks over task type, target ID, cell coordinates, and direction order. Normalize directions by our corner where helpful for fair testing; never rely on unordered set iteration or process-dependent hashes.

## 8. Cargo strategy: reliable score conversion

### 8.1 Resource route evaluation

For a resource, compare the entire pickup-and-bank route:

`route_turns = distance_to_resource + 1 collection + distance_resource_to_base`

For multiple pickups, add each leg and each collection action, recomputing capacity, threat, and feasible banking time. Avoid exact traveling-salesperson optimization; compare a handful of nearby candidates or a small greedy route.

Use actual resource values and unit counts from the kit. Do not assume every resource is worth the same or every dropped pile consumes one slot.

### 8.2 Banking rules to test

Bank when any of the following makes banking more valuable than continued collection:

- Hold is full or the next collectible does not fit.
- We can achieve Hauler’s 3-unit or 4+-unit threshold efficiently and extra pickups delay the return excessively.
- Health/threat makes carried value unlikely to survive another detour.
- Cargo value reaches 60 and global visibility makes the route substantially more dangerous.
- Remaining turns leave only the bank trip plus a collision/detour margin.
- The base lies on the route to another objective, making a partial bank nearly free.

Being below a multiplier threshold is not a reason to lose the entire load. Banking four Hauler units can beat waiting for a fifth or sixth. Base pockets can also become bottlenecks; retain alternate exit/entry routes.

### 8.3 Overtime banking decision

If `V` is the bankable value now and waiting until overtime would double it, compare the full alternatives:

- Bank now: secure `V`, then use the remaining turns for more points.
- Delay: expected `2V * probability_of_successful_bank`, plus any other achievable points, minus the opportunity cost of waiting.

Ignoring all other effects gives a rough survival break-even above 50%, but the actual decision includes missed collection cycles, exposure, and enemy scoring. Do not deliberately wait on our base, where banking is automatic. Stage near a safe return route only when the timing and expected gain are verified.

Use the actual turn/phase semantics to schedule the scoring event, not merely “start moving at turn 32.” At the end of the game, bankable cargo dominates a speculative pickup that cannot be delivered.

### 8.4 Hot pickups and spilled piles

Treat the 1.3 hot bonus as a benefit of an already acceptable pickup, not a reason to enter lethal range. A visible nearby bot can provide evidence for the bonus; absence in fog does not prove no bot is nearby. Do not depend on receiving the bonus when its condition is uncertain.

Spilled cargo can be a high-value route target, but its death location is often dangerous. Check fit, value, remaining horizon, nearby enemies, and escape route. Never multiply the hot bonus again on a spill.

## 9. Core strategy: race, plant, defend, or deny

### 9.1 Decide whether to enter the race

Before turn 8, compare useful resource routes with positioning for the first Core spawn. Do not spend seven turns waiting in the centre. Once the Core appears, score:

`distance_to_Core + 1 pickup + carrier_route_to_pad + countdown_to_payment`

Compare that full commitment with cargo and tag alternatives. Current marked/visible opponents inform competition; unknown opponents remain a risk. If someone else is already likely to win the race, switching early can be better than arriving second with no points.

Warden excludes Core pickup from candidate generation entirely. Its Core tasks are defuse, protect a scoring opportunity, intercept a vulnerable carrier, or obtain the Guard kill bonus legally.

### 9.2 Choose a pad by completion probability

Evaluate all four pads using:

- Our actual route, including currently usable mobility.
- Enemy travel time to the pad’s defuse neighborhood.
- Escape routes, bottlenecks, visible hazards, and likely firing lanes.
- Whether there is enough time for pickup, delivery, and countdown to finish before match end.
- Whether payment will occur in overtime.

The nearest pad is not necessarily the most profitable. A slightly farther pad with no reachable defuser can dominate. Score a plant by probable completion, not merely by reaching a pad.

### 9.3 After planting

The Core plants automatically. Do not waste an action trying to collect or manually plant it again. The planter can move away and still score if it completes.

Predict which enemies can establish two consecutive end-of-turn adjacent positions. Compare protecting the plant with taking another scoring route:

- Attack a vulnerable defuser if killing it reliably preserves the uplink.
- Shove a defuser out of the valid neighborhood when available and effective.
- Use blast threats for area denial only when their resolution occurs before the decisive defuse step.
- Do not stand on our own incoming blast or assume a distant defensive shot will connect after movement.
- If nobody can defuse in time, leave for the next useful task rather than guard an uncontested plant.

### 9.4 Defusing

Defuse progress is positional; a special defuse action is not described. Reach a valid neighboring cell and remain eligible for two consecutive turn ends. Prefer a position with protection, escape, or useful attack targets. Defend/heal/attack while maintaining position when those actions are legal and appropriate.

Track observed consecutive adjacency without claiming hidden engine progress. If forced out, re-evaluate whether there is still time. Positioning into a blast to complete a defuse is worthwhile only if the expected award and enemy denial justify the cost.

For Warden, defuse is a major scoring candidate. For other roles, it can still beat a lengthy cargo route, especially during overtime. Do not award ourselves a defuse merely for reaching the neighborhood once.

### 9.5 Repeated cycles

After resolution, use the four-turn Core respawn window for cargo, tags, positioning, or crates. Runner’s escalating uplink bonus makes successful repeat cycles valuable, but competition and the 40-turn horizon limit how many are feasible. Avoid hard-coding a promised number of captures per match.

## 10. Combat strategy: short, favorable engagements

### 10.1 Damage and action efficiency

Derived from the guide, unmodified melee on an undefended 70-health target needs 3 hits at best and 5 at worst; four 20-damage hits is only a rough midpoint model. Shooting needs 5 hits at best and 7 at worst before healing/defense. Isolated long duels can consume a large fraction of the match.

Prefer finishable enemies, shared-kill opportunities, valuable marked targets, and fights that protect a high-value objective. Do not chase full-health bots across the map for an uncertain kill.

Maintain three simple damage estimates: minimum, nominal, and maximum, with known Blood Moon, defense, protection, and item effects. Use minimum effective damage for a claim of a guaranteed kill. Do not assume the guide’s ranges imply a known probability distribution.

### 10.2 Melee versus shooting

- Melee is selected against an adjacent legal target at decision time and hits even if that target moves away. Walking away from an adjacent enemy does not erase its already-valid melee attack.
- Ordinary shots hit after movement and stop at the first bot. Railgun also resolves after movement but hits all bots on its line.
- Evaluate a small set of plausible enemy destinations: stay, continue toward objective, move toward/away from us, or escape hazard. Do not count a currently aligned target as a certain shot hit.
- Prioritize constrained corridors and objectives that make future positions predictable. Keep line geometry identical to the engine’s.
- Consider incoming melee even when planning an escape. A move to an empty cell may still be lethal; healing/mitigation can be better when its verified phase allows it.

### 10.3 Target scoring

Estimate immediate kill payment, expected bonuses, attainable tag value, and tactical denial, then subtract our likely damage, lost cargo, extra turns, and bounty offered to others.

Use First Blood, crown bounty, shutdown, revenge, nemesis, Guard, and streak effects only when their conditions are known or conservatively estimated. Never seek low health merely for Clutch. Avoid repeat-killing the same victim as a default because the base award drops to a tag-only 15 within 8 turns.

When several bots fight, dealing damage on the killing turn can earn a shared kill; earlier damage may earn an assist. This makes well-chosen interventions efficient, but does not justify firing blindly every turn. Regeneration covers a baseline melee’s 5 energy; repeated 6-energy shots slowly deplete energy before overtime, and other actions still need reserves.

### 10.4 Tags as route objectives

Treat reachable tags as scoring pickups with deadlines. Compare distance, expiry, likely contest, and whether collecting fits our current route. A 10-point denial can improve our relative position by both paying us and preventing the killer’s remaining award; it is not automatically a 25-point lead swing unless that killer otherwise would have confirmed it.

Own tag confirmation completes the ordinary 40-point base kill total. Do not count “40 for kill” and then add another 15 confirmation. Verify ownership and shared-kill cases in engine logs.

### 10.5 Health, defense, and spawn protection

- Heal when damage threatens useful future actions or the 40 heal is substantially usable; do not require exactly 40 missing health if waiting risks death.
- Defend/Shield when reduced incoming damage preserves cargo, a defuse, or another valuable commitment. Avoid indefinite defensive loops with no path to points.
- Use spawn protection for safe relocation where useful. Collecting or fighting breaks it, so evaluate that choice explicitly.
- Do not waste attacks into known protection or assume enemy protection is absent when unobserved.
- Preserve a valuable streak when it improves future score, but do not let streak protection override all map objectives.
- During Blood Moon, recalculate both our damage and enemy lethality. Do not multiply blast damage.

## 11. Ability and item policy

### 11.1 Ability selection experiment

Keep `ABILITY` fixed for each candidate file unless the contract explicitly permits another supported selection path. Compare ability variants using identical opponents, seeds, and corners.

| Candidate | Hypothesis | Promotion requirement |
|---|---|---|
| `None` | Simplest legal baseline | Always retained as a diagnostic control. |
| `DASH` | Saves a move in usable corridors and helps return trips | Measured productive uses exceed blocked/unused opportunities. |
| `BLINK` | Crosses walls and shortens Core/cargo routes | Landing legality is reliable and route savings survive varied maps. |
| `SHIELD` | Preserves high-value commitments under burst damage | Prevented deaths translate into more rank points than mobility. |
| `SHOVE` | Disrupts defuses, creates slams, and supports blast kills | Timing and target validation are implemented and the denial gain is repeatable. |
| `BOMB` | Controls constrained pads or multi-bot fights | Escape/chain-reaction reasoning is correct; no self-kill pattern. |

Start with `None` to establish the baseline, then one mobility ability. Blink is a promising candidate, not a proven winner. Do not implement all five merely because they exist. Bomb/Shove combinations and complex combat abilities are stretch work if the simpler candidate already passes and time remains.

### 11.2 Ordinary items

- **Medpack:** Use before likely lethal damage when healing resolves in time; account for maximum health and item opportunity cost.
- **Railgun:** Aim using engine ray geometry and post-movement estimates. Favor multiple plausible hits, constrained enemies, or a high-value kill.
- **Airstrike:** Aim at locations likely occupied two turns later—contested pads, bottlenecks, forced routes—rather than mechanically at a current position. Recheck our own planned route against the marker.
- **Smoke:** Use when it meaningfully breaks distant observation/targeting. It does not hide marks or block damage by itself. A crown, carrier, planter, high-cargo, or on-fire bot cannot assume invisibility.
- **Radar:** Use before a meaningful route or fight choice when existing visibility is insufficient. Avoid spending it while an equivalent streak radar effect is already active unless timing justifies it.

### 11.3 Crates and legendary items

Compare a crate’s expected option value with travel + collection and the value of the held item it replaces. Read `item_pool` and never plan around a guaranteed railgun drop. When trailing, increased power-item odds may make a nearby crate more attractive, but do not lose rank deliberately to manipulate the draw.

The legendary warning grants two turns to evaluate route and competition, not a command to rush the centre. An uncertain legendary reward can be worse than guaranteed banking.

- **Orbital:** Model its larger danger footprint, delayed impact, escape routes, and possible bomb chains. Radius geometry comes from the engine.
- **Overshield:** Activate before a dangerous high-value commitment using verified duration and hit-consumption semantics. It is not assumed to provide unlimited immunity.
- **Reaper:** Activate when realistic kill opportunities fit its five-turn window. Do not spend the window traveling with no targets. Multiply only the score components the engine actually treats as kill points.

Use a valuable item before opening another crate when doing so is genuinely productive; do not waste an action merely to empty the slot. Account for the streak-five airstrike award when interpreting sudden item replacement.

### 11.4 Blast planning

Maintain a small hazard schedule for known bombs, airstrikes, and orbitals. Associate danger with the turn-start explosion phase and our position after the preceding actions.

1. Find all blasts that can occur before our next decision.
2. Expand chain reactions among known bombs to closure.
3. Evaluate our candidate destination and planned residence, including defuse/plant commitments.
4. Prefer escape when it preserves more value than mitigation.
5. If remaining, use correctly timed defense/shield only when energy and cooldown permit.

Never claim certainty about unseen bombs. Avoid placements that require multiple perfect collision-free moves to escape. A fixed shallow horizon is sufficient initially; full game-tree search is unnecessary.

## 12. Turn phases and risk management

The following strategic phases are proposals, not engine event guarantees:

| Match period | Main priorities |
|---|---|
| Before first Core spawn | Exit base efficiently; acquire worthwhile local resources/items; scout useful routes; avoid full-health duels. |
| First Core cycle | Race only if delivery is profitable; otherwise collect, intercept a vulnerable carrier, or position to defuse. |
| Middle of match | Repeat high-value scoring cycles; respond to supply waves, Resource Rush, Blood Moon, and legendary warning. |
| Approach to turn 32 | Re-evaluate bank, plant, and combat payment times for overtime; retain survival and opportunity costs. |
| Overtime | All point sources are more valuable; exploit doubled energy regeneration; secure points before the horizon. |
| Final feasible scoring turns | Bank in time, take collectible tags, complete feasible defuses/uplinks, and avoid tasks whose rewards occur after match end. |

Derive “last safe departure” from real path length, actual phase semantics, and a modest obstruction margin. Do not hard-code one final-turn number for every map. A late bomb/airstrike/Core plan only counts if its resolution happens within the match.

When leading, prefer high-confidence points and avoid giving a bounty unnecessarily; hiding forever can still lose the lead. When trailing late, accept measured variance such as a contested high-value objective, but do not discard reachable guaranteed points for an impossible comeback plan.

## 13. Single-file architecture and implementation contracts

### 13.1 Proposed internal structure

These names are our internal design, not claims about SDK APIs:

```text
my_bot.py
  permitted imports and configuration constants
  class MyBot(Bot)
    ABILITY / ROLE defaults
    buy(shop)
    decide(view)
    normalize_view(view)
    update_memory(state)
    build_or_reuse_board_graph(state)
    estimate_threats(state)
    enumerate_tasks(state)
    score_task(task, state)
    choose_action(task, state)
    legal_candidate(action_spec, state)
    build_sdk_action(action_spec)
    fallback_action(state)
```

Keep internal action descriptions separate from final SDK objects if that simplifies validation. Construct the final `Action` only after its parameters have passed the relevant checks. Avoid reflection tricks or reliance on undocumented object internals.

The normalized state should contain only contract-supported information: self position/health/energy/cargo/item/cooldown, board/walls/base/pads, visible/marked enemies, visible resources/tags, Core state, hazards, events, turn, and allowed match metadata. Actual keys/types come from the kit. Optional fields can default sensibly; required-field mistakes should be detected during development rather than silently hidden.

### 13.2 Legality gate

For each candidate, check:

1. Bot is in a state where this action is accepted.
2. Energy covers the exact cost after applicable rules.
3. Direction/ID/coordinates are in the SDK’s supported form.
4. Target is currently valid, not merely remembered under fog.
5. Range, line/path geometry, and board bounds meet the action’s requirements.
6. Ability is equipped, ready, and satisfies special constraints such as one live bomb.
7. Item type and arguments match the held item.
8. Collect target is on our current cell, fits applicable capacity, and is permitted by role; Warden never collects Core.

Use `vibe_sdk.geometry.line_clear`, `shot_path`, and `step_open` after reading their signatures. An occupied destination can make ordinary movement ineffective without being illegal, while ability landing rules may differ. Keep this distinction explicit.

### 13.3 Fallback contract

The kit’s legal fallback for low energy, dead/inactive states, or unusual views must be established first. Do not invent `Action.wait()` or assume Defend always works: Defend costs 2 and movement costs 1.

Once verified, use an ordered fallback: a legal productive/safe move, a legal defensive action where useful, and finally the SDK-sanctioned idle behavior if one exists. If the engine guarantees minimum decision-time energy, encode only the guarantee actually confirmed by tests. No blanket `except: return Action.defend()` that could return another illegal action.

### 13.4 Runtime budget

Proposal: profile for ordinary decisions in roughly 10–30 ms and an observed maximum comfortably below 100 ms on our laptop, leaving headroom under 200 ms. These are engineering targets, not measured results or a promise about tournament hardware.

- Cache wall graph once per map; reuse distance fields within a turn.
- Limit strategic targets, for example the best 6–8 nearby candidates plus mandatory active objectives.
- Bound tactical search to a few actions and one or two future steps; start without it.
- Use at most 144 board cells, at most 3 enemy bots, fixed direction lists, and capped memory.
- Do not search all joint action sequences, perform Monte Carlo rollouts, or enumerate full match futures by default.
- Avoid recursion without small fixed depth, repeated all-pairs pathfinding, and unnecessary copies of the view.
- Measure first-call setup, worst crowded/hazard state, and shopping separately. Average speed alone is insufficient.

Deterministic work caps replace time-based cutoffs. All profiling clocks live in development tools, outside the submission.

## 14. Repository intake when the QR code is released

### 14.1 Establish a clean baseline

1. Download the official repository ZIP or clone the organizer-supplied URL. Keep an untouched copy.
2. Record its version/commit if available, Python version, and the official deadline/submission details.
3. Run the unmodified starter with the documented commands:

```powershell
python --version
python check.py
python play.py
python conformance.py
```

4. Resolve environment failures before bot work. During the setup period, raise a hand if the supplied kit fails as the guide instructs.
5. Save the starter as the first known-good fallback. Verify it is a complete standalone entry.

### 14.2 Read in a deliberate order

| File or area | Facts to extract |
|---|---|
| `docs/RULES.md` | Full numbers, event ordering, scoring/rounding, supply schedules, cooldowns, role and item interactions. |
| `docs/BOT_SDK_CONTRACT.md` | Complete view/shop schema, optional fields, legal actions, lifecycle, inactive handling, determinism contract. |
| `my_bot.py` | Required imports, subclass shape, working constructor/action examples, fallback. |
| `check.py`, `conformance.py` | Restricted operations, admission cases, timeout accounting, deterministic replay criteria. |
| `vibe_sdk/` | `Action`, `Bot`, direction deltas, geometry helper signatures and semantics. |
| `engine/` relevant sections | Definitive action validation, simultaneous resolution, damage, collection/banking, Core, scoring, respawn and shop order. |
| `play.py`, `sparring.py` | Seed controls, opponent selection, series support, logging/results, available chaos bots. |

The guide contains illustrative code with symbols that can be mangled by PDF extraction. Copy working code from the starter/SDK rather than pasting extracted PDF code.

### 14.3 Produce a compact contract map before adding strategy

For each rule we will use, record its source symbol/location and one confirming observation or test. Prioritize movement/geometry, active/dead state, energy regeneration, action validation, collection/banking, Core timing, and delayed blasts. Then bind scoring, role/shop interactions, and advanced items.

Confirm actual CLI options through source/help; the guide already documents these commands:

```powershell
python play.py
python play.py --seed 42
python play.py --seeds 20
python play.py --vs sparring:ChaosShove
python check.py
python conformance.py
```

Do not invent flags for corner rotation, saving replays, opponent lists, or series lengths. Use the harness as supported. If combined flags are supported, combine seed sweeps with chosen opponents; otherwise run the available modes separately.

## 15. Implementation ladder and stopping rules

Each level produces a complete, legal bot. A later level is optional until its benefit is demonstrated. Keep versioned copies outside the active submission name; only one selected file is ultimately attached.

| Level | Deliverable | Acceptance gate |
|---|---|---|
| V0: baseline | Official starter plus confirmed schema and legal fallback | Check, conformance, and basic play pass. |
| V1: navigation | Wall-aware paths, no persistent stuck loops, stable deterministic decisions | Varied maps/corners reach targets without illegal moves. |
| V2: cargo | Resource selection, capacity-aware collect, sensible bank return, endgame delivery | Repeated real banking; fewer stranded loads; no collection errors. |
| V3: hybrid | Tag routing, favorable melee, basic Core pickup/delivery/defuse, rank-aware task scoring | Improved paired placement results with no reliability loss. |
| V4: mobility and shop | One verified movement ability, a few valid bundles, health/item logic | Ability saves useful turns; shop never silently loses intended upgrades. |
| V5: timing | Overtime delivery, blast avoidance, Core deadline logic, event responses | Targeted boundary scenarios pass; no delayed-reward planning errors. |
| V6: refinement | Runner/Warden ablations, better shots, selective item use, small opponent-response improvements | Improvement persists across held-out opponents/maps and remains fast. |

Implementation order within a level follows dependencies, not feature excitement. A completely correct V2/V3 is preferable to an unfinished V6 that fails admission. The exact level reached depends on actual kit complexity and test duration.

**Drop first if time is tight:** extensive role switching, complex bomb/shove combinations, fine-grained enemy belief distributions, deep tactical search, legendary-item hunting, extra taunts, and large parameter sweeps. Retain legality, navigation, bank deadlines, immediate hazard handling, and competent use of currently held items.

After a failing change, either make a small diagnosed fix and rerun the gate or restore the last known-good file. No large last-minute rewrites.

## 16. Test and evaluation plan

### 16.1 Three layers

1. **Official admission:** `check.py` and `conformance.py`, including after final file selection. Passing a quick check is not evidence of competitive strength.
2. **Targeted behavior:** reproducible situations that expose geometry, phase, legality, and strategic mistakes. Use supported fixture/harness mechanisms without modifying official engine behavior.
3. **Competitive evaluation:** varied seeds, corners, mixed opponent styles, and full series when available.

The default `play.py` uses three copies of our own bot. Self-play is useful for collisions and mirror behavior, but it is not a sufficient performance benchmark. Chaos bots help find crashes; beating them does not establish tournament strength.

### 16.2 Required targeted cases

| Area | Cases | Expected invariant |
|---|---|---|
| View handling | Empty enemies/resources/tags/crates; absent optional data; visible marks through fog | No invalid indexing, invented data, or stale-target actions. |
| Energy | Every used action at just below/exactly/above cost; normal and overtime regeneration | Never knowingly return an unaffordable action. |
| Geometry | Every edge/corner; adjacent walls; diagonal corner blockers; narrow base exits | Movement, sight, and rays agree with SDK. |
| Mobility | Dash intermediate obstruction; Blink wall crossing/occupied landing; cooldown boundaries | Ability accepted exactly when contract permits. |
| Collision | Two bots choose one tile; occupied corridor; consecutive failed moves | Replan instead of freezing or looping. |
| Cargo | Full hold; 3/4-unit Hauler thresholds; partial bank; hot pickup; dropped pile fit | Correct capacity and no duplicated value multiplier. |
| Banking | Standing on base; arriving before/at overtime; final feasible return | Points only when actually banked. |
| Core | Turn-8 spawn; pickup; auto-plant; countdown; death after planting; respawn | No invented plant action or abandoned feasible payout. |
| Defuse | One vs two consecutive eligible ends; interruption/shove; multiple defusers | Progress and rewards match engine. |
| Combat | Melee target moves away; shooter target moves; first blocker; railgun multi-hit | Use correct phase and ray behavior. |
| Scoring | Repeat victim; shared kill; assist; own/other tag; expiry; crown and role bonuses | No double-counting; score log reconciles. |
| Hazards | Countdown 1; escape; prior-turn defense; chained bombs; overlapping blasts | Danger applies before next decision, not after. |
| Protection | Respawn immunity and each protection-breaking action; overshield | No assumed invulnerability beyond actual rule. |
| Events | Blood Moon starts/ends; Resource Rush; legendary warning; missing item types | Damage and item availability handled correctly. |
| Lifecycle | Death/respawn; map reset; changed walls; shopping before map | No stale tasks, graph, role, or cooldown assumptions. |
| Buy | Low/high credits, changed prices, duplicate names, two kits, cap pressure | Returned bundle is valid and useful. |
| Horizon | Delayed effects straddling turn 32 and match end | Score only events that actually resolve in time. |
| Determinism | Same state/history replayed; fresh process; changed hash seed if supported | Identical outputs under the checker’s exact contract. |
| Runtime | First turn, dense visibility, many hazards, long memory, shop | No timeout; fixed bounded work. |

Prioritize tests for features actually included, while ensuring unexpected held items/events cannot crash the bot. Do not spend the entire contest building a large custom test framework. Official tests, seed sweeps, and a few decisive reproductions come first.

### 16.3 Opponent portfolio

Use available supplied opponents first. Where the event permits local extra test bots and time allows, make minimal development-only opponents; never submit them or alter official opponents.

| Opponent style | Weakness it reveals |
|---|---|
| Official chaos / `ChaosShove` | Illegal edge cases, displacement, unpredictable occupancy. |
| Greedy cargo banker | Whether our uncontested economic throughput is competitive. |
| Core-first runner | Ability to avoid losing races and profitably deny plants. |
| Aggressive melee pursuer | Escape, health management, and adjacency timing. |
| Shooter / railgun user | Post-movement exposure and corridor risk. |
| Warden / pad defender | Whether our Core route is too predictable. |
| Passive survivor | Whether we can reliably create points without opponents feeding kills. |
| Our previous best version | Whether a change helps against a meaningful stable reference. |
| Three copies of current candidate | Mirror collisions and deterministic symmetry. |

Test mixed groups as well as three identical opponents: for example a farmer, a runner, and an attacker. Multiplayer interference can change which strategy is best.

### 16.4 Experimental method

- Hold opponents, seeds, and starting corners constant when comparing two versions. Change one strategic feature at a time.
- Begin with the documented 20-seed sweep as a smoke/performance screen. Twenty maps are a small sample, not proof of dominance.
- Separate development seeds from an untouched final holdout set. Use unseen opponent combinations on the holdout.
- Rotate all four corners when the harness supports it; compare corner-specific results, not only aggregate averages.
- Increase to 50–100 or more scenarios only if measured harness runtime and the contest clock allow. Testing depth must leave final submission time.
- Prefer series-aware evaluation with credits, role purchases, crown carryover, nemesis, and exact tie-breakers. Do not pretend isolated map tests measure those effects.
- Treat a whole series/opponent grouping as a dependent cluster when estimating uncertainty. Simple paired deltas and per-opponent tables are more useful than misleadingly precise percentages.
- Review representative losses, the worst collapse, and one success after each material change. Preserve the seed, corner, version, and relevant log lines.

### 16.5 Metrics to capture

Record candidate ID, configuration, seed, corner, opponent mix, placement, rank points, score, and admission/runtime failures. When logs expose them, also record:

- Score split: cargo, kills/bonuses, tags, Core uplinks, defuses, assists.
- Cargo banked, units per trip, lost/stranded cargo, productive travel turns.
- Core pickups, plants, successful uplinks, failed races, defuses.
- Kills, deaths, time out of play, health/energy waste.
- Failed moves, repeated-cell loops, unnecessary scans/defends, unused items.
- Ability uses and route turns saved; purchases and credits left unused.
- Decision latency distribution and maximum, including initialization and buying.

Collect these outside the bot or through permitted logs. Do not add forbidden runtime file logging to the entry.

### 16.6 Promotion rules

Promote a candidate only if it passes official admission, has no known deterministic/runtime defect, and improves paired placement results without an unexplained severe weakness. Small noisy score gains are not sufficient. If evidence is inconclusive, keep the simpler established candidate.

Example ablations: mobility on/off; Hauler versus None; early banking versus waiting for capacity; Core participation on/off; tag routing on/off; Armor/Pockets versus Armor/Medpack; fixed versus overtime-aware return timing. Avoid changing all of these together.

## 17. Preparation and competition-day execution

### 17.1 Before Sunday

**Wednesday 14 October:** Read this plan together, confirm registration/team membership, assign the typist and tester, and prepare a concise strategy brief. This document is preparation, not a claim that prewritten competition code is allowed.

**Thursday 15 October:** Install/check Python 3.9+, verify the editor and terminal workflow, sign in to the chosen AI tools, and ensure both teammates can run commands and exchange logs. Practice general Python and graph/decision concepts independently of an invented game SDK.

**Friday 16 October:** Rehearse the development loop on a small neutral exercise: change one behavior, run checks, inspect output, compare versions, and restore a working copy. Prepare the prompts below and a result-recording template.

**Saturday 17 October:** Charge laptop, pack charger, check AI access and editor, confirm arrival plan, and review the submission checklist. Obtain organizer answers to pre-event permission/logistics questions if available. Do not build a speculative engine and mistake it for the official competition.

### 17.2 Two-person division of work

| Person | Primary responsibilities | Coordination rule |
|---|---|---|
| Teammate A: implementation | Read SDK/engine, drive AI edits, own `my_bot.py`, explain decisions | Only one person edits the active bot at a time. |
| Teammate B: verification | Run checks and matches, inspect losses, record results, track time/submission | Test a labeled stable copy/version; report exact logs, not impressions. |

Both should understand the core policy. Tester recommends the highest-value fix while implementer handles the current one. Freeze a candidate before comparing it; never confuse results from different file contents.

### 17.3 Sunday schedule

The guide allows about 75 build minutes, 9:15–10:30. The following are internal targets; official event instructions take precedence.

| Time | Work | Exit condition |
|---|---|---|
| 8:45–9:00 | Report at Civil AV Auditorium; obtain/write down team ID; seat and power setup | Team ID and submission notes ready. |
| 9:00–9:15 | Download official kit; run baseline check/play; fix setup; read provided materials within announced rules | Official starter runs before contest coding begins. |
| 9:15–9:23 | Audit contract and decisive engine rules; run baseline conformance | Legal baseline saved; actual schema/phase assumptions documented. |
| 9:23–9:36 | Implement navigation, resource pickup, banking, legal fallback | V1/V2 passes check, play, and admission. |
| 9:36–9:48 | Add tags, selective melee, minimal Core and item handling; first varied-opponent sweep | A useful hybrid exists with logs showing real scoring. |
| 9:48–10:00 | Add one mobility ability and validated shop policy; compare limited variants; fix worst loss | Full conformance passes by approximately 10:00 at the latest. |
| 10:00–10:12 | Improve overtime/endgame and hazard timing; paired tests and held-out matchups | Final candidate chosen from evidence. |
| 10:12–10:20 | Feature freeze; final check/conformance, deterministic/runtime review, standalone-file check | Exact submission file is verified. |
| 10:20–10:25 | Prepare/send one email with tested `my_bot.py`, team name, team ID | Submission sent before official deadline; retain sent record. |
| 10:25–10:30 | Buffer for confirmed delivery/organizer instructions | No uncontrolled last-minute replacement. |
| 10:30 | Official code freeze | Stop code edits and all human/AI intervention in bot play. |
| 10:45–11:00 onward | Organizer checks and tournament start | Watch autonomous games; results remain tentative. |
| Saturday 24 October, 9:00 onward | Closing ceremony, Main Auditorium | Final results, awards, and prizes. |

If behind schedule, cut stretch features rather than final admission or submission time. If the latest candidate fails near freeze, restore the known-good file and verify that exact file again. If an email replacement is ever needed, follow the organizer’s confirmed procedure rather than sending ambiguous competing versions.

### 17.4 Practical failure recovery

- **AI access fails:** Continue from the last tested bot with local editor/terminal; use the prepared rules and prompts as reference. No runtime dependence on the AI service.
- **Environment fails:** Seek organizer help during setup; avoid speculative dependency installs since the official kit needs none.
- **A change crashes:** Capture full output, isolate one cause, fix/retest or revert.
- **Tests run slowly:** Reduce candidate count and prioritize admission, the failing scenario, and a compact varied-opponent set.
- **No clear strategic winner:** Submit the simplest passing candidate with the strongest robust placement evidence.
- **Submission friction:** Use the early buffer and official instructions; the deadline is a harder constraint than another optimization.

## 18. Ready-to-use AI prompts for the build session

Use the official files as context. Ask for one bounded change and exact test evidence. These prompts adapt the guide’s understand/design/implement/fix/compare/admit/explain workflow (pp. 23–24).

### A. Repository audit

> Read `docs/RULES.md`, `docs/BOT_SDK_CONTRACT.md`, the starter `my_bot.py`, and the relevant SDK/engine code. Do not edit yet. Map the actual view and shop schemas, legal actions, geometry helpers, lifecycle, and deterministic/runtime restrictions. Trace movement, melee, projectiles, blasts, collection, banking, Core, defuse, and overtime ordering. Compare them with `plan.md`; resolve section 20 questions from source where possible. Cite actual source locations. Do not invent missing API fields or helpers.

### B. Minimum competitive bot

> Edit only `my_bot.py`. Preserve `MyBot` and Python 3.9 compatibility. Implement the confirmed legal fallback, cached wall-aware navigation, capacity-aware resource collection, and timely banking. Use verified SDK geometry. Handle empty observations and lifecycle resets. Keep all helpers in this file and all actions behind legality validation. Do not add live AI calls, external dependencies, clocks, file I/O, or unseeded randomness. Run `check.py`, `conformance.py`, and supported play tests; report exactly what ran and what remains untested.

### C. One tactical improvement

> The current tested version has this specific failure: [seed, opponents, corner if available, full relevant log]. Expected behavior: [concrete expectation]. Explain the cause briefly and make the smallest change that addresses it while preserving all legality checks. Re-run admission and compare on the same scenarios. Do not change unrelated scoring or navigation policies.

### D. Controlled comparison

> Compare these two versions using the same seeds, corners, and opponent groups. Prioritize placement rank points and series advancement where supported. Show per-opponent results, fourth-place rate, failures, and score-source changes. Distinguish observed improvements from noisy differences. Recommend retaining the old version if the evidence is weak.

### E. Final admission and review

> Freeze features. Audit the exact `my_bot.py` we intend to submit for allowed imports/calls, deterministic decisions, legal actions in every branch, initialization and buying latency, single-file completeness, and Python 3.9 compatibility. Run the official quick check and conformance test. Reproduce any outstanding failure or revert to the known-good file. Give a concise pass/fail record; do not claim unrun tests passed.

### F. Teammate handoff

> Explain our bot’s current policy in execution order: what each rule detects, what action it can return, how that action is validated, and which scoring opportunity it targets. Identify the few tunable constants and the test evidence behind them. Keep this explanation consistent with the current file, not an earlier design.

## 19. Completion criteria and source coverage

### 19.1 Definition of a submission-ready bot

- [ ] Official kit version and actual SDK/engine rules have been inspected.
- [ ] `my_bot.py` alone contains `MyBot`, configuration, and every required helper.
- [ ] No disallowed imports, runtime external calls/files, nondeterministic choices, or timing-dependent decisions.
- [ ] `decide` and `buy` are bounded and comfortably within the 200 ms limit in measured cases.
- [ ] Official `check.py` and `conformance.py` pass on the exact selected file.
- [ ] Movement uses correct geometry and recovers from blocked/contested routes.
- [ ] Cargo is actually banked; endgame routes finish before the match ends.
- [ ] Core, defuse, projectile, and blast timing used by the policy matches the engine.
- [ ] Role restrictions, item forms, energy, target range, and cooldowns are checked.
- [ ] Mixed-opponent and varied-map tests supplement default self-play.
- [ ] At least one held-out comparison supports the selected candidate, or the tested simpler fallback is retained.
- [ ] Known-good copies and concise test records exist locally; no extra runtime file is required.
- [ ] Both teammates can explain the main policy and current limitations.
- [ ] The attachment is the tested `my_bot.py`, not a backup or stale copy.
- [ ] Team name, team ID, official submission address, and deadline have been checked.
- [ ] One clear submission email is sent before 10:30, preferably by 10:25.

These checkboxes describe future implementation gates. They are intentionally unchecked: this preparation task has produced a plan, not a tested tournament bot.

### 19.2 Complete guide coverage map

| PDF pages | Guide content | Covered in plan |
|---|---|---|
| 1 | Event overview, team size, Python/AI, single-file build window | 1, 3, 17 |
| 2–3 | Preparation, reporting, starter kit, build, admission, submission, results | 3, 14, 17, 19 |
| 4–6 | Arena, objects, visibility, symmetry, themes/characters | 4.1, 4.6, 7 |
| 7–8 | Simultaneous actions, costs, legality, death/protection, marks | 3, 4.1–4.2, 10, 13 |
| 9–11 | All scoring, tags, cargo, Core, overtime, map tie-break | 2, 4.3–4.4, 8–10, 12 |
| 12–14 | Abilities, blast timing, items, legendary drops, events, taunts | 4.5–4.6, 11 |
| 15–16 | Roles, full shop, credits, buying, cross-map carryover | 5, 13–16 |
| 17–18 | Tournament format, exact tie-breaks, failures, tentative results | 2–3, 16–17 |
| 19–22 | Kit files, AI workflows, build loop, commands, restrictions, geometry, common mistakes | 3, 7, 13–16 |
| 23–24 | Prompt templates, debugging/comparison, backups, teammate workflow | 15–18 |
| 25 | Numeric cheat sheet and engine authority | 1–5, 14, 20 |
| 26 | FAQ: tools, install, teams, kit/ID, admission, submission, hidden information | 1–3, 14, 17, 19–20 |
| 27 | Registration reminder and event venue/date | 1, 17, 20 |

## 20. Ambiguities, open questions, and how to resolve them

This is the consolidated uncertainty list. Earlier strategic proposals remain hypotheses until tested. Resolve implementation questions from the official repository first; ask organizers about permissions, format, or unresolved contradictions. Record answers with file/symbol references or organizer confirmation, then update the relevant plan sections.

### 20.1 Priority 0: admission and action contract

| Question | Why it matters | Resolution / interim approach |
|---|---|---|
| What is the exact `view`/`shop` schema, including types, optional fields, IDs, turn numbering, and status values? | A good policy can still fail on a missing or wrongly typed field. | Read SDK contract and fixtures; do not implement against guessed fields. |
| Is `decide()` called while dead/inactive, and what action is accepted then? Is there a sanctioned idle action? | Fallback legality; no documented zero-cost action appears in the guide. | Trace caller and validation. Never invent wait/no-op or assume Defend is always affordable. |
| At what phase does energy regenerate, and what minimum energy is guaranteed at a decision? | Determines fallback and repeated-action legality. | Read turn loop and test boundary cases. |
| What does determinism mean with allowed `self` memory: identical view alone, or identical view plus history/state? | The guide permits memory and seeded randomness but also says same input gives same action. | Read conformance replay logic. Use stable state updates and no randomness until understood. |
| Which modules/calls are allowed beyond the example blocked list? Does initialization count toward timeout? | Admission and first-call latency. | Inspect checker and runner; use the smallest necessary import set. |
| Does a bad or timed-out `buy()` merely buy nothing, or can it also fail admission? | The shop section describes forgiving runtime behavior; the FAQ describes strict timeouts. | Inspect conformance and distinguish runtime recovery from admission policy. Treat both entry points as strictly bounded. |
| What are the exact signatures and semantics of `step_open`, `line_clear`, `shot_path`, and action constructors? | Prevents geometry mismatch and malformed actions. | Read SDK implementations/examples before use. |
| Are ability destinations checked against pre-action occupancy, post-action occupancy, or only visible information? What happens with an unseen occupying bot? | A Blink/Dash into fog must not create an unavoidable illegal action. | Trace validator/resolver. Disable uncertain ability uses until their failure mode is known. |
| What happens for swaps, movement cycles, conflicting pushes, and move-vs-ability collisions? | Path prediction and legality. | Read simultaneous movement resolution; use conservative ordinary movement first. |
| Can abilities change between maps through supported selection, or only the class setting? | Determines whether runtime role adaptation can also change ability. | Keep fixed `ABILITY` per candidate unless contract explicitly allows more. |

### 20.2 Priority 1: phase order, geometry, and objective timing

| Question | Why it matters | Resolution / interim approach |
|---|---|---|
| Exact order of turn-start blasts, regen, decisions, healing, defense, melee, movement, projectiles, collection, tags, banking, plants, defuses, cooldowns, and scoring? | Defines survival and every multi-turn deadline. | Trace the authoritative turn function and write a short phase diagram. |
| Is the final decision turn 39, and which effects still resolve after it? Does scoring exactly at turn 32 receive overtime multipliers? | Avoid stranded cargo and late unpaid Core/blast plans. | Test transitions 31→32 and the final turn directly. |
| Do plant and first adjacent arrival count immediately toward their countdowns? What wins if uplink and defuse complete together? | Determines whether an interception is feasible. | Use focused engine tests and logs, not verbal countdown intuition. |
| Does each defuser need individual consecutive adjacency, or is defuse progress shared? Is the +40 pool split, or does each participant get +40? How does Warden’s +20 apply? | Defuse value and multi-bot cooperation/competition. | Read defuse state and award logic. Do not assume every participant gets the full amount. |
| Can walls obstruct adjacency for melee, defuse, Guard radius, or hot pickups? Are explosion radii Chebyshev, Euclidean, Manhattan, or engine-specific; do walls block blasts? | Distance alone may be insufficient. | Use actual validators and hazard area functions. |
| How do simultaneous lethal hits, healing, shields, spawn protection, and item use resolve? | Determines “guaranteed kill” and “heal to survive.” | Trace damage and protection phases; only claim guarantees after verification. |
| Are bombs globally visible or only nearby? What hazard metadata is exposed, and how do chain reactions interact with damage mitigation? | Hazard model completeness. | Follow the view schema; treat unobserved hazards as unknown. |
| How are countdown/cooldown counters decremented; do Runner and Overclock stack exactly as implied? | Determines earliest legal reuse and mobility plans. | Verify effective cooldown and zero/ready boundary. |

### 20.3 Priority 1: score, resources, and items

| Question | Why it matters | Resolution / interim approach |
|---|---|---|
| Resource values, unit counts, wave times, expiry, placement rules, and dropped-pile capacity/partial collection behavior? | Cargo profitability cannot be calculated from the PDF alone. | Extract constants and collect logic; use supplied values dynamically. |
| Can cargo and Core be carried together? Does Core consume capacity? How are multiple collectibles on one cell prioritized? | Hybrid route feasibility and correct collect IDs. | Read inventory and collection contract. |
| Does automatic banking happen before or after a resource collected on base? | May change one-turn routes and overtime payment. | Verify event ordering and a base-cell fixture. |
| How are hot pickup, Hauler multiplier, role penalties, overtime, and rounding composed? | Exact expected banking value. | Follow integer/float award code and score logs; do not multiply rounded subtotals speculatively. |
| Which bonuses survive repeat-kill suppression? How are shared-kill tags owned and awarded? Does assist overlap with shared kill? | Prevents exaggerated combat values. | Inspect kill attribution, tag ownership, and score reasons. |
| Does Reaper double only base immediate kill points, tags, or also bonuses/role rewards; how does it stack with overtime? | Late-game combat value. | Read actual multiplier application before estimating. |
| Does Overshield save one lethal hit then disappear, or have other per-hit/duration behavior? | Survival prediction. | Treat as no stronger than verified behavior; test multiple hits. |
| How are Hunter Rampage, On Fire, streak-three radar, and streak-five airstrike evaluated relative to the current/shared/multi-kill? Does the new airstrike replace an item? | Score and inventory state transitions. | Read streak update/award code. |
| How are crown ties handled, including the previous winner’s initial crown? What fields expose nemesis, streaks, protection, or bonus eligibility? | Target values and early threat. | Use only supplied metadata; no hidden-state introspection. |
| Which item types appear in `item_pool`, including legendary types and shop-kit exceptions? | Avoid rejecting a real held item because it is outside an ordinary spawn pool. | Separate held-item validity from random crate availability using the contract. |
| Are item draws/damage rolls seeded by the match, and does taking different actions alter future random draws? | Paired tests may diverge despite the same map seed. | Compare outcomes over multiple seeds; do not assume identical future randomness. |

### 20.4 Priority 2: tournament visibility and empirical strategy

- Does the bot receive current round, total maps, series standings, or qualification line, or only map number and our placement history in `buy()`? Use a general map-placement policy until additional information is confirmed.
- Does the same object survive across maps, and does `buy()` run on the same instance as `decide()`? Is any opponent-specific history legitimately supplied? Do not rely on memory beyond the guide’s per-match guarantee.
- Does the supplied harness support full series, corner control, mixed opponent groups, deterministic replays, latency reports, and machine-readable summaries? Inspect capabilities before planning automation around them.
- How frequently do resources and Core routes make Hauler, Runner, Warden, or neutral builds superior? Which ability actually saves the most productive turns across generated maps? Resolve by paired evaluation; there is no pre-repository answer.
- Are economic gains from Pockets larger than survival gains from Armor/Medpack? Do Scope, Quick Respawn, and Overclock earn their prices in the chosen policy? Measure marginal benefit rather than assume.
- Which risks dominate in real opponents: aggressive melee, Core rush, farming, pad defense, or collisions? Keep the policy robust to all; do not overfit the first weak opponent.
- What latency margin is adequate on the tournament machine? Ask for hardware/runtime information if available, while retaining strict deterministic work bounds.

### 20.5 Organizer and team questions

1. **Preparation rules:** Are prewritten bot code, reusable helper snippets, local test opponents, offline simulations, and prepared AI prompts allowed? The guide permits AI during the build but does not explicitly authorize importing prebuilt competition code. Until clarified, prepare understanding, tools, and this plan; do competition-specific implementation during the official window.
2. **Setup window:** Between 9:00 and 9:15, may teams inspect rules/source and run admission in addition to the explicitly instructed check/play, or is there a narrower setup-only policy? Follow announcements; no early contest edits.
3. **Official repository/version:** What URL/version is authoritative, and how are last-minute corrections announced? Use the QR release and preserve its exact version.
4. **Submission:** What email address, subject format, attachment/name limits, acknowledgment process, replacement policy, and deadline clock apply? The PDF gives content requirements but not the address.
5. **Final tournament format:** Does turnout change group size, map count, advancement, or round structure? Confirm before interpreting series tests.
6. **Registration and team identity:** Is registration complete for both teammates, what team name will be used, and who is typist/tester? Team ID is obtained on reporting day.
7. **Venue logistics:** Confirm reporting at Civil AV Auditorium on Sunday 18 October and closing ceremony at Main Auditorium on Saturday 24 October, 9:00 onward, against any organizer update.
8. **Testing permissions:** Are development-only extra test files/bots permitted, given the guide says only `my_bot.py` is edited and submitted? Use supplied tools first and do not modify engine, SDK, admission checks, or official opponents.
9. **Post-match review:** Is there any additional code-review policy beyond the PDF and SDK contract? Follow the published rules; seek clarification on contradictions rather than exploiting them.

None of these unanswered questions requires guessing a GitHub repository or treating this plan as a tested implementation. Once the kit arrives, resolve the Priority 0 items first, establish a passing baseline, and promote only improvements supported by actual results.
