# Designing AI for Strategy Games Through Modding

Three case studies in chronological order, each representing a different approach to the same fundamental challenge: making strategy game AI competent within the constraints of modding.

1. **Anbeeld's Custom AI** -- Stellaris 2.1.3
2. **ARoAI** (Anbeeld's Revision of AI) -- Victoria 3 1.3.6
3. **AI in Imperator: Invictus** -- Imperator: Rome (patches 1.9.1 through 1.10.1)

---

## 1. The Problem Space

### 1.1 What "AI" means in a strategy game

Not machine learning -- it's decision-making under constraints. There's no training data, no gradient descent, no neural network. The AI is a set of rules that execute every game tick.

The AI is an actor: it plays the game through the same interface a human uses, but via code. It reads the game state, evaluates options, and issues commands -- build this building, declare this war, research this technology, raise these armies.

The key challenge: the AI has access to the same (or fewer) levers as the player, but must decide without intuition or improvisation. A human player can glance at the map and "feel" that their economy is unbalanced. The AI must derive that conclusion from numbers, thresholds, and flags -- and the scripting language may not even expose the numbers it needs.

The goal isn't to make the AI play optimally. It's to make the AI play coherently -- all its subsystems pulling in the same direction, producing behavior that looks intentional rather than random.

### 1.2 Why vanilla AI fails in Paradox games

Paradox games are deeply systemic: economy, diplomacy, military, internal politics all interact. A decision in one domain cascades through others -- building a factory affects employment, which affects political support, which affects what laws you can pass, which affects what factories you can build.

Vanilla AI is split between hardcoded C++ (can't mod) and data weights (can mod but limited). The C++ layer handles core decision loops -- when to build, how to set taxes, how to spend money. Modders can't touch this. The data layer -- AI weights on buildings, technologies, production methods -- is moddable but limited in expressiveness.

Common failures are observable and specific: builds randomly without regard to market conditions, ignores shortages until the economy collapses, fleets don't repair after battles, transports get stuck with invalid orders, wars stagnate into decades-long stalemates, empires bankrupt themselves building defense platforms they can't afford, tribes never enact laws or reform their government, AI countries never build roads or manage character loyalty.

The AI doesn't fail because it's "dumb" -- it fails because it doesn't have a plan. Individual decisions may be acceptable in isolation, but they don't form a coherent strategy. The building AI doesn't know what the military AI is doing. The economy AI doesn't coordinate with the diplomacy AI. Each subsystem operates in its own bubble, and the result is an empire that stumbles from one crisis to the next.

### 1.3 The modder's constraint: scripting languages not built for AI

Clausewitz/Jomini script: no arrays, no structs, no loops over dynamic ranges, no runtime introspection of game data. You can't write `for each building in country`. You must explicitly enumerate every case: "if this building, then check this; if that building, then check that."

Everything must be expressed as events, triggers, effects, and static data. Events fire on schedules or conditions. Triggers are boolean tests. Effects change game state. Static data defines weights and properties. That's the entire vocabulary.

Performance is a first-class constraint: script runs on the game thread, every empire every tick. A poorly designed AI mod can halve the game speed. This isn't a theoretical concern -- it's the reason every design choice in these mods includes performance budgeting. Time distribution, idle periods, selective data collection, and data packing all exist because of this constraint.

The language shapes the design as much as the game does. You don't design the ideal AI and then implement it in script. You design an AI that script can express efficiently. The best solution in pseudocode may be impossible or prohibitively expensive in Clausewitz.

### 1.4 Data accessibility as the binding constraint

More fundamental than performance: the AI can only decide based on what the script can read. If the scripting language can't query a piece of game state, no amount of clever logic can use it. This is the single most important constraint across all three projects.

**Victoria 3:** The scripting language cannot query what goods a building type produces, what technology it requires, or what its input/output ratios are. The AI literally cannot look at a building definition and figure out what it does. Every building's properties must be manually hardcoded as scripted triggers and effects -- 49 buildings, each with separate evaluation, sanction, and allocation logic, plus 3,000 generated script values for data extraction. When a modder adds new buildings, those buildings are invisible to the AI unless someone writes a compatibility patch.

**Imperator:** Before patch 2.0.5, the script couldn't check research efficiency. The AI couldn't know if a country was at 40% or 175% efficiency, so there was no way to tell it "stop building research buildings, you've overcapped." When `modifier:` syntax was added, a previously impossible improvement became straightforward. Similarly, governor policy selection and mercenary budget management became possible only after modifier values were exposed.

**Stellaris:** The script can't read the vanilla ship build queue, so the mod emulates queue occupancy with six busy-slot flags per starbase -- each flag represents one occupied shipyard slot, set when a ship is queued and cleared when delivery completes. It can't detect fleet damage precisely, so repair decisions use coarse health thresholds that get stricter with distance to the repair starbase. Even the fleet upkeep calculation requires estimation: the economy evaluation subtracts an approximated fleet maintenance cost because the script can't query actual upkeep directly.

This shapes every design: the quality of AI decisions is bounded by the data accessible to the script. You can't make a smart decision about something you can't observe. When new engine capabilities expose new data, entire categories of improvement become possible overnight -- and the modder who maintains a mental "wish list" of data-blocked improvements can act immediately when new access appears.



## 2. Design Philosophy: Universal Principles

These principles emerge across all three projects. They are game-agnostic and represent the core design logic underlying every specific decision.

### 2.1 Quantize the world into stable decisions

Don't operate on raw numbers -- convert continuous state into categorical breakpoints.

- "Energy low / medium / high" instead of checking `energy_income > 47.3`
- Fragile arithmetic chains break in scripting languages; threshold-based decisions are robust and composable. A rounding error or off-by-one in a raw comparison can flip a decision permanently. A quantized flag only changes when the underlying value crosses a defined boundary.
- A shared vocabulary of quantized state ("energy low", "food medium", "navy above delete threshold") lets independent subsystems make coherent decisions without tight coupling. The building planner and the fleet producer don't need to agree on what "rich" means in absolute terms -- they just both check `minerals_income_extreme` and respond accordingly.

**Stellaris example:** The economy evaluation turns stockpiles and incomes into boolean flags (`aai_boolean_energy_income_low`, `acai_boolean_minerals_income_extreme`). Every downstream system -- buildings, ships, colonization, robots -- reads these same flags. Mineral income is quantized in 5-unit steps from 5 to 100+. Energy stock is computed as production × 6 divided by income thresholds. The building planner, colony reserve system, robot logic, and ship logic are not all inventing different ideas of "rich" or "poor" -- they all read the same derived flags. Even population is quantized into scaling tiers: empires under 200 pops use one formula, larger empires use another with different divisors and compensation offsets.

**Victoria 3 example:** Budget health score (-3 to +3) uses two-dimensional curves where the surplus requirement varies with debt level and vice versa. Negative health levels require both `weeks_of_reserves < 156` (~3 years) AND either a surplus or debt threshold. Reserves act as override (156+ weeks blocks all negative health regardless of other factors) and gate (positive health levels can be achieved through either sufficient surplus or sufficient reserves -- 156 weeks for health 0, 208 for +1, 260 for +2, 312 for +3). Debt-free countries get easier thresholds that scale with how full their gold reserves are. This creates smooth degradation rather than cliff-edge behavior where a tiny change flips the AI from healthy to crisis.

### 2.2 Distribute computation over time

Never run everything on one pulse -- stagger work across game days.

- Prevents frame drops and performance spikes that make the game feel sluggish
- Prevents synchronized AI behavior (all empires acting identically on the same tick, creating obvious "wave" patterns visible to the player)
- Creates a "persistent planner" feel instead of a "monthly panic script" where everything happens at once and then the AI goes dormant

**Stellaris:** Economy on days 1-7, ships 8-15, robots 16-19, buildings in separate waves for normal planets, upgrades/replacements, and habitats. Monthly and yearly cadences for different systems based on how often they need updating. Yearly systems include pop/food recalculation, civilian ship production, army production, starbase specialization, and naval capacity -- things that don't change fast enough to justify monthly processing.

**Victoria 3:** 28-day iteration cycle per country (minimum 14), with randomized start dates so countries don't all compute simultaneously. Day 1 = preparation (data collection, tax/wage management, downsizing, tech redirection), Day 2 = evaluation (priority assignment for every building type), Days 3-14 = construction (one building type per day, up to 12 types per iteration), Days 15-28 = deliberately idle buffer. A separate weekly loop runs 4x per iteration for budget tracking, construction progress monitoring, investment pool calculations, and cost-per-construction-point updates that need faster refresh than the main cycle.

**Why idle time matters:** If the game engine lags or events fire late, there's a buffer before the next iteration begins. Days 15-28 being deliberately empty means a delayed Day 14 doesn't cascade into the next cycle. This is defensive engineering against the unpredictable runtime environment -- the mod can't control when the engine actually fires its events.

### 2.3 Active planner + static override alignment

A scripted event planner alone will fight the vanilla AI's own logic. A weight rewrite alone can't fix dynamic failures. The event layer and the data layer must agree, or the AI contradicts itself.

- Do both: the event layer decides strategy, the data layer makes the vanilla engine agree
- This is probably the single most important architectural principle across all three mods
- Without alignment, you get observable failures: the event system builds corvettes while the vanilla ship designer picks battleship components, or the construction planner queues buildings while the vanilla AI queues conflicting ones

**Stellaris:** Ship doctrine flags (corvette-preference or battleship-preference) are assigned at game start and cascade through every layer they touch. Tech weights discourage chasing the wrong hull sizes. Component weights favor afterburners (speed/evasion) for corvette-pref, artillery computers and fire-control for battleship-pref. Ship section layouts are narrowed to proven configurations -- torpedo corvettes for corvette doctrine, artillery battleships for battleship doctrine. The runtime script decides the doctrine; the static data overrides (40+ tech/component checks reference the doctrine flag) make the vanilla ship designer, tech selector, and country defaults all support it. Destroyer and cruiser build paths exist in the code but are commented out -- the mod intentionally narrows to two viable strategies.

**Victoria 3:** Production method ai_values guide vanilla PM selection with clear progression (50,000 → 100,000 → 200,000 → 400,000) while the mod's own event system handles construction. Three defines disable vanilla construction, tax management, and government spending AI entirely (`PRODUCTION_BUILDING_CONSTRUCTION_ENABLED`, `TAX_LEVEL_CHANGES_ENABLED`, `GOVERNMENT_MONEY_SPENDING_ENABLED` all set to `no`), giving full control to the scripted system. Debt pause/resume thresholds set to unreachable values (6.66/7.77) so the vanilla debt management never fires.

**Imperator: Invictus:** Invention relative chance (IRC) values set high enough to actually overcome the randomized selection pool -- IRC 95% translates to weight 361, which is 35x more likely than an IRC 35% option at weight 10.23. The mod's own scripts handle law management, building decisions, and economic policies that the vanilla AI barely touches, while the static IRC weights ensure the vanilla research system picks reasonable inventions.

### 2.4 Specialize over generalize

A narrow competent AI beats a broad confused one. Reduce the decision space to a smaller set of good patterns rather than trying to optimize across all possibilities.

**Stellaris:** Corvette-or-battleship doctrine (no mixed fleets, no destroyers or cruisers in active production); starbases are shipyard, anchorage, or trade hub (no hybrid layouts). Many ship sections and component alternatives are suppressed entirely -- the mod picks proven configurations and zeroes out the rest. Sector type weighting pushes all sectors to "financial" (weight 100), zeroing out balanced, agricultural, industrial, and research alternatives.

**Victoria 3:** Building classes with distinct evaluation approaches -- government buildings (administration, university, ports, military) use formula-based evaluation checking population, GDP, innovation targets, military threats, lost tax revenue, and spending shares. Production buildings (factories, farms, mines) use market supply/demand queries against actual trade data. Two fundamentally different methods, each suited to its domain. Within production buildings, a three-layer decision hierarchy separates concerns: weight controls urgency (which building gets built first -- added to the supply/demand level to produce the priority number), order breaks ties (encoding dependency chains so construction comes before administration comes before infrastructure), and offset gates expansion (how productive existing buildings must be before the AI builds more -- added to the supply/demand level to produce the productivity requirement). Three orthogonal levers, each tunable independently.

**Imperator: Invictus:** Building priorities structured as ratio buildings first (Academies for nobles, Courts of Law for citizens, Forums for freemen, Mills for slaves -- matched to territory culture rights and trade goods), then modifier buildings that enhance what's already there (Library, Training Camp, Market, Tax Office). Clear sequencing rather than open-ended choice. Once research efficiency is high enough (checkable after patch 2.0.5), the AI stops building Academies and Courts of Law and switches to Forums and Mills -- preventing the observed failure where small AI countries filled every city with research buildings while starving for manpower and taxes.

### 2.5 Choose your relationship with vanilla: coexist, replace, or compensate

This is one of the first architectural decisions for any AI mod, and the three projects chose three different points on this spectrum. The choice isn't about technical preference -- it's driven by what the game allows and how badly the vanilla AI fails.

- **Coexist (Stellaris):** The mod runs alongside vanilla AI. Script events handle economy, fleets, starbases, and repair, while weight overrides push vanilla systems (tech selection, ship design, perk choices) in the right direction. Vanilla AI still runs -- the mod steers it rather than replacing it. The mod sets `ai_weight = 0` on starbase buildings it wants to place directly through script, effectively taking over placement while leaving the vanilla system nominally active. Lower maintenance burden, but the mod must constantly account for vanilla AI fighting its decisions.
- **Replace (Victoria 3):** Three defines disable vanilla construction, tax management, and government spending AI entirely. Debt pause/resume thresholds set to impossible values (6.66/7.77). The mod owns every economic decision -- from what to build, to where to build it, to how to set taxes and wages. This is the nuclear option: total freedom, total maintenance burden. Every edge case the vanilla AI would have handled is now the mod's responsibility. Every game patch can break everything. Even interest group promotion and suppression are disabled (`PROMOTION_BASE_VALUE = 0`, `SUPPRESSION_BASE_VALUE = 0`).
- **Compensate (Imperator):** Many critical systems (fort placement, ship building, unit movement) are hardcoded in C++ and cannot be replaced at all. The mod builds parallel systems that observe and correct the output of hardcoded logic. Fort placement can't be overridden, so the mod repositions forts after vanilla places them -- matching them with province capitals, prioritizing border provinces. Road building doesn't exist in vanilla at all, so the mod creates it from scratch with three phases: inter-region connections for levy delivery, dense province-level networks, then intra-province city-to-city roads. This is the lightest touch on what can be touched, combined with creative workarounds for what can't.

The right choice depends on the game: how much the vanilla AI exposes to modding, how badly the vanilla logic fails, and how complex the replacement would be. Victoria 3's economy AI is deeply broken and the scripting language is powerful enough (script values with inline calculations) to replace it. Imperator's economy is simpler and weight-based manipulation is viable, but fort/ship logic is locked behind C++. Stellaris sits in between -- its Clausewitz script is the most limited of the three, but the vanilla AI is functional enough to steer rather than replace.

### 2.6 Bypass broken systems by simulating their effects

When the vanilla AI makes bad decisions through a game system, one approach is to bypass the system entirely and apply the intended effects directly through script. Don't try to teach the AI to use the system better -- just give it the results when the conditions are met.

**Stellaris:** The mod sets `ai_weight = 0` on all real influence edicts, making the vanilla AI never cast them. Instead, `gai_edict.2` checks conditions and applies the equivalent modifiers directly. Capacity overload requires `tech_power_hub_1` and costs 300 influence. Production targets requires `tech_colonial_centralization` and costs 300 influence. Research focus requires materialist ethic and costs 300 influence. Healthcare campaigns require 3,000+ energy stockpile and 25+ energy/month income. Robot campaigns follow the same thresholds but only for machine intelligences. Recycling campaigns need 4,000+ energy. Enclave trade deals are also simulated: mineral trade modifiers at three tiers (2,400/6,000/12,000 energy thresholds), curator insight at 5,000 energy, artist patron equivalent at 5,000 energy. The AI gets the intended bonuses with correct timing, under script control instead of the base AI's unreliable heuristics.

**Imperator:** Economic policy management follows the same pattern. AI couldn't reliably choose between Free Trade and Trading Permits, so the mod implements the logic: check if the country has enough exports for Free Trade, otherwise use Trading Permits for extra capital import routes. Harsh Taxation (normally a player-only decision) is now used by AI when research efficiency is severely overcapped. Lax Tributes for subject integration. Each policy decision that the vanilla AI ignores or bungles is handled directly.

**Victoria 3:** The strike event override is a simpler case. The entire vanilla strike event chain (9 events -- strike initiation, negotiation, general strike, anti-strike measures, union popularity, police brutality, anti-union press, strikers appeased, strikers crushed) is overridden so AI always chooses to break strikes rather than negotiate (`negotiate=0, break=10` in the AI weighting). Negotiating creates economic promises the AI might not follow through on, leaving lingering negative modifiers. The override is blunt but prevents a specific, observed failure mode.

The pattern: if the AI can't use a system well, don't try to teach it -- just apply the results you want when the conditions are met. This trades elegance for reliability.

### 2.7 Target actual failure modes, not abstract optimization

Don't try to make the AI "optimal" -- make it stop breaking itself. Each practical intervention addresses a specific, observed self-destruction pattern that players actually notice and complain about.

**Stellaris:** Transport stuck-fix (detect invalid orders, retry, eventually delete and recreate as fresh transport fleet -- because the engine doesn't self-recover from certain invalid states). Strongest-fleet repair routing (detect damage on main fleet during war, find owned starbase at least starport tier, clear fleet orders, apply -1000 speed modifier to freeze it in place, apply +150% hull/armor regen to emulate repair -- then recheck after 30 days). Critical building enforcement (yearly scan of upgraded-capital colonies ensures unity buildings, growth buildings, paradise domes, power hubs, mineral processing, hive synapses, machine control centers all exist -- buildings vanilla AI "forgets" after initial construction). No-platforms modifier (poor empires with monthly energy < 45 AND minerals < 150 get -50 starbase defense capacity for 480 days, stopping them from wasting money on defense platforms while their economy starves).

**Victoria 3:** Stalemate war resolution (every 30 days, check all ongoing wars; for AI-vs-AI wars where both warleaders have 0 war support, advance a stalemate counter through 24 levels -- one per check = ~2 years -- then force resolution: secessionists win, revolutionary wars go to higher-population side, other wars get white peace). Building downsizing (buildings with occupancy < 40% for 6+ iterations are flagged as "abandoned" and removed, freeing construction capacity and budget). Strike event override (AI always breaks strikes to avoid lingering negative modifiers from failed negotiations).

**Imperator: Invictus:** Preventing civil wars through bribing and free hands (AI couldn't press the buttons before -- and worse, was silently cheating by not paying political influence costs for character interactions). Deleting excess ports that waste building slots (AI wants only 1-2 ports per state based on coastline, deletes extras). Farming settlements in states with many cities to prevent starvation. Moving province capitals away from weird post-annexation locations to better provinces. Fixing forts that protect useless land instead of borders by repositioning them to match province capitals and border territories.




## 3. Stellaris: Anbeeld's Custom AI

This was the first generation of the project, built for Stellaris 2.1.3 with its tile-based planet system.

### 3.1 Building on inherited work

The question of whether to start from scratch or build on existing work arises in any AI modding project, and the answer is almost always to build on top. The Stellaris mod was built on top of Glavius's AI Fix Mod. This is how most modding actually starts: not from a blank slate, but from whatever already exists. Glavius's code handled colonization (habitability checks, mineral reserve gating, colony shelter relocation to better tiles), critical building enforcement (yearly scans for missing unity/growth/infrastructure buildings), edict simulation (zeroing vanilla edict weights and applying equivalent modifiers through script), war pressure (scanning for weak neighbors and nudging peaceful empires toward conflict), and Rogue Servitors (bio-trophy growth and nutrient paste management). The new layer grew around it: economy model, fleet production, starbase planning, repair, robot assembly, habitat building.

This created two design eras in one mod -- different naming conventions (`gai_*` for legacy, `aai_*`/`acai_*` for new), different assumptions, overlapping responsibilities. Dormant code accumulated: destroyer preference flags checked in 40+ tech/component locations but never assigned, a fully-implemented sector tax simulation that's defined but never scheduled, a starbase defensive conversion system that's `is_triggered_only` with no caller, a debug logger that exists but isn't hooked up. It's messy, but it reflects a general truth about AI systems: you inherit decisions you didn't make, and you have to live with them or pay the cost of rewriting them. Usually you live with them.

In any strategy game AI project, the starting point shapes the architecture more than the plan does. Budget time for understanding inherited logic before extending it. And accept that archaeological layers will accumulate -- the cost of cleaning them up often exceeds the cost of working around them.

### 3.2 Giving every subsystem the same picture of the world

A strategy game AI has many subsystems -- building, fleet production, colonization, research, diplomacy. If each subsystem invents its own idea of "are we rich?" or "are we in danger?", they'll contradict each other. The building system might think the empire is poor while the fleet system thinks it's rich, because they check different variables with different thresholds.

The Stellaris mod solves this with a central economy calculator (`aai_calc_economy`) that converts raw stockpiles and incomes into a shared vocabulary of boolean-style flags: "energy low", "minerals extreme", "food medium." Mineral income is quantized in 5-unit steps from 5 to 100+. Energy stock is computed as production × 6 divided by income thresholds. Food income gets 26 discrete levels. Every downstream system -- planet building, habitat building, colonization, fleet production, robot assembly, civilian ships -- reads the same flags. No subsystem invents its own idea of the empire's economic state.

One subtle addition: the calculator subtracts estimated main-fleet upkeep when the fleet is parked at a friendly starbase, using fleet-upkeep deduction scaling that increases with game age (0.175 energy early game to 0.25+ late game, in 15-year brackets). Without this, the AI thinks it has more disposable income than it does and overbuilds during peacetime. Population-based calculations also feed into the shared model: mineral pressure thresholds scale with empire size -- low pressure at `(pops ÷ 2) + 10`, extreme at `(pops ÷ 4) + 5` -- and empire size itself uses different formulas above and below 200 pops to prevent small empires from getting distorted readings.

The lesson is to build a shared state model early. When you add a new subsystem, don't give it its own economic assessment -- make it read the existing one. Consistency between subsystems is more valuable than precision in any single one. And include deductions for committed resources (like fleet upkeep) so the model reflects disposable income, not gross income.

### 3.3 Taking over every productive decision

The vanilla AI makes acceptable individual decisions but they don't form a coherent strategy. It builds random buildings, colonizes at bad times, underuses robots, lets exploration stall, and forgets critical infrastructure. Rather than fixing each problem in isolation, the mod takes over essentially every productive activity: building on planets (reading tile yields to choose between food, energy, minerals, research, special deposits -- including Alien Pets requiring `tech_alien_life_studies` and Betharian power plants requiring `tech_mine_betharian`), building on habitats (simpler decision tree: capital first, solar power if energy low, agriculture if food low, then a weighted split of 45% mining bay and 30% research lab), upgrading buildings through full tech chains (mining/power/farms to tier 5, capital through all three tiers, research labs across all three science branches, unity chains per empire type), demolishing and replacing buildings during crises (including on habitats -- overbuilt food or energy modules are replaced when those incomes become high, and `building_junkheap` is automatically removed), enforcing critical support buildings yearly (unity, growth, paradise dome, power hub, mineral processing, purifier building, slave processing, hive synapse, machine control center -- buildings vanilla AI "forgets" after initial construction), handling Rogue Servitor bio-trophy mechanics (finding viable organic species, placing bio-trophies on food tiles, building nutrient paste facilities when food is too low), controlling colonization timing with a 300-mineral savings reserve (deposited in steps of 5 and spent from reserve instead of stockpile) and colony shelter relocation to better tiles (27 priority conditions for normal empires, 23 for machine intelligences -- prioritizing 4-adjacency energy tiles without research neighbors), keeping robot assembly active on suitable planets (picking the highest unlocked tier -- synthetic, droid, or robot, costing 100 minerals per assembly), spawning constructors and science ships on year-based targets (1 ship in years 0-5, 2 in years 5-10, 3 after year 10, at 100 minerals each), and managing the late-game megastructure progression.

The megastructure approach is selective, not uniformly aggressive. Perk weights push strongly toward Voidborn (10 → 100), Master Builders (10 → 100), and Galactic Wonders (10 → 200, but only after Voidborn is taken -- zero weight without it). The mod doesn't create habitat AI from nothing -- vanilla already had `ai_weight = 10` for habitats -- but it removes placement gates (the `exists = starbase` requirement is dropped and the foreign-border penalty changes from 0.1 to hard 0) and adds a suppression branch that reserves systems for larger wonder projects when the empire has Galactic Wonders. Think tanks are heavily promoted (10 → 100) with foreign-border hard-disabled. Spy orbs are heavily demoted (10 → 1) with foreign-border also hard-disabled. Gateways become reserve-dependent projects: vanilla's broad `factor = 5` model is replaced with `factor = 0` plus a single `factor = 500` case requiring 10,000+ minerals AND at least a starport-tier starbase. Ring worlds have the vanilla starfortress penalty removed (foreign-border stays at 0.1). Dyson spheres have the starfortress penalty removed AND foreign-border hard-disabled, pushing them toward safer rear-area systems. The real contribution is chain alignment: the empire is much more likely to unlock habitats via perk weighting, more willing to place them due to removed gates, and much less likely to leave them undeveloped afterward thanks to the dedicated habitat builder and monthly management pass that fills empty tiles, demolishes overbuilt modules, and upgrades through full capital chains.

Ships aren't just spawned -- they're created at a starbase with free shipyard capacity, then routed through a delayed delivery chain toward the main fleet (`aai_ships.2`, `.3`, `.20`, `.23`, `.40`, `.43`, `.60`, `.63` handle creation, routing, and arrival). If the route or destination fails, the build can be refunded. Six busy-slot flags per starbase (`acai_ships_busy_shipyards_1..6`) emulate queue occupancy to prevent over-queuing, since the script can't read the vanilla build queue. Battleships cost 2,000 minerals with a 480-day delay and reserve 8 naval capacity. Corvettes cost 140-320 minerals (year-scaled) with a 60-day delay and reserve 1 naval capacity.

The key insight isn't any single subsystem -- it's that all of them read the same economy flags and run on the same distributed scheduler. A coherent strategy emerges from many small decisions that share the same picture of the world.

Coherence comes from centralized state and uniform scheduling, not from making each subsystem individually smarter. If you're going to take over AI decision-making, take over enough that the decisions form a strategy, not a collection of unrelated fixes. And when you can't read vanilla systems (like the ship queue), emulate their state with your own tracking.

### 3.4 One high-level decision, propagated everywhere

In a game with many ship types, component options, and tech paths, the AI produces mixed fleets that are mediocre at everything. Tweaking individual choices doesn't help because the choices don't coordinate. The solution is to assign each empire a single fleet doctrine at game start -- corvette-preference or battleship-preference -- and propagate that decision through every layer it touches. Tech weights use `years_passed` and preference flags to discourage chasing the wrong hull sizes. Component preferences favor afterburners (speed/evasion) for corvette-pref, artillery computers and fire-control (damage) for battleship-pref. Ship section layouts are narrowed to a few proven configurations -- torpedo corvettes for corvette doctrine, artillery battleships for battleship doctrine. Ship production builds only corvettes or battleships -- no middle ground. Destroyer and cruiser production paths exist in the code but are commented out; the destroyer preference flag is actively removed on game start even though it's checked in 40+ locations. Naval capacity targets vary by doctrine, war state, and civic personality.

The naval capacity model reflects this specialization: default desired utilization runs from 0.60 (peace, minimal) to 1.10 (war, maximum). Fanatic militarists, regular militarists, genocidal civics, and machine assimilators shift upward. Inward Perfection, pacifists, and fanatic pacifists shift downward. Disbanding follows a strict hierarchy: corvettes first, then destroyers (if no corvettes), then cruisers, then battleships -- always trimming the smallest ships first.

One high-level decision, flowing through many layers, produces more coherent behavior than tweaking each layer independently. The mod gets competence by narrowing the choice space.

When the AI faces a combinatorial explosion of choices (unit types x equipment x tactics), don't try to optimize the full space. Pick a small number of viable strategies, assign one per AI player, and propagate it everywhere. Narrow competence beats broad confusion. The doctrine flag pattern -- one decision at game start, referenced in dozens of downstream systems -- is reusable in any game with complex military composition.

### 3.5 Classify first, then build

Starbases in Stellaris can serve many roles (shipyard, trade hub, anchorage, defense), but vanilla AI builds mixed layouts that are mediocre at everything -- a starbase with one shipyard module, one anchorage, one trading hub serves no role well. The mod separates *what role a starbase should have* from *how to fill that role* through a multi-pass system. First, `aai_calc_starbases` computes desired role targets from population, systems, and tech: shipyards start at 1 and gain +1 per 10 points of desired starbase limit (once limit ≥ 11); trading hubs start at 2 and gain +1 per 4 points of limit (once limit ≥ 7); anchorages absorb the remainder. Then `aai_starbases.1` assigns roles to existing starbases, preferring to reuse already-partial builds -- systems with colonies, colonization targets, or trader enclaves are suitable for trading hubs; colonizable-only systems are marked as potential future trading hubs; oversupplied roles are converted to anchorages. Then construction passes per role (`aai_starbases.2`, `.3`, `.4`) fill slots with role-appropriate modules and buildings. Outpost upgrading (`aai_starbases.5`) checks for hostile neighbors at 2-jump, 1-jump, then adjacent range, with different mineral thresholds and `ap_voidborn` as a reserve requirement.

When an entity can serve multiple roles but shouldn't mix them, decompose the decision into classification followed by execution. Compute the desired role distribution first, assign roles second, then build within those roles third. Two or three simple sequential decisions are more robust than one complex mixed decision. This applies to any strategy game with multi-purpose infrastructure.

### 3.6 Making AI players feel different from each other

If every AI player follows the same logic, the game feels like playing against clones. But adding randomness makes the AI erratic rather than distinctive. The mod uses civic personality to create systematic behavioral differences that persist throughout the game. Militarists and genocidal empires want more fleet relative to capacity in both war and peace -- their naval capacity targets are shifted upward. Pacifists maintain smaller fleets and trim aggressively -- their targets are shifted downward. Fleet doctrine (corvette vs battleship) creates lasting differences in military style that cascade through tech choices, component selection, and fleet composition. Army composition switches between organic, slave, psionic, clone, robotic armies based on tech and empire type. Expansion scoring uses higher randomness to create diverse patterns -- empires don't all grab the same systems.

The result: two empires with identical economies but different civics will build different amounts of different kinds of ships, research different tech paths, and expand differently. Personality is expressed through parameter shifts on the same underlying logic, not through separate code paths. The same naval capacity formula serves every empire -- the civics just shift the inputs.

The broader principle is to use the game's existing faction/personality system to parameterize AI behavior, not to branch it. Shifting thresholds and weights creates distinctive behavior that's still competent. Separate code paths per faction create maintenance nightmares and make it impossible to improve all AI players at once. One formula with per-faction parameter shifts scales better than N independent implementations.

### 3.7 Breaking mid-game stagnation

In many strategy games, the mid-game stagnates. Strong empires sit peacefully next to weak ones forever. Nobody declares war because the AI doesn't assess relative strength well or isn't aggressive enough. The galaxy freezes into a static political map within the first few decades and never changes. The mod directly combats this through both push and pull mechanisms. The pull: `gai_war.1` scans peaceful AI empires with sufficient strength and economy, rejects pacifists, federation members, subjects, and weak empires, then looks for weaker neighboring AI empires as war targets. War goals are selected based on civics -- conquest for normal empires, cleansing for purifiers, absorption for devourers, assimilation for assimilators, with special handling for awakened-empire domination and end-threat scenarios. The push: poor empires (monthly energy < 45 AND minerals < 150) get a `no_platforms` modifier for 480 days that reduces starbase defense capacity by 50, preventing them from wasting resources on defense platforms while their economy starves -- a common failure where the AI builds defenses it can't afford. Global defines make the base AI more expansionist (colony systems and resource systems score much higher), more claim-focused (max claim distance increased, colony systems valued more heavily), and more willing to operate with large fleets (transport size 20, arsenal size 200, minimum offensive fleet sizes raised).

AI passivity is one of the most common complaints in strategy games. Addressing it requires both pull (opportunity assessment for when to declare war -- scan for weak neighbors, check relative strength) and push (preventing defensive waste and economic stagnation -- stop the AI from spending money on things that make it weaker). The AI needs to be told not just *how* to fight but *when* fighting is a good idea. And poor empires need special protection from self-destructive spending patterns.

### 3.8 Making diplomacy strategic instead of sentimental

Vanilla diplomacy is dominated by inertia. Alliances and federations generate large flat opinion bonuses that make them self-sustaining regardless of whether they still make strategic sense. Trust accumulates quickly and caps high, so long-standing treaties become nearly unbreakable. The result: static alliance blocs that form early and never dissolve, even when the geopolitical situation has completely changed. The diplomatic map freezes just like the political one. The mod rewrites the diplomatic model around current strategic alignment instead of accumulated goodwill. It works through three coordinated changes:

First, passive relationship bonuses are stripped. Alliance opinion drops from +25 to 0, federation membership from +50 to 0, defensive pact from +20 to 0, ally-of-ally spillover from +25 to 0. Shared rival opinion is halved from +50 to +25 (it still matters, but as direct strategic calculation rather than generic friendship). Indirect diplomatic entanglements are reduced: allied-to-rival drops from -100 to -75, rivals-with-ally from -100 to -50. Gift diplomacy is also weakened: minimum gift size drops from 25 to 20, maximum from 50 to 40, and the opinion cap from gifts drops from 100 to 50. Existing treaties no longer generate their own justification for continuing.

Second, trust -- the long-term diplomatic glue -- accumulates much more slowly and caps much lower. Federation trust growth drops from 1.0/month to 0.25/month. Defensive pact trust from 0.75 to 0.25/month. NAP trust from 0.50 to 0.15/month. Association status from 0.50 to 0.20/month. Even guarantees drop from 0.25 to 0.10/month. Trust caps are also lowered: defensive pact from 100 to 75, associate from 100 to 50, NAP from 75 to 25, minimum trust from 50 to 25. Base trust change (the passive drift) doubles from -0.25 to -0.50, meaning unused relationships decay faster. The deal-breaking threshold tightens from -30 to -25, so bad deals are abandoned sooner.

Third, pact and federation acceptance is reweighted toward active strategic factors. For federations: shared rivals jump from 10 to 100, shared rival already in federation from 5 to 25, opinion factor from 0.10 to 0.30, shared threat factor from 0.25 to 0.40, relative strength factor from 5 to 20 (max from 20 to 200), conqueror policy mismatch from -50 to -75. For defensive pacts: shared rivals from 30 to 150, shared allies from 30 to 125, existing-pact penalty doubled from -50 to -100 (discouraging pact spam), attitude-alliance from 50 to 25, attitude-coexist from 20 to 0. For NAPs: shared rivals from 50 to 150, attitude-coexist raised from 50 to 75 (NAPs suit cautious empires), other-attitude penalized at -25 (was 0). Distance matters less across all pact types (-0.10 to -0.05), so strategic partnerships can survive range if the interests are strong enough.

Border friction grows stronger and more local. Max friction rises from 100 to 150. Each bordering system generates double the tension (5 → 10). Threat from planets (8 → 12), starbases (4 → 6), and systems (2 → 5) all increase. The high-threat threshold drops from 50 to 40, so AI classifies threats sooner. Fleet balance threat shifts from 0.5 to 0.7, meaning AI recognizes stronger neighbors as threats at a milder disadvantage. But threat also decays faster (-0.25 → -0.50) and scales more with distance (0.01 → 0.02), keeping it more local and more current. Shared-threat max drops from 200 to 50, preventing anti-threat alignment from creating huge permanent friendship blocs.

The combined effect: diplomacy becomes transactional. Empires form coalitions based on shared rivals and threats, not on the fact that they've been allied for fifty years. Alliance blocs dissolve when the threat that created them disappears. Nearby expansion creates sharp local tension that drives counter-coalitions. The diplomatic landscape stays dynamic through the mid and late game instead of freezing after the first few decades.

If a strategy game has stale diplomacy, look at where the relationship system generates self-reinforcing inertia. Flat bonuses from existing treaties ("we're allies, so we like each other, so we stay allies") are the main culprit. Shift acceptance logic toward current strategic conditions -- shared enemies, relative power, geographic proximity -- and reduce the passive bonuses that make old relationships permanent. Make threat local and decaying rather than global and permanent. Dynamic diplomacy comes from making every treaty continuously re-evaluated against present circumstances, not from accumulated history.

### 3.9 Duct tape for specific failure modes

Some AI failures aren't architectural -- they're specific mechanical breakdowns that ruin gameplay. Fleets don't repair. Transports get stuck. Events spawn useless entities. Edicts are misused. These aren't design problems -- they're bugs that need targeted patches.

- **Fleet repair:** Vanilla AI throws half-destroyed ships back into combat. The mod detects damage on the strongest fleet during war, searches for an owned starbase at least starport tier, uses damage thresholds that get stricter with distance (repair almost always when very damaged, tolerate less damage when repair target is close). At a suitable starbase without hostiles: clears orders, applies `acai_healing_locking_by_speed` (-1000 speed multiplier to freeze the fleet in place), applies `acai_healing_emulate_repairing` (+150% hull and armor regen). If not at the right system: routes the fleet toward repair and rechecks after 30 days. Not elegant -- but effective.
- **Transport stuck-fix:** Transports with land-army orders but no valid target system are detected. The mod retries for a while, then eventually deletes the broken transport fleet and recreates the army load as a fresh transport. During wartime, transport cleanup fires every 180 days. On entering war, all transports and military fleets are forced to return first (720-day timed flag tracks this).
- **Machine uprising override:** Vanilla uprisings spawn weak and die immediately. The mod gives them large resource stockpiles, applies capacity overload/production targets/research focus modifiers, flips capital and flagged systems, ensures ≥5 machine pops on flipped worlds, copies most host technologies (excluding biological/psionic/organic-specific), declares war on the host, and creates real fleets: exterminators get 40% naval cap, others get 30% plus a science ship and constructor. The uprising becomes an actual event instead of a footnote.
- **Edict simulation:** Rather than trusting vanilla edict heuristics, the mod zeros out all edict AI weights and applies equivalent modifiers directly through script. Each simulated edict has specific tech requirements, resource thresholds, and influence costs. This ensures the AI gets the right buffs at the right time without relying on the vanilla AI's unreliable decision-making about when to activate edicts.

Don't only design elegant systems. Also identify the specific, commonly observed ways the AI self-destructs and patch each one. Players notice the AI crashing and burning more than they notice optimal play. A targeted fix for a visible failure is worth more than a general optimization nobody sees. Patch the spectacular failures first, optimize later.



## 4. Victoria 3: ARoAI

### 4.1 Deciding to replace vanilla entirely

ARoAI represents a full-stack AI replacement -- the most ambitious approach of the three mods. The vanilla AI makes bad economic decisions, and unlike Stellaris where you can steer the AI with weight overrides, Victoria 3's construction logic is hidden in C++ with no hooks for modders. You can't fix what you can't reach, which raises the fundamental question: do you work around the edges, or take over completely?

ARoAI chose the nuclear option: three `NAI` defines disabled construction (`PRODUCTION_BUILDING_CONSTRUCTION_ENABLED`), tax management (`TAX_LEVEL_CHANGES_ENABLED`), and government spending (`GOVERNMENT_MONEY_SPENDING_ENABLED`). Debt pause/resume thresholds set to impossible values (6.66/7.77 for both normal and critical construction). Even interest group promotion and suppression are disabled (`PROMOTION_BASE_VALUE = 0`, `SUPPRESSION_BASE_VALUE = 0`). The mod owns every economic decision.

This is a fundamentally different approach from Stellaris (coexist) or Imperator (compensate). The motivation wasn't ambition -- it was necessity. Victoria 3's economy is complex enough that half-measures produce incoherent behavior. If you control building but not taxes, the AI builds things it can't afford. If you control taxes but not spending, the AI saves money it never uses. The consumption tax AI is the only vanilla economic behavior left -- and even that gets tuned via defines (`CONSUMPTION_TAX_LOW_INCOME_THRESHOLD = 3.0`, `HIGH = 5.0`, max taxed goods base = 6).

The trade-off is brutal: every game patch potentially breaks everything, and the mod must handle every edge case that vanilla would otherwise cover. 49 buildings, each with individually hardcoded evaluation, sanction, and allocation logic. Every building's properties manually encoded because the scripting language can't read building definitions.

When the vanilla AI's failures are systemic rather than local, partial fixes can make things worse by creating inconsistencies between the parts you control and the parts you don't. Sometimes full replacement is the only coherent option -- but go in with eyes open about the maintenance cost. And when you take over, take over completely: construction, taxes, spending, and debt management all need to agree, or you create new failure modes.

### 4.2 Using the best available signal for each domain

The AI must decide what to build, but the scripting language can't read building definitions. It can't look up what goods a building produces, what inputs it needs, or what technology it requires. The AI is essentially blind to the thing it's deciding about.

Two fundamentally different evaluation approaches, chosen by domain:

- **Production buildings** (factories, farms, mines): Query actual market supply vs demand. The supply/demand level runs from 1 (extreme shortage, no sell orders at all) through a 22-point scale to 99 (complete oversupply, zero demand). Levels 2-11 represent progressively milder shortage (buy orders still exceed sell orders). Levels 12-21 represent progressively deeper surplus. Level 22 catches residuals after clamping. The formulas use `BUY_SELL_DIFF_AT_MAX_FACTOR` to normalize the ratio -- level 1 represents >75% shortage, levels 12-21 map 0% to -75% surplus. This is the best signal available -- it captures the emergent state of each game's unique economy without needing to understand building internals.
- **Government buildings** (administration, university, ports, military): No market goods to query, so use predefined formulas based on population, GDP, innovation targets, military threats, lost tax revenue, spending shares. Each building type has its own formula with its own priority range: government administration 1-10, university 5-12, construction sector 1-8, railway only 4 discrete levels (1, 5, 9, 16), port 1-12, barracks 2-10, naval base 2-10.

The priority system (1 = highest, 0 = do not build) creates graceful degradation. Priority 1 is soft-reserved for critical government needs -- an emergency port when the market capital coastline has no ports but needs overseas connections, or infrastructure in critical deficit. Production buildings start at priority 2 (extreme shortage of a vital good) and range up through the scale. Critical shortages get immediate attention; moderate needs queue behind them; oversupply stops construction entirely.

When a building produces multiple goods (a logging camp makes wood and hardwood, a tea plantation produces tea, coffee, and wine), each good is evaluated independently and the most-needed good drives the building's priority. Secondary goods are evaluated conditionally -- military goods in shipyards and arms factories are deprioritized unless the country is actively at war or the shortage is severe. This prevents peacetime overbuilding of munitions while still responding to wartime demand.

Each good of a building has two separate parameters that serve different purposes:

- **Weight** controls **priority** -- which building gets built first. Added to the supply/demand level to produce the priority number. Lower weight = the AI treats shortages of this good more urgently. Weight families (`aroai_resource_weight_N`, `aroai_agriculture_weight_N`, `aroai_industry_weight_N`) allow grouped adjustment, with optional `_factor` variables floored at 1.
- **Offset** controls **expansion gating** -- how productive existing buildings must be before the AI expands them. Added to the supply/demand level to produce the productivity requirement (see section 4.3). Higher offset = stricter productivity bar.

These are independent: weight controls *whether* this building gets picked over other buildings, offset controls *how easily* it can expand once picked. This creates a three-layer decision hierarchy: weight decides what gets built first, order (see 4.6) breaks ties between buildings at the same priority, and offset gates expansion independently of both.

Primary goods (a building's core purpose) get low weight (1-4) and zero offset: the AI responds urgently to shortages AND the productivity bar is set purely by the market signal. Critical industrial inputs like tools, steel, coal, iron, and paper all get weight 1 -- maximum urgency. Secondary and luxury goods get high weight (5-11) AND high offset (4-6), creating a double barrier -- they rarely win on priority, and when they do, the productivity bar is raised. For example, a tea plantation evaluates tea (weight 5, offset 0) and coffee (weight 9, offset 4). If both have the same supply level, tea wins on priority and expands easily. But if tea is in surplus and coffee in shortage, coffee can win -- though it still faces a stricter productivity bar due to offset 4. A whaling station evaluates oil (weight 1, offset 0) as primary and meat (weight 10, offset 6) as secondary -- meat from whaling almost never drives construction.

This dual-lever design handles multi-good buildings elegantly. The same evaluation function handles both cases uniformly -- the weight and offset values create all the necessary differentiation without conditional logic.

The broader principle is to avoid forcing a single evaluation method across all decision types. Identify what signal best captures "should we do more of this?" for each domain, and use the strongest signal available. When a single entity serves multiple purposes, separate "how urgently should we build this?" from "how confident are we that expansion will succeed?" into independent parameters. This lets you express complex policies (e.g., "respond to luxury shortages only when existing production is already efficient") without branching logic.

### 4.3 Gating expansion on real-world performance

Knowing there's a shortage of iron doesn't mean you should build more iron mines. If your existing iron mines are unprofitable -- losing money per employee -- building more of them just accelerates the bleeding. The AI needs a way to distinguish "this good is needed" from "this good is needed *and* we can produce it efficiently."

The productivity requirement connects supply/demand urgency to actual building performance. The productivity level is calculated as `supply_vs_demand_level + offset` (the offset parameter from section 4.2), then mapped to a minimum earnings threshold. The system computes a country-wide median earnings figure -- what a decent building earns per employee -- and uses it as a baseline.

The thresholds form a smooth curve across 41 levels. During extreme shortage with zero offset (productivity level 1, a primary good in crisis), the bar is lenient: a building only needs to earn about 42% of the median to justify expansion. At levels 2-3, the bar rises to 54-62%. At levels 4-6 (significant shortage or primary good with small offset), 68-79%. At levels 7-10 (mild shortage plus moderate offset), 84-96%. At levels 11-12 (near-balance with offset 6), the bar essentially equals the median (100-103%). Above that, only buildings earning well above the median (up to 145% at levels 27+) get expanded -- you need to be demonstrably better than average to justify building more of something the market doesn't urgently need.

The offset parameter from 4.2 feeds directly into this: secondary goods with offset 4-6 start at a higher productivity level even during shortage, meaning they face a stricter bar. A primary good in moderate shortage (supply level 5) has productivity level 5, needing 74% of median earnings. The same shortage for a secondary good with offset 4 would have productivity level 9, needing 93%. This is the "double barrier" in action -- secondary goods are deprioritized by weight AND face stricter expansion requirements through offset.

The crucial discount prevents deadlocks. Each building has a `crucial` threshold (ranging from 5 for gold mines to 99 for railways and ports). When the average of supply/demand level and productivity level falls at or below the crucial threshold, productivity requirements are discounted: a 30% reduction (divide by 1.30) for very crucial shortages (half-average <= crucial), 15% (divide by 1.15) for moderately crucial (average <= crucial). This prevents the AI from refusing to build something it desperately needs because existing instances are underperforming due to the very shortage the new building would help resolve.

Construction gates at two levels. At the country level: if existing buildings of a type collectively meet the productivity threshold, the AI is allowed to build *new* instances in states that don't have this building yet (cell 4 = 1 in the packed data). At the state level: each individual building instance is checked for profitability and productivity, and only productive instances justify expansion. A coal mine that's struggling in one state won't trigger expansion there, even if coal mines elsewhere in the country are thriving.

The lesson is clear: don't let demand signals override reality. When the AI sees a shortage, it should check whether the existing production actually works before building more of it. Use a sliding threshold: desperate need lowers the quality bar, mild need raises it. Include an escape hatch (crucial discount) for goods so important that the AI must build them even when profitability is questionable. This prevents the AI from throwing good money after bad -- a common failure mode where the AI responds to shortage by building more of something that was never profitable in the first place.

### 4.4 Designing smooth degradation instead of cliff edges

The AI needs to know "how healthy is my economy?" to make spending decisions. Simple thresholds (surplus > 5% = healthy, surplus < 0 = crisis) create cliff-edge behavior where a tiny income change flips the AI between opposite strategies, causing oscillation.

Budget health (-3 to +3) uses two-dimensional threshold surfaces instead of simple cutoffs. Four inputs: scaled debt (0.0-1.0), budget surplus percentage, gold reserves percentage, and weeks of reserves (reserves divided by negative surplus, capped at 9,000; 0 if surplus is positive).

- **Negative health levels (-3, -2, -1):** All require `weeks_of_reserves < 156` (~3 years) as a gate. A country sitting on 3+ years of reserves cannot be in negative health regardless of current cash flow. Within the gate, each level uses OR logic between a surplus threshold and a debt threshold -- you can qualify for health -3 either by having a terrible surplus or by having extreme debt, and the thresholds for each scale with the other variable. At 75% debt, health -3 needs 11% surplus. At lower debt, the surplus requirement relaxes proportionally.
- **Reserves as override:** 156+ weeks of reserves blocks all negative health regardless of debt or surplus. A country with large gold reserves doesn't panic just because this month's budget is tight.
- **Positive health levels (0, +1, +2, +3):** Use OR logic between surplus thresholds and reserve requirements. Health 0 needs either 44% × debt as surplus, or 156 weeks of reserves. Health +1 needs either a stricter surplus formula or 208 weeks (~4 years). Health +2: 260 weeks (~5 years). Health +3: 312 weeks (~6 years).
- **Debt-free bonus:** Countries with zero debt have easier surplus thresholds that decrease as reserves fill. At 75% full reserves, health +2 requires effectively 0% surplus -- the reserves speak for themselves.

The asymmetric tax/wage adjustment logic reflects real constraints. When budget is healthy (health >= 0): keep taxes not very high (priority), raise government wages from very low to medium, then progressively lower taxes (very high → high → medium → low → very low), then raise wages further (medium → high → very high) when surplus allows. When budget is negative: raise taxes first (very low through very high), then cautiously lower wages (very high → high → medium → low). Wage cuts require interest group approval checks -- government wages need Intelligentsia and Petty Bourgeoisie support, military wages need Armed Forces approval. Military wages floor at medium during active wars -- you don't cut soldier pay during a fight.

Spending shares define how active income is divided: government administration gets 20% (plus up to 5% bonus from lost taxes, calculated as lost_taxes / 12.5% threshold × 0.05, clamped 0-1), university 10%, port 10%, military 35% (split between barracks and naval base by army/navy ratio). Construction sector gets the residual after all other shares plus the investment pool. An investment pool multiplier (1.0 + up to 0.20 based on private-sector construction funding) scales non-construction shares upward when investment pool covers part of construction costs.

Country-specific military spending multipliers apply before 1870: Egypt gets 2.5x before 1850 declining to 1.0x by 1870, Turkey and Prussia get 1.5x declining to 1.0x. This reflects historical military buildup priorities for these countries without requiring separate code paths.

When converting continuous economic state into decision categories, use overlapping multi-dimensional curves rather than simple thresholds. Include override paths (large reserves = not in crisis regardless of cash flow), asymmetric adjustment logic (raising taxes is faster than cutting wages), and spending shares that adapt to external funding. The goal is smooth degradation: the AI should gradually tighten its belt, not alternate between spending sprees and panic.

### 4.5 Deciding where to build, not just what

"Build more iron mines" is only half the answer. *Where* to build them matters -- wrong location means poor infrastructure, no workforce, wasted capacity. But optimizing location perfectly means the AI always builds in the same "best" state, creating unrealistic geographic concentration.

State aptitude scoring varies by building type: government administration uses 4 levels based on tax capacity balance and lost-tax severity, railway uses 7 levels based on infrastructure balance (level 1 at balance < 0.60 through level 7 at balance >= 1.10), barracks and naval base use 2 levels (non-discriminated homelanders preferred), resource buildings use 2 levels (protecting critical resource potential -- rubber, oil, coal, iron, sulfur, gold), agriculture buildings use 2 levels (protecting luxury crop potential -- tea, coffee, dye, tobacco, opium, sugar, silk), and industry buildings use 1 level (all states equally valid).

The resource and agriculture protection is a strategic twist: if a state has potential for critical resources or luxury crops, building a non-critical good there is penalized to aptitude level 2 -- it would "waste" the state's strategic potential for a future critical building. This prevents the AI from filling mining states with gold mines when iron deposits are still available, or planting grain farms where tea plantations could go.

The key design choice: random selection *within* tiers. States at the same aptitude level are treated as interchangeable, so the AI builds in different places each game. This trades a small amount of optimality for geographic diversity that makes each playthrough feel different.

Branching adds a second dimension: within each aptitude level, states are further filtered into 4 branches based on incorporation status, infrastructure headroom, and workforce availability. The interleaving order is deliberate: A1B1, A2B1, A1B2, A2B2, A1B3, A2B3, A1B4, A2B4. Branch takes priority over aptitude. An aptitude-2 state with branch 1 conditions (incorporated, good infrastructure, available workforce) is built before an aptitude-1 state with branch 4 conditions (fallback). This means "right conditions in a decent location" beats "great location with wrong conditions" -- a pragmatic choice that prevents the AI from building in critical-need states that lack the infrastructure to support the building.

Final eligibility checks add hard gates: the state must be owned by the country, workforce must be available (at least 5,000 unless the building's workforce flag is 0 or the good is crucial), and agriculture buildings require free arable land.

For location decisions, tier-then-randomize beats both pure ranking (always picks the same "best" spot) and pure randomization (ignores quality entirely). Group candidates into quality tiers, then choose randomly within the top viable tier. When the entity has infrastructure requirements, let conditions (workforce, infrastructure, incorporation status) override raw need -- a building that works in the second-best location outperforms one that fails in the best location. And protect strategic locations from being consumed by low-value uses.

### 4.6 Solving the cold start: implicit bootstrapping through build order

Victoria 3 countries start with minimal industry. What do you build first? Building consumer goods before construction capacity means you can never scale up. Building military before raw materials means you can't supply your army. The wrong sequence cascades into shortages that take decades to fix.

This is where the third layer of the decision hierarchy comes in. Weight (section 4.2) determines which building type has the highest priority. Offset (4.2/4.3) gates whether expansion is justified. But when multiple building types end up at the same priority -- which happens regularly, especially when several goods are in similar shortage -- the `order` attribute breaks the tie. Weight is the dominant factor (it shifts the priority number itself), so a weight-1 good in moderate shortage will always beat a weight-5 good in the same shortage. Order only matters when priorities are numerically equal. But when they are, order encodes dependency chains that ensure the economy bootstraps correctly.

The `order` attribute creates an implicit bootstrapping sequence:
```
1  Construction Sector
2  Government Administration
3  Railway, Port
4  Oil Rig, Tooling Workshops, Power Plant
5  Logging Camp, Whaling Station, Coal Mine, Iron Mine, Sulfur Mine, Livestock Ranch, Cotton Plantation
6  Lead Mine, Rubber Plantation, Paper Mills, Chemical Plants, Steel Mills
7  Glassworks, Motor Industry, Arms Industry, Munition Plants
8  Electrics Industry, War Machine Industry, Shipyards
9  University, Barracks, Naval Base
10 Wheat/Rye/Rice/Maize/Millet Farms
11 Gold Mine, Dye Plantation, Opium Plantation, Silk Plantation
12 Textile Mills, Furniture Manufactories, Synthetics Plants
13 Fishing Wharf, Banana Plantation, Sugar Plantation
14 Tobacco Plantation, Food Industry
15 Tea Plantation, Coffee Plantation, Arts Academy
```

Construction capacity first (need points to build anything), then administration (need bureaucracy to fund the state), then infrastructure (need to move goods), then tools and power (inputs for everything downstream), then raw materials, then heavy industry, then military, then grain farms, then luxury goods, and finally consumer entertainment. University and military are deliberately late -- order 9 -- because they don't directly grow the economy. Grain farms at order 10 reflects that food shortages are rarely critical in the early game. Luxury goods (tea, coffee, arts) are last at order 15.

Some buildings share construction counters to prevent over-queuing: wheat/rice/maize/millet share counter 18 (rye's ID), cotton shares counter 23 (livestock's ID), synthetics share counter 25 (dye's ID). This means if the AI is already building wheat farms, it won't also queue rice farms in the same iteration.

The `order` attribute only breaks ties between buildings at the same priority level. A critical iron shortage (priority 1+1=2) still beats a moderate construction need (priority 5+1=6). But when multiple buildings are equally needed, order ensures the economy bootstraps correctly.

This is a static heuristic, not adaptive. It doesn't know that *this particular country* needs iron before coal. But it gets the broad sequencing right, and the dynamic priority system handles country-specific adjustments.

When the AI must build up from nothing, encode a sensible default sequence as a tiebreaker rather than trying to compute optimal build order dynamically. The bootstrapping problem is mostly about avoiding catastrophic mis-sequencing (military before economy, luxury before infrastructure), not about finding the optimal path. More broadly, this illustrates a useful pattern: separate your decision into orthogonal layers (urgency via weight, dependency via order, confidence via offset) rather than trying to capture everything in a single score. Each layer handles a different concern and can be tuned independently.

### 4.7 Engineering around language limits: data packing and code generation

The scripting language has no arrays, no structs, no loops over dynamic ranges, and every variable consumes memory and save file space. With 49+ building types x multiple countries x multiple states, the naive approach (one variable per data point) is prohibitively expensive.

**Data packing:** Multiple values packed into single integers by digit position. A single `aroai_building_type_{id}_collected_data` variable stores 5 cells: priority level (ones/tens), supply vs demand level (hundreds), productivity requirement (ten-thousands to millions), production method level for railways (hundred-millions), and construction counter (rounded to millions). An average of cells 1 and 2 is computed for construction limit lookup. This is ugly but necessary -- it's the scripting language equivalent of bit-packing in systems programming, and it reduces memory footprint by roughly 5x compared to separate variables.

**Code generation:** JavaScript toolchain in the `tools/` directory generates repetitive Paradox script. The master generator (`vanilla_building_types.js`) produces 4 code blocks covering 200 compatibility slots, dispatch logic, and iteration handling. Specialized generators create: 3,000 script values for data extraction (500 buildings × 6 values each -- priority, supply/demand, productivity, PM level, counter, and average), a 999-case switch for technology progress (each level adds `innovation × 4 weeks` progress to a chosen tech), 99 construction limit values (9 limit levels × 11 data quality levels using a quadratic decay formula: `0.12 × ((11-level)^2 × 0.01 + (11-level) × 0.15 + 1) × (1 + (factor-5) × 0.125)`), state filtering/sorting across 40 combinations (10 aptitude levels × 4 branches), 200 compatibility patch stubs, and 500 variable initializations to suppress error log complaints.

**Compatibility patch system:** The mod can't discover modded buildings at runtime. No reflection, no dynamic dispatch. Solution: 200 reserved slots with a complete workflow -- JavaScript code generators produce boilerplate effect/trigger files, detailed documentation with step-by-step instructions and a full example patch, a GitHub issue tracker for ID registration (issues #4 and #5). Each patched building requires the 11-attribute format (key, ID, class, counter, order, limit, crucial, workforce, allocate, branching, scaling) plus custom evaluate/consider/sanction/allocate triggers. Agriculture and most government buildings can't be patched (complex arable land interactions and formula-based evaluation), but resource and industry buildings work well. Runtime detection runs monthly via `aroai_refresh_list_of_compatibility_patches`.

When your target language lacks expressiveness, build a meta-layer that generates code in that language. The cost of maintaining a code generator is lower than maintaining thousands of lines of hand-written repetitive script. And if your design has a hard limitation (like invisible modded buildings), don't just document it -- build an extensibility toolkit so others can work around it with minimal effort. The 11-attribute building format is a data contract that makes the patch system predictable.

### 4.8 Steering vanilla systems without replacing them

Not everything can or should be replaced. Victoria 3's production method selection (how buildings operate -- which inputs they use, what technology tier they run at) is handled by vanilla AI, and replacing it entirely would be another massive maintenance burden. But vanilla has no `ai_value` set on most production methods, so it uses defaults that don't reflect which upgrades are actually worthwhile.

The solution: add explicit `ai_value` weights with clear progression to guide vanilla toward upgrading production methods as technology advances. The progression varies by building type:

- **Administration:** 50,000 (Simple Organization) → 100,000 (Horizontal Drawer Cabinets) → 200,000 (Vertical Filing Cabinets) → 400,000 (Switch Boards). Clear doubling progression that strongly favors modern methods.
- **Education:** 50,000 (Scholastic Education) → 100,000 (Philosophy Department) → 200,000 (Analytical Philosophy Department).
- **Glassworks:** 50,000 (Forest Glass) → 100,000 (Leaded Glass) → 200,000 (Crystal Glass) → **0** (Houseware Plastics). The zero is deliberate: Houseware Plastics depends on electricity/oil inputs that the AI may not have secured, and if supply is insufficient, the entire building chain stalls.
- **Urban Center:** 50,000 (No Street Lighting) → 100,000 (Gas Streetlights) → **0** (Electric Streetlights). Same reasoning: blocking a risky electricity-dependent upgrade is better than the AI adopting it and crashing.
- **Ports:** **0** (Anchorage, the worst port method) → 50,000 (Basic Port) → 100,000 (Industrial Port) → 100,000 (Modern Port). Anchorage gets zero so the AI always prefers a proper port.

Crucially, no gameplay values are changed -- all inputs, outputs, employment, convoy capacity, infrastructure effects remain identical to vanilla. The mod only adds AI guidance. Full PM redefinition is a technical necessity (Paradox modding replaces whole blocks, not individual fields), but the content within is vanilla-identical except for the added ai_value lines.

When you can steer a vanilla system through weights or preferences, prefer that over replacement. Steering is lower maintenance, inherits future engine improvements, and doesn't change the game for human players. Know when to block dangerous options (the AI will adopt risky upgrades it can't handle) and use clear progressive values so the intent is obvious to any future maintainer. The lightest possible touch: add guidance, change nothing else.

### 4.9 Breaking infinite wars: patient stalemate resolution

AI-vs-AI wars where both sides reach 0 war support can persist indefinitely, tying up armies and ruining economies for decades. The vanilla game has no mechanism to break these deadlocks. Players watch helplessly as half the map's AI countries slowly bankrupt themselves in pointless wars.

The solution is deliberately patient: every 30 days, check all ongoing wars. For wars where both warleaders are AI with 0 war support, advance a stalemate counter through 24 levels (one per check = ~2 years). Wars are tracked through numbered lists (`aroai_stalemate_list_1` through `_24`), with garbage collection removing ended wars. Only at level 24 is the war forcefully resolved:
- **Secessionist wars:** Secessionists win (the revolt succeeded -- they held out long enough)
- **Revolutionary wars:** Higher population side wins (the larger faction prevails)
- **Other wars:** White peace (nobody won)

Two years is long enough that most wars resolve naturally through the game's own mechanics. The forced resolution is a last resort, and the type-aware outcomes produce historically plausible results rather than arbitrary coin flips. Secessionist wars favoring the secessionists reflects that if neither side can win decisively, the status quo changes -- the new state exists and isn't going away.

When AI-vs-AI interactions can deadlock, build a timeout mechanism -- but make it patient and context-aware. Immediate intervention feels artificial; long patience with type-specific resolution feels like the world working itself out. The resolution logic should reflect the game's narrative logic, not just "flip a coin." And track progress through escalation levels rather than binary timers -- this lets you add intermediate responses at future levels if needed.

### 4.10 Letting players tune the AI: configurability as design philosophy

One AI behavior doesn't fit all players. Competitive players want maximum challenge. Casual players want the AI to be competent but not ruthless. Historical roleplayers want AI behavior that varies by country character. Modders want to adjust the AI for their own content.

ARoAI provides 10 game rules:
- **Power Level** (0-100%, 10% increments) scales AI income calculations, cascading into all spending targets and construction. Separate settings for AI countries and player subjects. Special handling for AI China and AI India -- identified by population thresholds (≥50M AND ≥50% of that culture), with linear scaling from 50M (0x effect) to 100M (full effect). Non-customs-union subjects get a 1.25x recovery multiplier capped at the game rule value. China and India multipliers stack multiplicatively with the base power level.
- **Construction scaling** (0-200%) -- values above 100% add visible state modifiers (`aroai_X_percent_accelerated_construction`, 10 tiers from +10% to +100% `state_construction_mult`) and are explicitly warned as "no longer fair play" in the UI.
- **Building Priorities** (Roleplay / Uniform) -- whether AI weights vary by country laws and character (Roleplay) or treat all countries identically (Uniform).
- **Research Assistance** (Default / Railroaded / Disabled) -- Default adds gentle weighted guidance toward tier 3-4 techs. Railroaded forces a strict path: Rationalism → Academia → prerequisites (Enclosure, Manufacturies, Shaft Mining, Steelworking, Cotton Gin, Lathe, Mechanical Tools) → Railways, plus Nationalism for appropriate countries (USA, Italian states, German states), Nitroglycine, Breech-loading artillery, and Ironclads. Uses an `aroai_innovation_redirection` modifier that sets `country_weekly_innovation_mult = -1` (cancels all natural innovation) while manually adding `innovation × 4 weeks` progress to the chosen tech each iteration. Disabled removes all guidance.
- **Autobuild for Players** and **Stalemate War Prevention** as additional toggles.

The configurability isn't just user-friendliness -- it's an acknowledgment that AI behavior has no single correct answer. By exposing the knobs, the mod is honest about the trade-offs. The power level system specifically avoids stat cheating at default: 100% power, 100% construction gives AI no advantages. The AI is smarter, not stronger.

Design AI behavior with explicit configuration points from the start, not as an afterthought. The parameters you expose tell players what trade-offs exist. Warn when settings leave fair-play territory. And if a setting affects game balance (like construction bonuses), make the consequences visible through in-game modifiers that players can inspect.

### 4.11 Fairness and validation through shared logic

How do you know your AI's decisions are actually good? Testing AI quality in a strategy game is hard -- you can't unit-test "did the AI build a reasonable economy?"

Default game rules (100% power, 100% construction) give AI no stat advantages. The mod makes AI smarter, not cheaty. But the real validation comes from player autobuild: the exact same evaluation, priority, and construction logic the AI uses is available to human players. A decision button opens autobuild settings with per-category toggles (government administration, university, construction sector, railway, port, barracks, naval base). Each toggle sets a country variable, and the main loop treats the player like an AI for enabled types. Settings persist for 93 days before the decision becomes available again.

If players use the autobuild and find it good enough to trust with their own economy, that's strong evidence the decision quality is real. Every player who enables autobuild is an implicit tester of the AI's judgment. The shared logic also prevents a common failure mode in AI mods: the AI and the player-assist feature making different decisions because they use different code paths.

If you can expose your AI's decision logic as an optional tool for human players, do it. Players are the best testers -- they'll immediately notice when the AI makes a decision they wouldn't. Shared logic between AI and player-assist features keeps both honest. And per-category toggles let players keep control where they care most while delegating the rest.



## 5. Imperator: Invictus

### 5.1 Working within someone else's project

Imperator: Invictus represents the refined generation of this AI work -- AI embedded in a team project, with new tools and a communication-first approach. The core challenge is contributing AI improvements to an existing team mod with its own vision, its own codebase, and its own players. You can't rewrite everything, and every change must be explained and justified.

Imperator: Invictus is not a standalone AI overhaul -- it's AI work embedded in a larger content and overhaul mod. Different constraints from solo projects: must coexist with the content team's work, can't break balance assumptions, must explain changes through dev diaries for every update. Every patch from 1.9.1 through 1.10.1 came with a detailed public diary covering what changed, why, what the observed problems were, what the limitations are, and what still doesn't work.

This is also the project with the lightest-touch approach: primarily adjusting AI weights and adding new scripted behaviors, rather than disabling and replacing vanilla systems wholesale. The Imperator economy is simpler than Victoria 3's, making weight-based manipulation viable. Many critical systems (fort placement, ship building, unit movement) are hardcoded in C++ and simply can't be replaced. The architecture reflects the context -- not because replacement would always be wrong, but because the project doesn't need it and the team wouldn't benefit from the maintenance burden.

The patch-by-patch evolution tells a story: 1.9.1 laid the foundation with a complete invention and building weight rework. 1.9.2 added law management, character loyalty, state investment, diplomatic stances, and national ideas -- entire systems that didn't exist before. 1.10 leveraged the new `modifier:` syntax for research efficiency checks, mercenary recruiting, and governor policies. 1.10.1 added roads, fort placement, and integrated all previous systems into the Advanced AI game rule. Each patch built on the previous one's infrastructure.

The lesson is to match your approach to the project context, not your ambition. A team mod with regular releases needs stable, incremental improvements that can be explained to teammates and players. A solo project can afford architectural experiments. The best technical approach in isolation may be the wrong approach for the project. And if you're working in public, the discipline of explaining every change forces clearer thinking about design rationale.

### 5.2 Making AI weights human-readable: the relative chance system

When manually setting AI weights for hundreds of inventions or buildings, humans lose track of what the numbers mean. "Pretty good" at 200 and "very good" at 500 seem different enough, but in a randomized system choosing from dozens of options, a 2.5x weight difference is negligible. Weight values tend to inflate over time as each new addition needs to be "higher than the last one" -- 100, 500, 2000, 9000...

The solution: **Invention Relative Chance (IRC)** -- a probability-based scale. IRC 35% means "this option has a 35% chance of being chosen if all other options are at base weight." The modder thinks in probabilities, not magic numbers. The same system is used for buildings as **Building Relative Chance (BRC)**.

The formula: `(poolSize - 1) / (1 - x/100) - (poolSize - 1)`, where x is the desired percentage and poolSize is hardcoded at 20 (representing the average number of choices per invention decision).

Key properties:
- **Limited scale based on probability** (0-99%), so values are always interpretable. You can't accidentally create a weight of 50,000.
- **Self-documenting:** `irc_35` immediately tells you and any future maintainer what tier of importance this is. No need to check a reference table.
- **Prevents inflation:** The probability framing keeps values anchored to a meaningful scale. IRC 95% is the practical ceiling for "almost always pick this."
- **Non-linear mapping exposes real differences:** IRC 35% = weight 10.23, IRC 75% = weight 57, IRC 95% = weight 361. The 95% option is 35x more likely than the 35% option -- a difference that would require typing ~9,000 as a raw weight, which is psychologically uncomfortable and loses all interpretability.
- **Works for any pool of similar choices:** Pre-generated values at 5% increments (5, 10, 15... 95, 99%) cover the practical range.

With pool size 20, the system creates clear tiers: IRC 35% is "moderately preferred," IRC 75% is "strongly preferred," IRC 95% is "almost always chosen." Within priority tiers, weights are close enough that randomness creates variety between AI countries. Every AI knows discipline inventions are important, but IRC 75% vs IRC 65% means some focus military first while others prioritize economy.

When designing weight systems for AI choices, the measurement system matters as much as the values. If weights are just numbers, they'll inflate and lose meaning over time. Anchor weights to an interpretable scale (probabilities, percentages, named tiers) so that any developer can look at a value and immediately understand its intent. The pool-size-aware formula ensures the weights actually produce the intended probabilities, not just relative ordering. This is a human factors insight applied to AI design -- the system serves the maintainer as much as the engine.

### 5.3 Structured building decisions: sequencing over optimization

Imperator has fewer building types than Victoria 3 or Stellaris, but the choices still interact: research buildings are useless if you've already overcapped research efficiency, economic buildings only matter once you have the right population types, and ports waste building slots if you have too many. The AI needs a coherent building strategy, not just individual good choices.

The building system uses a layered sequence:
1. **Non-cities first:** Mines and Farming Settlements provide trade goods and income. Slave Estates in territories with important trade goods unsuitable for mines/farming, or in slave-heavy territories. Barracks for manpower. Tribal Settlements for tribesmen-heavy territories. Secure the treasury and resource base before investing in city buildings.
2. **Ratio buildings in cities:** Academy (noble focus, boosting research), Court of Law (citizen focus), Forum (freemen focus, boosting manpower and taxes), Mill (slave focus). Choice based on territory culture rights and trade goods -- matching buildings to the population that will actually work them. A territory where nobles have full rights gets an Academy; where only freemen have rights, it gets a Forum. The side effects of these choices matter: Mills skew population toward slaves, reducing nobles and citizens, which shrinks both research output and levy size.
3. **Modifier buildings after ratio:** Library (cheaper than Academy, immediate effect, built until research approaches maximum efficiency), Training Camp, Market, Tax Office -- these enhance what the city already does well, so they only make sense after ratio buildings establish the city's identity.
4. **Unique buildings conditionally:** Foundry in cities with expensive trade goods (the production bonus is proportional to trade good value). Great Temple where many pops don't follow the state religion (accelerates religious conversion). Grand Theatre where assimilation is needed (accelerates cultural integration). Aqueducts for metropolis growth in the late game.

The crucial 1.10 refinement: once a country reaches high enough research efficiency (now checkable via the `modifier:` syntax introduced in patch 2.0.5), the AI stops building Academies and Courts of Law and switches to Forums and Mills. This prevents the observed failure where small AI countries like Crete would fill every city (all 1-5 of them) with Academies, overcapping research while starving for manpower and taxes. The Library continues to be built more relaxedly -- it's cheaper, has immediate effect, and only stops when research is "somewhat close to max." If the AI still ends up with too much research despite these changes, Harsh Taxation kicks in as an economic policy to convert the excess into revenue.

Port management required particular care: AI wants only 1-2 ports per state (based on coastline percentage), with an exception that every city keeps its port. Excess ports are deleted to free building slots. Territories with trade goods suitable for Mines or Farming Settlements are avoided for port placement -- those slots are too valuable. "Mega" ports (level 3, requiring 20+ building slots or equivalent) are maintained for medium ships. "Giga" ports (level 5, requiring 30+ building slots) only for countries with the military tradition for heavy ships. All other ports are downscaled to level 1. Target numbers of high-level ports are calculated based on country size.

City founding also got a complete rework: priority conditions include capital territory, high integrated population, coastal and river locations, "good" terrain types (steppe is prohibited due to -50% population capacity). City creation is blocked until existing cities are somewhat developed. Tribes have extra restrictions to prevent the "Mesopotamia 2.0 in Germany" scenario where tribal AI would spam cities in unsuitable terrain. Small non-tribal countries without any cities will establish them, but larger empires only urbanize modestly in the late game.

When building choices interact (one building's value depends on what's already built), use explicit sequencing rather than trying to evaluate all options simultaneously. "Build infrastructure, then specialization, then enhancement" is a pattern that works across strategy games because it mirrors real economic development. Add dynamic cutoffs (stop building X when you've overcapped its benefit) to prevent runaway specialization. And be explicit about side effects -- if building type A reduces the population type that building type B needs, the AI should know about that interaction.

### 5.4 Guiding research without scripting it: trees, not picks

The AI needs to research useful inventions, but you don't want every AI country following the same path (that feels scripted) and you can't predict which inventions any particular country needs most. Pure randomization produces incompetent AI; pure scripting produces identical AI.

Instead of weighting individual inventions, weight the entire tree leading to important targets. The AI gradually researches the path, benefiting from intermediate inventions along the way while reliably reaching the important ones. This is critical because Imperator's invention system is randomized -- the AI gets a pool of options and picks one, so weights need to be high enough to overcome the pool.

Priority layers, structured by importance:
- **Discipline inventions:** Weighted heavily as primary focus (AI was previously ignoring these entirely). Discipline directly determines combat effectiveness.
- **National Tax inventions:** Equally important -- revenue is the lifeblood of expansion.
- **Secondary priorities:** Economic inventions, Proscribed Canon (critical for monarchy assimilation/conversion policies), siege capability (needed to actually take cities in wars), Auxiliary Recruitment, Formulaic Worship, City Planning, Rural Planning, culture-specific trees.
- **Lower priority:** Diplomatic and navy inventions (useful but not decisive).

Within each layer, weights are close enough that randomness creates real variety. Every AI country knows discipline and national tax are both important, but some focus on military first while others prioritize economy. The IRC scale makes this possible: IRC 75% vs IRC 65% creates a meaningful preference without being deterministic. The 35x difference between IRC 95% and IRC 35% ensures truly important inventions are almost always taken, while moderate preferences leave room for divergent paths.

Later patches (1.10.1) shifted focus further: AI major powers now aim for population happiness and province loyalty inventions (stability enables expansion), and essential inventions are prioritized more heavily over niche ones. The Dictatorship invention was re-enabled: great powers get decent probability, major powers get much lower but non-zero probability.

For research/tech AI, weight paths rather than endpoints. This ensures the AI makes progress toward important goals while benefiting from intermediate steps. Use similar-but-not-equal weights within priority tiers to create natural variety between AI players. The goal is guided randomness: good choices with different sequences. And use a probability-anchored scale (like IRC) so that "strongly preferred" and "moderately preferred" have precise, predictable meanings.

### 5.5 Building systems that simply didn't exist

Some of the most impactful work addressed not "the AI does this poorly" but "the AI doesn't do this at all." Several game systems had no AI logic -- not underperforming, not buggy, just absent. The distinction matters: fixing a bad system is tuning; building a missing system is design from scratch. And missing systems are easy to overlook because their absence doesn't produce errors -- the AI just ignores an entire dimension of gameplay.

- **Law management:** No system existed for AI enacting laws. Tribes had nearly empty law panels -- they literally couldn't change their government structure. Monarchies and republics depended on rare random events to change laws, meaning most AI countries would go entire games without adjusting their government. The mod built a complete system: tribes progress centralization or decentralization based on starting values (most double down on their starting direction; tribes starting at 0% centralization get a random roll via events/modifiers). Centralization leads to higher civilization, more research, and the ability to reform to monarchy/republic. Decentralization leads to levy size and morale bonuses -- becoming a late-game raid boss with minimal-cost migration. Most tribal laws provide non-military bonuses (income, happiness, stability), with the Steppe Horde as a notable exception providing strong military bonuses. Monarchies prioritize conversion/assimilation laws (conversion first as it's faster, then assimilation -- following the optimal player strategy) and military reform. All consider stability before enacting and wait appropriate cooldowns.
- **State investment:** AI weights were literal constants -- 1 and 1.5 -- with no conditions at all. Near-random selection. The rework evaluates each investment type's actual value: oratory investments (trade routes) are almost always useful and serve as the default when uncertain, military investments help with fort limit, civic investments are only worthwhile when real overcrowding exists, religious investments need multiple built-up cities to justify the expense. Investment limits are set by government type and state population combination. Capital state is always prioritized for development and strategic goods imports. Performance limitation: oratory investments only allowed if the state already has trade routes from other sources (AI trading is heavy on performance).
- **National idea selection:** Previously believed to be unmoddable. Nobody in the modding community had tried adding AI weights because the feature wasn't documented as supporting them. Turned out the weight syntax from other systems works -- it just hadn't been attempted. Once discovered, AI countries could be guided toward appropriate choices. AI doesn't update ideas over time (sticks to initial setup), but government type changes wipe ideas, allowing fresh selection. Revolts start with blank ideas, so civil wars can bring fresh national direction.
- **Character loyalty management:** AI couldn't (or wouldn't) use bribing, free hands, and subsidies to prevent civil wars. Worse: the vanilla AI wasn't paying political influence for character interactions -- it was silently cheating. The mod removed the cheating and implemented proper loyalty management with a clear escalation chain: bribes first, then grant free hands for governors (with strict corruption limits tied to province loyalty impact), then revoke hands if corruption gets too high, then family subsidies as last resort (only if treasury is sufficient). Restrictions keep civil wars possible through actual political struggle: no loyalty fixing for ruler rivals unless the threat is existential, and additional caps prevent the AI from becoming too stable. The goal is competent management, not invincibility.
- **Diplomatic stance selection:** Vanilla had overlapping AI weights causing constant stance-switching that burned political influence for no benefit. The rework uses exclusive logic with anti-switching buffers: high aggressive expansion and not attacking → Appeasing (until AE drops), big war active → Bellicose (lower war goal costs and AE impact), heavy subjugation focus → Domineering, low diplomatic relations or political influence → Neutral (conserve resources), money-focused → Mercantile (default for peacetime commerce). One stance at a time, with hysteresis to prevent flip-flopping.

Before trying to improve AI behavior, audit whether the behavior exists at all. In modding, the highest-impact work is often not optimization but creation -- building systems the developers shipped without AI support. Each individual system may be small, but together they give the AI access to game mechanics it previously couldn't use. An AI that can use all the game's systems competently will outperform one that plays half the game brilliantly and ignores the rest. And check your assumptions about what's moddable -- the national idea discovery shows that untested capabilities may already exist in the engine.

### 5.6 Removing crutches when the AI outgrows them

Games often contain compensatory mechanics -- shortcuts, cheats, or lowered requirements -- that exist because the AI can't handle the real thing. When you teach the AI to play properly, these compensations become exploits. But they're easy to miss because they look like normal game mechanics.

When AI tribes learned to found cities and enact laws, they started abusing special low-requirement reform decisions. These decisions existed because the AI couldn't manage standard reform requirements -- ruler popularity, correct laws corresponding with government type, sufficient rank (at least regional power). They were training wheels. Once the AI could actually play the game -- progressing centralization, managing laws, founding cities -- the training wheels became exploits. AI tribes were mass-reforming to monarchies within 100 years, far faster than intended.

The fix: force AI tribes to meet the same requirements as players. Minimum rank of regional power. Correct law prerequisites. Ruler popularity thresholds. The AI now earns reform through actual development -- centralization progress, city building, population growth -- rather than shortcuts.

This pattern recurs across all three mods:
- **Imperator:** Removing free PI for character interactions was the most egregious case. The vanilla AI wasn't just getting help -- it was silently cheating, using bribes and interventions without paying the political influence cost. Once proper loyalty management existed (with the escalation chain of bribes → free hands → subsidies), the cheat was removed. The AI now pays real costs for real actions.
- **Stellaris:** Free ship upgrades via `acai_upgrading_band_aid` (-100% ship upgrade cost) exist because the vanilla AI can't reliably figure out when and where to upgrade fleets. This crutch would ideally be replaced by proper upgrade logic, but the mod hasn't reached that point.
- **Victoria 3:** Disabling vanilla debt pause/resume thresholds (set to unreachable 6.66/7.77) because the mod's own budget health system handles the same function better and with more nuance.

When you improve AI capability, audit the game for compensatory mechanics that existed because the AI was weak. Improvements compound with crutches to create new exploits -- the AI reform case is a perfect example where better law management + easy reform requirements = broken progression speed. Every AI upgrade should come with a crutch review: "what shortcuts does the game give the AI that it no longer needs?" The crutch may be a lowered threshold, a free resource, a simplified requirement, or an outright cheat.

### 5.7 Unlocking new decisions when the engine exposes new data

The quality of AI decisions is bounded by what the script can observe. When an engine patch exposes new data, previously impossible improvements become straightforward -- but only if you're watching for these opportunities and ready to act on them.

Patch 2.0.5 introduced `modifier:` syntax -- the ability to check values of most game modifiers at runtime. This was transformative for multiple systems:

- **Research efficiency:** Previously impossible to check. The AI couldn't know if a country was at 40% or 175% efficiency. Previous workaround: use population ratio logic (culture rights determining building types) as an indirect proxy. After 2.0.5: the mod reads current and max research efficiency directly, stops over-building Academies and Courts of Law when efficiency is high enough, and switches to Forums and Mills for manpower and tax income. Harsh Taxation becomes available as an economic policy when research is severely overcapped. The improvement was conceptually designed long before the data existed -- it was just waiting for the API.
- **Governor policy selection:** Precise province loyalty monthly change values enable timely, targeted intervention rather than reactive fixes. Corrupt governors choose Acquisition of Wealth over Encourage Trade (roleplaying self-interest). Merciful/submissive characters choose Local Autonomy over Harsh Treatment. Governors must match state religion and primary culture to prioritize conversion/assimilation policies -- mismatched governors ignore those policies. Disloyal governors won't prioritize country expectations. Multiple "evil" trait combinations on the same governor are avoided.
- **Mercenary recruiting:** Budget limits, maintenance costs, and reachability -- all needed for the Advanced AI mercenary system. Without modifier access, the AI couldn't check whether it could afford mercenary upkeep or whether the mercenary company could even reach the country.

Each of these was a known problem before 2.0.5. The design solutions existed conceptually -- they just couldn't be implemented without the data. When the data became available, the implementations were straightforward because the thinking had already been done.

Maintain a "wish list" of improvements that are blocked by data access limitations. When engine updates or new API capabilities arrive, check the wish list first. The solutions are often already designed in your head -- they're just waiting for the data. This applies beyond modding: any AI system bounded by observable state improves discontinuously when new observations become available. The modder who has pre-designed solutions ships faster than the one who starts designing after the API appears.

### 5.8 Difficulty through smarter play, not bigger numbers

Players want adjustable difficulty, but the standard approach -- stat bonuses (+50% income, +25% combat) -- feels unfair and creates balance distortions. It also teaches the AI nothing: a stat-boosted AI that makes bad decisions just makes bad decisions with bigger numbers.

The "Advanced AI" game rule is a modular difficulty system based on behavioral changes, introduced in 1.10 and fully integrated by 1.10.1:
- **Mercenary recruiting:** Custom algorithm (not a vanilla weight adjustment -- built from scratch) that aggressively hires mercenaries during wars. AI checks if mercenaries can reach the country, validates budget supports maintenance, and can drain the entire regional mercenary pool during active wars. Richest countries can bribe mercenaries as last resort. AI subjects spend less on mercenaries to prevent "vassal swarm" scenarios where every subject floods the field with hired armies.
- **Internal development:** Higher resistance to civil wars through better loyalty management. More tribes reforming (properly, meeting real requirements). Manual pop movement into underused cities (moving population to where it's needed). Heavy province investment usage -- a game-changer for small countries that can stack 50+ building slots in city states through sustained investment.
- **Road building from scratch:** Three-phase construction (inter-region connections for levy delivery, dense province-level networks, then city-to-city roads in small developed countries). Almost no performance impact despite custom pathfinding. Configurable via game rules (disable entirely or limit scope) because some players consider roads historically implausible in certain regions.
- **Fort optimization:** A corrective layer over hardcoded vanilla logic that repositions existing forts to border provinces, matches them with province capitals (essential due to occupation mechanics), and maintains higher-level forts in capitals and densely populated provinces. The AI also moves province capitals to logical locations (large cities with forts) to fix weird post-annexation placement.
- **Fully configurable:** Individual features can be disabled. This matters because Advanced AI leans on meta-strategies (aggressive mercenary hiring is a strategy experienced players use but casual players might find unfair).

The design principle: difficulty scaling through smarter decisions, not stat bonuses. Stat bonuses are covered by separate vanilla difficulty settings and can be combined with Advanced AI independently. A player who wants harder AI can enable behavioral improvements without artificial advantages. A player who wants realistic behavior without extreme difficulty can use base improvements without Advanced AI.

This configurability philosophy evolved across all three mods: Stellaris (least configurable, earliest project), Victoria 3 (10 game rules with per-category autobuild toggles), Imperator (individual Advanced AI toggles plus road-building scope control). The evolution shows growing awareness that AI behavior isn't one-size-fits-all.

Separate "AI competence" from "AI advantage." Competence improvements (better decisions, using more game systems, fixing failure modes) should be the default -- they make the game more fun for everyone. Difficulty bonuses (stat boosts, behavioral aggression) should be optional and transparent. Players resent invisible advantages but respect AI that plays well. And make difficulty features individually toggleable -- different players have different thresholds for what feels "fair" vs. "frustrating."

### 5.9 Working around hardcoded logic: roads and forts

Critical game systems (fort placement, ship building, unit movement) are hardcoded in C++ and cannot be replaced at all. You can observe their output, but you can't change their logic. Some important systems (road building) simply don't exist. The question is how to improve what you can't touch.

**AI road building** -- built from scratch because the game had no road AI at all:
1. First connect all controlled regions (levy delivery -- levies raise across the country, roads help them gather faster). This is the highest strategic priority because levy assembly speed directly affects military responsiveness.
2. Then create dense province-level networks for fast army movement within theaters.
3. Then intra-province roads between cities in small, developed countries -- the final polish.
- Configurable via game rules (disable entirely or limit scope to inter-region only)
- Major technical challenge: the mod had to implement custom pathfinding, construction sequencing, and budget management -- none of which the engine provides -- all with minimal performance impact. Roads are only built when the country has qualifying legions and available money.
- Historical plausibility was a community concern: game rules let players who find AI road-building ahistorical in certain contexts disable it.

**AI fort placement** -- a corrective layer over hardcoded vanilla logic:
- Vanilla fort placement has only a few tunable variables with insufficient influence on behavior. The AI places forts in bizarre locations -- protecting useless land instead of borders, ignoring province capitals, leaving the actual frontier undefended.
- The workaround: an optimization layer that repositions existing forts after vanilla places them. Border provinces get primary priority (forts are expensive and budgets are tight -- better to defend the frontier than the interior). Province capitals are matched with forts (essential: Imperator's occupation mechanics make capital-fort alignment critical for defense). Higher-level forts go in capitals and densely populated provinces.
- Related: AI now moves province capitals to logical locations -- large cities, ideally with existing forts. Post-annexation, province capitals can end up in tiny villages. The AI fixes this.
- The inner provinces are deliberately exposed. The trade-off: a few unfortified interior provinces vs. a solid frontier line with easy de-occupation using minimal forces. The mod chooses frontier defense because it's what experienced players do.

When you can't replace a system, build a corrective layer that observes its output and adjusts. "Repositioning forts after vanilla places them" is less elegant than controlling fort placement directly, but it works. And when a system is completely absent (roads), building it from scratch -- even with limited tools and custom pathfinding -- is often more valuable than optimizing systems that already work acceptably. Prioritize creating missing capabilities over perfecting existing ones. The road system and fort optimization together transformed AI military competence more than any amount of weight tuning could.

### 5.10 Communication as part of the design process

AI behavior is invisible. Players can't tell why the AI did something, teammates don't know what assumptions the AI code relies on, and the next maintainer won't understand the design intent. In a team project, unexplained AI changes create subtle bugs when other developers unknowingly conflict with AI assumptions.

Every Imperator: Invictus update came with a detailed dev diary explaining not just *what* changed but *why*: the design reasoning, the observed problems, the limitations, and honest assessment of what still doesn't work. The 1.9.1 diary introduced the IRC system and explained why raw weights fail at scale. The 1.9.2 diary walked through each new system (law management, character loyalty, investments) with specific examples of the failures they address. The 1.10 diary explained the modifier syntax breakthrough and what it enabled. The final 1.10.1 diary includes a retrospective of all AI work across the project -- roads, forts, integration, and remaining limitations.

This isn't just documentation -- it's a design tool. Writing the explanation forces you to articulate the design logic, which often reveals gaps or inconsistencies. "I can't explain why this weight is 75% instead of 65%" is a signal that the choice is arbitrary and should be reconsidered. The discipline of explaining each change to a public audience -- teammates and players who will test it, critique it, and report edge cases -- raises the bar for design quality.

The dev diaries also serve as a historical record. When a later patch changes behavior, the diary from the original patch explains the reasoning behind the original choice. This prevents the common modding failure where a new contributor "fixes" something that was deliberately designed that way, because they can read the diary and understand the intent.

If you're working on AI in a team context, explain the *why* at least as thoroughly as the *what*. AI code is uniquely opaque -- it produces emergent behavior that can't be understood by reading individual lines. Dev diaries, design documents, or even detailed commit messages serve as the bridge between "what the code does" and "what the AI is trying to accomplish." Players and teammates need this bridge to provide useful feedback. And the act of writing the explanation improves the design itself -- if you can't explain it clearly, you may not understand it well enough.



## 6. Cross-Cutting Themes

### 6.1 The evolution arc

- **Stellaris:** Fix-and-enhance approach, coexisting with vanilla AI, organic layer accumulation. Built on top of an inherited mod (Glavius). Broad scope (economy, fleets, starbases, repair, war, diplomacy), implemented through full vanilla replacement for some systems (edict simulation, ship production) and weight overrides for others (tech, perks, components, traditions). Legacy layers from inherited code created architectural complexity that was never fully resolved -- dormant destroyer preference checks, unscheduled sector tax simulation, untriggered defensive starbase conversion. But the core insight of a shared economy model feeding every subsystem through quantized boolean flags was foundational.
- **Victoria 3:** Systematic full replacement. Designed as a complete architecture from the start: phased 28-day iteration cycle with randomized country offsets, market-driven evaluation for production buildings with a three-layer decision hierarchy (weight → order → offset), formula-based evaluation for government buildings with individual priority ranges and spending share targets, code generation toolchain producing 3,000+ script values and 999-case switches, 200-slot compatibility patch system with JavaScript generators, 10 game rules for configurability. Budget management through 7-level health scoring with 2D threshold surfaces and asymmetric tax/wage adjustment. More engineering, less duct tape. The trade-off: total maintenance burden (49 individually hardcoded buildings) and brittleness to game patches.
- **Imperator: Invictus:** Lightest touch, heaviest impact per line of code. AI weights with a human-friendly measurement system (IRC/BRC with probability-anchored scales and hardcoded pool size of 20), targeted new systems for capabilities that simply didn't exist (law management, character loyalty, state investment, diplomatic stances, national ideas, road building, fort optimization), leveraging new engine features (`modifier:` syntax) as they became available. Communication-first approach within a team -- every patch accompanied by a detailed dev diary. Configurability through modular Advanced AI game rules with individually toggleable features.

The arc is not just "getting better" -- it's adapting the approach to the project context. A standalone mod for an older game (Stellaris 2.1.3) warrants a different strategy than contributing to a living team mod (Invictus). The context shapes the craft:
- **Solo vs. team:** Stellaris and Victoria 3 were solo projects where all decisions were internal. Imperator required explaining every change to teammates and players through dev diaries, which forced clearer thinking about design rationale and created a historical record of why each decision was made.
- **Standalone vs. embedded:** Stellaris and Victoria 3 mods own their entire scope and can break things freely. Imperator AI work must coexist with content changes, map updates, and balance patches from other team members. Every AI assumption is a contract with the content team.
- **Game complexity:** Stellaris has a tile-based planet system (simpler building decisions but complex fleet and megastructure management), Victoria 3 has a deep market economy (requiring market-driven evaluation, productivity gating, and multi-dimensional budget management), Imperator has a simpler economy but deep internal politics (requiring law management, loyalty escalation chains, and governor policy AI). Each game demands a different emphasis.
- **Engine evolution:** Each game's scripting language offers different capabilities. Victoria 3's Jomini script has script values (inline calculations) enabling complex formulas but no runtime introspection of building definitions. Imperator's Jomini gained `modifier:` syntax in patch 2.0.5, opening up research efficiency checks, governor policies, and mercenary recruiting. Stellaris's Clausewitz is the most limited -- no inline calculations, forcing quantized ladders of if/else chains for every formula.

### 6.2 Performance is a first-class design constraint

Not an afterthought -- it shapes the architecture at every level. Every design choice includes a performance budget:

- Time distribution and country staggering (Stellaris: economy/ships/robots/buildings in day windows; Victoria 3: randomized 28-day iteration cycles with half-iteration idle buffers)
- Data packing into single integers by digit position (Victoria 3: 5 cells per variable, 5x memory reduction vs. separate variables)
- Selective data collection: only gather data when downsizing/construction is actually allowed (Victoria 3: preparation phase skips data collection if permissions deny action)
- Half-iteration idle periods as defensive engineering against engine lag (Victoria 3: days 15-28 deliberately empty)
- Hardcoded pool sizes in relative chance formulas rather than dynamic calculation (Imperator: pool size fixed at 20)
- Trade route limits on oratory investments due to AI trading being heavy on performance (Imperator: only allow oratory investments if state already has trade routes)
- Yearly cadence for systems that don't need monthly updates (Stellaris: pop/food recalculation, army production, starbase planning run yearly instead of monthly)

If your AI is technically correct but tanks the framerate, it's useless. Performance constraints produce better designs, not just faster ones -- they force you to decide what actually matters. The question isn't "what should the AI check?" but "what must the AI check, and how often?" This pruning eliminates speculative computation and focuses the AI on decisions that actually change behavior.

### 6.3 The trade-off: control vs. brittleness

More control = more maintenance, more compatibility burden, more version drift risk. Every override is a liability on the next game patch:

- **Stellaris:** ~1,700+ lines of overrides across tech, perks, traditions, components, sections, defines, opinion modifiers. Two full diplomatic systems rewritten (trust values, acceptance factors, border friction). Every game patch could invalidate any of them. The dormant code surface (40+ destroyer preference checks, unscheduled systems, unused modifiers) adds maintenance weight without delivering value.
- **Victoria 3:** Full PM redefinitions just to add `ai_value`, with all vanilla gameplay values preserved exactly (can't change one field without replacing the whole block -- Paradox modding replaces whole blocks, not individual fields). 49 buildings with individually hardcoded evaluation, sanction, and allocation logic. Three-layer decision hierarchy (weight, order, offset) with extensible weight families. 3,000 generated script values. Modded buildings are invisible without compatibility patches -- a hard limitation documented and mitigated with a 200-slot system, JavaScript generators, and GitHub registration workflow.
- **Imperator:** Lightest override burden (IRC/BRC weights plus targeted scripted behaviors), but compensated by the constraint that many systems (forts, ship building, unit movement) are hardcoded in C++ and require workarounds rather than direct replacement. Roads built entirely from scratch including custom pathfinding. Fort optimization as a corrective layer rather than a replacement.

The compatibility patch system in ARoAI is an explicit acknowledgment of this trade-off: if your design has a hard limitation, make it easy for others to work around it. The 11-attribute building format, the JavaScript generators, the step-by-step documentation, and the GitHub registration system turn a hard limitation into a manageable workflow.

### 6.4 Reactive vs. predictive AI

All three mods are fundamentally reactive -- they respond to current state, not anticipated future needs. This is a deliberate constraint, not an oversight:

- No forward-looking mechanisms (e.g., "research is about to unlock steel → pre-build iron mines"). The AI reacts to shortages after they appear, not before.
- Victoria 3's supply/demand evaluation is a lagging indicator: by the time a shortage appears in market data, the AI is already behind. The 28-day iteration cycle adds further delay.
- The `order` attribute (construction dependency chain) partially compensates with static front-loading -- construction sector before administration before infrastructure before raw materials -- but it's a coarse tiebreaker. It can't adapt to situations where the usual dependency chain is wrong (e.g., a country that already has abundant tools but desperately needs grain farms).
- Weight values are hardcoded per good, not per building or per country -- the AI treats iron shortage with the same urgency (weight 1) regardless of which building produces the iron or which country is experiencing the shortage.
- Stellaris's economy flags update monthly but downstream decisions may run on different schedules, creating lag between state change and AI response.
- Imperator's building decisions can't account for buildings under construction -- only completed buildings affect the research efficiency check.

Why: predictive AI requires more state tracking, more complexity, more performance cost. In a modding context where the language has no arrays, no structs, and every variable costs memory and save-file space, the marginal benefit may not justify the cost. The mods get remarkable results from purely reactive approaches -- the Victoria 3 doc's design analysis lists "reactive rather than predictive" as the top weakness but also acknowledges the architectural constraints that make prediction impractical. The Imperator doc notes several TODO items that would benefit from prediction but remain unimplemented due to complexity.

### 6.5 Teaching humans vs. programming AI

A recurring insight across all three projects: designing AI weights and systems is partly a human factors problem. The code serves two audiences -- the engine and the maintainer.

- The IRC relative chance system exists because humans lose track of magic numbers, not because the engine requires probabilities. Raw weights of 100, 500, 2000, 9000 are psychologically indistinguishable in their effect. IRC 35%, 75%, 95% are immediately meaningful.
- Dev diary communication exists because the team and players need to understand what the AI is trying to do. AI behavior is invisible and emergent -- you can't deduce intent from observing output, and you can't file useful bug reports without understanding the design.
- Removing AI cheats (free character interactions at zero PI cost, lowered reform requirements) must happen alongside AI improvements, or the improvements compound with the cheats to create new exploits. The crutch audit is as important as the capability improvement.
- Game rules and configurability let players tailor the AI to their skill level and preferences. The 10 Victoria 3 rules, the modular Advanced AI toggles in Imperator, and even the simple difficulty defines in Stellaris all acknowledge that AI behavior has no single correct answer.
- The compatibility patch system in Victoria 3 turns a technical limitation (invisible modded buildings) into a human workflow (11-attribute format, JavaScript generators, GitHub registration, step-by-step documentation). The engineering is in the tooling as much as the code.
- Clear progressive values (50,000 → 100,000 → 200,000 → 400,000 for PM ai_values) make intent obvious to future maintainers without comments.

The quality of AI modding work is bounded not just by the engine's capabilities, but by the modder's ability to create maintainable, communicable designs. Code that the engine executes correctly but the next maintainer can't understand is a ticking time bomb.

---


## 7. Deep Analysis: Game AI Design Patterns

All three mods can be located within the broader landscape of game AI design -- some patterns they use are well-established, some they reinvent under constraint, and some are genuinely unusual.

### 7.1 The architectural family: rule-based expert systems

All three mods are, at their core, **hand-authored rule-based expert systems** -- a lineage that traces back to 1970s AI research and remains the dominant approach in commercial game AI. The AI doesn't learn, doesn't search a game tree, doesn't optimize a utility function over time. It applies pre-written rules to observed state and acts on the first matching condition. Every `if shortage > threshold then build` trigger is a production rule. Every quantized economy flag is a fact in a working memory. Every event that fires downstream is a rule chaining to the next inference step.

This isn't a limitation of the modder's imagination -- it's a hard constraint of the Clausewitz/Jomini scripting languages. The engines provide events, triggers, and effects. That's the vocabulary. You can't implement a neural network in Paradox script. You can't run Monte Carlo tree search. You can barely do arithmetic. The language enforces rule-based design, and the mods are expert systems whether or not the author thinks of them that way.

What's notable is how much sophistication the mods extract from this primitive substrate. Classical expert systems in industry struggled with knowledge acquisition (how do you get the rules right?), maintenance (how do you keep thousands of rules consistent?), and brittleness (what happens at the boundary between rules?). The mods face all three problems and address them with specific strategies:

- **Knowledge acquisition:** The modder is simultaneously the domain expert and the knowledge engineer. They play the game, observe failures, and encode fixes. The IRC system in Imperator is explicitly a knowledge elicitation tool -- it lets the modder express intent in probabilities rather than opaque weights, reducing the gap between "what I think should happen" and "what the numbers actually produce."
- **Maintenance:** Code generation (Victoria 3's JavaScript toolchain producing 3,000 script values), naming conventions that separate design eras (`gai_*` vs `aai_*` in Stellaris), and the compatibility patch system (200-slot framework with documented workflow) all address the maintenance problem.
- **Brittleness:** Quantization (converting continuous state into categorical flags) is a direct anti-brittleness measure. A rule that fires on "minerals low" is robust to small fluctuations. A rule that fires on `minerals > 47.3` is fragile. The 2D threshold surfaces in Victoria 3's budget health are a sophisticated brittleness mitigation that doesn't appear in most classical expert systems.

### 7.2 The blackboard pattern: shared state driving independent subsystems

The Stellaris mod's central economy calculator is a textbook implementation of the **blackboard architecture** -- one of the oldest patterns in AI systems design (originally from the Hearsay-II speech recognition system, 1970s). A central data structure (the "blackboard") is written by specialist producers and read by independent consumers. The economy calculator writes quantized flags; the building planner, fleet producer, colonization system, robot assembler, and civilian ship spawner all read them independently.

The blackboard pattern solves a specific coordination problem: how do you get independent subsystems to agree on the state of the world without tightly coupling them? The Stellaris mod's answer is clean: one producer, many consumers, no consumer writes to the blackboard. This is a stronger discipline than most blackboard implementations, which typically allow multiple writers and use conflict resolution. The simplicity is enforced by the scripting language's limitations -- there's no easy way to have subsystems feed back into the economy model, so the one-way data flow is a constraint that happens to be good architecture.

Victoria 3's evaluation system is a more complex variant. The "blackboard" includes budget health scores, spending shares, investment pool calculations, military threat assessments, and per-building packed data variables. Multiple phases write to it (preparation, evaluation), and construction reads from it. The weekly loop writes budget tracking data that the main loop reads. This is closer to a multi-writer blackboard with phase-ordered access -- writers are guaranteed not to conflict because the scheduler ensures they run in sequence.

Imperator doesn't have a formal blackboard -- it's the lightest architecture. But the `modifier:` syntax serves a similar role: it provides a standardized read interface to game state that multiple systems query independently. The research efficiency check, governor policy logic, and mercenary recruiting all read from the same modifier values without coordinating with each other.

### 7.3 Utility AI elements in Victoria 3

Victoria 3's evaluation system has the most overlap with **utility AI** -- the approach where each possible action is scored by a utility function and the highest-scoring action is chosen. The priority number (supply/demand level + weight) is a utility score. The construction phase selects the building with the highest priority (lowest number = highest utility). State aptitude scoring adds a second utility dimension for location selection.

But it's not pure utility AI, and most of the deviations are driven by data access constraints rather than preference:

- **No continuous utility function.** A "proper" utility AI implementation would combine all relevant data about each building into one score: profitability projections, PM-specific throughput, goods dependency chains, local pop qualification ratios, interaction effects with existing buildings. The vanilla game's C++ AI can do this -- it has direct access to building definitions, PM values, and can simulate outcomes. The mod cannot. Some values (profitability, state-level pop data) can be queried but at prohibitive computational cost if done per-building-per-state. Others (PM compositions and their exact input/output values) are entirely invisible to the scripting layer. With incomplete inputs, a continuous utility function would be garbage-in-garbage-out -- the math would look precise but the result would be no more accurate than the coarse approximation. Quantizing supply/demand into 22 discrete levels isn't a preference for discreteness; it's using the one signal the mod *can* reliably read (market supply/demand queries) at the granularity that matches the signal's actual information content.
- **Orthogonal concerns instead of a single score.** Pure utility AI collapses everything into one number. Victoria 3 explicitly separates weight (urgency), order (dependency), and offset (confidence) into three independent layers. This is closer to a **multi-criteria decision system** with lexicographic ordering: weight dominates, order breaks ties, offset gates independently. Again, this is largely necessity: if you could dynamically compute a building's full value (including its dependency contribution to other buildings, its profitability given current PMs, its workforce implications), you could fold all of that into a single score. The mod can't, so it separates the concerns it *can* evaluate independently.
- **Hardcoded order and weights.** The build order (`order` attribute) and goods weights are hardcoded values, but they aren't arbitrary expert opinion. They approximate what a dynamic system would compute if it had access to the dependency graph: which goods are inputs to many other buildings, which buildings are prerequisites for economic development. A C++ implementation could build this dependency tree at runtime, compute centrality scores for each good, and derive priority dynamically. The mod encodes the same structural knowledge as static data because the scripting layer can't traverse building definitions or PM compositions. The values *can* be validated against the game data -- and were, during development -- but they can't be *computed* from it at runtime.
- **Gating instead of scoring.** The offset/productivity system doesn't contribute to the priority score -- it gates expansion as a binary yes/no. This separates "should we build this at all?" from "how urgent is it?" -- different questions with different answer types (boolean vs ordinal). A full-information utility function could incorporate profitability projections directly into the score, making the gate unnecessary. Without reliable per-building profitability data, the mod uses the coarser signal it can access (country-wide median earnings as a baseline) and makes a binary decision.
- **Tier-then-randomize instead of argmax.** Classical utility AI picks the single highest-scoring option. Victoria 3's state selection groups candidates into tiers and randomizes within the top tier. Given that the priority scores are already coarse approximations with limited input data, treating near-equal scores as genuinely equivalent is more honest than pretending the highest score is meaningfully better. The randomization also produces variety across games, which pure argmax on imprecise scores would not.

The result is a hybrid: utility-inspired priority scoring layered with rule-based gating, dependency-chain tiebreaking, and randomized location selection. It's shaped more by what the scripting layer can and cannot access than by a theoretical preference for one AI architecture over another. Where the mod has good signal (market supply/demand), it uses it quantitatively. Where signal is absent (PM values, building definitions), it substitutes hardcoded structural knowledge. Where signal is too expensive (per-state profitability), it uses coarse proxies.

### 7.4 Finite state machines in Stellaris

The Stellaris mod uses **finite state machines** (FSMs) in several subsystems, though they're implemented as flag-based state tracking rather than explicit FSM data structures:

- **Fleet repair:** The strongest fleet transitions through states: combat → damage detected → routing to starbase → locked at starbase (speed = -1000) → repair (hull/armor regen +150%) → repair complete → return to duty. Transitions are driven by health thresholds, war state, and 30-day recheck timers. This is a classic FSM with state represented by modifier presence/absence and flag values.
- **Ship delivery:** Ships move through build → queued (busy-slot flag set) → routing to fleet → delivered (busy-slot cleared) or failed (refund). The 6 busy-slot flags per starbase are effectively a capacity-bounded state machine for the shipyard.
- **Transport stuck-fix:** Detect invalid orders → retry → delete and recreate. An escalating recovery FSM.
- **Fleet doctrine:** The corvette-preference/battleship-preference flag is a one-transition FSM -- set at game start, never changed. It's the simplest possible state machine, but it coordinates 40+ downstream decisions.

The wartime cleanup system (`gai_clear.1-.4`) is also an FSM with timed transitions: war entry → force return (720-day timer) → periodic transport cleanup every 180 days → peace → cleanup flag removed.

What's unusual is that these FSMs are implicit -- there's no state machine framework, no transition table. State is tracked through flags and modifiers, transitions happen through event chains. The scripting language doesn't have FSM primitives, so the modder builds FSMs out of the language's primitive elements (flags, timed modifiers, chained events) without necessarily thinking of them as FSMs. The pattern emerges from the constraints.

### 7.5 What's absent -- and what's actually used in practice

Locating these mods in the broader game AI landscape also means identifying what they *don't* do. But this needs grounding in what commercial strategy game AI actually uses, rather than what's theoretically possible. Game AI textbooks describe techniques like MCTS, HTN planning, and reinforcement learning, but the gap between textbook and shipping strategy game AI is large.

**No tree search.** The mods never explore "if I build iron mines now, what happens in 5 years?" There's no minimax, no alpha-beta pruning, no Monte Carlo tree search. Every decision is based on current state. This is the "reactive" limitation discussed in section 6.4. But it's worth asking: do commercial strategy games actually use deep lookahead? Mostly, no. Civilization uses limited tactical search for military unit positioning, but its economic and research AI is reactive -- score current options, pick the best. Total War's campaign AI is similarly next-step. Hearts of Iron IV's front-line AI plans operations but doesn't search a game tree. The complexity of grand strategy state spaces (hundreds of provinces, dozens of interacting economic systems, multi-year time horizons) makes meaningful tree search computationally intractable even with C++ engine access. The mods' reactive approach isn't a modding limitation that commercial AI has solved -- it's the standard approach across the genre, because the state space is too large and the evaluation function too uncertain for lookahead to reliably improve on informed single-step decisions.

**No dynamic planning.** There's no Goal-Oriented Action Planning (GOAP), no Hierarchical Task Network (HTN). The mods don't form explicit goals and work backward to find action sequences. The closest analog is Victoria 3's build order (the `order` attribute), which encodes an implicit plan: "construction before administration before infrastructure before materials." But it's a static, hand-authored dependency chain, not a dynamically computed plan. A true planner would compute the order based on the current state ("this country already has administration, skip to infrastructure"). In practice, GOAP sees use in action games (F.E.A.R., Shadow of Mordor) where the action space is small and well-defined. Strategy games rarely use formal planning systems -- the combinatorial explosion of possible action sequences over economic/diplomatic/military domains makes planning impractical. What strategy games do use is prioritized reactive logic: evaluate current needs, pick the most urgent action, repeat next tick. That's essentially what these mods do. The static build order is a good-enough approximation of what a planner would compute, without the computational cost -- and importantly, without the data access a planner would require.

**No learning.** No reinforcement learning, no parameter tuning from experience, no adaptation across games. Every AI player starts with the same rules and the same weights. What feels like learning is actually hand-tuned parameterization: the modder plays, observes, adjusts weights, and ships a new version. The IRC system in Imperator is an acknowledgment of this -- it's a tool for the modder's learning process (making weight adjustments more interpretable), not for the AI's. Commercial strategy games don't use learning either -- no shipping Civilization, Total War, or Paradox title trains its AI through reinforcement learning or adapts across playthroughs. The reasons are practical: learning AI is hard to QA, can discover degenerate strategies, and behaves inconsistently across games. DeepMind's AlphaStar (StarCraft II) demonstrated that RL can produce superhuman strategy game play, but it required millions of games of training, enormous compute, and produced behavior that was sometimes alien and frustrating to play against. No commercial grand strategy game has adopted this approach. Hand-tuned rules remain the industry standard.

**No opponent modeling.** The mods don't track what other AI players are doing and adjust accordingly. Victoria 3's military threat assessment (averaging the top 6 countries' power) is the closest thing -- it's a simple aggregation of observable state, not a model of other players' strategies. No AI country thinks "my neighbor is building lots of barracks, so I should prepare for war." The diplomacy rework in Stellaris makes threat more local and coalition-building more strategic, but it works through static weight shifts on acceptance factors, not through a dynamic model of other players' intentions. Again, this matches the commercial landscape -- most strategy game AI uses observable state (army size, proximity, diplomatic relations) rather than building predictive models of opponent intent. The exception is some Civilization diplomacy AI that tracks player behavior patterns, but even that is closer to reactive scoring than true opponent modeling.

**No meta-reasoning.** The mods don't reason about their own decision-making -- no "I've been building iron mines for 10 iterations and the shortage isn't improving, maybe I should try something else." The closest mechanism is Victoria 3's productivity requirement (don't expand unprofitable buildings) and downsizing (remove buildings with <40% occupancy for 6+ iterations), which function as crude feedback loops. But they're rule-based corrections, not reflective reasoning. This too is standard for the genre. Commercial strategy AI doesn't meta-reason; it uses the same kind of feedback rules (threshold checks, timeout resets) to prevent stuck behavior.

### 7.6 What these mods do that's unusual in game AI

Despite the primitive substrate, several design choices are genuinely interesting from a game AI perspective -- things that wouldn't be obvious from textbook approaches:

**Dual-system architecture (planner + overrides).** The active planner + static override alignment pattern (section 2.3) doesn't map cleanly to any standard game AI architecture. It's not a layered FSM (the layers aren't FSMs). It's not a behavior tree with decorators (there are no behavior trees). It's closer to a **policy alignment problem**: two independent decision-making systems (the mod's event-driven planner and the vanilla engine's weight-based selector) must produce compatible outputs. The mod solves this by controlling both the planner (through events) and the environment the vanilla system sees (through weight overrides). This is a unique constraint of modding -- you're programming two AI systems at once, one of which you can only influence indirectly. It's reminiscent of principal-agent problems in economics: how do you get an agent (vanilla AI) to do what you want when you can't directly control its actions, only its incentives?

**Quantization as a forced design choice with good side effects.** Most game AI treats continuous values as continuous. These mods aggressively quantize everything into discrete categories -- because Clausewitz script can't reliably do arithmetic. The scripting languages lack floating-point operations, variable-length data structures, and in many cases even basic comparison operators on non-integer values. Quantization is the only way to work with continuous game state. But this forced constraint produces unexpectedly good results: quantized decisions are more stable (no oscillation from floating-point noise), more debuggable (you can name each state), and more composable (boolean flags combine cleanly). The 26-level food income quantization in Stellaris, the 7-level budget health in Victoria 3, and the probability-anchored IRC scale in Imperator are all variations of the same adaptation to the same constraint. The side benefit -- that discrete categories humans can name and reason about are easier to maintain and debug than opaque continuous values -- is real, but it emerged from necessity rather than being chosen for that reason.

**Absent-system creation as the highest-impact category.** Game AI literature focuses heavily on improving existing behaviors -- better pathfinding, better combat tactics, better economic optimization. The Imperator work highlights a category that gets almost no academic attention: systems where the AI does *nothing* because no AI behavior was implemented. Law management, character loyalty, national ideas, road building, diplomatic stances -- each was a complete blank in the vanilla game. The highest-impact improvements came not from optimizing an existing system but from creating a new one where none existed. This suggests a general principle for game AI work: audit for zeros before optimizing for better. An AI that uses 8 game systems competently will almost always outperform one that uses 4 systems brilliantly.

**The corrective layer pattern.** Imperator's fort repositioning and road building represent a pattern that's rare in game AI literature: building a secondary system that observes and corrects the output of a primary system you can't modify. This is closer to **supervisory control** in industrial systems than to typical game AI patterns. The fort optimizer doesn't replace vanilla fort placement -- it lets vanilla place forts, then moves them to better positions. This post-hoc correction pattern could be applied to any game AI where the core logic is inaccessible but the results are observable and modifiable.

**Time-distributed computation as architecture.** Most game AI runs all decision-making in a single frame or tick. These mods spread computation across days and weeks, with explicit idle periods and staggered country offsets. This isn't just an optimization -- it changes the AI's character. A 28-day iteration cycle means the AI is patient by design. It can't panic-react to short-term fluctuations because it won't reconsider until next iteration. The idle buffer (days 15-28 in Victoria 3) is defensive engineering that happens to produce more deliberate-seeming behavior. This is an accidental instance of **bounded rationality** -- the AI's computational limitations force it to make satisficing decisions at a slower cadence, which paradoxically produces better outcomes than trying to optimize every tick.

### 7.7 The satisficing principle

Across all three mods, the overarching design philosophy is **satisficing** rather than optimizing -- a concept from Herbert Simon's bounded rationality theory. The AI doesn't seek the best possible decision. It seeks a decision that's good enough and moves on.

- Stellaris: build the first building that matches the economic flags, don't evaluate all possible buildings.
- Victoria 3: select the first aptitude tier that has valid states, randomize within that tier, don't rank all states globally.
- Imperator: weight research trees toward important targets, don't compute the optimal research path.

This is pragmatic, not lazy. Satisficing is the correct approach when: (a) the search space is too large for exhaustive evaluation, (b) the evaluation function is imprecise, (c) the cost of deciding exceeds the benefit of a marginally better decision, and (d) the decision will be revisited soon anyway. All four conditions hold for strategy game AI in a modding context. The scripting language can't search efficiently, the evaluation captures only a fraction of the relevant state, computation is expensive, and the next monthly/iteration pulse will reconsider everything.

The three mods implement satisficing at different granularities:
- **Stellaris** satisfices at the action level: check a few conditions, pick the first acceptable action.
- **Victoria 3** satisfices at the evaluation level: compute a priority score that's deliberately coarse (integer, not float), accept ties as genuine equivalence.
- **Imperator** satisfices at the knowledge level: use probability-based weights that explicitly accept randomness as a feature, not a bug.

The IRC system is the most self-aware implementation of satisficing in the three mods. By framing weights as probabilities, it explicitly acknowledges that the AI won't always pick the "best" option -- and that this is correct behavior. IRC 75% means "this should usually be chosen, but 25% of the time something else wins, and that's fine." The randomness isn't noise to be eliminated; it's variety to be preserved.

### 7.8 Comparison to commercial strategy game AI

How do these mods compare to the AI in commercial strategy games?

**Civilization series:** Uses a utility-based system where the AI scores each possible action (build, research, unit move) and picks the highest. Limited tactical search exists for military unit positioning, but economic and research AI is fundamentally reactive -- evaluate current options, pick the best. The key architectural difference isn't sophistication but data access: Civilization's AI can read building definitions, compute yields, and evaluate options with full information. The Victoria 3 mod must hardcode every building's properties because the script can't query them. The mod's market signals (supply/demand queries) are a proxy for the internal data it can't access -- coarser, but covering the same decision space.

**Total War series:** Uses behavior trees for tactical combat AI and utility-based scoring for campaign-level decisions. The behavior tree approach -- hierarchical, with fallback nodes and conditional decorators -- is architecturally richer than what Paradox scripting allows. But behavior trees require a runtime framework that Clausewitz/Jomini doesn't provide. The mods approximate behavior tree logic through chained events with cascading conditions. At the campaign level, Total War's AI faces similar challenges to these mods: too many interacting systems for deep lookahead, so it relies on scored reactive decisions -- the same basic approach, with better data access.

**Paradox's own AI in later titles:** Paradox has gradually improved their internal AI, especially in Victoria 3 patches post-1.3.6 and in Hearts of Iron IV. The internal AI and the mods use fundamentally the same approach: rule-based reactive decision-making with scored priorities. The difference is data access and execution capability. The C++ AI can query building definitions, simulate PM outcomes, traverse dependency graphs, and compute profitability projections. The mods must hardcode what the engine can compute, approximate what the engine can simulate, and skip what the engine alone can evaluate. This is the real ceiling of script-layer modding: not architectural sophistication (the mods use the same patterns as the internal AI), but the information barrier between the scripting layer and the game's internal state.

**The common ground:** What's striking is how similar these mods are to commercial strategy game AI in architectural terms. The genre is dominated by rule-based reactive systems with utility-inspired scoring -- not by the search, planning, or learning techniques that game AI textbooks emphasize. The mods and the commercial implementations face the same fundamental problem: grand strategy state spaces are too large, too interconnected, and too poorly defined for formal optimization methods. The practical difference between the mods and commercial AI isn't the approach -- it's the information. Commercial AI can read the game's data structures directly; the mods can only read what the scripting layer exposes. The mods compensate by using observable market signals, hardcoded structural knowledge, and quantized state categories as proxies. That the results are competitive suggests the approach is sound and the binding constraint is data access, not architectural design.

---

## 8. Conclusion

Strategy game AI modding is systems design under severe constraints: limited scripting languages, performance budgets, no runtime introspection, no learning, and the requirement to produce coherent behavior across dozens of interacting subsystems.

The most important decisions are architectural, not algorithmic:
- Quantize the world into stable categories
- Distribute computation over time
- Align active planning with static overrides
- Choose the right relationship with vanilla: coexist, replace, or compensate
- Specialize over generalize
- Target specific failure modes
- Bypass broken systems by simulating their effects
- Build for the data you can access, not the data you wish you had
- Audit for absent systems, not just broken ones

The best AI improvements often aren't the cleverest -- they're the ones that stop the AI from self-destructing. Making the AI "smart" is less valuable than making it "coherent": all its subsystems pulling in the same direction, reading the same quantized state, following the same doctrine. A fleet repair system that freezes damaged ships at a starbase with +150% hull regen is crude, but it prevents the visible, player-frustrating failure of half-destroyed fleets charging back into combat.

Some of the highest-impact work isn't optimization at all -- it's discovering that a system simply doesn't exist and building it from scratch. Law management, character loyalty, national idea selection, road building, diplomatic stance logic in Imperator weren't bad systems that needed fixing. They were absent systems that needed creating. The difference between "fix" and "create" is a fundamental distinction in this kind of work. And the discovery that national ideas were moddable -- when the entire community assumed they weren't -- shows that testing assumptions about what's possible is itself a high-value activity.

The work evolves: from inheriting and extending someone else's code (Stellaris), to designing a complete replacement architecture from scratch (Victoria 3), to building human-friendly tools and filling in missing systems within a team project (Imperator). Each project taught lessons that shaped the next. The constraint shapes the craft -- and the craft shapes the modder. The Stellaris mod's shared economy model inspired Victoria 3's centralized evaluation. Victoria 3's configurability philosophy influenced Imperator's modular Advanced AI rules. Imperator's IRC system was a direct response to the weight inflation observed in Stellaris modding.

Strategy game AI modding is not about making the AI play like a human. It's about making the AI stop playing against itself. Coherence beats cleverness. Robustness beats optimality. And communication -- with the engine, with other modders, with the player community -- is as much a part of the design as the code. The dev diary that explains *why* the AI builds Forums instead of Academies after reaching research cap is as valuable as the code that implements it, because without the explanation, the next person to touch that code will "fix" it back to Academies.


