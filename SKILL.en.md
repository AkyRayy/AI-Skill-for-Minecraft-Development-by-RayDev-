---
name: minecraft-plugin-craft
description: Building Minecraft plugins (Paper/Folia, Java 21) and related JVM projects at a senior engineering level. Apply when writing, reviewing, refactoring, debugging or releasing — code, configs, texts, build, security, performance.
version: 2
author: AkyRayy
---

# Minecraft Plugin Craft

<sub>Crafted by **AkyRayy**</sub>

> 😁 **No access to a good AI model?** 🫱 [@LomyPayBot](https://t.me/LomyPayBot) 🫲 — the best deal on the market for buying AI access. Promo code `AkyRayy` — **30% off**.

This is not a tutorial or a cookbook. It is a set of engineering decisions made up front, so you don't make them badly under pressure.

The rule above all rules: **if you cannot explain why a line exists, it should not exist.**

Three questions for any piece of code:
1. What breaks if I delete this?
2. Who reads this in six months, and what do they think?
3. How many times per second does it run?

---

## Table of contents

- [0. Stack and baseline decisions](#0-stack-and-baseline-decisions)
- [1. Working with the API](#1-working-with-the-api)
- [2. Anti-slop](#2-anti-slop)
- [3. Project architecture](#3-project-architecture)
- [4. Commands](#4-commands)
- [5. GUIs and inventories](#5-guis-and-inventories)
- [6. Data storage](#6-data-storage)
- [7. Performance](#7-performance)
- [8. Concurrency and Folia](#8-concurrency-and-folia)
- [9. Configs and texts](#9-configs-and-texts)
- [10. Errors, logs, debugging](#10-errors-logs-debugging)
- [11. Testing](#11-testing)
- [12. Build, dependencies, CI](#12-build-dependencies-ci)
- [13. Security](#13-security)
- [14. Readability and style](#14-readability-and-style)
- [15. Compatibility and public API](#15-compatibility-and-public-api)
- [16. Antipattern catalogue](#16-antipattern-catalogue)
- [17. Review rubric](#17-review-rubric)
- [18. Release checklist](#18-release-checklist)

---

## 0. Stack and baseline decisions

| Area | Choice | Why |
|---|---|---|
| Platform | Paper, `paper-plugin.yml` | Spigot API is poorer, Bukkit is archaeology |
| Folia | support it if the project targets networks | retrofitting costs 5× more |
| Java | 21 LTS | records, sealed types, pattern matching, virtual threads |
| Build | Gradle Kotlin DSL + `paperweight-userdev` | type-safe, Mojang-mapped runtime |
| Text | Adventure `Component` + MiniMessage | `ChatColor` and `§` are legacy |
| Config | Configurate or your own `record` POJOs | "string by key" is a bug factory |
| Database | HikariCP + JDBC, Flyway migrations | an ORM is usually overkill for a plugin |
| Tests | JUnit 5, AssertJ, MockBukkit sparingly | don't boot a server to check a formula |
| Quality | Spotless, ErrorProne, NullAway | let the compiler catch bugs, not the reviewer |

Deviating from the stack costs you one line of justification in the PR description. Not in the code.

### What "supporting a version" means
A declared version is one you ran a manual scenario on with green CI. Not "should work". A plugin page claiming `1.8–1.21` is a sign of an unserious project.

---

## 1. Working with the API

### Source of truth
Consult in this exact order:

1. **The decompiled sources of the jar on your classpath.** IDE → Decompile. The only thing that never lies.
2. Paper Javadoc — `jd.papermc.io`, for exactly your version.
3. Paper sources on GitHub, including patches and PR discussions — that's where the semantics live.
4. `docs.papermc.io`, `docs.advntr.dev`.
5. Minecraft Wiki — for game mechanics, not for API.
6. PaperMC Discord `#paper-dev` — for subtle questions.
7. SpigotMC / StackOverflow — a **hypothesis** to verify against #1. Most answers there are from 1.8.

Never invent a method signature "from the logic of the name". A non-existent method is the most recognisable sign of generated code.

### Platform boundaries

**NMS lives behind an interface. Always.**

```java
public interface EntityBridge {
    void sendFakeEquipment(Player viewer, LivingEntity target, ItemStack item);
}
```

One implementation, one version check, and the rest of the codebase never learns that NMS exists. NMS smeared across twelve classes kills the project on the next Minecraft release.

**Reflection is the last resort.** If you use it, resolve once in a static initialiser and keep a `MethodHandle`:

```java
private static final MethodHandle GET_HANDLE = resolveGetHandle();
```

Never `getDeclaredMethod` inside a tick. It is orders of magnitude slower and fails silently.

**Dependency detection** goes through `ServicesManager` or `PluginManager`, not `try { Class.forName(...) } catch (Throwable ignored) {}`.

**`@Deprecated` in Bukkit** usually means "this does not do what you think and will disappear". If you use it, say why in the PR.

### Lifecycle

| Phase | Allowed | Forbidden |
|---|---|---|
| `onLoad` | worldgen, flags, pre-world registration | touching worlds or players |
| `onEnable` | registration, pool startup, async loading | blocking I/O, heavy computation |
| `onDisable` | fast synchronous flush | scheduling tasks, async saving |

`onDisable` runs on a shutting-down server: the scheduler is already dead and async saves will not finish. Persist anything you cannot lose periodically and on events, not only at shutdown.

### Leaks that must not happen
Never keep in long-lived structures: `Player`, `Entity`, `World`, `Block`, `Chunk`, `Inventory`.
Keep `UUID`, a `Location` snapshot, or a packed chunk key.

```java
// bad — pins the player object after logout, dragging the whole world with it
private final Map<Player, Session> sessions = new HashMap<>();

// good
private final Map<UUID, Session> sessions = new HashMap<>();
```

Clean up on `PlayerQuitEvent`. A cache without an eviction strategy is a leak that simply has not surfaced yet.

### Events

**Priorities by meaning:**

| Priority | Role |
|---|---|
| `LOWEST` / `LOW` | preparation, mutating input data |
| `NORMAL` | main logic |
| `HIGH` / `HIGHEST` | final say, final cancellation |
| `MONITOR` | **read only** — no mutation, no cancellation |

`ignoreCancelled = true` by default for reactive handlers.

**Hot events** (`PlayerMoveEvent`, `BlockPhysicsEvent`, `EntityDamageEvent`, `InventoryClickEvent`) fire tens of thousands of times per second. First line is a cheap guard:

```java
@EventHandler(ignoreCancelled = true)
public void onMove(PlayerMoveEvent event) {
    if (!event.hasChangedBlock()) return;
    ...
}
```

Never cancel an event "just in case": cancellation is a contract with every other plugin, and everyone sees it.

Unregister (`HandlerList.unregisterAll`) when a module unloads, or listeners duplicate after `/reload`.

---

## 2. Anti-slop

Slop is code that is syntactically correct and semantically empty. It is not necessarily machine-generated, but it looks exactly like it.

### Delete unconditionally

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

The deletion list:
- comments restating the next line;
- Javadoc stubs that add no information;
- banner separators and ASCII art;
- emoji in logs and in source;
- `// TODO` in merged code — either it is done or it is a tracker issue;
- getter wrappers around fields nobody will ever override;
- speculative abstraction: one-implementation interfaces, factories of factories;
- `Utils`, `Helper`, `Manager`, `Handler`, `Common`, `Misc`, `Data`, `Info` in names — those are not names, they are a refusal to name;
- a 400-line `ItemBuilder` of which four methods are used.

### A comment earns its place when it explains *why*

```java
// Paper throttles TNT physics above 64 entities per chunk — hence batches of 32
// see PaperMC/Paper#9431
// order matters: the inventory closes before the update, otherwise the client desyncs
```

### How code starts looking human

**Early exit instead of a staircase.**

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

Failure branches are short and on top. The happy path lives at the bottom, unindented.

**Names come from the domain language.** `withdrawBalance`, `cooldownRemaining`, `isOnCooldown`, `refundTransaction`. Not `data`, `temp`, `result2`, `flag`, `obj`, `doStuff`.

**Booleans read as statements:** `if (player.isOnCooldown())`, not `if (checkCooldown(player) == true)`.

**A local variable beats a comment:**

```java
boolean shouldRefund = !event.isCancelled() && transaction.isReversible();
if (shouldRefund) { ... }
```

**Variety of form is a sign of a living author.** A switch expression here, an early return there, a stream elsewhere. Perfectly uniform code across 3000 lines is machine code.

**Unevenness is allowed.** One method is longer than the rest because splitting it would lie about the shape of the problem. Symmetry for its own sake is worse than honest asymmetry.

### Three slop tests

1. **Comment removal test.** Strip every comment from the file. Did it get harder to read? Restore only the ones that were needed. Nine times out of ten nothing is lost.
2. **Read-aloud test.** Read the method out loud as a sentence. If you stumble, rename something.
3. **The "why" test.** Point at a random class and state the problem it solves. No answer within five seconds means the class is redundant.

---

## 3. Project architecture

```
com.example.shop
├── ShopPlugin.java          entry point: wiring only
├── command/                 parse input → call the domain, zero business logic
├── listener/                thin adapters from events to the domain
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

**The core rule: `domain/` does not import Bukkit.** Consequences — logic is tested in milliseconds without a server, rules read like rules, and the project survives a platform change.

### Wiring

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

Dependencies go through the constructor. A `getInstance()` singleton is acceptable only for the plugin itself, and preferably not even then: static access hides the dependency graph and breaks tests.

### Project rules
- One class, one reason to change. `ShopPlugin` knows nothing about SQL.
- No dead code. A commented-out block "just in case" gets deleted — git exists.
- The third repetition triggers extraction. The first two are tolerated; premature abstraction costs more than duplication.
- README is for someone opening the project for the first time: what it does, how to build, how to configure. No marketing, no "✨ Features" list.
- Meaningful commits: `fix: roll back balance on cancelled transaction`. Not `update`, not `fix bug`, not `.`.
- CHANGELOG is written for the server admin, not for you.

---

## 4. Commands

Do not hand-parse `String[] args` in every command. Use Brigadier (Paper's `LifecycleEventManager`) or Cloud — you get types, completions and sane errors for free.

Mandatory for any command:

```java
// 1. Permissions are checked before any work
if (!sender.hasPermission("shop.admin.reload")) { ... }

// 2. Player sender and console sender are handled separately
if (!(sender instanceof Player player)) { ... }

// 3. Arguments are validated with bounds
int amount = parsePositiveInt(args[0], 1, 64);

// 4. Heavy work goes async, the reply comes back on main
```

- Every response comes from `messages.yml`, usage included.
- Tab completion is filtered by permission: a player must not see admin subcommands.
- Rate limit expensive commands (searches, reports, generation).
- Confirmation for destructive actions (`/shop wipe` → `/shop wipe confirm`) with a TTL.
- Do not register commands dynamically through `CommandMap` if the manifest can declare them.

---

## 5. GUIs and inventories

The single most common source of dupes and exploits.

- **Cancel the event first**, then handle it: `event.setCancelled(true)` before any logic.
- Handle **every** movement path: `InventoryClickEvent`, `InventoryDragEvent`, shift-click, number keys, offhand swap, `InventoryMoveItemEvent`.
- Identify menus by holder object, **never by title**:

```java
if (!(event.getInventory().getHolder() instanceof ShopHolder holder)) return;
```

Titles are spoofable and localised; holders are not.
- Check `event.getClickedInventory()` — the click may have landed in the player's inventory, not the menu.
- Permissions are checked in the click handler. A hidden button is not a permission check.
- Debounce: two clicks in one tick equal one purchase. Otherwise lag produces double purchases.
- Do not redraw the menu every tick. Refresh on data-change events.
- Build menu `Component`s once and reuse them — they are immutable.
- Close open menus on plugin reload: a dangling inventory with a dead holder is an NPE waiting to happen.

---

## 6. Data storage

### Choosing
- **PDC (`PersistentDataContainer`)** — data bound to an item, entity or chunk. Namespaced keys. Do not stuff kilobytes of JSON in there.
- **SQLite** — single server.
- **MySQL / PostgreSQL** — server networks.
- **Flat files** — configuration and things humans edit. Not your player database.

### Rules
- Connections through HikariCP with a deliberate pool size (usually 4–10), not 100.
- Queries **only** through `PreparedStatement`.
- Bulk inserts use `addBatch()` / `executeBatch()`, not a loop of `execute()`.
- Schema is versioned by migrations (Flyway or your own with a version table). A manual `ALTER TABLE` in production is not a plan.
- Back up before migrating. Always.
- Transactions for anything involving value. Dupes are born between two `save()` calls.

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

- Periodic autosave plus saves on significant events. Do not rely on `onDisable`.
- Idempotency: re-granting a reward after a crash must not double it. Operation key plus a check.

---

## 7. Performance

### The order of operations. Break it and you waste your time

1. **Measure.** Spark: `/spark profiler start --timeout 300`, `--alloc` for allocations, `/spark tps`, `/spark healthreport`.
2. **Find the actual hot path.** Performance intuition is wrong almost every time.
3. **Fix one thing.** Measure again. Without before/after numbers there was no optimisation, only an edit.

### What actually pays off

**Take work out of the tick.** The best loop is the one that never runs. Cache and invalidate on events instead of recomputing every tick.

**Data structures that fit the job.**

```java
// bad — O(n) on every call, inside a tick
players.stream().filter(p -> p.getUniqueId().equals(id)).findFirst();

// good — O(1)
sessions.get(id);
```

`LinkedList` is never the answer. For primitives at scale use fastutil (`Long2ObjectOpenHashMap`, `LongOpenHashSet`): no boxing, far less garbage.

**Chunk keys are packed `long`s:**

```java
long key = ((long) chunkX << 32) | (chunkZ & 0xFFFFFFFFL);
```

Not `"x:z"`, and not `new ChunkKey(...)` in a loop.

**No `getNearbyEntities` inside a tick.** Use a spatial index or enter/exit events.

**Batching.** One task for 400 entities, not 400 tasks of one. `runTaskTimer` with period 1 is almost always a design mistake; 5, 20 or 100 ticks is usually enough. Spread load across ticks: process 1/20 of the list each tick instead of the whole list once a second.

**Allocations in the hot path.** Do not create `Location`, `ItemStack`, `Component` or capturing lambdas every tick. `Component` is immutable — build once, store it.

**Async chunks:** `getChunkAtAsync`, never a synchronous load at player coordinates.

**I/O never runs on the main thread.** Not files, not JDBC, not HTTP, not DNS resolution.

### What not to do
- Micro-optimisation without a profile (`i++` vs `++i`, manual unrolling) — noise, the JIT eats it.
- A cache without invalidation is already a leak. Set TTL and a maximum size (Caffeine).
- `System.gc()`, GC flags from plugin code, unbounded custom thread pools.
- Replacing a readable stream with a loop "for speed" outside a hot path. Escape analysis beats your intuition.
- Optimising something that runs once a minute.

### The tick budget
50 ms per tick covers the entire server. On a busy server your plugin is entitled to roughly 1–2 ms. If one of your handlers consistently takes 5 ms, it is architecturally wrong, not "a bit slow".

---

## 8. Concurrency and Folia

- The Bukkit API is **not** thread-safe. Anything touching worlds, entities or inventories runs on main.
- I/O, HTTP and heavy computation go async, always. Return to main:

```java
CompletableFuture
    .supplyAsync(() -> repository.load(id), ioExecutor)
    .thenAcceptAsync(profile -> applyToPlayer(player, profile), mainExecutor);
```

- Never block main with `.join()` / `.get()`. That is a server freeze, not "a short pause".
- Shared structures between threads: `ConcurrentHashMap`, immutable snapshots, or explicit locks. `synchronized` around everything is not a solution, it is a future deadlock.
- After an async round trip, **re-validate**: the player may have left, the world may have unloaded.

```java
Player online = Bukkit.getPlayer(id);
if (online == null || !online.isOnline()) return;
```

**Folia.** There is no single main thread: there are region and entity schedulers. Write against an abstraction from day one:

```java
public interface Scheduler {
    void runAt(Location location, Runnable task);
    void runFor(Entity entity, Runnable task);
    void runAsync(Runnable task);
}
```

Two implementations, Paper and Folia. Retrofitting after the whole project is written costs several times more.

---

## 9. Configs and texts

### Humans read the config

- Blank lines between logical blocks, a short header per section.
- Defaults that work out of the box.
- Comments in the config are **required**, unlike in code: the user does not read your sources. Explain units, ranges and consequences.
- `kebab-case` keys, consistently. Units in the name: `cooldown-seconds`, `save-interval-minutes`. A bare `20` could be ticks or seconds — nobody can tell.
- A config version plus automatic migration. Breaking someone's settings on update is disrespectful.
- Messages always live in a separate `messages.yml`. Hardcoded strings in Java are forbidden.

```yaml
# config-version is changed by the developer only. Do not touch.
config-version: 3

# Economy. Balances are stored with two decimal places.
economy:
  # Starting balance for a new player.
  starting-balance: 100.0

  # Account cap. 0 means unlimited.
  balance-cap: 0.0

  # How often data is flushed to disk.
  # Lower is safer on a crash, higher means less database load.
  save-interval-minutes: 5

storage:
  # sqlite | mysql
  type: sqlite

  # mysql only. Prefer moving the password into secrets.yml.
  mysql:
    host: localhost
    port: 3306
    database: shop
    pool-size: 6
```

### Config in code is typed

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

Read once at load. Never `config.getDouble("economy.starting-balance")` inside a tick.

### In-game text

- **MiniMessage**, not `&` codes and not `§`.
- Named placeholders: `<player>`, `<amount>`. Not `{0}`, not `%s`.
- Write like a person, not like a system notification:

```yaml
# bad
error-no-money: "&c&lERROR! &cInsufficient funds!!! Transaction failed!"

# good
not-enough-money: "You need <red><amount></red> more coins."
region-occupied: "There is already a claim here. Pick a spot at least 50 blocks from spawn."
```

- An error message says **what to do next**, not just that something went wrong.
- One consistent tone across the plugin. Decide once.
- No ALL CAPS, no chains of exclamation marks, no emoji spam. At most one meaningful prefix, defined in one place and not repeated in every string.
- Format numbers and durations for humans: `1,234 coins`, `2m 30s`. Not `1234.0`, not `150000ms`.
- Pluralisation: `1 coin / 2 coins`. Write the helper once, it shows.
- Logs are **not** localised: English, stable wording, grep-friendly, with context.

---

## 10. Errors, logs, debugging

### Exceptions
- Do not swallow them. `catch (Exception ignored) {}` is a delayed-action bomb.
- Do not catch `NullPointerException` — remove the cause.
- Catch the specific type when you know what can be thrown.
- Wrap boundaries (event handler, scheduled task) so one failure does not take down the tick.

```java
// bad
catch (Exception e) {
    e.printStackTrace();
}

// good
catch (SQLException e) {
    logger.warn("Failed to save profile player={} world={}", id, worldName, e);
    messages.send(player, "storage.save-failed");
}
```

### Logs
- Log **decisions**, not control flow: "denied, reason X", not "entered method".
- Always with context: `player=`, `world=`, `region=`, `amount=`.
- Levels with meaning: `ERROR` — broken, needs a human; `WARN` — suspicious but survived; `INFO` — significant lifecycle events, rarely; `DEBUG` — behind a config flag, off by default.
- No `System.out.println`. Plugin logger only.
- Never log tokens, passwords, IPs or private player data.
- Do not spam every tick, or the admin stops reading logs entirely — including the important ones.

### Debugging
- Reproduce with a minimal scenario before fixing.
- Read the whole stack trace, `Caused by` included.
- An error "in another plugin" is, nine times out of ten, your argument reaching their code.
- Every production bug gets a regression test. Otherwise it comes back.

---

## 11. Testing

Test `domain/` first — there is no Bukkit there, so the tests are fast.

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
- MockBukkit sparingly, for API interaction. Do not build your whole suite on it.
- Test **boundaries**: zero, negative, maximum, empty string, `null`, duplicates, concurrent calls.
- Property-based testing (jqwik) for formulas and serialisation — it finds what you would never think of.
- A test name is a sentence about behaviour: `refundRestoresBalanceAfterCancelledPurchase`. Not `test1`, not `testWithdraw`.
- Do not test getters and mappers for coverage points. Coverage is a metric, not a goal.
- Manual scenario before release: clean install, upgrade from the previous version, reload, player quitting mid-operation.

---

## 12. Build, dependencies, CI

- **Reproducible builds:** pinned versions, no `latest`, no dynamic ranges.
- **Do not shade** what the server already ships: Adventure, Gson, Guava, SnakeYAML. Class conflicts break other people's plugins.
- Whatever you do shade must be `relocate`d into your own package.
- Minimise dependencies. Each one is a CVE, a version conflict and another 200 KB.
- `paper-plugin.yml` with an honest `api-version`, correct `depend` / `softdepend`, no gratuitous entries in `permissions`.
- SemVer: a breaking change to the public API means a major bump.

**CI, at minimum:**
1. `./gradlew build` on a clean clone.
2. Spotless check — the build fails on formatting violations.
3. ErrorProne / NullAway.
4. Tests.
5. OWASP Dependency-Check for known CVEs.
6. Artefact carrying version and commit hash in the manifest, so a jar can identify itself.

Red CI does not merge. No exceptions.

---

## 13. Security

### Input is always hostile

- Validate length, range, characters, encoding. Everything that comes from a player.
- SQL **only** through `PreparedStatement`. Concatenation is a vulnerability, not a style choice.
- File names from user input: normalise the path and confirm it stays inside `dataFolder`.

```java
Path target = dataFolder.resolve(name).normalize();
if (!target.startsWith(dataFolder)) throw new SecurityException("path traversal: " + name);
```

- Never deserialise untrusted data: `ObjectInputStream`, `BukkitObjectInputStream` from a foreign source, YAML with arbitrary tags.
- Limits on everything: string length, book pages, stack size, entities per spawn, JSON nesting depth. Otherwise your plugin becomes the crash vector.
- MiniMessage from user input must use a restricted tag set, or a player will inject `<click:run_command>`.

### Permissions and game logic

- Permission checks happen server-side, in the handler, before the action. A hidden GUI button is not a check.
- Never trust the client: slot, coordinates, inventory contents and movement speed all arrive from the client.
- Value operations are atomic: withdrawal and delivery in one transaction. Dupes are born in between.
- Rate limit expensive commands and GUI clicks.
- Never call `setOp` from code and never bypass permission checks "for convenience".
- Keep an audit log of significant actions (item grants, balance changes, admin commands) separately, with timestamp and actor.

### Secrets and network

- Tokens and passwords never go into git. Separate file plus `.gitignore`, or environment variables.
- HTTPS with certificate validation. Connect and read timeouts are mandatory.
- Cap response size, or a foreign server OOMs yours.
- Never execute code received over the network. Never, in any form.
- Update dependencies regularly.

### Resilience

- Degrade gracefully: database down means read-only mode with a clear message, not an NPE for every player.
- Circuit breaker on external services: three timeouts in a row means a pause, not infinite retries.
- An exception in one handler must not bring down the tick.

---

## 14. Readability and style

- The reader matters more than the writer: code is read ten times more often than written.
- Formatting is automated (Spotless). Style debates in review are banned.
- Class order: constants → fields → constructor → public methods → private ones. A private method sits right after its caller.
- Lines around 120 characters, wrapped by meaning rather than by counter.
- A blank line separates logical blocks inside a method. Three in a row, never.
- Nesting maxes out at three levels; beyond that, extract a method.
- Methods usually run 5–25 lines. Not "exactly 10" and not "a full page".
- `final` on fields that never change — documentation enforced by the compiler.
- `record` for DTOs and value objects.
- `sealed interface` plus pattern matching instead of an `instanceof` chain:

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

The compiler verifies exhaustiveness. Add a variant and you find out immediately, not in production.

- One idea per line. `if (a) { doX(); }` on a single line is an economy that costs bugs.
- Do not abbreviate: `transaction`, not `txn`, `trns`, `t`. Exceptions: conventional counters `i`, `x`, `z`.
- Magic numbers become named constants: `MAX_STACK_SIZE`, `COOLDOWN_TICKS`.

---

## 15. Compatibility and public API

- If you expose an API, extract a `-api` module. Internals never leak into it.
- The public API is stable: deprecate one release ahead and name the replacement.
- Custom events extend `Event` and implement `Cancellable` only when cancellation is meaningful. Fire them **before** the action, not after.
- Register services through `ServicesManager` so others can find you properly.
- Respect other plugins' cancellations, WorldGuard flags, Vault providers and permission systems.
- Do not wreck the neighbourhood: no mutation in `MONITOR`, no silently overriding other plugins' commands.
- bStats without personal data and with an opt-out.
- Verify upgrades against real data: old config plus old database plus new jar.

---

## 16. Antipattern catalogue

| Antipattern | Why it hurts | Do this instead |
|---|---|---|
| `Map<Player, X>` as a field | memory leak | `Map<UUID, X>` + cleanup on quit |
| `runTaskTimer(this, 0, 1)` | 20 runs/s for no reason | longer period, batching |
| `getNearbyEntities` in a tick | O(n) across chunks | spatial index, events |
| String chunk keys | garbage and slow hashing | packed `long` |
| `catch (Exception ignored)` | hidden bugs | log with context or rethrow |
| SQL concatenation | injection | `PreparedStatement` |
| Menu detection by title | spoofable | `InventoryHolder` |
| Hardcoded text in Java | untranslatable, unchangeable | `messages.yml` |
| `config.getX()` in a tick | YAML parsing in a hot path | load into a record at startup |
| `.join()` on main | server freeze | `thenAcceptAsync` on the main executor |
| Shading Adventure/Guava | class conflicts | do not shade, or relocate |
| Static `getInstance()` everywhere | hidden graph, untestable | constructor injection |
| `Utils`, `Manager`, `Helper` | no responsibility | name it after the domain |
| Cache without TTL | leak | Caffeine with a size limit and TTL |
| Restating comments | noise that goes stale | rename, delete |
| One-implementation interface "for later" | dead layer | the concrete class |
| Synchronous chunk load | lag spike | `getChunkAtAsync` |
| `onDisable` as the only save | data loss on crash | periodic flush |
| 300-line `onEnable` | unreadable, untestable | services plus wiring |
| NMS directly in 12 classes | dies on the next update | one bridge interface |

---

## 17. Review rubric

The reviewer walks this in order. The first "no" is already worth a conversation.

**Correctness**
- Do all called API methods exist in the target version?
- Are boundaries handled: zero, negative, maximum, empty, `null`?
- What happens if the player logs out mid-operation?
- Is there a path where data is saved twice or not at all?

**Threading**
- Is anything Bukkit-specific called from async?
- Is anything blocking called from main?
- Are shared structures protected?

**Performance**
- What here runs more than 20 times per second?
- Any allocations or linear scans in the hot path?
- Was a profile taken if hot code changed?

**Cleanliness**
- Any comments restating the code?
- Any dead code or speculative abstraction?
- Do the names speak the domain language?

**Security**
- Is input validated at the boundary?
- Are permissions checked server-side before the action?
- Are value operations atomic?

**Review reads what was deleted, not only what was added.**

---

## 18. Release checklist

- [ ] Every API method used was verified against the target version's sources
- [ ] No I/O, stray allocations or linear scans in the hot path
- [ ] Spark profile taken before and after optimisation, numbers in the PR
- [ ] No comments restating the code
- [ ] No hardcoded in-game text — everything in `messages.yml`
- [ ] Config is understandable without reading sources; versioned and migratable
- [ ] `domain/` imports no Bukkit and is covered by tests
- [ ] Input validated, SQL parameterised, limits in place
- [ ] Value operations are transactional and idempotent
- [ ] Exceptions are not swallowed, logs carry context
- [ ] No leaks: no stored `Player`/`Entity`, caches cleared on quit
- [ ] CI green: build, Spotless, ErrorProne, tests, CVE scan
- [ ] Dependencies do not clash: nothing unnecessary was shaded
- [ ] Upgrade from the previous version tested on a real config and database
- [ ] Verified on Folia if support is claimed
- [ ] CHANGELOG and README updated for humans, not for the checkbox
- [ ] Version follows SemVer, `api-version` is honest

---

<div align="center">

### 😁 Want to use this skill but have no good AI model?

**🫱 [@LomyPayBot](https://t.me/LomyPayBot) 🫲** — the best deal on the market for buying AI access

Promo code `AkyRayy` — **30% off**

<br>

<sub>Crafted by <b>AkyRayy</b></sub>

</div>
