# CombatKeepInventory — API Documentation

> A CombatTag (combat-tagging) system and death inventory keep/drop policy manager, with cross-server combat-state synchronization support via Velocity.
>
> **Version:** 1.2.0 · **Author:** ThanhNhan (GitHub: `ThanhNhan-thanhnhan0928`) · **License:** Apache License 2.0 · **Java:** 21

This document describes every public package, class, interface, and enum currently in the source code. It was produced by reading the source code directly (not just method signatures, but the default implementations as well) to ensure the descriptions match actual behavior at the time of writing.

---

## Table of Contents

1. [Overview](#overview)
2. [Module Structure & System Requirements](#module-structure--system-requirements)
3. [Adding CombatKeepInventory as a Dependency](#adding-combatkeepinventory-as-a-dependency)
4. [Architecture Model](#architecture-model)
5. [Quick Start](#quick-start)
6. [⚠️ Things to Know Before Integrating](#things-to-know-before-integrating)
7. [Core API — package `core.api`](#7-core-api--package-coreapi)
8. [Package `core.combat`](#8-package-corecombat)
9. [Package `core.damage`](#9-package-coredamage)
10. [Package `core.death`](#10-package-coredeath)
11. [Package `core.inventory`](#11-package-coreinventory)
12. [Package `core.platform`](#12-package-coreplatform)
13. [Package `core.event` (Not Yet Activated)](#13-package-coreevent-not-yet-activated)
14. [Package `core.hook`](#14-package-corehook)
15. [Package `core.exception`](#15-package-coreexception)
16. [Package `core.internal` (Default Implementations)](#16-package-coreinternal-default-implementations)
17. [Bukkit Module (`cki-bukkit`)](#17-bukkit-module-cki-bukkit)
18. [Velocity Module (`cki-velocity`)](#18-velocity-module-cki-velocity)
19. [Cross-Server Network Protocol — Channel `votri:combat`](#19-cross-server-network-protocol--channel-votricombat)
20. [Commands & Permissions](#20-commands--permissions)
21. [Default Decision Rules](#21-default-decision-rules)
22. [License](#22-license)

---

## Overview

CombatKeepInventory (CKI) is a system made up of 3 Maven modules:

| Module | Artifact | Role |
|---|---|---|
| `cki-core` | `com.votri:cki-core` | Contracts (interfaces/enums) and **platform-independent** logic — CombatTag, death evaluation, inventory keep/drop policy, platform detection. |
| `cki-bukkit` | `com.votri:cki-bukkit` | Paper/Spigot plugin — where combat state is actually created, listens for Bukkit events, integrates with WorldGuard, sends state to Velocity. |
| `cki-velocity` | `com.votri:cki-velocity` | Velocity plugin — receives and **mirrors** CombatTag state from Bukkit, tracks server switches/cluster exits, provides a manual punishment service. |

Main features:

- **CombatTag**: marks a player as being in combat for a configurable duration (`combat.duration-seconds`, default 10 seconds), automatically refreshing on additional hits.
- **Death policy**: decides whether to keep or drop the inventory/experience based on the cause of death and CombatTag state (see [section 21](#21-default-decision-rules)).
- **WorldGuard integration** (optional, loosely coupled via reflection): restricts the regions where PvP is allowed.
- **Bukkit → Velocity bridge**: sends CombatTag state over a Plugin Messaging channel so Velocity knows whether a player is in combat, even if they switch servers or leave the cluster.
- **Velocity-side punishment service**: `CombatPunishmentService` allows disconnecting players or running custom console commands — currently **exposed via the API for third parties to call themselves**, see the note in [section 6](#things-to-know-before-integrating).

No `README.md` file was found in the source code at the time this document was written — this is a standalone API reference that can be placed at `API.md` or `docs/API.md` in the repository.

---

## Module Structure & System Requirements

```
CombatKeepInventory (pom, groupId com.votri, version 1.2.0)
├── cki-core      → artifactId cki-core
├── cki-bukkit    → artifactId cki-bukkit   (depends on cki-core)
└── cki-velocity  → artifactId cki-velocity (depends on cki-core)
```

| Requirement | Value (per `pom.xml` / `plugin.yml`) |
|---|---|
| Java | 21 (`maven.compiler.release=21`) |
| Paper API (compiles `cki-bukkit`) | `1.21.11-R0.1-SNAPSHOT` |
| `plugin.yml` `api-version` | `1.21` |
| Velocity API (compiles `cki-velocity`) | `3.4.0-SNAPSHOT` |
| WorldGuard (optional, `provided`) | `7.0.16` |
| SnakeYAML (`cki-velocity` only) | `2.4` |

> **Version note:** the file `cki-velocity/src/main/resources/velocity-plugin.json` currently declares `"version": "1.1.0"`, while the `@Plugin(version = "1.2.0")` annotation in `CombatKeepInventoryVelocity.java` and the root `pom.xml` are both `1.2.0`. If you check the version via the `/plugins` command or proxy metadata, these two sources may not match.

---

## Adding CombatKeepInventory as a Dependency

The source code has no `<distributionManagement>` configuration or public repository set up to publish these 3 artifacts — CI (`.github/workflows/build.yml`) only builds and uploads the jar as a GitHub Actions artifact; it does not publish to Maven Central/JitPack/GitHub Packages. So, the practical way to compile against this API is:

1. Clone the repo and run `mvn -B clean install` to install `com.votri:cki-core:1.2.0` (and `cki-bukkit`/`cki-velocity` if needed) into your local Maven repository (`~/.m2`).
2. In your plugin, declare a `provided`-scope dependency (Bukkit) on `cki-bukkit`, or just `cki-core` if you only need the platform-independent data types:

```xml
<dependency>
    <groupId>com.votri</groupId>
    <artifactId>cki-bukkit</artifactId>
    <version>1.2.0</version>
    <scope>provided</scope>
</dependency>
```

3. Declare CombatKeepInventory as a `softdepend`/`depend` in your `plugin.yml` (Bukkit), or in the Velocity plugin's dependency list, to ensure it loads before your plugin.

---

## Architecture Model

**Bukkit is always the authoritative source of truth**; Velocity **never creates** a CombatTag on its own — it only stores a read-only mirror, received via Plugin Messaging from `CombatStateBridge` on the Bukkit side. This is explicitly documented in the source code's Javadoc for `ProxyCombatState`, `ProxyCombatStateManager`, and `VelocitySessionListener`.

```
Player A hits B on Bukkit server "survival-1"
        │
        ▼
CombatListener (Bukkit) → CombatManager.tag(A, B) → CombatSessionManager (core)
        │
        ▼
CombatStateBridge.publishStart(...)  ──[Plugin Messaging: channel "votri:combat"]──▶  VelocitySessionListener (Velocity)
                                                                                          │
                                                                                          ▼
                                                                          ProxyCombatStateManager.start(...)
                                                                          (stores only a copy, with its own timeout)
```

---

## Quick Start

### On Bukkit / Paper

The only entry point intended for third-party plugins is **`BukkitCombatKeepInventoryAPI`** (package `com.votri.combatkeepinv.bukkit.api`). Do not instantiate it yourself or call `register()`/`unregister()` — those two methods are for the internal lifecycle of the CombatKeepInventory plugin itself (`onEnable`/`onDisable`).

```java
import com.votri.combatkeepinv.bukkit.api.BukkitCombatKeepInventoryAPI;
import com.votri.combatkeepinv.core.api.CombatKeepInventoryAPI;

Optional<CombatKeepInventoryAPI> apiOpt = BukkitCombatKeepInventoryAPI.getRegistered();

if (apiOpt.isPresent()) {
    CombatKeepInventoryAPI api = apiOpt.get();
    boolean inCombat = api.getCombat().isInCombat(player.getUniqueId());
}

// Or, if you're certain CKI is enabled and want it to throw when not ready:
CombatKeepInventoryAPI api = BukkitCombatKeepInventoryAPI.requireRegistered();
```

### On Velocity

The entry point is **`VelocityCombatAPI.get()`** (package `com.votri.combatkeepinv.velocity.api`), which throws `IllegalStateException` if CKI Velocity hasn't finished initializing or has been disabled via configuration (`api.enabled` / `api.expose-api`).

```java
import com.votri.combatkeepinv.velocity.api.VelocityCombatAPI;

VelocityCombatAPI api = VelocityCombatAPI.get();

if (api.isInCombat(player.getUniqueId())) {
    long remaining = api.getRemainingCombatSeconds(player.getUniqueId());
}
```

---

## ⚠️ Things to Know Before Integrating

1. **Class names collide across packages — always check the full import.** There are 3 pairs of names that are identical but carry different meanings, and even the CKI source code itself has to use fully-qualified names in places to avoid confusion:

   | Name | Variant 1 | Variant 2 |
   |---|---|---|
   | `CombatState` | `core.api.CombatState` — enum `SAFE / IN_COMBAT / ENDING`, used by `CombatService.getCombatState()` | `core.combat.CombatState` — enum `NONE / ACTIVE / EXPIRED / ENDED / FORCED_END`, used by `CombatSession.getState()` |
   | `DeathContext` | `core.api.DeathContext` — a simple **enum**: `PLAYER / PROJECTILE / MOB / ENVIRONMENT / VOID / UNKNOWN` (the "legacy" path, used by `CombatService.evaluateDeath(...)` and `CombatHook.onDeath(...)`) | `core.death.DeathContext` — a data-rich **interface** (killer, combat session, PvP classification, etc.), used by `DeathService`/`DeathAPI.createContext(...)` |
   | `InventoryPolicy` | `core.api.InventoryPolicy` — enum `KEEP / DROP / DEFAULT`, used in `DeathResult` | `core.inventory.InventoryPolicy` — a detailed interface (`keepArmor()`, `keepHotbar()`, ...), used by `InventoryAPI.getDefaultPolicy()` |

2. **`CombatAPI` ≠ `CombatService`.** Both live in the `core.api` package, but they are two independent interfaces with no inheritance relationship. `CombatAPI` is the public-facing facade (obtained via `CombatKeepInventoryAPI.getCombat()`), working directly with `CombatSession`/`CombatSessionManager`. `CombatService` is a lower-level interface implemented by `BukkitCombatService`, obtainable only via `CombatKeepInventory#getCombatService()` (i.e., you must cast to the main plugin class, not go through `CombatKeepInventoryAPI`).

3. **`core.event.*` and `core.hook.CombatHook` are currently NOT activated.** The entire source code has been checked: nowhere is `CombatStartEvent`, `CombatEndEvent`, `CombatRefreshEvent`, `CombatDeathEvent`, `DeathEvaluateEvent`, or `InventoryPolicyEvent` ever instantiated, nor is any method of `CombatHook` ever called anywhere. The plugin also never fires any custom Bukkit `Event` of its own — `CombatListener` only **listens** to two vanilla Bukkit events (`EntityDamageByEntityEvent`, `PlayerDeathEvent`). The `CombatPlayer` interface also has no implementing class anywhere. In other words, these classes exist as contracts reserved for future use; **to integrate right now, call the APIs in section 7 directly (polling) — there is no event-listener/hook mechanism of CKI's own.**

4. **Automatic punishment configuration on the Velocity side is not wired up.** `VelocityConfig` has the keys `punishment.on-server-switch.*` and `punishment.on-cluster-exit.*`, and `CombatPunishmentService.executeCommands(...)` already exists and is ready to use — but the `handleTransition(...)` method in `CombatKeepInventoryVelocity` (where server-switch/cluster-exit events are handled) currently **never calls** `punishmentService` anywhere. If you want an "auto-punish on combat-log" feature, you need to listen for it yourself (e.g., via your own Velocity event, combined with `VelocityCombatAPI.get().isInCombat(...)`) and call `VelocityCombatAPI.get().punishments().executeCommands(...)` manually.

5. **`core.exception.*` has never been `throw`n** anywhere in the current source code — no special `catch` is needed for `CombatKeepInventoryException`/`InvalidCombatSessionException` when calling the existing APIs.

6. **`PlatformInfo.supports(PlatformCapability)` on the Velocity side always returns `false`**, regardless of the parameter passed (`VelocityPlatformDetector.detect(...)` hard-codes `return false;`). On the Bukkit side, `BukkitPlatformDetector` declares support for `COMBAT, DEATH, INVENTORY, DAMAGE_ATTRIBUTION, PLUGIN_MESSAGING, EVENTS`.

7. **No class implements the `core.platform.PlatformDetector` interface** — `BukkitPlatformDetector`/`VelocityPlatformDetector` are two independent static utility classes (`static PlatformInfo detect(...)`) that do not implement this interface (their parameter signatures also differ).

---

## 7. Core API — package `core.api`

This is the main API surface, platform-independent. Full package: `com.votri.combatkeepinv.core.api`.

### `CombatKeepInventoryAPI`

The main facade — everything else is reached from here.

```java
public interface CombatKeepInventoryAPI {
    CombatAPI getCombat();
    DamageAPI getDamage();
    DeathAPI getDeath();
    InventoryAPI getInventory();
    PlatformAPI getPlatform();
}
```

### `CombatAPI`

```java
public interface CombatAPI {
    CombatSessionManager getSessionManager();
    Optional<CombatSession> getSession(UUID playerId);
    boolean isInCombat(UUID playerId);
    CombatSession startCombat(UUID playerId);
    CombatSession startCombat(UUID playerId, UUID opponentId);
    CombatSession refreshCombat(UUID playerId);
    CombatSession refreshCombat(UUID playerId, UUID opponentId);
    boolean endCombat(UUID playerId, CombatReason reason);
}
```

`CombatSession`, `CombatSessionManager`, and `CombatReason` belong to the `core.combat` package — see [section 8](#8-package-corecombat).

### `CombatService`

A lower-level interface, implemented by `BukkitCombatService` (see [section 17](#17-bukkit-module-cki-bukkit)). Not exposed via `CombatKeepInventoryAPI` — only obtainable via `CombatKeepInventory#getCombatService()` on the Bukkit side.

```java
public interface CombatService {
    CombatResult startCombat(UUID attacker, UUID victim);
    CombatResult refreshCombat(UUID attacker, UUID victim);
    CombatResult endCombat(UUID player);
    CombatResult forceEndCombat(UUID player);
    boolean isInCombat(UUID player);
    CombatState getCombatState(UUID player);          // core.api.CombatState
    CombatTag getCombatTag(UUID player);
    long getRemainingCombatMillis(UUID player);
    DeathResult evaluateDeath(UUID player, DeathContext context); // core.api.DeathContext (enum!)
    boolean isEnabled();
    DamageAttributionService getDamageAttributionService();
    DeathService getDeathService();
}
```

### `DamageAPI`

```java
public interface DamageAPI {
    DamageAttributionService getAttributionService();
    Optional<DamageSource> getLastDamage(UUID playerId);
}
```

### `DeathAPI`

```java
public interface DeathAPI {
    DeathService getDeathService();
    DeathContext createContext(UUID playerId, DamageSource damageSource); // core.death.DeathContext (interface!)
    DeathDecision evaluate(DeathContext context);
}
```

> Note: although both live in `core.api`, `createContext(...)` here returns `com.votri.combatkeepinv.core.death.DeathContext` (the data-rich interface), which is **different** from the `DeathContext` enum used in `CombatService.evaluateDeath(...)` — see [section 6, point 1](#things-to-know-before-integrating).

### `InventoryAPI`

```java
public interface InventoryAPI {
    InventoryPolicy getDefaultPolicy();  // core.inventory.InventoryPolicy (interface!)
    InventoryDecision createDecision(InventoryAction action, InventoryPolicy policy);
}
```

### `PlatformAPI`

```java
public interface PlatformAPI {
    PlatformInfo getPlatform();
    boolean supports(PlatformCapability capability);
}
```

### `CombatPlayer`

```java
public interface CombatPlayer {
    UUID getUniqueId();
    String getName();
    boolean hasPermission(String permission);
    boolean isOnline();
}
```

> No class in the codebase implements this interface (see [section 6, point 3](#things-to-know-before-integrating)).

### `CombatTag`

A read-only representation of an active CombatTag.

```java
public interface CombatTag {
    UUID getPlayerId();
    boolean isActive();
    long getRemainingMillis();
    default long getRemainingSeconds();   // rounded up to the nearest second
    long getExpiresAt();                  // epoch millis timestamp
    UUID getLastOpponent();               // null if none
    default boolean hasOpponent();
}
```

There are 2 classes implementing `CombatTag`: `CombatManager.CoreCombatTagView` (internal, Bukkit) and `ProxyCombatState` (Velocity, see [section 18](#18-velocity-module-cki-velocity)).

### `DeathResult`

An immutable data class representing the final outcome of a death.

```java
public final class DeathResult {
    public DeathResult(InventoryPolicy inventoryPolicy, boolean keepExperience, boolean wasCombatDeath);
    public InventoryPolicy getInventoryPolicy();  // core.api.InventoryPolicy (enum)
    public boolean shouldKeepInventory();          // == InventoryPolicy.KEEP
    public boolean shouldDropInventory();           // == InventoryPolicy.DROP
    public boolean shouldKeepExperience();
    public boolean wasCombatDeath();
}
```

### Enum: `CombatResult`

| Value | Meaning |
|---|---|
| `SUCCESS` | The operation succeeded. |
| `ALREADY_IN_COMBAT` | A CombatTag already existed; no need to create a new one. |
| `NOT_IN_COMBAT` | The player currently has no CombatTag. |
| `PLAYER_NOT_FOUND` | The player's UUID could not be resolved. |
| `INVALID_ARGUMENT` | An invalid argument was passed. |
| `DISABLED` | The combat system is disabled. |
| `WORLD_DISABLED` | Combat is disabled in the relevant world. |
| `IMMUNE` | The player is immune to combat-tagging. |
| `FAILED` | Failure for a reason specific to the implementation layer. |

### Enum: `CombatState` (package `core.api`)

| Value | Meaning |
|---|---|
| `SAFE` | Not in combat. |
| `IN_COMBAT` | Currently in combat. |
| `ENDING` | Combat is ending. |

### Enum: `DeathContext` (package `core.api`)

A simplified ("legacy") version of the cause of death, used by `CombatService.evaluateDeath(...)` and `CombatHook.onDeath(...)`.

| Value | Meaning |
|---|---|
| `PLAYER` | Died to another player. |
| `PROJECTILE` | Died to a projectile (fired by a player). |
| `MOB` | Died to a mob. |
| `ENVIRONMENT` | Died to the environment. |
| `VOID` | Fell into the void. |
| `UNKNOWN` | Unknown. |

### Enum: `InventoryPolicy` (package `core.api`)

| Value | Meaning |
|---|---|
| `KEEP` | Keep the inventory. |
| `DROP` | Drop the inventory. |
| `DEFAULT` | *(defined but currently unused by any internal logic — `DefaultDeathService` always explicitly constructs either `KEEP` or `DROP`.)* |

---
