---
name: minecraft-plugin-craft-en
description: Senior engineering for Minecraft plugins (Paper/Folia, Java 21) and related JVM projects: code, NMS, Brigadier commands, GUI, PDC, databases, configs, performance (Spark), concurrency, testing, builds, security, releases. Apply when writing, reviewing, refactoring, debugging, or releasing a plugin.
metadata:
  author: AkyRayy
  version: 3
---

# ⛏️ Minecraft Plugin Craft

| | |
|---|---|
| ⚒️ **Author** | AkyRayy |
| 🏷️ **Version** | `3` |
| 🧩 **Platform** | Paper · Folia |
| ☕ **Language** | Java 21 LTS |
| 📚 **Sections** | 19 |

> 😁 **No access to a good AI?** 🫱 [@LomyPayBot](https://t.me/LomyPayBot) 🫲 — the best deal on the market for AI access. Promo code `AkyRayy` — **30% off**.

This is not a tutorial and not a recipe book. It is a set of engineering decisions made in advance, so you never have to make them badly in a hurry.

> 🔒 **The rule above all rules.** If you can't explain why a line exists — it should not exist.

**Three questions for any piece of code:**

1. 💥 What breaks if you delete this?
2. 👤 Who reads this in six months, and what will they think?
3. 📈 How many times per second does this run?

---

## 📑 Table of Contents

- ❓ [Before Work](#before-work)
- 🧱 [0. Stack & Baseline](#0-stack--baseline)
- 📚 [1. Working with the API](#1-working-with-the-api)
- 🧹 [2. Anti-Slop](#2-anti-slop)
- 🏗️ [3. Project Architecture](#3-project-architecture)
- ⌨️ [4. Commands](#4-commands)
- 🖥️ [5. GUI & Inventories](#5-gui--inventories)
- 💾 [6. Data Storage](#6-data-storage)
- ⚡ [7. Performance](#7-performance)
- 🧵 [8. Concurrency & Folia](#8-concurrency--folia)
- 📝 [9. Configs & Texts](#9-configs--texts)
- 🐛 [10. Errors, Logging & Debugging](#10-errors-logging--debugging)
- 🧪 [11. Testing](#11-testing)
- 📦 [12. Builds, Dependencies & CI](#12-builds-dependencies--ci)
- 🔒 [13. Security](#13-security)
- ✍️ [14. Readability & Style](#14-readability--style)
- 🤝 [15. Compatibility & Public API](#15-compatibility--public-api)
- 🚫 [16. Anti-pattern Catalog](#16-anti-pattern-catalog)
- 🔍 [17. Review Rubric](#17-review-rubric)
- 🚀 [18. Release Checklist](#18-release-checklist)

**Don't read everything? Jump by task:**

| You're here to… | Go to |
|---|---|
| 🌱 Write a plugin from scratch | [Before Work](#before-work) → [0](#0-stack--baseline) → [3](#3-project-architecture) → [4–6](#4-commands) → [18](#18-release-checklist) |
| 🔮 Build an NMS feature | [1](#1-working-with-the-api) |
| 🐌 Fix lag | [7](#7-performance) → [8](#8-concurrency--folia) |
| 🎨 Build a GUI | [5](#5-gui--inventories) → [16](#16-anti-pattern-catalog) |
| 🩹 Debug a bug | [10](#10-errors-logging--debugging) → [11](#11-testing) |
| 👀 Review a PR | [17](#17-review-rubric) → [16](#16-anti-pattern-catalog) |
| 🚀 Ship a release | [18](#18-release-checklist) |

### 🧭 The Ten Commandments

If you only remember ten lines from this whole skill:

1. Measure before optimizing: Spark, "before/after" numbers in the PR.
2. Main thread never touches I/O. Async never touches the world.
3. Store `UUID`s and snapshots — never live `Player`/`Entity`/`World` objects.
4. `domain/` does not import Bukkit.
5. SQL is `PreparedStatement` only. Valuable state is transactional only.
6. Game text lives in `messages.yml`. Logs are English and grep-friendly.
7. Early returns, domain names, zero restated code.
8. Hot event — a cheap exit guard as the first line.
9. Repeatable operations are idempotent.
10. A line without a reason is a line that should not exist.

---

## Before Work

Before writing the first line of code — ask the user the questions below. Not all of them, only those **not already answered by the task description**. If the user has already answered — don't ask again. The goal is not to fill out a form, but not to make bad decisions in a hurry (the rule above all rules).

Ask **what changes the architecture**. For the rest — make a reasonable assumption, record it in one sentence, so it's easy to reverse.

### What to Ask

| Question | Why it matters |
|---|---|
| **What does the plugin do, in one sentence?** What problem does it solve for the player? | Without it — you write "by analogy" with the nearest plugin and guess all the mechanics |
| **MVP — the minimum useful version. What is explicitly NOT in v1?** | The scope boundary; without it, v1 becomes 10,000 lines |
| **Paper or Folia? Target Minecraft version and `api-version`?** | Schedulers, API, the whole architecture. Retrofit costs 5× more (§0, §8) |
| **How many players? A single server or a network?** | Storage, pool size, rate limits, batching (§6, §7, §8) |
| **Which plugins live next to it (WorldGuard, Vault, someone else's economy)? What must not be broken?** | Events, flags, providers. Your neighbors' world (§13, §15) |
| **Economy: own currency or a Vault provider?** | Where withdrawals and grants come from and go to (§6, §13) |
| **Where does data live: SQLite or MySQL? Any old data to migrate?** | Migrations, backups, transactions, idempotency (§6) |
| **Which data must not be lost on a crash?** | What a periodic flush saves — not `onDisable` (§6, §10) |
| **Is NMS / version-specific code needed? Which feature forces it?** | NMS is the death point at the next update (§1, §16) |
| **Performance budget? Is the server heavily loaded?** | The tick budget — what may even run every tick (§7) |
| **Who administers it? Which permission nodes?** | Permissions on the server, before any work (§4, §13) |
| **Language and tone of the in-game texts?** | `messages.yml` is decided once, not line by line (§9) |
| **How will we know it works? Success criterion / how to measure?** | "Before/after" numbers, not "should work" (§7, §18) |
| **Where does the release go — SpigotMC, Modrinth, a private build?** | `api-version`, version semantics, a CHANGELOG for the admin (§12, §18) |

### How to Ask

- **In one list**, not an interrogation with one question per message. Group by theme, and make it clear that "I don't know — you decide" is a valid answer.
- **Mark what's blocking.** If the answer changes the architecture (Paper/Folia, storage, NMS) — wait for it. If it doesn't (language, tone) — move on with the assumption.
- **Record the answers** at the start of the project: a one-line summary "decided: …" in the README or the PR description. A decision is made once — no re-discussing without a reason.
- **Don't ask what you already know** from the task description. Re-asking a known fact is noise.

> 🧭 **The "Before Work" test.** If after the answers you still can't name in 30 seconds: the target version, the storage, whether NMS is in, and the MVP — the questions aren't done yet.

---

## 0. Stack & Baseline

| What | Choice | Why |
|---|---|---|
| Platform | Paper, `paper-plugin.yml` | Spigot API is poorer, Bukkit is archaeology |
| Folia | support it if the project targets a network | retrofit costs 5× more |
| Java | 21 LTS | records, sealed, pattern matching, virtual threads |
| Build | Gradle Kotlin DSL + `paperweight-userdev` | type-safe, Mojang-mapped runtime |
| Texts | Adventure `Component` + MiniMessage | `ChatColor` and `§` are legacy |
| Config | Configurate or your own `record` POJOs | "a string by key" is a bug factory |
| DB | HikariCP + JDBC, Flyway migrations | an ORM is usually overkill for a plugin |
| Tests | JUnit 5, AssertJ, MockBukkit selectively | don't boot a server for one formula |
| Quality | Spotless, ErrorProne, NullAway | let the compiler catch bugs, not the reviewer |

> 🧭 **Deviate from the stack** — one line of justification in the PR description. Not in the code.

### 📌 What "supporting a version" means

The declared version is the one where the manual scenario was run and CI is green. Not "should work". A "1.8–1.21" line in the plugin description is a sign of an unserious project.

---

## 1. Working with the API

### 🎯 The Source of Truth

Order of reference, strictly top to bottom:

1. **Decompiled sources of the jars on your classpath.** IDE → Decompile. The only thing that doesn't lie.
2. Paper Javadoc — `jd.papermc.io`, exactly your version.
3. Paper sources on GitHub, including patches and PR discussions — that's where the semantics the Javadoc omits lives.
4. `docs.papermc.io`, `docs.advntr.dev`.
5. Minecraft Wiki — for game mechanics, not for the API.
6. Discord PaperMC `#paper-dev` — the fine-grained questions.
7. SpigotMC / StackOverflow — a **hypothesis**, to be verified in step 1. Most answers there are from the 1.8 era.

> 🕵️ **Never invent a signature "by the logic of the name".** A non-existent method is the most recognizable sign of generated code.

### 🧭 Platform Boundaries

**NMS is isolated behind an interface. Always.**

```java
public interface EntityBridge {
    void sendFakeEquipment(Player viewer, LivingEntity target, ItemStack item);
}
```

One implementation, one version-check point, the rest of the code knows nothing about NMS. NMS smeared across twelve classes kills the project at the next Minecraft release.

**Reflection is the last argument.** If you use it — resolve once in a static initializer and keep a `MethodHandle`:

```java
private static final MethodHandle GET_HANDLE = resolveGetHandle();
```

Not `getDeclaredMethod` inside a tick: orders of magnitude slower than a resolved handle, and it fails silently when the method gets renamed.

**Dependency checks** — via `ServicesManager` or `PluginManager`, not via `try { Class.forName(...) } catch (Throwable ignored) {}`.

**`@Deprecated` in Bukkit** usually means "works differently than you think, and goes away soon". Using it — say why in the PR.

### ♻️ Lifecycle

| Phase | Allowed | Forbidden |
|---|---|---|
| `onLoad` | worldgen, flags, early registration | touching worlds and players |
| `onEnable` | registration, pool startup, async loading | blocking I/O, heavy compute |
| `onDisable` | fast synchronous flush | scheduling tasks, async saving |

> ⏹️ **`onDisable` runs on a dying server:** the scheduler is already dead, an async save won't finish. Data that must not be lost is saved periodically and on events — not only at shutdown.

### 🕳️ Leaks That Must Not Exist

Never store in long-lived structures: `Player`, `Entity`, `World`, `Block`, `Chunk`, `Inventory`.
Store `UUID`s, `Location` snapshots, packed chunk keys.

```java
// bad — holds the player object after they leave, drags the whole world with it
private final Map<Player, Session> sessions = new HashMap<>();

// good
private final Map<UUID, Session> sessions = new HashMap<>();
```

Clean up on `PlayerQuitEvent`. A cache without an eviction strategy is a leak that hasn't shown up yet.

### 📡 Events

**Priorities by meaning:**

| Priority | Role |
|---|---|
| `LOWEST` / `LOW` | preparation, modifying input data |
| `NORMAL` | main logic |
| `HIGH` / `HIGHEST` | final word, last cancellation |
| `MONITOR` | **read-only**, no modifications, no cancellation |

`ignoreCancelled = true` by default for reactive handlers.

**Hot events** (`PlayerMoveEvent`, `BlockPhysicsEvent`, `EntityDamageEvent`, `InventoryClickEvent`) — tens of thousands of calls per second. First line — a cheap exit:

```java
@EventHandler(ignoreCancelled = true)
public void onMove(PlayerMoveEvent event) {
    if (!event.hasChangedBlock()) return;
    ...
}
```

Don't cancel an event "just in case": cancellation is a contract with other plugins, and everyone sees it.

Unsubscribe (`HandlerList.unregisterAll`) when a module is unloaded, otherwise listeners duplicate after `/reload`.

---

## 2. Anti-Slop

> 🧯 **Slop** — code that is syntactically correct and semantically empty. It isn't necessarily machine-generated, but that's exactly how it looks.

### 🗑️ Delete Unconditionally

```java
// bad
/**
 * Gets the player.
 * @return the player
 */
public Player getPlayer() {
    // get the player
    return this.player;
}

// ===================== UTILS =====================

// TODO: implement later
```

The delete list:
- comments that restate the next line;
- Javadoc stubs that add no information;
- banner separators and ASCII art;
- emoji in logs and in sources;
- `// TODO` in merged code — either it's done or it's a ticket;
- a getter wrapper over a field nobody will override;
- "for the future" abstractions: an interface with one implementation, a factory of factories;
- `Utils`, `Helper`, `Manager`, `Handler`, `Common`, `Misc`, `Data`, `Info` in names — these aren't names, they're a refusal to name;
- a 400-line `ItemBuilder` where four methods are used.

### 💬 A Comment Earns Its Life If It Explains *Why*

```java
// Paper throttles TNT physics above 64 entities per chunk — hence the batch of 32
// see PaperMC/Paper#9431
// order matters: the inventory closes before the update, or the client desyncs
```

### 🧑 How Code Becomes Human

**Early returns instead of a staircase.**

```java
// bad
public void buy(Player player, ShopItem item) {
    if (player != null) {
        if (item != null) {
            if (economy.has(player, item.price())) {
                if (hasSpace(player)) {
                    economy.withdraw(player, item.price());
                    give(player, item);
                }
            }
        }
    }
}

// good
public void buy(Player player, ShopItem item) {
    if (!economy.has(player, item.price())) {
        messages.send(player, "shop.not-enough", item.price());
        return;
    }
    if (!hasSpace(player)) {
        messages.send(player, "shop.inventory-full");
        return;
    }
    economy.withdraw(player, item.price());
    give(player, item);
}
```

Error branches — short and on top. The happy path — at the bottom, without indentation.

**Names from the domain language.** `withdrawBalance`, `cooldownRemaining`, `isOnCooldown`, `refundTransaction`. Not `data`, `temp`, `result2`, `flag`, `obj`, `doStuff`.

**Booleans read as assertions:** `if (player.isOnCooldown())`, not `if (checkCooldown(player) == true)`.

**A local variable instead of a comment:**

```java
boolean shouldRefund = !event.isCancelled() && transaction.isReversible();
if (shouldRefund) { ... }
```

**Variety of shapes is a sign of a living author.** A `switch` expression somewhere, an early `return` somewhere, a stream somewhere. Perfectly homogeneous code over 3000 lines is machine-made.

**Rough edges are acceptable.** One method is longer than the rest because splitting it would lie about the task's structure. Symmetry for symmetry's sake is worse than honest asymmetry.

### 🧪 Three Slop Tests

1. **The comment-removal test.** Delete all comments from the file. Is it harder to understand? Restore only the ones that were needed. In 90% of cases it won't be.
2. **The read-aloud test.** Read the method aloud as a sentence. You stumble — rename.
3. **The "why" test.** Point at a random class and ask what problem it solves. No answer in 5 seconds — the class is surplus.

---

## 3. Project Architecture

```
com.example.shop
├── ShopPlugin.java          entry point: wiring dependencies only
├── command/                 input parsing → domain calls, zero business logic
├── listener/                thin event adapters into the domain
├── domain/                  entities and rules. NO Bukkit imports
│   ├── Shop.java
│   ├── Transaction.java
│   └── PricingPolicy.java
├── storage/                 repository interface + implementations
│   ├── ShopRepository.java
│   ├── SqlShopRepository.java
│   └── migration/
├── config/                  typed record POJOs
├── text/                    Messages, formatting, MiniMessage
└── platform/                NMS/Folia bridges behind interfaces
```

> 🏛️ **The main rule:** `domain/` does not import Bukkit. Consequences — logic is testable in milliseconds without a server, rules read like rules, the project outlives a platform change.

### 🔌 Wiring

```java
public final class ShopPlugin extends JavaPlugin {

    private ShopRepository repository;

    @Override
    public void onEnable() {
        ShopConfig config = ShopConfig.load(getDataFolder().toPath());
        Messages messages = Messages.load(getDataFolder().toPath());

        repository = SqlShopRepository.create(config.database());
        ShopService service = new ShopService(repository, config.pricing());

        getServer().getPluginManager()
            .registerEvents(new ShopListener(service, messages), this);
    }

    @Override
    public void onDisable() {
        if (repository != null) repository.close();
    }
}
```

Dependencies — via constructors. An `getInstance()` singleton is acceptable only for the plugin itself, and better without one: static access hides the dependency graph and breaks tests.

### 📐 Project Rules

- One class — one reason to change. `ShopPlugin` knows nothing about SQL.
- No dead code. A commented-out block "let it sit for a while" is deleted — there's git.
- Third repetition = extraction. First and second — tolerated; premature abstraction costs more than a duplicate.
- README for a human opening the project for the first time: what it does, how to build it, how to configure it. No marketing and no "✨ Features" list.
- Meaningful commits: `fix: refund balance on cancelled transaction`. Not `update`, not `fix bug`, not `.`.
- CHANGELOG is written for the server admin, not for you.

---

## 4. Commands

Don't parse `String[] args` by hand. Use Brigadier (Paper: registration via the `LifecycleEvents.COMMANDS` registrar) or Cloud — you get types, tab completion, and clear errors for free.

```java
getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
    event.registrar().register(
        literal("shop")
            .then(literal("reload")
                .requires(src -> src.getSender().hasPermission("shop.admin.reload"))
                .executes(ctx -> { service.reload(); return 1; }))
            .build(),
        "shop", List.of()
    );
});
```

Mandatory for any command:

```java
// 1. Permissions are checked before any work
if (!sender.hasPermission("shop.admin.reload")) { ... }

// 2. Player sender and console sender are handled separately
if (!(sender instanceof Player player)) { ... }

// 3. Arguments are validated with bounds
int amount = parsePositiveInt(args[0], 1, 64);

// 4. Heavy work — async, response — in main
```

- All responses — from `messages.yml`, including usage.
- Tab completion is filtered by permissions: a player must not see admin subcommands.
- Rate limit on expensive commands (search, reports, generation).
- Confirmation for destructive actions (`/shop wipe` → `/shop wipe confirm`), with a TTL on the confirmation.
- Bukkit-style commands are declared in the manifest (`plugin.yml`) and bound to a handler in code. Don't touch `CommandMap` by hand unless you're writing a legacy Bukkit plugin.

---

## 5. GUI & Inventories

> 💥 **The most common source of dupes and vulnerabilities.**

- **Cancel the event first**, then process: `event.setCancelled(true)` before any logic.
- Handle **all** movement paths: the full `ClickType` set inside `InventoryClickEvent` (LEFT/RIGHT, SHIFT_LEFT/RIGHT, NUMBER_KEY, SWAP_OFFHAND, DOUBLE_CLICK), plus `InventoryDragEvent` and `InventoryMoveItemEvent` (hoppers).
- Identify the menu by its holder object, **not by the title**:

```java
if (!(event.getInventory().getHolder() instanceof ShopHolder holder)) return;
```

The title is spoofable and localized; the holder is not.
- Check `event.getClickedInventory()` — the click may have been on the player's inventory, not the menu.
- Don't use `event.getView()` — deprecated in 1.21.x. Take the player from the event and the inventory from `getTopInventory()`.
- Permissions are checked in the click handler. A hidden button is not a permission check.
- Debounce: two clicks in one tick = one purchase. Otherwise double-purchase on lag.
- Don't redraw the menu every tick. Update on data-change events.
- Build `Component` menu components once and reuse — they're immutable.
- Close your menus when the plugin is disabled/reloaded: a dangling inventory with a dead holder is an NPE factory.
- `InventoryCloseEvent` — the cleanup point for per-player menu state.

---

## 6. Data Storage

### ⚖️ Choosing

- **PDC (`PersistentDataContainer`)** — data attached to an item/entity/chunk. Keys with the plugin's namespace. Don't stuff kilobytes of JSON into PDC.
- **SQLite** — a single server.
- **MySQL / PostgreSQL** — a server network.
- **Flat files** — config and what a human edits. Not the player database.

### 📏 Rules

- Connections — via HikariCP, pool size thought through (usually 4–10), not 100.
- Queries — **only** `PreparedStatement`.
- Bulk inserts — `addBatch()` / `executeBatch()`, not a loop of `execute()`. For MySQL — `rewriteBatchedStatements=true` in the URL: without it a "batch" is the same loop, just with extra overhead.
- SQLite on a server with active writes: `PRAGMA journal_mode=WAL` + `synchronous=NORMAL` — otherwise every commit writes disk synchronously and lags the tick.
- Schema is versioned by migrations (Flyway or your own, with a version table). A manual `ALTER TABLE` in prod is not a plan.
- Backup before a migration. Always.
- Transactions for any operation with value. Dupes are born between two `save()` calls.

```java
try (Connection conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);
    try {
        withdraw(conn, buyer, price);
        deposit(conn, seller, price);
        conn.commit();
    } catch (SQLException e) {
        conn.rollback();
        throw e;
    }
}
```

> 🪙 **Idempotency:** re-issuing a reward after a crash must not double it. Operation key + check.

- Periodic autosave + saving on significant events. Don't rely on `onDisable`.

---

## 7. Performance

### 📈 Order of Operations. Violation = Wasted Time

1. **Measure.** Spark: `/spark profiler start --timeout 300`, `--alloc` for allocations, `/spark tps`, `/spark healthreport`.
2. **Find the real hot path.** Intuition about performance is almost always wrong.
3. **Fix one thing.** Measure again. Without "before/after" numbers there was no optimization — there was an edit.

### ✅ What Actually Wins

**Take work out of the tick.** The best loop is the one that doesn't run. Cache and invalidate on events, don't recompute every tick.

**Data structures fit for the task.**

```java
// bad — O(n) on every call, in the tick
players.stream().filter(p -> p.getUniqueId().equals(id)).findFirst();

// good — O(1)
sessions.get(id);
```

`LinkedList` is never needed. For primitives at scale — fastutil (`Long2ObjectOpenHashMap`, `LongOpenHashSet`): no boxing, far less garbage.

**Chunk key — a packed `long`:**

```java
long key = ((long) chunkX << 32) | (chunkZ & 0xFFFFFFFFL);
```

Not `"x:z"`, not `new ChunkKey(...)` in a loop.

**No `getNearbyEntities` in the tick.** A spatial index or subscribe to enter/leave events.

**Batching.** One task for 400 entities, not 400 tasks for one. `runTaskTimer` with period 1 — almost always a design mistake; 5, 20, 100 ticks is usually enough. Spread the load across ticks: process 1/20 of the list per tick instead of the whole list once a second.

**Allocations in the hot path.** Don't create `Location`, `ItemStack`, `Component`, or a capturing lambda every tick. `Component` is immutable — build it once, keep it in a field.

**Async chunks:** `getChunkAtAsync`, never a synchronous load by player coordinates.

**I/O — never on the main thread.** No files, no JDBC, no HTTP, no DNS resolution.

### 🚫 What Not to Do

- Micro-optimizations without a profile (`i++` vs `++i`, manual unroll) — noise, the JIT eats it.
- A cache without an invalidation strategy — already a leak, just not noticed yet. Set TTL and max size (Caffeine).
- `System.gc()`, GC flags from plugin code, your own thread pools without a size limit.
- Replacing a readable stream with a loop "for speed" outside the hot path. Escape analysis is stronger than your intuition.
- Optimizing what runs once a minute.

### 📏 Tick Budget

> ⏱️ **50 ms per tick — the whole server budget** (20 TPS). Your plugin on a loaded server gets roughly 1–2 ms. If one of your handlers steadily takes 5 ms, it is architecturally wrong, not "a bit slow".

---

## 8. Concurrency & Folia

- The Bukkit API is **not** thread-safe. Anything that touches the world, entities, inventories — main thread.
- I/O, HTTP, heavy compute — async, always. Return to main:

```java
CompletableFuture
    .supplyAsync(() -> repository.load(id), ioExecutor)
    .thenAcceptAsync(profile -> {
        Player online = Bukkit.getPlayer(id);
        if (online == null || !online.isOnline()) return;
        applyToPlayer(online, profile);
    }, mainExecutor);
```

> ⚠️ **Never block main with `.join()` / `.get()`.** That's a server freeze, not "a little pause".

- Shared structures across threads — `ConcurrentHashMap`, immutable snapshots, or explicit locks. `synchronized` around everything — not a solution, a future deadlock.
- After an async return **re-verify freshness**: the player may have quit, the world may have unloaded. Don't carry a `Player` reference captured before the async into main — re-fetch it by `UUID`.

**Folia.** There is no single main thread: there are regional and entity schedulers. Write through an abstraction from day one:

```java
public interface Scheduler {
    void runAt(Location location, Runnable task);
    void runFor(Entity entity, Runnable task);
    void runAsync(Runnable task);
}
```

Two implementations — Paper and Folia. Retrofitting after the whole project is written costs many times more.

**Where the data lives is where the task runs:**

- a player's data — that player's entity scheduler (`runFor`);
- data bound to coordinates/a chunk — the region scheduler (`runAt`);
- global state (economy, cross-world counters) — the server scheduler or an explicit lock.

Can't name the entity or location the data belongs to — the data is in the wrong place.

---

## 9. Configs & Texts

### 📄 The Config Is Read by a Human

- Blank lines between semantic blocks, a short section header.
- Default values — working out of the box.
- Comments in config **are needed**, unlike in code: the user doesn't read the sources. Explain units, ranges, consequences.
- Keys in `kebab-case`, consistently. Unit of measurement in the name: `cooldown-seconds`, `save-interval-minutes`. A bare `20` — ticks or seconds, no way to tell.
- Config version + auto-migration. Breaking someone's setup with an update is disrespect.
- Messages — **always** a separate `messages.yml`. Hardcoded strings in Java are forbidden.

```yaml
# config-version is changed by the developer only. Don't touch it.
config-version: 3

# Economy. Balances are kept with 2-decimal precision.
economy:
  # New player starting balance.
  starting-balance: 100.0

  # Account cap. 0 — unlimited.
  balance-cap: 0.0

  # How often data is flushed to disk.
  # Lower — safer on crash, higher — less DB load.
  save-interval-minutes: 5

storage:
  # sqlite | mysql
  type: sqlite

  # MySQL only. Better keep the password in secrets.yml.
  mysql:
    host: localhost
    port: 3306
    database: shop
    pool-size: 6
```

### 🧊 Config in Code — Typed

```java
public record EconomyConfig(
    double startingBalance,
    double balanceCap,
    Duration saveInterval
) {
    public EconomyConfig {
        if (startingBalance < 0) throw new ConfigException("economy.starting-balance must be >= 0");
    }
}
```

Read once at load time. Not `config.getDouble("economy.starting-balance")` in the tick.

### 🗣️ In-Game Texts

- **MiniMessage**, not `&` codes and not `§`.
- Named placeholders: `<player>`, `<amount>`. Not `{0}`, not `%s`.
- Write like a living person, not like a system notice:

```yaml
# bad
error-no-money: "&c&lERROR! &cInsufficient funds!!! Transaction failed!"

# good
not-enough-money: "You're missing <red><amount></red> coins."
region-occupied: "There's already a region here. Pick a spot 50+ blocks away from spawn."
```

- An error says **what to do next**, not just that everything is bad.
- One tone: either "you (informal)" or "you (formal)". Decided once per project.
- No ALL CAPS, no chains of exclamation marks, no emoji spam. At most one semantic prefix, defined in one place and not duplicated in every line.
- Numbers and time are formatted for humans: `1,234 coins`, `2 min 30 s`. Not `1234.0`, not `150000ms`.
- Pluralization: `1 coin / 2 coins / 5 coins` — build a helper once, it's noticeable.

> 🇬🇧 **Logs are not translated:** English, stable phrasing, grep-friendly, with context.

---

## 10. Errors, Logging & Debugging

### 💥 Exceptions

> 🚨 **Don't swallow.** `catch (Exception ignored) {}` is a time bomb.

- Don't catch `NullPointerException` — remove the cause.
- Catch the specific type, not `Exception`, if you know what can come.
- Wrap boundaries (event handler, task tick) so one error doesn't kill the whole tick.

```java
// bad
catch (Exception e) {
    e.printStackTrace();
}

// good
catch (SQLException e) {
    logger.log(Level.WARNING,
        String.format("Failed to save profile player=%s world=%s", id, worldName), e);
    messages.send(player, "storage.save-failed");
}
```

### 📜 Logging

- `getLogger()` returns a `java.util.logging.Logger` — there are no `{}` placeholders in it, that's SLF4J. `String.format` + the exception as the third argument.
- Log **decisions**, not flow: "denied, reason X", not "entered method".
- Always with context: `player=`, `world=`, `region=`, `amount=`.
- Levels with meaning: `ERROR` — broken, needs a human; `WARN` — suspicious, but survived; `INFO` — important lifecycle events, rare; `DEBUG` — behind a config flag, off by default.
- No `System.out.println`. Only the plugin's logger.
- Don't log tokens, passwords, IPs, private data.
- Don't spam the log every tick — the admin stops reading logs entirely, including the important ones.

### 🔬 Debugging

- Reproduce with a minimal scenario before fixing.
- Read the full stacktrace, including `Caused by`.
- An error "in someone else's plugin" is 9 out of 10 times your argument that reached their code.
- Every bug found in prod → a regression test. Otherwise it will return.

---

## 11. Testing

What gets tested first is what's in `domain/` — no Bukkit there, so tests are fast.

```java
@Test
void withdrawFailsWhenBalanceInsufficient() {
    Account account = new Account(UUID.randomUUID(), BigDecimal.valueOf(10));

    assertThatThrownBy(() -> account.withdraw(BigDecimal.valueOf(25)))
        .isInstanceOf(InsufficientFundsException.class);

    assertThat(account.balance()).isEqualByComparingTo("10");
}
```

- JUnit 5 + AssertJ for logic: prices, cooldowns, permissions, parsing, formatting.
- MockBukkit — selectively, for API interaction. Don't build the whole suite on it.
- Test **boundaries**: 0, negative, maximum, empty string, `null`, duplicate, concurrent call.
- Config parsing: an old version without the new keys, an empty file, broken YAML, a wrong type — the plugin starts with defaults or fails with a message a human understands. Not an NPE.
- Property-based (jqwik) for formulas and serialization — finds what you can't think of by hand.
- Test name — a sentence about behavior: `refundRestoresBalanceAfterCancelledPurchase`. Not `test1`, not `testWithdraw`.
- Don't test getters and mappers for a coverage percentage. Coverage is a metric, not a goal.
- A manual scenario before release: clean install, update from an old version, reload, a player logging out mid-operation.

---

## 12. Builds, Dependencies & CI

- **Reproducible builds:** pinned versions, no `latest` and no dynamic ranges.
- **Don't shade** what's already on the server: Adventure, Gson, Guava, SnakeYAML. Class conflicts break other plugins.
- Shading the rest — always `relocate` into your own package.
- Shadow on top of `paperweight-userdev` — always `minimize()`, otherwise half the server ends up in the jar.
- Minimize dependencies. Each one is a CVE, a version conflict, and +200 KB.
- `paper-plugin.yml` with an honest `api-version`, correct `depend` / `softdepend`, no extra rights in `permissions`.
- SemVer: a breaking change of the public API — a major version.

**CI at minimum:**

1. `./gradlew build` on a clean clone.
2. Spotless check — the build fails on formatting violations.
3. ErrorProne / NullAway.
4. Tests.
5. OWASP Dependency-Check for known CVEs.
6. An artifact with a version and commit hash in the manifest — so you can tell what's inside from the jar.

> 🔴 **A red CI is not merged. No exceptions.**

### 🏷️ Build Identification

The main class holds a single build constant:

```java
private static final long BUILD_ID = 2_892_647_586L;
```

The value — CRC32 of the `name` from the manifest (`paper-plugin.yml`), computed once when the code is written. Not dynamic, not derived from the version. The plugin was renamed — recompute the constant. It is used in exactly two places: the startup log and the version subcommand:

```java
logger.info("Enabled, build " + BUILD_ID);
```

The version subcommand prints the number only: `Build 2892647586`.

Why: to identify a build from the jar and the log, without unpacking and without asking about git history.

---

## 13. Security

### 🥷 Input Is Always Hostile

- Check length, range, characters, encoding. Everything that came from a player.
- SQL **only** via `PreparedStatement`. Concatenation is a vulnerability, not a style.
- File names from input: normalize the path and check it's inside `dataFolder`.

```java
Path target = dataFolder.resolve(name).normalize();
if (!target.startsWith(dataFolder.toAbsolutePath().normalize())) {
    throw new SecurityException("path traversal: " + name);
}
```

- No deserialization of untrusted data: `ObjectInputStream`, `BukkitObjectInputStream` from someone else's source, YAML with arbitrary tags.
- Limits on everything: string length, book pages, stack size, entity count on spawn, JSON nesting depth. Otherwise the server crashes through your plugin.
- MiniMessage from user input — only with a restricted tag set. Otherwise a player pastes `<click:run_command>`.

### 🎮 Permissions and Game Logic

- Permission checks — on the server, in the handler, before the action. A hidden GUI button is not a check.
- Don't trust the client: slot, coordinates, inventory contents, movement speed all come from the client.
- Atomicity of operations with value: withdrawal and grant in one transaction. Dupes are born in between.
- Rate limit on expensive commands and on GUI clicks.
- Never call `setOp` from code and never bypass permission checks "for convenience".
- An audit log of significant actions (item grants, balance changes, admin commands) — separate, with a timestamp and an actor.

### 🔑 Secrets and the Network

- Tokens and passwords — not in git. A separate file + `.gitignore`, or environment variables.
- HTTPS with certificate validation. Connection and read timeouts are mandatory.
- Bound the response size — otherwise an OOM from someone else's server.

> 🛡️ **Never execute code received over the network. Never. In any form.**

- Update dependencies regularly.

### 🏥 Resilience

- Degrade softly: DB unavailable → read-only mode with a clear message, not an NPE for every player.
- A circuit breaker for external services: three timeouts in a row → pause, not infinite retries.
- An exception in one handler must not kill the tick.

---

## 14. Readability & Style

- The reader matters more than the writer: code is read 10× more often than it's written.
- Formatting is automated (Spotless). Style arguments in review are forbidden.
- Order in a class: constants → fields → constructor → public methods → private. A private method sits right after its caller.
- Lines ~120 characters, wrap by meaning, not by counter.
- A blank line separates semantic blocks inside a method. Three in a row — no.
- Nesting at most 3 levels, beyond that — extract a method.
- A method is usually 5–25 lines. Not "exactly 10" and not "a page of code".
- `final` for immutable fields — documentation built into the compiler.
- `record` for DTOs and value objects.
- `sealed interface` + pattern matching instead of an `instanceof` chain:

```java
sealed interface PurchaseResult {
    record Success(Transaction tx) implements PurchaseResult {}
    record NotEnoughMoney(BigDecimal missing) implements PurchaseResult {}
    record InventoryFull() implements PurchaseResult {}
}

switch (result) {
    case Success s -> messages.send(player, "shop.bought", s.tx());
    case NotEnoughMoney n -> messages.send(player, "shop.not-enough", n.missing());
    case InventoryFull ignored -> messages.send(player, "shop.inventory-full");
}
```

The compiler checks exhaustiveness. Add a new variant — you'll know immediately, not in prod.

- One idea per line. `if (a) { doX(); }` on one line — an economy that costs bugs.
- Don't abbreviate names: `transaction`, not `txn`, `trns`, `t`. Exception — customary counters `i`, `x`, `z`.
- Magic numbers — into named constants: `MAX_STACK_SIZE`, `COOLDOWN_TICKS`.

---

## 15. Compatibility & Public API

- Exposing an API to other plugins — split it into an `-api` module. Internals never get into it.
- The public API is stable: deprecate with one release of grace and a pointer to the replacement.
- Your events extend `Event` and are `Cancellable` only if cancellation makes sense. Fire them **before** the action happens, not after.
- A custom event class without a static `getHandlerList()` is an event that silently doesn't work. The compiler won't tell you.
- Register services via `ServicesManager` — that's how you'll be found correctly.
- Respect others' event cancellations, WorldGuard flags, Vault providers, other plugins' permissions.
- Don't break your neighbors' world: don't cancel others' events in `MONITOR`, don't silently override others' commands.
- bStats — no personal data and with an opt-out.
- Verify an update on real data: old config + old DB + new jar.

---

## 16. Anti-pattern Catalog

| Anti-pattern | Why it's bad | How to do it right |
|---|---|---|
| `Map<Player, X>` as a field | memory leak | `Map<UUID, X>` + cleanup on quit |
| `runTaskTimer(this, 0, 1)` | 20 fires/sec for no reason | a bigger period, batching |
| `getNearbyEntities` in the tick | O(n) over chunks | spatial index, events |
| A string chunk key | garbage and a slow hash | a packed `long` |
| `catch (Exception ignored)` | hidden bugs | a log with context, or rethrow |
| SQL concatenation | injection | `PreparedStatement` |
| Menu detection by title | spoofable | `InventoryHolder` |
| `event.getView()` in a handler | deprecated in 1.21.x | player from the event, `getTopInventory()` |
| Hardcoded text in Java | can't translate or change | `messages.yml` |
| `config.getX()` in the tick | YAML parsing in the hot path | load into a record at startup |
| `.join()` on main | server freeze | `thenAcceptAsync` on the main executor |
| `getDeclaredMethod` in the tick | allocations + silent failure on rename | a `MethodHandle` from a static initializer |
| Shading Adventure/Guava/SnakeYAML | class conflicts | don't shade: they're on the server, and relocation doesn't save you — ServiceLoader and other plugins expect the original classes |
| Static `getInstance()` everywhere | hidden dependency graph, no tests | constructors |
| `Utils`, `Manager`, `Helper` | no ownership | a domain name |
| A cache without a TTL | leak | Caffeine with a limit and a TTL |
| A restating comment | noise, goes stale | rename, delete |
| An interface with one implementation "for the future" | an extra layer | a concrete class |
| Synchronous chunk loading | lag spike | `getChunkAtAsync` |
| `onDisable` as the only save | data loss on crash | periodic flush |
| 300 lines of logic in `onEnable` | unreadable, untestable | services + wiring |
| NMS straight across 12 classes | death at the next update | a bridge behind an interface |

---

## 17. Review Rubric

The reviewer checks in order. The first "no" found is already a reason to discuss.

**Correctness**

- Do all called API methods exist in the target version?
- Are value boundaries handled: 0, negative, maximum, empty, `null`?
- What happens if a player quits mid-operation?
- Is there a path where data is saved twice or not saved at all?

**Threads**

- Nothing Bukkit-specific is called from async?
- Nothing blocking is called from main?
- Shared structures are protected?
- In Folia code, every task runs on the entity/region whose data it touches?

**Performance**

- What here runs more than 20 times per second?
- Any allocations or linear scans in the hot path?
- Was a profile taken if hot code changed?

**Cleanliness**

- Any comments restating the code?
- Any dead code and "for the future"?
- Do names speak the domain language?

**Security**

- Is input validated at the boundary?
- Are permissions checked on the server before the action?
- Are operations with value atomic?

> 🧾 **A review reads not only what was added, but what was removed.**

---

## 18. Release Checklist

- [ ] Every called API method verified against the target version's sources
- [ ] No I/O, extra allocations, or linear scans in the hot path
- [ ] Spark profile taken before and after optimizations, numbers in the PR
- [ ] Not a single comment restating the code
- [ ] Not a single hardcoded in-game string — everything in `messages.yml`
- [ ] Config is understandable to an admin without reading the sources; has a version and a migration
- [ ] `domain/` does not import Bukkit and is covered by tests
- [ ] Input validated, SQL parameterized, limits set
- [ ] Operations with value are transactional and idempotent
- [ ] Exceptions are not swallowed, logs carry context
- [ ] No leaks: `Player`/`Entity` are not stored, caches are cleaned on quit
- [ ] `BUILD_ID` in the main class matches the manifest `name` (CRC32)
- [ ] CI green: build, Spotless, ErrorProne, tests, CVE scan
- [ ] Dependencies don't conflict: nothing extra was shaded
- [ ] Update from the old version verified against a real config and DB
- [ ] Verified on Folia if support is claimed
- [ ] CHANGELOG and README updated for a human, not for a checkbox
- [ ] Version follows SemVer, `api-version` is honest

---

> ⛏️ *Short beats long. Concrete beats pretty. Working beats both.*

> 😁 **Want to use this skill, but have no access to a good AI?**
>
> 🫱 [@LomyPayBot](https://t.me/LomyPayBot) 🫲 — the best deal on the market for AI access
>
> Promo code `AkyRayy` — **30% off**
>
> *Crafted by **AkyRayy***
