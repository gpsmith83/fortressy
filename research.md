# **Architectural Analysis and Technical Feature Backlog for a Rampart Clone in Godot 4.6.1**

## **Executive Summary and Historical Context**

The 1990 arcade title *Rampart*, developed by Atari Games, occupies a seminal position in the history of strategic video game design, widely acknowledged as a foundational precursor to the modern tower defense genre that would gain immense popularity in subsequent decades.1 By synthesizing the frantic, real-time, cursor-based artillery combat of earlier arcade titles like *Missile Command* with the time-restricted, grid-based spatial puzzle reconstruction popularized by *Tetris*, the application demands a highly specialized, tightly coupled systemic architecture.3 Translating these legacy mechanics into a contemporary, object-oriented engine environment—specifically Godot 4.6.1—requires a meticulous deconstruction of the original phase-based game loop, grid logic, ballistics, and territory validation algorithms.3

The transition from a 1990s arcade hardware paradigm to the Godot 4.6.1 engine necessitates a shift toward modular, node-based development. Godot 4.6.1 introduces several critical environmental updates that directly influence the architectural approach to a *Rampart* clone.6 Notably, the complete deprecation of the monolithic TileMap node in favor of the single-layer TileMapLayer system requires a decoupled approach to grid management, where terrain, structures, and hazards exist as discrete, overlapping layers.7 Furthermore, the integration of Jolt Physics as the default 3D and robust 2D backend ensures deterministic collision detection, which is vital for the high volume of overlapping projectiles and structural damage calculations inherent to the game.6 Finally, engine-level improvements to Screen Space Reflections (SSR) and workflow enhancements, such as decoupled Select and Transform modes, provide developers with advanced tools to render the naval environments and rapidly prototype grid-based levels.9

This report provides an exhaustive technical analysis of *Rampart's* mechanics. It examines the underlying mathematical models required for parabolic projectile trajectories, state machine transitions for phase management, territory calculation protocols utilizing flood fill algorithms, and deterministic enemy artificial intelligence.3 By mapping these arcade mechanics to modern engine paradigms, the analysis culminates in a comprehensive, highly detailed feature backlog optimized for production in Godot 4.6.1.

## **Engine Environment and Godot 4.6.1 Optimizations**

Before defining the specific game logic, it is imperative to establish how the foundational features of Godot 4.6.1 will be leveraged to recreate and enhance the *Rampart* experience. The engine's architecture dictates that every entity, manager, and interface element must inherit from the Node class, structured within a hierarchical scene tree.

The most significant shift for grid-based game developers in recent Godot iterations is the handling of tile maps. Historically, a single TileMap node managed multiple layers of grid data. As of Godot 4.6.1, this approach has been entirely superseded by the TileMapLayer node, which enforces a strict one-to-one relationship between a node and a grid layer.7 To recreate *Rampart*, the project architecture must instantiate an array of TileMapLayer nodes sharing a common TileSet resource.8 This separation allows developers to isolate collision queries and visual updates. For instance, a raycast from an enemy ship evaluating navigable water space only needs to query the Water\_Layer node, completely ignoring the Structure\_Layer or Hazard\_Layer. This localized querying significantly reduces the computational overhead during the intensive Build Phase and AI navigation cycles.8

Furthermore, the default inclusion of Jolt Physics in Godot 4.6.1 provides a highly optimized backend for spatial queries.6 While *Rampart* operates on a rigid 2D grid, the combat phase involves dozens of independent cannonballs, ship hitboxes, and explosion radii interacting simultaneously. Jolt's optimized broad-phase collision detection ensures that overlapping Area2D nodes—representing explosion blast zones and ship hurtboxes—resolve their interactions deterministically without triggering frame drops, even when the arcade maximum of sixteen active enemy vessels is reached.6

Visually, Godot 4.6.1 offers an overhauled Screen Space Reflection (SSR) pipeline.9 While *Rampart* is traditionally a 2D top-down game, modern adaptations often employ 2.5D visual styles, utilizing 3D models rendered via orthographic cameras. If developers choose to implement a 2.5D aesthetic, the new SSR implementation allows for highly realistic, performant water reflections. Developers can configure SSR to operate at half-resolution, maintaining visual fidelity of burning ships and crumbling castles reflecting on the ocean surface while preserving the strict 60 FPS minimum required for arcade-style input latency.9

Finally, debugging complex state machines and grid algorithms is streamlined in 4.6.1. The ability to click terminal errors to jump directly to the offending line of GDScript, coupled with the capacity to dynamically throttle the engine's execution speed during runtime, allows engineers to visually track the expansion of the Flood Fill algorithm or the state transitions of the AI frame-by-frame, vastly improving the iteration cycle during development.13

## **Finite State Machine (FSM) Architecture**

The operational flow of *Rampart* is defined by a rigid, cyclical sequence of distinct gameplay phases. The game does not allow overlapping modes of interaction; when the player is building, they cannot shoot, and when they are shooting, they cannot build.3 This strict segregation necessitates the implementation of a Finite State Machine (FSM) architectural pattern to ensure that game logic, user input processing, and rendering rules are isolated to their appropriate contexts.5

In a Godot 4.6.1 environment, an FSM is optimally implemented using a node-based hierarchy.14 Rather than relying on monolithic scripts filled with nested if/match statements, the FSM utilizes a parent GameManager node with discrete child nodes representing each state (e.g., StateSetup, StateBattle, StateBuild, StateCannon). Each child node extends a virtual base State class containing \_enter\_state(), \_process\_state(), \_physics\_process\_state(), and \_exit\_state() functions.5 The GameManager holds a reference to the current\_state and routes all engine callbacks (\_process, \_input) explicitly to that active node.

The cyclical nature of *Rampart* relies entirely on timer-based transitions. The GameManager must instantiate global Timer nodes that dictate the strict duration of the Battle and Build phases.4 When a state is entered, it connects to the relevant timer's timeout signal. Upon receiving this signal, the active state triggers its \_exit\_state() cleanup protocol—such as resolving remaining projectiles in the air or locking the grid—and emits a transition signal to the GameManager to instantiate the subsequent phase.14 This decoupled architecture ensures that if a developer wishes to add a new phase (e.g., a "Bonus Round" or "Boss Transition"), they can simply snap a new state node into the FSM without rewriting the core loop.15

## **Phase 1: Setup and Territory Initialization**

Upon level initialization, the player is presented with a macro-view of the available geography. The system requires the player to select a specific starting location, choosing a "Home Castle" from several pre-placed structures on the map.3 This initial choice dictates the foundational geometry of the player's territory and profoundly influences the strategic viability of their defense against naval bombardment.

When the player selects a castle using the cursor, the FSM triggers the SetupState initialization protocol. The engine must automatically generate a preliminary perimeter wall around the chosen structure.3 In Godot, this is achieved by executing an automated subroutine that identifies the grid coordinates immediately adjacent to the castle's ![][image1] or ![][image2] footprint on the TileMapLayer\_Structures. The script loops through these adjacent coordinates, setting their cell values to the appropriate wall tile identifier.

Once the perimeter is established, the game instantly converts the enclosed spatial area into claimed territory.3 Visually, this is represented by modulating the color of the ground tiles within the perimeter or swapping the TileMapLayer\_Ground cells to a "claimed" texture variant. Mechanically, the system initializes a base allocation of cannons—typically two or three depending on whether the player is starting fresh or utilizing a continuation credit—and immediately transitions the FSM into the Battle Phase, denying the player any preliminary preparation time.12

## **Phase 2: Combat and Artillery Ballistics**

The Battle Phase shifts the engine from a static grid interface into a frantic, real-time artillery simulation.12 Governed by a strict, visible countdown timer, players manipulate a crosshair across the two-dimensional screen space to target an invading naval armada.3 The mechanical depth of this phase relies on the intricate simulation of projectile ballistics, target leading, and sequential firing constraints.

## **The Sequential Cannon Queue**

A defining systemic limitation in *Rampart* is that each placed cannon may only possess one active projectile in the air at any given moment.12 When a player possesses multiple cannons, they do not fire simultaneously. Instead, cannons fire sequentially in the exact chronological order in which they were placed during the Cannon Placement Phase.12

To replicate this in Godot 4.6.1, the StateBattle node must maintain an Array functioning as a continuous loop queue containing references to every active cannon node within the player's territory. When the player inputs a "fire" command, the script iterates through the array starting from an active\_cannon\_index. It evaluates a boolean flag on the cannon node, is\_reloading or has\_projectile\_in\_air. If the cannon at the current index is available, it calculates the trajectory to the crosshair, spawns the projectile, sets its flag to true, and increments the active\_cannon\_index to the next available weapon.12 If all cannons currently have projectiles in flight, the input command is ignored, emitting a "dull click" audio cue to provide negative feedback to the player.

This mechanical constraint inherently forces the player to prioritize proximate targets over distant ones to maximize their overall rate of fire.12 Distant targets require high-arcing trajectories that consume more time before impact, thereby keeping the firing cannon locked in an unavailable state for a longer duration.12

## **Mathematical Modeling of Parabolic Trajectories**

Unlike linear, top-down arcade shooters, *Rampart's* ballistics rely on pseudo-3D parabolic arcs.3 The time a cannonball remains in flight must be directly proportional to the Euclidean distance between the firing cannon's coordinates and the crosshair's target coordinates.

In a 2D engine like Godot, this trajectory is simulated by mathematically separating the horizontal ground movement from the vertical visual arc. Let the initial firing coordinate be ![][image3] and the target coordinate be ![][image4]. The ground distance ![][image5] is calculated as ![][image6]. The total flight time ![][image7] is determined by ![][image8], where ![][image9] is an arbitrary constant scaling factor determining the general speed of all projectiles.12

The projectile node moves linearly from ![][image10] to ![][image11] over the duration of ![][image7], maintaining a constant horizontal velocity vector ![][image12]. Simultaneously, the visual sprite of the cannonball must follow a parabolic curve on the Z-axis (represented as an offset on the Y-axis in 2D space) to simulate height.12 This vertical offset ![][image13] at any given time ![][image14] follows the quadratic equation: ![][image15] where ![][image16] is the peak height of the arc (which should scale dynamically based on the total distance ![][image5]), and ![][image17] is the coefficient defining the steepness of the parabola. By applying this ![][image18] calculation as an offset to the Sprite2D node's position relative to its parent CharacterBody2D or Area2D root, the system generates a convincing 3D arc within a strictly 2D coordinate space.12

Upon reaching ![][image19], the vertical offset returns to zero, simulating ground impact. The system then frees the projectile node, spawns an explosion Area2D to detect overlapping enemy hitboxes, and sends a signal back to the originating cannon to reset its has\_projectile\_in\_air flag, making it available for the next firing sequence.

## **Phase 3: The Construction and Repair Phase**

At the exact moment the Battle Phase timer expires, the FSM instantly halts all input, waits for any remaining projectiles in the air to resolve their impacts, and transitions to the Build Phase.12 Operating under severe time constraints—often measured in a handful of seconds—the Build Phase introduces mechanics heavily inspired by *Tetris*.3 The primary objective is to continuously repair breached walls and expand the perimeter to enclose additional space.

## **Procedural Tetromino Generation**

The player is provided with randomly generated, multi-tile wall sections, varying from simple linear tetrominoes to complex, asymmetric polyominoes.4 In Godot 4.6.1, these shapes must be generated procedurally from a predefined dictionary. Each shape is structurally defined as an Array of Vector2i coordinates relative to a central pivot point (0, 0).17

For example, a standard 3-tile "L" shape would be defined as:

\[Vector2i(0, \-1), Vector2i(0, 0), Vector2i(1, 0)\]

When the player inputs a rotation command (typically a secondary button press), the system applies a 90-degree rotational matrix to the vector array.17 The mathematical transformation for a 90-degree clockwise rotation on a 2D Cartesian plane is standard: ![][image20]. The system iterates over the array, applying this transformation to each relative coordinate, thereby updating the visual orientation of the floating wall piece tracking the player's cursor.

## **Spatial Validation and Constraint Checking**

The most critical sub-system of the Build Phase is the spatial validation algorithm.19 Players cannot place wall pieces indiscriminately; they cannot build over existing structures, water tiles, enemy units, or battle-damaged terrain.4

As the player moves the cursor across the grid, the engine must convert the global mouse coordinates into grid coordinates using the TileMapLayer.local\_to\_map() function.21 For every frame, the system takes the cursor's current grid position and adds the relative Vector2i coordinates of the active tetromino array to calculate the exact global grid tiles the piece intends to occupy.18

The engine then queries the relevant TileMapLayer nodes and object arrays at those specific coordinates. If a target coordinate intersects with a tile mapped to a non-buildable Custom Data Layer (e.g., water or a flaming crater) or intersects with the grid coordinate of a dynamic entity (like an enemy Grunt or an existing cannon), the placement is deemed invalid.12 The visual indicator of the tetromino is typically modulated to a semi-transparent red color to signal the invalid state to the player.19

If all target coordinates return empty, buildable terrain, the visual indicator turns green. Upon the player issuing the placement command, the validation logic is executed one final time. If successful, the engine permanently inscribes the specific tile identifiers into the TileMapLayer\_Structures at the target coordinates, updates the physics and navigation grids, and immediately generates the next random tetromino.7 It is a foundational rule of *Rampart* that once a block is committed to the grid, it cannot be retracted, moved, or deleted by the player during the Build Phase, punishing misplaced inputs severely.3

## **The Survival Condition**

The Build Phase acts as the primary fail-state mechanism for the entire game. The player must successfully construct an unbroken loop of wall tiles that entirely surrounds at least one castle entity before the timer expires.4 This enclosed castle does not strictly need to be the original "Home" castle chosen at the start of the game, allowing for strategic abandonment of heavily damaged areas.3 If the timer reaches zero and the territory validation algorithm fails to detect a fully enclosed castle, the FSM triggers a catastrophic failure state, deducting a continuation credit from the player.4

## **Territory Validation: Flood Fill Algorithm Implementation**

The most computationally intensive and mathematically complex requirement in replicating *Rampart* is the instantaneous calculation of enclosed territory. At the exact moment the Build Phase timer expires, the game must algorithmically determine if a closed loop of wall entities fully surrounds a given castle.12

Attempting to calculate every possible permutation of wall connections, diagonals, and corners is mathematically inefficient and highly prone to edge-case failures. The mathematically optimal approach relies on a space-filling algorithm—specifically, the Flood Fill algorithm—initiated from the spatial origin point of the castle itself.10

## **Algorithm Architecture in GDScript**

The Flood Fill algorithm operates by systematically exploring the grid outward from a starting node, bounding its expansion upon encountering predefined obstacles.10 In Godot 4.6.1, this is best implemented using a queue-based Breadth-First Search (BFS) or a stack-based Depth-First Search (DFS) routine executed entirely within GDScript.10 Given the constraints of a grid system, BFS ensures a uniform radial expansion.

1. **Initialization and Bounding:** The algorithm identifies the grid coordinates of a Castle node. A queue array ![][image21] is instantiated, and the initial coordinate is appended.10 Simultaneously, an empty dictionary or bitmask array named visited is created to track checked coordinates, preventing infinite recursion loops.  
2. **Iterative Expansion:** A while loop commences, executing as long as ![][image21] is not empty.10 The system pops the first coordinate from the front of the queue and inspects its four adjacent cardinal neighbors (North, South, East, West) using Vector2i additions.10 Diagonals are strictly ignored in *Rampart's* enclosure rules, as walls must connect cardinally to form a seal.  
3. **Boundary Condition Checks:** For each neighboring tile, the system queries the TileMapLayer stack.7  
   * If the neighbor tile is a Wall or a Cannon, the algorithm halts expansion in that specific directional vector. It does *not* add the coordinate to ![][image21].  
   * If the neighbor is an empty space or existing territory, and it does not already exist within the visited dictionary, it is marked as visited and pushed to the back of ![][image21].  
4. **Validation Failure Triggers:** The critical logic involves detecting a breach. If the expansion algorithm queries a coordinate and determines that it lies outside the strict map boundaries (the absolute edge of the playfield) or intersects with a Water tile on the base layer without first encountering a Wall, the territory is definitively "open" or "breached".4 The loop instantly breaks, returning a false boolean, flagging the castle as un-enclosed.  
5. **Validation Success and Data Processing:** If the while loop completely exhausts ![][image21] without ever encountering a map edge or water tile, the entire area is proven to be sealed.10 The algorithm returns true.

## **Performance Optimizations**

Because this algorithm must execute instantaneously across multiple castles within a single frame to determine the player's survival state, performance is paramount. In Godot 4.6.1, native GDScript arrays and dictionaries are highly optimized, but iterating over hundreds of tiles can still cause micro-stutters.24

To optimize the Flood Fill, developers should leverage the TileMapLayer API to its fullest.7 Instead of querying multiple layers sequentially (e.g., checking the hazard layer, then the structure layer, then the terrain layer), developers should maintain a custom binary bitmask representing "solid" vs. "passable" tiles that updates only when a piece is placed or destroyed.7 The Flood Fill algorithm then only needs to query this single, flat, one-dimensional boolean array, allowing the BFS to execute in a fraction of a millisecond.

Upon a successful enclosure validation, the engine utilizes the stored visited dictionary. By iterating through all the coordinates stored during the expansion, the system can instantly paint the TileMapLayer\_Ground tiles at those exact locations with the "Claimed Territory" visual variants, effectively rewarding the player with usable landmass.3

## **Phase 4: Cannon Placement and Reward Systems**

If the Flood Fill algorithm successfully validates at least one enclosed castle, the player survives the round and the FSM transitions to the Cannon Placement Phase.22 This phase acts as the primary reward mechanism, expanding the player's offensive capabilities for the subsequent Battle Phase.

The system calculates the artillery reward based on the number of successfully enclosed castles.12 The baseline reward is two additional cannons for successfully securing the primary "Home" castle.12 For every supplementary, distinct castle enclosed within the unbroken perimeter, the system awards one additional cannon.12

The player is given a very brief window—approximately ten seconds—to place these cannons.12 Mechanically, cannons are treated identically to tetrominoes, functioning as a static ![][image1] grid-sized structure.12 The placement validation algorithm utilized in the Build Phase is reused here, ensuring cannons do not overlap walls, water, or other structures.19 However, an additional constraint is applied: cannons *must* be placed within the coordinates validated as "Claimed Territory" by the recent Flood Fill operation.12 They cannot be placed in neutral or enemy-controlled space.

Strategic placement during this phase is paramount to high-level play. Because cannons occupy a significant ![][image1] footprint, placing too many tightly clustered cannons dramatically reduces the available open grid space.12 This self-inflicted space restriction makes it exceptionally difficult to fit the randomly generated, oddly shaped tetrominoes during future Build Phases.12 Furthermore, expert strategy dictates that cannons should be placed as far away from the outer perimeter walls as possible.27 If a cannon is placed directly adjacent to an outer wall near a map edge or water, and that wall section is subsequently destroyed, the tight clearance makes it geometrically impossible to fit a replacement wall piece, dooming the castle to permanent exposure.12

## **Artificial Intelligence and Entity Behaviors**

The artificial intelligence governing the naval armada and ground forces in *Rampart* operates on a blend of rigid, deterministic pathing and rudimentary target acquisition. Implementing this AI in Godot 4.6.1 requires selecting the appropriate spatial querying tools.28

While modern Godot features sophisticated continuous pathfinding meshes like NavigationRegion2D and NavigationAgent2D, utilizing these tools would result in smooth, organic movement that fundamentally breaks the distinct arcade feel of *Rampart*.28 The original game is intrinsically bound to its grid. Therefore, simulating the deterministic, rigid movement rules of the ships is far more architecturally authentic when achieved using grid-snapped mechanics and RayCast2D sensors.29

## **Naval Armada State Management**

Enemy ships travel in straight lines across the eight cardinal and diagonal directions. They do not utilize complex A\* pathfinding to navigate around obstacles.12 Instead, they project RayCast2D vectors along their current trajectory.29 When the raycast detects a collision with a shoreline, the screen boundary, or another vessel, the ship's internal state machine triggers a course correction routine.12 The ship halts, pivots 45 or 90 degrees to an unobstructed vector, and resumes linear movement.12

The fleet composition is managed by a centralized FleetManager node, which caps the total active ships on-screen at an absolute maximum of sixteen to prevent performance degradation and visual clutter.12 The manager spawns distinct classes of vessels based on a progressively scaling difficulty table.27

| Enemy Classification | Physical Characteristics | Hit Points | Behavioral Profile | Strategic Threat Level |
| :---- | :---- | :---- | :---- | :---- |
| **Regular Ship** | Single sail, dark hull | 1 \- 3 Hits | Navigates cardinally/diagonally; strictly stops at shorelines.12 | Low. Often ignored by expert players in favor of higher threats.27 |
| **Lander / Transport** | Double-mast sails | 4 Hits | Actively navigates toward shorelines. Pauses at edges to deploy ground troops (Grunts).27 | High. Requires immediate interception before troops are deployed.27 |
| **Red Ship** | Red hull / red deck | 5 Hits | Aggressive firing patterns. Fires incendiary projectiles at player walls. Successful hits create craters.27 | Critical. Craters permanently block building and can force fail states.27 |
| **Dark Ship** | Dark profile | Variable | Deploys larger clustered armies specifically on diagonal beach corners.27 | Moderate. Can be ignored if the corner is pre-emptively walled off.27 |
| **Boss Ship** | Massive multi-cannon | Variable | Introduced in Super SNES ports; requires sustained, concentrated bombardment.3 | Extreme (If implemented as an extended console feature). |

## **Ground Forces: The Grunt Menace**

The most complex AI challenge involves the "Grunts," which are small, mobile tank units deployed onto the shorelines by Landers and Dark Ships.27 Unlike ships, which operate continuously, Grunts are bound to a highly specific temporal rule: they move and multiply *only* during the Build Phase.12

During the Battle Phase, Grunts remain completely static, acting as passive targets.12 However, the moment the FSM transitions to the Build Phase, the Grunts activate. Their AI utilizes a rudimentary pathfinding algorithm directly targeting the grid coordinates of the player's nearest castle. As they march inward, they function as dynamic physical blockers; the player's tetromino validation script will reject any placement attempting to overlap a Grunt's current grid position.12 Furthermore, if left unchecked across multiple rounds, Grunts will automatically batter down castle structures, permanently removing them from the board.33

The counter-play to Grunts involves an algorithmic intersection with the Flood Fill system. If a player successfully builds a sealed perimeter of walls that completely surrounds a group of Grunts, the Flood Fill algorithm immediately detects the enclosure.12 The engine then iterates through all entities contained within that specific enclosed coordinate space, triggering a mandatory queue\_free() deletion on the trapped Grunts, instantly eliminating the threat.12

## **Emergent Strategies and Environmental Hazards**

The intersection of *Rampart's* strict rulesets generates highly complex emergent strategies that the engine architecture must flawlessly support. The most prominent hazard dictating these strategies is the "Crater."

When a Red Ship successfully fires an incendiary projectile that strikes a land tile, it spawns a flaming crater entity.27 Craters function as absolute, hard obstacles; the spatial validation algorithm strictly forbids placing wall pieces over them.12 Furthermore, craters are persistent, remaining active on the grid for exactly three complete rounds before their internal life\_timer expires and the tile reverts to buildable terrain.27

## **The Friendly Fire Mechanic**

Because craters are so devastating to the reconstruction effort, expert players employ a counter-intuitive strategy: intentionally firing upon their own walls.16

This strategy relies on a nuanced distinction in the damage calculation architecture. A wall destroyed by friendly fire simply disappears, reverting the tile to open, buildable terrain.16 Conversely, a wall destroyed by an enemy Red Ship transforms into an unbuildable crater.16 The combat architecture must therefore implement strict friendly fire rules. When an explosion Area2D overlaps with a Wall node, the script must verify the damage\_source variable of the projectile.27 If the source is determined to be the AI, the script spawns a crater. If the source is determined to be the player, the script simply calls queue\_free() on the wall, leaving the ground pristine.27

## **The Summer Home Strategy**

The accumulation of craters and Grunts can occasionally render a player's primary castle mathematically impossible to enclose within the strict time limits. This inevitability gives rise to the "Summer Home" strategy.12

If the primary castle is overwhelmed, the player may intentionally abandon it, utilizing their remaining tetrominoes to desperately enclose a distant, secondary castle on the opposite side of the map.12 Because the survival condition only mandates that *at least one* castle be enclosed, this action satisfies the logic.12 The FSM and Flood Fill algorithms must seamlessly accommodate this fluid shift in operational bases, stripping the original castle of its "Home" status and transferring the territory calculation origin point to the newly secured structure.12

## **Scoring, Progression, and Audio-Visual Systems**

*Rampart* utilizes a continuous progression loop that scales in difficulty by increasing the spawn rate, velocity, and composition of the enemy armada across undocumented round thresholds.3 Advancing to subsequent geographical levels requires surviving a hidden quota of rounds or destroying a critical mass of vessels.12

Scoring is heavily segmented, functioning as both a leaderboard metric and a strategic incentive for aggressive expansion.33 The exact point allocations are as follows:

| Action Trigger | Point Value | Architectural Implementation Notes |
| :---- | :---- | :---- |
| **Destroy Ship** | 30 | Applied immediately upon the health value of any vessel reaching zero.33 |
| **Destroy Grunt** | 12 | Applied per unit eliminated via direct cannon fire or via the algorithmic enclosure method.33 |
| **Home Castle Bonus** | 150 | Awarded at the conclusion of a successful Build Phase for securing the primary castle.33 |
| **Other Castle Bonus** | 50 | Awarded iteratively for each secondary castle validated by the Flood Fill algorithm.33 |
| **Territory Bonus** | 7 per square | Calculated by multiplying the total number of enclosed grid tiles returned by the Flood Fill by 7\.33 |
| **Difficulty Bonus** | 5,000 | A flat bonus applied if the player intentionally bypasses the "Recruit" level starting map.32 |

## **Audio Feedback and Godot Integration**

The sensory feedback loop is essential to maintaining the high-pressure arcade experience. The original hardware relied heavily on digitized vocal cues (e.g., "Ready\! Aim\! Fire\!\!", "Prepare for Battle") to notify players of impending FSM state transitions.3

In Godot 4.6.1, an AudioManager singleton must be implemented to preload these .wav files and trigger them dynamically via global signals emitted by the central GameManager.35 Sound effects must be meticulously categorized into distinct audio buses (SFX, Voice, Music, UI). This categorization is critical because it allows the developer to apply automated audio ducking—briefly lowering the volume of the chaotic explosion SFX and frantic background music whenever a critical voice line is triggered, ensuring the player never misses a phase transition warning.36

## **Multiplayer Architecture**

The multiplayer component is a defining hallmark of the *Rampart* experience, originally supporting up to three players simultaneously via an expansive control board featuring individual trackballs.3 Replicating this in Godot requires adapting the engine's viewport and input routing systems.

Local multiplayer can be implemented utilizing either a shared-screen orthographic camera (requiring all players to share a single, massive landmass) or a split-screen setup utilizing multiple SubViewport nodes. The systemic rules shift considerably in competitive multiplayer formats 3:

1. **Target Acquisition Substitution:** The AI armada spawner is entirely disabled. Players must aim their crosshairs at opposing players' walls rather than ships.3  
2. **Environmental Simplification:** To balance the pacing, Red Ship craters and AI Grunts are completely removed from the logic pool.3  
3. **Spontaneous Combustion:** To prevent defensive stagnation and infinitely stalemated Build Phases, a hidden global timer is introduced. Walls may spontaneously degrade or explode over time without any direct player fire, forcing players to continuously repair even in the absence of an attack.3  
4. **Win Conditions:** Victory is evaluated based on a maximum round limit (where the winner is determined by the highest accrued score) or via a survival metric where a player loses their ability to repair their castle across three continuation credits.3 Upon elimination, a specific visual sequence—traditionally depicting the losing player's commander being forced to "walk the plank" or face an executioner—is triggered by the FSM.34

## ---

**Comprehensive Feature Backlog**

The following backlog translates the exhaustive systemic deconstruction detailed above into actionable development epics, user stories, and technical acceptance criteria. This backlog is strictly tailored for implementation utilizing the nodes, tools, and scripting paradigms native to Godot 4.6.1.

## **Epic 1: State Machine and Core Game Loop Architecture**

This epic covers the foundational skeletal structure of the application, managing the strict, timer-based phase transitions that govern all gameplay systems.

| ID | Feature Name | Description | Technical Acceptance Criteria |
| :---- | :---- | :---- | :---- |
| 1.1 | **Node-Based FSM** | Implement a Finite State Machine using parent-child node hierarchies to handle phases. | \- Base virtual State class created with \_enter\_state, \_process\_state, and \_exit\_state methods. \- Parent GameManager node instantiates and manages transitions between StateSetup, StateBattle, StateBuild, and StateCannon. |
| 1.2 | **Phase Control Timers** | Global countdown timers dictating the duration of Battle and Build phases. | \- Independent Timer nodes connected to the GameManager. \- Expiration of BattleTimer halts all user input, waits for active\_projectiles \== 0, and emits transition signal to StateBuild. |
| 1.3 | **Continuation Logic** | Manage the 3-credit arcade life system and catastrophic failure states. | \- Global singleton tracks an integer player\_credits variable. \- On territory validation failure, decrement credit. Trigger StateGameOver if credits reach 0\.4 |
| 1.4 | **Main Menu & Level Select** | Interface for selecting the starting battlefield and difficulty constraints. | \- Control nodes utilizing TextureRect and Button classes. \- Selecting a map passes a data dictionary (enemy spawn rates, map layout) to the GameManager via SceneTree.change\_scene\_to\_file(). |

## **Epic 2: Grid Systems and Territory Calculation**

This epic focuses on the TileMapLayer integration, spatial queries, and the mathematical detection of player territory boundaries.

| ID | Feature Name | Description | Technical Acceptance Criteria |
| :---- | :---- | :---- | :---- |
| 2.1 | **TileMapLayer Setup** | Establish decoupled, distinct layers for terrain, structures, and dynamic hazards. | \- Instantiate individual TileMapLayer nodes for Water, Ground, Walls, Hazards, and Cursor.8 \- Define Custom Data Layers within the TileSet resource to flag specific tiles mathematically as buildable (bool) or is\_water (bool). |
| 2.2 | **Flood Fill Validation** | Calculate enclosed territory via an outward search originating from the home castle. | \- Breadth-First Search (BFS) algorithm written in optimized GDScript.10 \- Traverses grid cells radially; halts pathing at Wall tiles. \- Immediately returns false if map\_boundary or is\_water tile is reached.12 |
| 2.3 | **Territory Claim Visuals** | Visually update grid cells that have been successfully enclosed by the Flood Fill. | \- Upon FloodFill returning true, iterate through the algorithm's visited array and use set\_cell() to swap ground tiles to a visually distinct "Claimed" texture variant.3 |
| 2.4 | **Dynamic Castle Management** | Handle Home and Secondary Castle selection, tracking, and status swapping. | \- Castles exist as distinct objects containing Vector2i grid coordinates. \- Support dynamic shifting of "Home" status if the primary castle is abandoned to support the "Summer Home" mechanic.12 |

## **Epic 3: Combat Simulation and Ballistics**

This epic details the real-time artillery mechanics, mathematical trajectory modeling, projectile interactions, and environmental hazards.

| ID | Feature Name | Description | Technical Acceptance Criteria |
| :---- | :---- | :---- | :---- |
| 3.1 | **Crosshair Controller** | Mouse or Gamepad-driven targeting reticle bounding within the viewport. | \- Sprite2D node mapped to global mouse position via get\_global\_mouse\_position(). \- Bounded strictly to screen dimensions using clamp(). |
| 3.2 | **Trajectory Simulation** | Pseudo-3D cannonball flight paths calculated dynamically based on target distance. | \- Flight time scales directly with distance ![][image5]. \- Parabolic math equation applied to sprite's Y-axis offset to simulate height.12 \- Godot Jolt Area2D explosion instantiated upon reaching the target coordinate. |
| 3.3 | **Cannon Queue** | Sequential firing order enforcement for all placed cannons. | \- Cannons appended to an array queue based on their placement instantiation time.12 \- Firing input cycles through the array checking the has\_projectile\_in\_air boolean. |
| 3.4 | **Crater Generation** | Red Ship impacts create persistent, unbuildable terrain obstacles. | \- Red Ship projectile impacts utilize local\_to\_map to swap the Layer\_Hazards tile to a Crater sprite. \- Craters maintain an internal integer life\_timer, decrementing by 1 per round, executing self-deletion after 3 rounds.27 |
| 3.5 | **Friendly Fire Logic** | Allow players to destroy their own walls to pre-emptively prevent crater generation. | \- Projectile hit detection checks a source\_id string or enum. \- If source \== player hitting a Wall tile, the tile is erased via erase\_cell(), and no crater logic is spawned.16 |

## **Epic 4: The Construction Phase**

This epic covers the procedural Tetris-style placement logic, matrix rotation mathematics, and grid-snapping validation constraints.

| ID | Feature Name | Description | Technical Acceptance Criteria |
| :---- | :---- | :---- | :---- |
| 4.1 | **Tetromino Generator** | Randomly select and instantiate complex wall shapes for placement. | \- Dictionary mapping shape IDs to arrays of relative Vector2i coordinates.17 \- Implements a weighted RNG selector to prevent extreme edge-case shapes from spawning concurrently. |
| 4.2 | **Piece Manipulation** | Matrix rotation and cursor grid-snapping for active wall pieces. | \- 90-degree mathematical matrix rotation applied to the array upon secondary input.18 \- Active piece snaps perfectly to the grid utilizing TileMapLayer.local\_to\_map(). |
| 4.3 | **Collision Validation** | Prevent placement on invalid, obstructed, or mathematically illegal tiles. | \- \_process loop iterates through the shape's projected target coordinates. \- Queries TileMapLayer custom data and active Entity arrays. \- Modulates piece color to opaque red if any target tile contains water, wall, crater, cannon, or grunt.12 |
| 4.4 | **Cannon Placement** | Phase 4 logic for placing ![][image1] artillery footprints. | \- Triggers exclusively if Phase 3 validation succeeds. \- Algorithm grants 2 cannon units for the home castle, \+1 unit for each enclosed secondary castle.12 \- Uses identical validation logic as 4.3 but forces an additional constraint: placement *must* be strictly within territory flagged as claimed.12 |

## **Epic 5: Artificial Intelligence and Entities**

This epic handles the behaviors, state machines, and spatial navigation of the naval armada and dynamic ground forces.

| ID | Feature Name | Description | Technical Acceptance Criteria |
| :---- | :---- | :---- | :---- |
| 5.1 | **Ship Navigation** | Grid-aligned, deterministic linear movement for enemy naval vessels. | \- Movement logic relies on RayCast2D to detect incoming shoreline collisions or map bounds.28 \- Internal state machine reverses or alters course by 45/90 degrees upon detecting a collision vector.12 |
| 5.2 | **Fleet Manager** | Controls procedural spawning and composition of the enemy armada. | \- Hard limit capping total active ships at 16 instances.12 \- Instantiates Single Sail, Double-Mast, Red, and Dark ship scenes based on a scalable round difficulty integer.27 |
| 5.3 | **Grunt Deployment** | Double-Mast (Lander) ships spawn ground troops upon reaching the shoreline. | \- StateDeploying triggers when a Lander's raycast detects a beach. \- Instantiates Grunt CharacterBody2D entities at the adjacent shoreline edge.27 |
| 5.4 | **Grunt Behavior** | Phase-dependent movement and algorithmic destruction mechanics for ground troops. | \- Active grid-snapped movement executes *only* when the GameManager is in StateBuild.12 \- Pathfinding aggressively tracks the global\_position of the nearest castle tile. \- Grunt calls queue\_free() immediately if the FloodFill algorithm detects its coordinates enclosed within a closed wall loop.12 |

## **Epic 6: Scoring, Progression, and UI Overlays**

This epic manages the game economy, data tracking, visual overlays, audio ducking, and competitive multiplayer configurations.

| ID | Feature Name | Description | Technical Acceptance Criteria |
| :---- | :---- | :---- | :---- |
| 6.1 | **Scoring Engine** | Centralized bus calculating and broadcasting score updates. | \- Global Event Bus system. Emits ship\_destroyed (awards 30pts) and grunt\_destroyed (awards 12pts) signals. \- End of Build Phase calculation: enclosed\_tiles \* 7, plus 150 points for the Home Castle.33 |
| 6.2 | **Audio Trigger System** | Coordinates digitized voice clips and environmental sound effects. | \- AudioManager singleton preloading .wav files. \- Utilizes Godot Audio Buses to implement ducking on SFX tracks when Voice tracks (e.g., "Ready\! Aim\! Fire\!") are triggered by the FSM.34 |
| 6.3 | **UI Overlay** | Display temporal constraints, scores, and phase instructions to the viewport. | \- CanvasLayer isolating UI components to prevent rendering interference with the gameplay grid. \- AnimationPlayer triggers flashing red alerts for critical time constraints during Phase 3\. |
| 6.4 | **Multiplayer Framework** | Configure inputs and environment rules for local competitive multiplayer. | \- Shared-screen orthographic camera scaling or dual SubViewport implementation. \- Bypasses FleetManager AI spawner entirely. Maps distinct input devices to multiple reticles.3 \- Implements the spontaneous\_wall\_decay timer function to enforce competitive balancing and prevent stalemates.3 |

This architectural foundation comprehensively maps the physical and systemic requirements of the original 1990 arcade iteration into the node-based logic structures of Godot 4.6.1. By adhering strictly to the phase-based FSM, utilizing the advanced TileMapLayer and matrix transformation APIs, and implementing an optimized Flood Fill protocol in GDScript, the resulting application will accurately reproduce the challenging, high-stakes spatial puzzles of the original title while ensuring a robust, maintainable, and highly performant modern codebase.

#### **Works cited**

1. The Game That Invented Tower Defence: Rampart\! \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=p1STK3gCfiM](https://www.youtube.com/watch?v=p1STK3gCfiM)  
2. Rampart (video game) \- Wikipedia, accessed on March 23, 2026, [https://en.wikipedia.org/wiki/Rampart\_(video\_game)](https://en.wikipedia.org/wiki/Rampart_\(video_game\))  
3. Rampart \- Hardcore Gaming 101, accessed on March 23, 2026, [http://www.hardcoregaming101.net/rampart/](http://www.hardcoregaming101.net/rampart/)  
4. Rampart (Arcade) \- Let's Play 1001 Games \- Episode 83 \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=Nlmg9oa0uRw](https://www.youtube.com/watch?v=Nlmg9oa0uRw)  
5. Make a Finite State Machine in Godot 4 \- GDQuest, accessed on March 23, 2026, [https://www.gdquest.com/tutorial/godot/design-patterns/finite-state-machine/](https://www.gdquest.com/tutorial/godot/design-patterns/finite-state-machine/)  
6. Godot 4.6 Release – All about your flow \- Reddit, accessed on March 23, 2026, [https://www.reddit.com/r/godot/comments/1qnmjnj/godot\_46\_release\_all\_about\_your\_flow/](https://www.reddit.com/r/godot/comments/1qnmjnj/godot_46_release_all_about_your_flow/)  
7. TileMapLayer — Godot Engine (4.6) documentation in English, accessed on March 23, 2026, [https://docs.godotengine.org/en/4.6/classes/class\_tilemaplayer.html](https://docs.godotengine.org/en/4.6/classes/class_tilemaplayer.html)  
8. Using TileMaps — Godot Engine (4.6) documentation in English, accessed on March 23, 2026, [https://docs.godotengine.org/en/4.6/tutorials/2d/using\_tilemaps.html](https://docs.godotengine.org/en/4.6/tutorials/2d/using_tilemaps.html)  
9. Godot 4.6 Release: It's all about your flow, accessed on March 23, 2026, [https://godotengine.org/releases/4.6/](https://godotengine.org/releases/4.6/)  
10. Flood fill \- Wikipedia, accessed on March 23, 2026, [https://en.wikipedia.org/wiki/Flood\_fill](https://en.wikipedia.org/wiki/Flood_fill)  
11. Godot 4.3+ TileMap Tutorial: Layers, collisions and more\! \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=43sJIWaj2Yw](https://www.youtube.com/watch?v=43sJIWaj2Yw)  
12. Arcade Mermaid: Rampart, Part 3: The Phases \- Set Side B, accessed on March 23, 2026, [https://setsideb.com/arcade-mermaid-rampart-part-3-the-phases/](https://setsideb.com/arcade-mermaid-rampart-part-3-the-phases/)  
13. Godot 4.6: What changes for you | GDQuest Library, accessed on March 23, 2026, [https://www.gdquest.com/library/godot\_4\_6\_workflow\_changes/](https://www.gdquest.com/library/godot_4_6_workflow_changes/)  
14. Starter state machines in Godot 4 \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=oqFbZoA2lnU](https://www.youtube.com/watch?v=oqFbZoA2lnU)  
15. State machines. Learned how and I'm never going back. : r/godot \- Reddit, accessed on March 23, 2026, [https://www.reddit.com/r/godot/comments/10hm1np/state\_machines\_learned\_how\_and\_im\_never\_going\_back/](https://www.reddit.com/r/godot/comments/10hm1np/state_machines_learned_how_and_im_never_going_back/)  
16. Atari's Rampart \- A 12 Year Regression in MAME, Now Fixed \- Reddit, accessed on March 23, 2026, [https://www.reddit.com/r/MAME/comments/klgpht/ataris\_rampart\_a\_12\_year\_regression\_in\_mame\_now/](https://www.reddit.com/r/MAME/comments/klgpht/ataris_rampart_a_12_year_regression_in_mame_now/)  
17. Beginner Godot Tutorial \- How to Make Tetris \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=2T2Fkzwf6FM](https://www.youtube.com/watch?v=2T2Fkzwf6FM)  
18. How to make Tetris in Godot 4 (Complete Tutorial) 🖥️ \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=4XxsbtvQJE0](https://www.youtube.com/watch?v=4XxsbtvQJE0)  
19. DEVLOG \#1 \- Grid Building and Targeting System for Godot 4 \- Chris' Tutorials \- itch.io, accessed on March 23, 2026, [https://chris-tutorials.itch.io/grid-placement-godot/devlog/545697/devlog-1-grid-building-and-targeting-system-for-godot-4](https://chris-tutorials.itch.io/grid-placement-godot/devlog/545697/devlog-1-grid-building-and-targeting-system-for-godot-4)  
20. Grid Building for Godot 4 Plugin Overview \- How to Place Objects into your Game Levels\!, accessed on March 23, 2026, [https://www.youtube.com/watch?v=j05kIn26XoM](https://www.youtube.com/watch?v=j05kIn26XoM)  
21. TileMapLayer — Godot Engine (stable) documentation in English, accessed on March 23, 2026, [https://docs.godotengine.org/en/stable/classes/class\_tilemaplayer.html](https://docs.godotengine.org/en/stable/classes/class_tilemaplayer.html)  
22. Rampart | Video Game History Wiki | Fandom, accessed on March 23, 2026, [https://videogamehistory.fandom.com/wiki/Rampart](https://videogamehistory.fandom.com/wiki/Rampart)  
23. The flood fill algorithm \- GDQuest, accessed on March 23, 2026, [https://www.gdquest.com/tutorial/godot/2d/tactical-rpg-movement/lessons/06.game-board-and-flood-fill/](https://www.gdquest.com/tutorial/godot/2d/tactical-rpg-movement/lessons/06.game-board-and-flood-fill/)  
24. GDScript reference — Godot Engine (4.4) documentation in English, accessed on March 23, 2026, [https://docs.godotengine.org/en/4.4/tutorials/scripting/gdscript/gdscript\_basics.html](https://docs.godotengine.org/en/4.4/tutorials/scripting/gdscript/gdscript_basics.html)  
25. Filling a grid area when user-drawn walls fully enclose it : r/algorithms \- Reddit, accessed on March 23, 2026, [https://www.reddit.com/r/algorithms/comments/441hjg/filling\_a\_grid\_area\_when\_userdrawn\_walls\_fully/](https://www.reddit.com/r/algorithms/comments/441hjg/filling_a_grid_area_when_userdrawn_walls_fully/)  
26. In Depth TILEMAP Tutorial For Godot 4.3+ \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=ZutpG0\_CYrQ](https://www.youtube.com/watch?v=ZutpG0_CYrQ)  
27. Rampart \- FAQ \- Arcade Games \- By CChavez \- GameFAQs, accessed on March 23, 2026, [https://gamefaqs.gamespot.com/arcade/583614-rampart/faqs/762](https://gamefaqs.gamespot.com/arcade/583614-rampart/faqs/762)  
28. Godot 4 Beginner Tutorial: Smart Enemy AI Movement (NavigationRegion & NavigationAgent) \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=ZA\_hBXiUJYs](https://www.youtube.com/watch?v=ZA_hBXiUJYs)  
29. RayCast In Godot Tutorial: How To Create Smarter Enemies (Enemy AI) \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=\_AheThiIiyg](https://www.youtube.com/watch?v=_AheThiIiyg)  
30. How to achieve pixel-perfect movement on an enemy that's moved by Navigation? \- Reddit, accessed on March 23, 2026, [https://www.reddit.com/r/godot/comments/1nm7tip/how\_to\_achieve\_pixelperfect\_movement\_on\_an\_enemy/](https://www.reddit.com/r/godot/comments/1nm7tip/how_to_achieve_pixelperfect_movement_on_an_enemy/)  
31. player detection by enemy using 2d raycasts \- Help \- Godot Forum, accessed on March 23, 2026, [https://forum.godotengine.org/t/player-detection-by-enemy-using-2d-raycasts/76149](https://forum.godotengine.org/t/player-detection-by-enemy-using-2d-raycasts/76149)  
32. Arcade Mermaid: Rampart, Part 4: Level Strategy \- Set Side B, accessed on March 23, 2026, [https://setsideb.com/arcade-mermaid-rampart-part-4-level-strategy/](https://setsideb.com/arcade-mermaid-rampart-part-4-level-strategy/)  
33. Rampart \- Guide and Walkthrough \- Sega Master System \- By TheMightyRoast \- GameFAQs, accessed on March 23, 2026, [https://gamefaqs.gamespot.com/sms/570271-rampart/faqs/64964](https://gamefaqs.gamespot.com/sms/570271-rampart/faqs/64964)  
34. Rampart (NES) Playthrough \- NintendoComplete \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=-jMVqZE5bHQ](https://www.youtube.com/watch?v=-jMVqZE5bHQ)  
35. Rampart Sound Effects Download | SFX Library | Soundsnap, accessed on March 23, 2026, [https://www.soundsnap.com/tags/rampart](https://www.soundsnap.com/tags/rampart)  
36. All Rampart Voice Lines \- Apex Legends 2023 \- YouTube, accessed on March 23, 2026, [https://www.youtube.com/watch?v=SkVNHa2mlJU](https://www.youtube.com/watch?v=SkVNHa2mlJU)  
37. Rampart, Arcade | The King of Grabs, accessed on March 23, 2026, [https://thekingofgrabs.com/2019/11/26/rampart-arcade/](https://thekingofgrabs.com/2019/11/26/rampart-arcade/)  
38. How does multiplayer scoring work on Rampart for NES? \- GameFAQs, accessed on March 23, 2026, [https://gamefaqs.gamespot.com/nes/937631-rampart/answers/431042-how-does-multiplayer-scoring-work-on-rampart-for-nes](https://gamefaqs.gamespot.com/nes/937631-rampart/answers/431042-how-does-multiplayer-scoring-work-on-rampart-for-nes)  
39. Rampart (USA), accessed on March 23, 2026, [https://www.videogamemanual.com/snes/Rampart%20(USA).pdf](https://www.videogamemanual.com/snes/Rampart%20\(USA\).pdf)