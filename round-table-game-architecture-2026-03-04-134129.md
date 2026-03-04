# Round Table: Game Architecture — ECS vs OOP

**Date:** 2026-03-04 13:41
**Topic:** Game Architecture — When building a mid-sized game, what are the real-world trade-offs between Entity-Component-System (ECS) and traditional Object-Oriented (inheritance-based) architecture?

## Participants

| Participant | Role | Tool |
|---|---|---|
| Pi (GLM-5) | Performance Engineer | pi (bailian-coding-plan/glm-5) |
| Pi (Kimi-K2.5) | Indie Game Developer | pi (bailian-coding-plan/kimi-k2.5) |
| Pi (Qwen3.5-Plus) | Engine Architect | pi (bailian-coding-plan/qwen3.5-plus) |
| Claude | Gameplay Programmer | claude-code (host) |

---

## Round 1: ECS vs OOP Trade-offs for Mid-Sized Games


### Pi (GLM-5) — Performance Engineer

From a performance standpoint, the decision is largely settled by data access patterns. With 100K entities, you're not fighting algorithmic complexity—you're fighting the memory hierarchy.

**OOP's fatal flaw here is pointer chasing.** Traditional inheritance hierarchies allocate objects scattered across the heap. Each entity is a node in a web of virtual calls and indirections. When your physics system needs position and velocity data, it pulls them through pointers scattered across memory. Each cache miss costs 100-300 cycles on modern hardware. With 100K entities, you're looking at millions of cache misses per frame just to read the data you need.

**ECS wins on cache coherence alone.** Components stored in contiguous arrays mean you iterate through position data sequentially. A typical L1 cache line (64 bytes) holds 8 Vector3s. You're processing 8 entities per memory fetch instead of 1. That's not a 2x improvement—it's potentially 10-20x for tight loops. Physics integration becomes: `for (i = 0; i < count; ++i) { velocity[i] += acceleration[i] * dt; position[i] += velocity[i] * dt; }` — pure sequential access, prefetcher-friendly, SIMD-ready.

**The OOP advantage disappears under real workloads.** Virtual function dispatch costs 2-5 cycles plus potential misprediction penalties. With 100K entities making multiple virtual calls per frame, you've burned milliseconds before doing useful work. Component-based composition lets you hot-path the 90% case: the vast majority of entities are inert data that systems process without branching.

**Where ECS hurts:** Archetype transitions (adding/removing components) require data moves. Spawning 10K entities with complex component combinations can spike frame time if you're not pooling archetypes. Fragmentation from many small component arrays can re-introduce cache issues.

**The honest truth:** OOP is tenable for 10K entities where cache misses are absorbed by CPU headroom. At 100K with AI, physics, and rendering competing for 16ms, ECS isn't architectural preference—it's survival. The performance gap is measured in orders of magnitude, not percentage points.

---

### Pi (Kimi-K2.5) — Indie Game Developer

At 100K entities, you're in the painful middle ground where OOP starts suffocating but ECS might be premature optimization for a small team.

**OOP falls apart** when inheritance hierarchies become archaeological digs. I've seen `PlayerCharacter : Humanoid : LivingEntity : GameObject : MonoBehaviour` chains where changing base class behavior breaks AI, physics, and UI systems simultaneously. Virtual function overhead and cache misses become real at this scale—your AI pathfinding stutters not because the algorithm is slow, but because chasing pointers through fragmented memory layouts destroys your L1 cache. Debugging polymorphic behavior across 15 inheritance levels is cognitive torture when you're trying to ship.

**ECS shines** when you need to batch process: 10K AI agents updating simultaneously, physics colliders processed in tight SoA arrays, render culling that doesn't trash your cache. The data locality is undeniable—you'll get 5-10x performance wins on hot paths. The composition model also eliminates the diamond problem entirely.

**But here's the indie reality:** ECS has a brutal productivity tax. Boilerplate skyrockets—every behavior needs systems, queries, archetype management. You're writing `entityManager.GetComponentData<Velocity>(entity)` instead of `player.velocity`. Debugging becomes abstract indirection hell; you can't breakpoint "the player" because it's scattered across archetype chunks. For a 3-person team, that architectural friction kills iteration speed.

**Where ECS falls apart** is rapid gameplay prototyping. When you're iterating on "how does the jump feel?" or "should this AI flank or charge?", you need to tweak values and see results in seconds, not recompile system assemblies. OOP lets you hack in `if (player.isPoweredUp)` in 30 seconds; ECS forces you to create new component types, update queries, and manage structural changes.

My pragmatic take: Start OOP with composition (not inheritance), profile ruthlessly, and migrate hot paths to ECS-style data layouts only when profiler screams. Don't architect for 100K entities on day one if you might not ship the game.

---

### Pi (Qwen3.5-Plus) — Engine Architect

For 100K entities with complex subsystems, the architectural decision fundamentally impacts your team's velocity over the project lifecycle, not just frame times.

**ECS strengths at scale:** The data-oriented layout provides predictable memory access patterns—critical when iterating 100K entities every frame. Cache coherence isn't just optimization; it's the difference between 16ms and 4ms frame times. More importantly for architects, ECS enforces separation of concerns at the system level. Rendering, physics, and AI become independent systems that communicate through component data, not tangled object references. This modularity pays dividends when teams scale or requirements shift mid-development.

**Where ECS creates friction:** Tooling complexity increases significantly. Serializing component pools, debugging system execution order, and creating editor workflows for component-based entities requires substantial infrastructure investment. Stateful behaviors (think dialogue trees or quest logic) become awkward—you're fighting the architecture rather than working with it. Complex hierarchies (parent-child transforms with inheritance semantics) require explicit systems rather than natural object relationships.

**OOP's legitimate niche:** Traditional inheritance excels at UI systems, editor tools, and high-level game logic where entity count is low but behavioral complexity is high. A button inheriting from Control inheriting from UIElement makes intuitive sense. The problem emerges when you try to scale this to 100K game objects—you get deep inheritance trees that fracture under changing requirements.

**My architectural recommendation:** Hybrid approaches work best. Use ECS for simulation-heavy systems (rendering, physics, AI batches) where data locality matters. Retain OOP for tooling, UI, and complex stateful behaviors. The architecture boundary between these zones must be explicit—components shouldn't hold direct object references, and systems shouldn't depend on monobehaviours.

The real cost isn't performance—it's maintainability. ECS demands discipline but rewards you with systems that remain comprehensible at 500K lines of code.

---

### Claude — Gameplay Programmer

From the gameplay programming trenches, the architecture question comes down to one thing: how fast can I iterate on game feel?

**OOP gives me velocity on day one.** When I need to prototype a new enemy behavior, I subclass, override `OnUpdate`, and I'm testing in 60 seconds. The mental model maps directly to game design language — "the goblin *is a* creature that *has* health and *can* attack." Designers think in objects, and when your architecture matches their mental model, communication friction disappears.

**But OOP betrays me at scale.** The moment I need a goblin that also swims, also flies, also has mounted combat — the inheritance tree becomes a liability. I've spent entire sprints refactoring base classes because one new feature cross-cuts the entire hierarchy.

**ECS gives me composition freedom.** Slap a `Swimmer` component on the goblin, done. No refactoring. The system just picks it up next frame. And the performance headroom means I can be wasteful with gameplay systems without constantly worrying about the frame budget.

**Where ECS frustrates me:** Expressing sequential, stateful gameplay logic. A cutscene sequence, a multi-phase boss fight, a branching dialogue tree — these are inherently stateful, ordered processes. Encoding them as component data + systems is unnatural and verbose.

**My honest take:** Use ECS as the simulation backbone, but don't be dogmatic. Let gameplay scripts and high-level logic live in a more expressive layer. The 100K entities live in ECS. The 50 unique gameplay behaviors live wherever you can iterate fastest.

---

### Moderator Synthesis

**Key Agreements:** All four panelists agree that at 100K entities, ECS offers decisive performance advantages through cache-coherent memory layouts (5-20x on hot paths). All acknowledge pure ECS struggles with stateful, sequential logic.

**Key Disagreements:**
- **When to adopt ECS:** Performance Engineer says non-negotiable at 100K. Indie Dev says start OOP, migrate when profiler screams. Engine Architect and Gameplay Programmer advocate hybrid approaches.
- **Productivity cost:** Indie Dev sees "brutal productivity tax." Engine Architect sees "discipline that rewards at 500K LOC."

**Novel Insights:**
- GLM-5: Concrete hardware numbers (100-300 cycles/miss, 8 Vector3s/cache line) ground the debate in measurable reality.
- Kimi-K2.5: Debugging ergonomics (can't breakpoint "the player" in ECS) are underappreciated.
- Qwen3.5-Plus: Reframes cost as maintainability, not performance.

**Questions for Round 2:** Where should the boundary between ECS simulation and OOP/scripting layers be drawn? How do modern frameworks address the tooling gap?

