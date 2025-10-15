# Dungeon Procedural Generation Reference

This document catalogs the systems that build Shattered Pixel Dungeon's procedural floors, with pointers to the concrete code that assembles rooms, paints tiles, and populates hazards, monsters, and loot.

## 1. Level bootstrap and RNG isolation
* `Level.create()` seeds a depth-specific RNG stack, queues guaranteed drops (food, upgrade scrolls, stones, catalysts), and optionally assigns one of several "feelings" that later builders consult for map size, lighting, or feature density adjustments.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/Level.java†L215-L316】
* After a subclass' `build()` succeeds, `setSize()` initializes map arrays with feeling-aware defaults, pathfinding masks, and FOV caches before flags are rebuilt and mobs/items are generated.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/Level.java†L295-L345】
* `createMob()` rotates through a depth-tuned mob queue and rolls champion modifiers, ensuring spawn variety without reusing randomness earmarked for layout work.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/Level.java†L506-L516】

## 2. Room catalogs and selection queues
* `RegularLevel.build()` shuffles the room seed list, repeatedly invokes the active builder until it returns a connection graph, then hands the result to the painter.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/RegularLevel.java†L103-L120】
* `initRooms()` always includes entrance/exit templates, rolls a configurable number of standard rooms (expanded by large-feeling floors), injects the shop when present, enqueues special rooms via `SpecialRoom.initForFloor()`, and draws a region/feeling controlled number of secret rooms.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/RegularLevel.java†L123-L164】
* Standard rooms negotiate their size category via weighted probabilities, where larger categories increase `sizeFactor()`, spawn weights, and connection weights—so big rooms count as multiple placements when quotas are consumed.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/rooms/standard/StandardRoom.java†L36-L141】
* Special rooms alternate equipment-leaning and consumable-leaning archetypes, enforce crystal-key and potion quotas per floor, and schedule laboratory/pit rooms across runs; selection pops from run-scoped queues so rarities balance over multiple floors.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/rooms/special/SpecialRoom.java†L82-L190】
* Secret rooms pre-compute per-region quotas, distribute fractional expectations across the remaining floors, and rotate through their shuffled catalog each time one is spawned.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/rooms/secret/SecretRoom.java†L34-L101】

## 3. Layout builders and graph construction
* `RegularBuilder.setupRooms()` clears residual links, classifies rooms by connection capacity, weights large standards to bias them toward the main loop, and selects a main path length with jitter so large rooms appear early without being deterministic.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/builders/RegularBuilder.java†L89-L129】
* `createBranches()` grows branches from eligible rooms by inserting `ConnectionRoom` tunnels, retrying failed placements, and occasionally re-adding multi-connection rooms to the branchable pool to maintain connectivity variety.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/builders/RegularBuilder.java†L143-L240】
* `LoopBuilder` turns the weighted path list into a single loop, inserting tunnel segments based on probabilistic counts, laying rooms around a curved path parameterized by exponent/intensity/offset, and finally stitching the loop closed before adding branches and random extra connections.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/builders/LoopBuilder.java†L49-L174】
* `FigureEightBuilder` selects a landmark hub, splits the main rooms between two interlocking loops, builds and closes each loop independently, and then uses the shared landmark plus branch growth to produce a figure-eight topology.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/builders/FigureEightBuilder.java†L69-L259】

## 4. Door topology and adjacency data
* Rooms share `Room.Door` instances so both endpoints agree on type upgrades; only passable/regular doors count as edges when builders compute reachability, preventing sealed rooms from breaking pathfinding.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/rooms/Room.java†L407-L485】
* `RegularPainter.placeDoors()` finds valid intersections, creates doors where absent, and records them for both rooms so later logic can merge openings or upgrade door types in-place.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/painters/RegularPainter.java†L124-L184】
* `paintDoors()` rolls hidden-door odds that scale with depth, prevents isolating standard rooms by re-running graph distance checks, enforces tutorial-specific hidden entrances, then maps final door types to terrain tiles.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/painters/RegularPainter.java†L186-L300】
* Before committing, `mergeRooms()` attempts to carve arches between adjacent standards, provided both templates allow merging and the overlap is wide enough to stay readable.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/painters/RegularPainter.java†L304-L357】

## 5. Painting rooms, fluids, and foliage
* `RegularPainter.paint()` recenters the entire room graph within a padded bounding box, resizes the map, randomizes paint order, and defers to each room's `paint()` implementation before handling shared features under a separate RNG seed to keep decoration variance from perturbing structural randomness.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/painters/RegularPainter.java†L83-L155】
* Water and grass masks come from the `Patch.generate()` cellular automaton, which smooths random seeds and, when requested, post-processes to hit a target fill ratio—producing organic blobs constrained by each room's placement whitelist.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/painters/RegularPainter.java†L360-L421】【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/Patch.java†L21-L118】
* Trap placement gathers candidate tiles, optionally filters hallway cells, caps density at one per five tiles, and for trap-feeling floors multiplies attempts while revealing the surplus traps; revelation probability accumulates so some traps start visible even without the feeling.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/painters/RegularPainter.java†L423-L494】

## 6. Monster population and spacing rules
* `RegularLevel.mobLimit()` scales spawn counts with depth and large feelings, while floor 1 short-circuits to zero unless the player already owns the Amulet.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/RegularLevel.java†L204-L216】
* `createMobs()` weights standard rooms by `mobSpawnWeight()`, keeps spawns outside the entrance FOV and an eight-tile walk radius, respects passability plus large-creature open-space checks, and occasionally doubles up spawns in the same room on later floors.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/RegularLevel.java†L218-L315】
* After placement, tall grasses underneath mobs are flattened to keep LOS predictable.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/RegularLevel.java†L307-L313】

## 7. Item drops, mimics, and guaranteed loot
* `createItems()` rolls 3–5 base drops (plus two on large feelings), chooses item categories from the main generator, and converts some treasure tiles into chests, skeletons, or mimics based on depth-scaled odds and active mimic chance multipliers.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/RegularLevel.java†L376-L446】
* Guaranteed spawns queued during `Level.create()` are delivered here; catalysts force a paired golden key drop, and tall grasses are flattened beneath every placed item to avoid blocking visibility.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/RegularLevel.java†L449-L466】
* Separate RNG pushes isolate optional drops (torches for darkness challenge, bones loot, Dried Rose petals) so meta-progression checks do not perturb the main layout RNG stream.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/RegularLevel.java†L468-L520】

## 8. Feeling and subclass-specific tuning
* Feelings influence trap counts, grass/water density, map padding, and other painter decisions by checking `Level.feeling` throughout painting and placement logic.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/painters/RegularPainter.java†L79-L494】
* Concrete level subclasses override room counts, trap tables, and painter presets; for instance, `SewerLevel` biases toward watery/grass layouts when matching feelings and injects region-specific trap pools before falling back to shared spawn logic.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/SewerLevel.java†L87-L140】

## 9. Runtime door interaction
* Once generated, door tiles use `Door.enter()`/`Door.leave()` to toggle between open and closed states, respecting occupant counts and adjacent heaps so procedural placement integrates with gameplay rules.【F:core/src/main/java/com/shatteredpixel/shatteredpixeldungeon/levels/features/Door.java†L33-L59】

Together these components combine catalog-driven room templates with RNG-controlled builders and painters, producing varied-yet-consistent dungeon floors filled with appropriately spaced enemies, hazards, and rewards.
