# Design patterns (GoF subset) and game architecture

Scope: the design-level patterns TDT4240 lectures and exercises use (Gamma, Helm, Johnson & Vlissides, *Design Patterns*, Addison-Wesley, 1994, "GoF"). Also Rollings & Morris ch. 17 on game architecture, the game-programming patterns groups use in the libGDX project, and a pattern-to-QA map.

Cross-links:
- Architectural patterns (Layered, MVC, Pub-Sub, Client-Server and others): [architectural-patterns.md](architectural-patterns.md)
- Tactics per QA: [quality-attributes-classic.md](quality-attributes-classic.md) / [quality-attributes-4th-edition.md](quality-attributes-4th-edition.md)
- Coplien 1998 (pattern form, generative patterns): [architectural-patterns.md](architectural-patterns.md) §8
- 4+1 views: [documentation.md](documentation.md) §2
- Project pitfalls: [course-and-project-guide.md](course-and-project-guide.md) §5
- Exam drills (Composite, Template Method, GoF categories, game loop): [exam-prep.md](exam-prep.md)

> Parts of Part A and Part B are adapted from the Wikipendium TDT4240 compendium (https://www.wikipendium.no/TDT4240_Software_Architecture), originally CC BY-SA 3.0 by the Wikipendium TDT4240 authors. The adaptation is paraphrased, restructured and corrected where noted, and is shared under CC BY-SA 4.0 (a later version, which CC BY-SA 3.0 §4(b) permits for adaptations). Contributors and the list of changes: see [CREDITS.md](../CREDITS.md).
>
> Year tags come from the public 2015/2016 exam papers (dvikan.no archive).

**Level distinction (common exam point):** a *design pattern* solves a problem inside a subsystem or module (classes and objects). An *architectural pattern* describes system-level element types and how they interact. Observer is a design pattern. MVC and Publish-Subscribe are architectural patterns, and they are often *implemented* with Observer.

---

## Part A: GoF design patterns

### A.1 The three categories (a past-exam short question, 2016)

| Category | Concern | The 23 GoF patterns |
|---|---|---|
| **Creational** | How objects are created, hiding concrete classes | Abstract Factory, Builder, Factory Method, Prototype, Singleton |
| **Structural** | How classes and objects are composed into larger structures | Adapter, Bridge, **Composite**, Decorator, Facade, Flyweight, Proxy |
| **Behavioral** | Algorithms, and how responsibility and communication are assigned between objects | Chain of Responsibility, **Command**, Interpreter, Iterator, Mediator, Memento, **Observer**, **State**, **Strategy**, **Template Method**, Visitor |

GoF also split patterns by *scope*: **class** patterns work through inheritance and are fixed at compile time (Factory Method, class Adapter, Interpreter, Template Method). **Object** patterns work through composition and can change at run time (most of the others).

**Correction to Wikipendium:** the compendium files "Template" under *Structural*. That is wrong. Template Method is **behavioral** in GoF.

Pattern description, per GoF: name, intent, also known as, motivation, applicability, structure, participants, collaborations, consequences, implementation, sample code, known uses, related patterns. For the past-exam question on the three parts of a pattern description (2016), the safest answer is SAiP's context / problem / solution; the expected answer is not published. Do not confuse it with GoF's four essential elements: name, problem, solution, consequences. Coplien's Alexandrian form is in [architectural-patterns.md](architectural-patterns.md) §8.

### A.2 Per-pattern cards

Every card follows the same order: intent, structure, sketch, libGDX use, consequences, pitfalls. Sketches are Java, since most groups write libGDX in Java; Kotlin maps directly.

#### Singleton (creational)

- **Intent:** a class has exactly one instance, reached through one global access point.
- **Structure:** a private constructor, a private static field holding the instance, and a public static `getInstance()`. In Kotlin, `object Foo` is a language-level singleton, initialised lazily and thread-safely.

```java
public final class Assets {
    private static volatile Assets instance;
    private final AssetManager manager = new AssetManager(); private Assets() {}
    public static Assets get() {
        if (instance == null) {
            synchronized (Assets.class) {
                if (instance == null) instance = new Assets(); // double-checked locking
            }
        }
        return instance;
    }
    public AssetManager manager() { return manager; }
    public void dispose() { manager.dispose(); instance = null; }
}
```

- **libGDX use:** one asset manager (textures, sounds, skins), an audio or settings service, or the backend/Firebase gateway. libGDX runs game logic on a single render thread, so a plain lazy init is often enough there. Backend callbacks (Firebase listeners) can arrive on other threads.
- **Consequences:** + controlled access, lazy creation, one shared resource. − it is effectively a global variable, and it couples every caller to the concrete class.
- **Pitfalls:**
  - **Hidden global state.** Dependencies do not appear in constructor signatures, so the logical view understates coupling.
  - **Test leakage.** State survives between tests. You need a `reset()`, or better, inject an interface.
  - **Initialisation order.** A singleton used before `create()` runs, or before the GL context exists (textures need it), crashes. On Android, a static instance can outlive the `Activity`/GL context and hold dead textures after the app resumes.
  - **Thread safety.** A naive `if (instance == null)` check races when called from several threads. Use Kotlin `object`, an eager `static final`, the holder idiom, or `volatile` double-checked locking.
  - Overuse ("singletonitis"). Graders mark it as a modifiability and testability cost. Justify every singleton.

#### Factory Method vs Simple Factory (creational)

- **Simple Factory** (not a GoF pattern; an idiom): one class with a method (often static) that `switch`es on a parameter and returns a concrete product. It centralises creation, but adding a product means editing the switch.
- **Factory Method** (GoF): *"Define an interface for creating an object, but let subclasses decide which class to instantiate."* An abstract Creator declares `createProduct()`. Each ConcreteCreator overrides it. Other creator code uses the Product interface only. It is a *class* pattern and relies on inheritance.

```java
abstract class Level {                       // Creator
    abstract Enemy createEnemy(float x, float y);   // factory method
    void spawnWave(int n) {                  // uses only the Product interface
        for (int i = 0; i < n; i++) enemies.add(createEnemy(i * 64f, 400f));
    }
    final List<Enemy> enemies = new ArrayList<>();
}
class DesertLevel extends Level {
    Enemy createEnemy(float x, float y) { return new Scorpion(x, y); }
}
class IceLevel extends Level {
    Enemy createEnemy(float x, float y) { return new Yeti(x, y); }
}
// Simple Factory, for contrast:
class EntityFactory {
    static Entity create(String type) {
        switch (type) { case "ship": return new Ship(); default: return new Rock(); }
    }
}
```

- **libGDX use:** `EntityFactory` to build ECS entities (it attaches the right components), power-up factories, and per-level spawning.
- **Consequences:** + clients depend on abstractions (lower coupling, supports *defer binding*). − one extra subclass per product variant.
- **Pitfalls:** calling a Simple Factory "Factory Method" in the architecture document. Growing a god-factory with a huge switch.

#### Abstract Factory (creational)

- **Intent:** an interface for creating **families** of related objects without naming concrete classes.
- **Structure:** an AbstractFactory with `createA()` and `createB()`. ConcreteFactory1 and ConcreteFactory2 each produce one consistent family. The client holds only the abstract factory, which is chosen once, often at startup.

```java
interface ThemeFactory { Button button(String label); Background background(); }
class DarkTheme implements ThemeFactory {
    public Button button(String l) { return new DarkButton(l); }
    public Background background() { return new StarfieldBackground(); }
}
class LightTheme implements ThemeFactory {
    public Button button(String l) { return new LightButton(l); }
    public Background background() { return new SkyBackground(); }
}
// MenuScreen(ThemeFactory f) { play = f.button("Play"); bg = f.background(); }
```

- **libGDX use:** UI themes/skins. A backend family, such as `FirebaseBackendFactory` or `LocalBackendFactory` producing matching `AuthService` + `LobbyService` + `LeaderboardService` (for example a fake backend for tests or desktop). Platform families, injected from the `android`/`lwjgl3` launcher.
- **Consequences** (Wikipendium's MySQL/Oracle connection-factory example is the same idea): + swapping a whole family is one change, and family consistency is enforced. − adding a new *kind* of product changes every factory.
- **Pitfalls:** using it when there is only one product family (a Simple Factory or plain constructor is enough). Letting the factory grow into a service locator that everything reaches into, which hides dependencies just as a Singleton does. Mixing products from two families by creating some objects directly instead of through the factory.

#### Observer (behavioral)

- **Intent:** a one-to-many dependency. When the subject changes state, all registered observers are notified automatically.
- **Structure:** a Subject with `attach`/`detach`/`notify` and a list of Observers. Observers implement `update(...)`. In *push* style the data is passed in the notification. In *pull* style observers query the subject afterwards.

```java
interface ScoreListener { void onScoreChanged(int newScore); }
class ScoreModel {                                   // Subject
    private final List<ScoreListener> listeners = new ArrayList<>();
    private int score;
    void addListener(ScoreListener l) { listeners.add(l); }
    void removeListener(ScoreListener l) { listeners.remove(l); }
    void add(int points) {
        score += points;
        for (ScoreListener l : listeners) l.onScoreChanged(score);   // push
    }
}
// listeners: HUD label, achievements (s >= 1000 -> unlock), backend.pushScore(...)
```

- **libGDX use:** score or health events updating the HUD, achievements and backend sync. Model to view updates in MVC. Firebase `ValueEventListener` is itself an Observer.
- **Consequences:** + loose coupling (the subject knows only the interface), and observers can be added at run time. − notification order is unspecified, cascades are hard to trace, and costs grow with observer count.
- **Pitfalls:** forgetting `removeListener` when a Screen is disposed (lapsed listener: a memory leak plus updates to dead UI). Mutating the list during notify. Backend callbacks arriving off the render thread: use `Gdx.app.postRunnable(...)` before touching the scene.
- **vs Publish-Subscribe:** in Observer the subject holds *direct references* to its observers. In Pub-Sub an event bus decouples publishers from subscribers, which do not know each other.

#### State (behavioral)

- **Intent:** an object changes behaviour when its internal state changes, as if it had changed class.
- **Structure:** a Context holds a reference to a State interface and delegates to it. Each ConcreteState implements the behaviour for one state and can trigger transitions. This replaces `switch (state)` chains.

```java
abstract class GameState {                 // State
    protected final GameStateManager gsm;
    GameState(GameStateManager gsm) { this.gsm = gsm; }
    abstract void handleInput(); abstract void update(float dt);
    abstract void render(SpriteBatch sb); abstract void dispose();
}
class GameStateManager {                   // Context
    private final Deque<GameState> states = new ArrayDeque<>();
    void push(GameState s) { states.push(s); }
    void pop()  { states.pop().dispose(); }
    void set(GameState s) { if (!states.isEmpty()) states.pop().dispose(); states.push(s); }
    void update(float dt) { states.peek().update(dt); }
    void render(SpriteBatch sb) { states.peek().render(sb); }
}
class MenuState extends GameState {
    MenuState(GameStateManager g) { super(g); }
    void handleInput() { if (Gdx.input.justTouched()) gsm.set(new PlayState(gsm)); }
    void update(float dt) { handleInput(); }
    void render(SpriteBatch sb) { /* draw menu */ }  void dispose() {}
}
```

- **libGDX use:** the classic TDT4240 pattern exercise, a `GameStateManager` with Menu/Play/Pause/GameOver. The stack gives *pause overlays* (push Pause, pop to resume). libGDX's own `Game` + `Screen` (`setScreen`) is the same idea without a stack. The State pattern also works for a unit's AI or turn phases (e.g. PlacingShips → YourTurn → OpponentTurn → GameOver).
- **Consequences:** + state-specific behaviour is localised and adding a state is cheap (modifiability). + transitions are explicit. − more classes, and the transition logic is spread across states.
- **Pitfalls** (Wikipendium's ATM example, NoCard / CardInserted / PinEntered, has the same structure): not disposing popped states (texture leaks). States reaching into each other's internals. Confusing it with **Strategy**: the structure is the same, but in State the *object switches itself*, while in Strategy the *client picks* an algorithm.

#### Template Method (behavioral; not structural)

- **Intent:** define the **skeleton of an algorithm** in a base-class operation and defer some steps to subclasses. Subclasses redefine steps without changing the algorithm's structure ("Hollywood principle": *don't call us, we'll call you*).
- **Structure:** AbstractClass has a `final templateMethod()` that calls, in a fixed order:
  - **primitive operations**, which are abstract and *must* be overridden;
  - **hook operations**, which have a default (often empty) body and *may* be overridden;
  - concrete invariant steps.

  ConcreteClass overrides only the primitives and the chosen hooks.

```java
abstract class AbstractScreen extends ScreenAdapter {
    protected final Stage stage = new Stage();
    @Override public final void render(float dt) {   // template method: fixed per-frame skeleton
        ScreenUtils.clear(0, 0, 0, 1);               // invariant step
        if (!isPaused()) update(dt);                 // hook + primitive
        draw();                                      // primitive (must override)
        stage.act(dt); stage.draw();                 // invariant step (HUD)
        afterFrame();                                // hook (may override)
    }
    protected abstract void update(float dt);
    protected abstract void draw();
    protected boolean isPaused() { return false; }   // hook: default policy
    protected void afterFrame() {}                   // hook: default no-op
}
// One skeleton run per render() call. Do not put a blocking while-loop in a
// template method: it would freeze libGDX's render thread.
```

- **Exam class diagram (the 2015 paper asked for one):** draw `AbstractClass` with `templateMethod()` (note: *calls primitiveOp1(), primitiveOp2()*) and abstract `primitiveOp1()` and `primitiveOp2()`. Draw `ConcreteClass` inheriting from it and implementing those two operations. Mark the hook as a non-abstract overridable method.
- **libGDX use:** a base `AbstractScreen` as in the sketch (clear → `update(delta)` → `draw()` → `stage.act/draw`). A turn skeleton for turn-based games (one phase step per frame or per event), or a level-loading pipeline.
- **Consequences:** + code reuse, and the base class controls the extension points (inversion of control). − inheritance-bound (class scope), and deep hierarchies get brittle. Strategy is the composition-based alternative.
- **Pitfalls:** forgetting `final` on the template method, so a subclass overrides the skeleton itself. Too many hooks, which makes the call order hard to follow. Overriding a hook that has a real default body without calling `super`, when the base behaviour is still needed.
  - *Note:* the Wikipendium zoo-animal example shows only abstract-class inheritance and **no** algorithm skeleton, so it does not illustrate Template Method.

#### Composite (structural; past-exam question "when to use Composite", 2016)

- **Intent:** compose objects into **tree structures** for part-whole hierarchies, and let clients treat single objects and compositions **uniformly**.
- **Use it when:** (1) you need to represent part-whole hierarchies, and (2) clients should ignore the difference between leaves and composites, calling the same operation on both, for example operations that recurse over the tree such as draw, update, move or compute total.
- **Structure:** a Component interface (`operation()`, optionally `add`/`remove`/`getChild`). A Leaf implements the operation. A Composite holds child Components and implements the operation by forwarding to them.

```java
interface Node { void draw(Batch b); void translate(float dx, float dy); }
class SpriteNode implements Node {                         // Leaf
    final Sprite s; SpriteNode(Sprite s) { this.s = s; }
    public void draw(Batch b) { s.draw(b); }
    public void translate(float dx, float dy) { s.translate(dx, dy); }
}
class GroupNode implements Node {                          // Composite
    final List<Node> children = new ArrayList<>();
    void add(Node n) { children.add(n); }
    public void draw(Batch b) { for (Node n : children) n.draw(b); }
    public void translate(float dx, float dy) { for (Node n : children) n.translate(dx, dy); }
}
// ship = group(hull, turretGroup(barrel, base)); ship.translate(5, 0) moves everything
```

- **libGDX use:** scene2d is a Composite (`Actor` is the component, `Group` the composite, `Stage` the root). Also menus with submenus, multi-part ships, and UI layouts.
- **Consequences:** + simple clients, and new component kinds are easy to add. − the design can be *too general*: it is hard to restrict what a composite may contain. There is also a transparency vs safety trade-off: declaring `add()` on Component makes leaves carry meaningless methods.
- **Pitfalls:** cycles, or one child added to two parents (it gets drawn and moved twice); check or reparent on `add()`, as scene2d's `Group.addActor` does. Leaves that throw on `add()` when you choose transparency, which turns a design choice into run-time errors. Deep or very wide trees traversed every frame, which costs time per frame; cull or cache where you can.

### A.3 Also common in projects (brief)

| Pattern | Category | Intent | Typical libGDX use |
|---|---|---|---|
| **Command** | Behavioral | Encapsulate a request as an object, so you can queue, log, undo or send it | Map input to `MoveCommand`/`FireCommand`; send moves to the backend as serialised commands; undo in puzzle games; replays |
| **Strategy** | Behavioral | A family of interchangeable algorithms behind an interface, chosen by the client | AI difficulty (`EasyAI`, `HardAI`), scoring rules, movement behaviours; swappable at run time |
| **Adapter** | Structural | Convert one interface into the one clients expect | Wrap Firebase/Supabase SDKs behind an app-owned `BackendService` interface; the `core` module defines the interface and each platform module supplies an adapter. Firebase has no official libGDX SDK, and its official client SDKs target Android/iOS/web, so the Android SDK can only be used from the `android` module; desktop builds get a fake or REST-based adapter |

The backend Adapter + interface is the pattern teachers most often want justified as **modifiability and portability**: a change of backend or a desktop build touches only the adapter.

---

## Part B: Game architecture (Rollings & Morris, ch. 17)

**Source:** Rollings, A. & Morris, D. *Game Architecture and Design: A New Edition*. New Riders, 2004. The syllabus portion is chapter 17, **pp. 462-500**. That page range, from the section "Hardware Abstraction" up to (not including) "State Transitions and Properties", is **as given by Wikipendium for the 2015 syllabus**. Check that it is still on the current reading list (Leganto/Blackboard).

> The details below beyond Wikipendium's three steps and token definition are from general knowledge of the book. **Verify against the text** before quoting them as the book's wording.

### B.1 Hardware abstraction (verify against the text)

- **Idea:** keep game logic independent of platform specifics (graphics API, sound, input devices, OS services) by putting them behind an abstraction layer with a stable interface. Each platform gets its own implementation.
- **Why:** portability (new platform = new implementation of the layer), modifiability, and testability (a fake input or renderer).
- **Mapping to the course project:** libGDX *is* such a layer. The `core` module codes against `Gdx.graphics`, `Gdx.input` and `Gdx.audio`, and the `android`/`lwjgl3` backends implement them. Groups add their own layer for services libGDX lacks (backend, ads, platform login) through a `core` interface plus a platform implementation. In the *development view* this appears as the module split. In SAiP terms it is the modifiability tactics *encapsulate*, *use an intermediary* and *abstract common services*. Cost: the layer can hide platform-specific optimisations (performance vs portability trade-off).

### B.2 Token analysis (the core method)

**Token** (Wikipendium's paraphrase of the book): any game element the player manipulates directly or indirectly, and which the computer supervises and manages. Examples: the player's ship, enemies, bullets, pickups, the score, the board, a timer.

Steps (Wikipendium lists these three):
1. **Identify the tokens.** Go through the game design and list every element that acts or is acted upon. Include abstract ones (score, level timer, turn).
2. **Analyse interactions and events.** For each pair of tokens, what happens when they meet: collision, damage, pickup, capture? Which events do they cause (destroyed, score changed, level complete)?
   - *Token interaction matrix* (verify against the text): a table with tokens on both axes. Each cell describes the interaction (or "none") of the row token with the column token. It is a systematic check that no interaction is forgotten and shows which tokens are coupled.
3. **Build the logical view from the tokens.** Tokens become the key abstractions (classes, or entities/components). Interactions become methods, collision handlers or events. Clusters of strongly interacting tokens suggest subsystems. The book also refines tokens by merging or specialising them; verify the exact steps.

**Worked mini-example (Asteroids-like game).** Illustrative example written for this guide, not the book's own worked example (the book analyses its own example game; verify against the text). Tokens go on both axes; each cell names the interaction and the event it raises.

| ↓ acts on → | Ship | Asteroid | Bullet | Score |
|---|---|---|---|---|
| **Ship** | none | collision → `ShipDestroyed` | fires (creates) → `BulletFired` | none |
| **Asteroid** | collision → `ShipDestroyed` | none (or bounce) | none | none |
| **Bullet** | none | hit → `AsteroidDestroyed` (split into smaller asteroids, bullet removed, +points) | none | none |
| **Score** | none | none | none | none |

Events raised: `ShipDestroyed` (lose a life, respawn or game over), `BulletFired` (sound), `AsteroidDestroyed` (spawn fragments, Score adds points). Score is a passive token: it is changed by events but acts on nothing, so its row is empty.

The resulting logical view has `Ship`, `Asteroid`, `Bullet` and `ScoreModel`, plus `CollisionSystem` (handles the matrix cells) and an event such as `AsteroidDestroyed` that ScoreModel observes. This is how token analysis leads naturally to Observer or an event bus, and to ECS.

### B.3 How it feeds 4+1

- **Logical view:** tokens → classes/entities. Interactions → associations, operations and events. External tokens (backend player records) must be shown too; graders ask for this.
- **Process view:** the event flow found in step 2 (who notifies whom, at run time, on which thread) belongs here, not in the logical view.
- **Scenarios (+1):** each non-empty matrix cell is a candidate scenario for validating the views.

**Caution:** Wikipendium's glossary says FPS is "limited by the rate of AI ticks". This is **unverified** and an oversimplification. Do not state it as fact. What is safe to say: frame rate depends on how much work each loop iteration does (update + render), and on whether the loop ties simulation to rendering. See C.1.

---

## Part C: Game-programming patterns used in projects

> **Common practice, NOT confirmed syllabus.** These show up in TDT4240 projects and teacher feedback (e.g. "specify the ECS properly"), but no public source shows them as lecture or reading-list material. Further reading (not syllabus): Robert Nystrom, *Game Programming Patterns* (Genever Benning, 2014; free at https://gameprogrammingpatterns.com/).

### C.1 Game loop (past-exam question on how loop architecture affects frame rate, 2016)

The loop repeats: process input → update the simulation → render. Its design decides how game speed relates to frame rate.

| Loop style | How it works | Effect on frame rate / behaviour |
|---|---|---|
| **Fixed step, no timing** | update + render as fast as possible | Game speed depends on hardware: fast machine = fast game. Unusable across devices |
| **Fixed step + sleep/vsync cap** | each frame a fixed dt, then wait for the frame time | Stable if the device keeps up. If update + render exceed the budget, the game slows down |
| **Variable timestep** | `update(delta)` with measured elapsed time | Frame rate can vary freely and speed stays consistent. Large or unstable `delta` makes physics non-deterministic (tunnelling, explosions), which hurts networked games |
| **Fixed update, variable render** (accumulator) | accumulate real time; run `update(FIXED_DT)` while accumulator ≥ dt; render once, optionally interpolating by `alpha = acc/dt` | Deterministic simulation that is independent of render FPS. Rendering drops frames under load, not simulation steps. Risk: *spiral of death* if one update costs more than dt (clamp the frame time) |
| **Decoupled subsystems / threads** | AI, physics or networking tick at their own (lower) rate or on a worker thread; render reads the latest state | Render FPS no longer waits for slow AI. Needs thread-safe state hand-off (double buffer, `Gdx.app.postRunnable`); harder to debug |

```java
private float acc = 0f; private static final float DT = 1f / 60f;
@Override public void render() {                 // libGDX ApplicationListener
    acc += Math.min(Gdx.graphics.getDeltaTime(), 0.25f);   // clamp: avoid spiral of death
    while (acc >= DT) { world.update(DT); acc -= DT; }     // fixed-step simulation
    renderer.draw(world, acc / DT);                        // interpolated render
}
```

- libGDX owns the loop (inversion of control): the backend calls `render()` once per frame, usually vsync-capped, and `getDeltaTime()` gives the variable delta. Box2D in particular should be stepped with a fixed dt.
- Architectural points: the loop is a *process-view* element (thread, rate, ordering). The rate of *network sync* should be decoupled from the render rate (for example, send state at 10-20 Hz), which serves performance and cost. Name the performance tactics: *introduce concurrency* (worker threads for AI/network), *limit event response* (lower tick rates for AI and sync), and *bound execution times* (cap the work per frame, clamp the frame time).

### C.2 Update method

Each game object exposes `update(dt)`, and the loop iterates the collection once per frame. It is simple and matches libGDX `Actor.act(delta)`. Pitfalls: order dependencies between objects, and adding or removing objects during iteration (defer them to the end of the frame). ECS systems generalise this: behaviour is organised per *system*, not per object.

### C.3 Component / Entity-Component-System (ECS)

- **Entity:** just an ID (or an empty container) with no behaviour.
- **Component:** plain data (`PositionComponent{x,y}`, `VelocityComponent`, `SpriteComponent`, `HealthComponent`).
- **System:** behaviour that runs over all entities having a certain component *family* (`MovementSystem` = Position + Velocity, `RenderSystem`, `CollisionSystem`).
- **libGDX Ashley:** `Engine`, `Entity`, `Component`, `Family.all(...)`, `IteratingSystem`, `ComponentMapper` (fast lookup). `PooledEngine` reuses entities and components (Object Pool).
- **Why architects pick it:** composition over inheritance (no deep `Enemy extends Ship extends GameObject` trees). New behaviour = new component/system (modifiability). Tight loops over similar data (performance). Systems can be tested in isolation.
- **Costs:** indirection and a learning curve, and control flow is harder to follow. Cross-system communication needs events or signals.
- **Documenting it (teacher feedback: "specify the ECS properly"):** in the logical view, list the components (with fields) and systems (with the component family each processes), and name the entity archetypes built by the factory. In the process view, show system execution order per frame. Do not draw ECS entities as if they were classes with behaviour.

### C.4 Event queue / message bus

Senders post events to a queue or bus. Receivers subscribe and process them later (decoupled in **time**, unlike Observer's synchronous call). Uses: cross-system ECS communication (`CollisionEvent`), audio triggers, network messages applied on the render thread. Trade-offs: looser coupling vs harder debugging, and events may be processed a frame late. Bound the queue size (a performance tactic). This is the in-process counterpart of the Publish-Subscribe architectural pattern.

### C.5 Object pool

Pre-allocate and reuse objects (bullets, particles, effects) instead of `new`-ing them every frame, which avoids garbage-collector pauses (frame hitches on Android). libGDX offers `Pool<T>`, `Pools.obtain/free`, `Pool.Poolable.reset()`, and Ashley's `PooledEngine`. Pitfalls: forgetting `reset()` (stale state), and holding references after `free()`.

### C.6 Double buffer

Keep two buffers: write to the back buffer while the front one is read, then swap. Classic use is the frame buffer (the OpenGL backend swaps it for you in libGDX). At the architectural level, the same idea applies to simulation state: update into "next state" while reading "current state", so order of updates does not matter. It also helps apply network snapshots consistently.

### C.7 Screen / state management in libGDX

| Approach | Mechanism | Notes |
|---|---|---|
| `Game` + `Screen` | `game.setScreen(new PlayScreen(game))`; Screen lifecycle `show/render/resize/pause/resume/hide/dispose` | Built-in and simple. `setScreen` calls `hide()` on the old screen but **not** `dispose()`, so dispose it yourself |
| `GameStateManager` stack | State pattern (A.2) with push/pop/set | Pause overlays, back navigation. Common in the pattern exercise |
| Screen as MVC view | Screen = view, controllers handle input, models hold state | Pair it with Observer for model → view updates. See [architectural-patterns.md](architectural-patterns.md) |

Android `pause()`/`resume()` can lose the GL context. Reload managed assets through `AssetManager` and save game state in `pause()`.

---

## Part D: Pattern → quality attribute map

Use this when writing the *rationale* and the *patterns* section of the architecture document. Name the QA and the tactic, not just the pattern. Tactic names follow SAiP 3rd ed.; 4th-ed. renames are in parentheses and unverified (see [quality-attributes-4th-edition.md](quality-attributes-4th-edition.md) §7).

| Pattern | Primary QA served | Tactic(s) it realises | Main cost / QA hurt |
|---|---|---|---|
| Singleton | (convenience) controlled resource use | — (not a QA tactic) | Testability, modifiability (hidden coupling) |
| Factory Method / Simple Factory | Modifiability | Encapsulate; defer binding; restrict dependencies | Extra classes |
| Abstract Factory | Modifiability, portability, testability (fake families) | Abstract common services; defer binding (startup) | Adding a product kind touches every factory |
| Observer | Modifiability | Use an intermediary (the observer interface); category: reduce coupling | Performance (notify cost), traceability |
| Event queue / message bus | Modifiability; performance (smoothing load) | Use an intermediary; defer binding (run time); bound queue size | Debuggability, latency of one frame |
| State / GameStateManager | Modifiability; usability (clear flows) | Split module; increase semantic coherence (4th ed.: redistribute responsibilities; verify) | More classes |
| Template Method | Modifiability (reuse), consistency | Abstract common services (3rd ed. also lists refactor) | Inheritance rigidity |
| Strategy | Modifiability, testability | Defer binding (run time); encapsulate | Extra objects |
| Command | Usability (undo), testability (record/playback), interoperability (serialisable moves) | Undo (usability); record/playback (testability) | Class count |
| Composite | Modifiability (uniform treatment of parts) | Abstract common services | Over-general type constraints |
| Adapter / backend interface | **Modifiability, portability**, testability | Encapsulate; use an intermediary; abstract data sources (testability) | Small indirection overhead |
| Hardware abstraction layer | Portability, modifiability | Abstract common services; encapsulate | Performance (lost platform tuning) |
| **ECS** | **Modifiability and performance**; testability | Split module (modifiability); increase resource efficiency (performance; 4th ed.: increase efficiency of resource usage; verify) | Understandability, indirection |
| Fixed-timestep game loop | Performance predictability; correctness of physics/network sync | Bound execution times; manage sampling rate (4th ed.: manage work requests; verify) | Spiral-of-death risk; complexity |
| Object pool | Performance (no GC hitches) | Reduce overhead (4th ed.: reduce computational overhead; verify); falls under the category control resource demand | Memory held; stale-state bugs |
| Double buffer | Performance (tear-free rendering), correctness of state updates | — (implementation technique) | Double memory |

*Category names are not tactics.* Reduce coupling and increase cohesion (modifiability), and manage resources and control resource demand (performance), are tactic groups. In the document, name the tactic inside the group (for example *use an intermediary*, not "reduce coupling"). *Defer binding* is also a group; where the table says "defer binding (startup / run time)", name the concrete binding tactic, such as startup-time binding, runtime registration or polymorphism.

**Grader rules to apply (from the 2026 feedback, see [course-and-project-guide.md](course-and-project-guide.md) §5):** do not list patterns as tactics. In the patterns section give the *problem* and a high-level use, and put detailed design in the views. Show where each pattern and tactic appears in each view. Tie every choice to a quality goal in the rationale.
