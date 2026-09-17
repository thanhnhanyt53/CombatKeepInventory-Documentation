# CombatKeepInventory — Tài liệu API

> Hệ thống CombatTag (đánh dấu giao chiến) và quản lý giữ/rơi đồ khi chết, có hỗ trợ đồng bộ trạng thái giao chiến xuyên máy chủ qua Velocity.
>
> **Phiên bản:** 1.2.0 · **Tác giả:** ThanhNhan (GitHub: `ThanhNhan-thanhnhan0928`) · **Giấy phép:** Apache License 2.0 · **Java:** 21

Tài liệu này mô tả toàn bộ package, class, interface và enum công khai (public) hiện có trong mã nguồn, được tạo ra bằng cách đọc trực tiếp source code (không chỉ chữ ký hàm mà cả phần triển khai mặc định) để đảm bảo mô tả đúng hành vi thực tế tại thời điểm viết tài liệu.

---

## Mục lục

1. [Tổng quan](#tổng-quan)
2. [Cấu trúc module & Yêu cầu hệ thống](#cấu-trúc-module--yêu-cầu-hệ-thống)
3. [Thêm CombatKeepInventory làm dependency](#thêm-combatkeepinventory-làm-dependency)
4. [Mô hình kiến trúc](#mô-hình-kiến-trúc)
5. [Bắt đầu nhanh](#bắt-đầu-nhanh)
6. [⚠️ Những điều cần biết trước khi tích hợp](#những-điều-cần-biết-trước-khi-tích-hợp)
7. [Core API — package `core.api`](#7-core-api--package-coreapi)
8. [Package `core.combat`](#8-package-corecombat)
9. [Package `core.damage`](#9-package-coredamage)
10. [Package `core.death`](#10-package-coredeath)
11. [Package `core.inventory`](#11-package-coreinventory)
12. [Package `core.platform`](#12-package-coreplatform)
13. [Package `core.event` (chưa được kích hoạt)](#13-package-coreevent-chưa-được-kích-hoạt)
14. [Package `core.hook`](#14-package-corehook)
15. [Package `core.exception`](#15-package-coreexception)
16. [Package `core.internal` (triển khai mặc định)](#16-package-coreinternal-triển-khai-mặc-định)
17. [Module Bukkit (`cki-bukkit`)](#17-module-bukkit-cki-bukkit)
18. [Module Velocity (`cki-velocity`)](#18-module-velocity-cki-velocity)
19. [Giao thức mạng liên server — kênh `votri:combat`](#19-giao-thức-mạng-liên-server--kênh-votricombat)
20. [Lệnh & Quyền](#20-lệnh--quyền)
21. [Quy tắc quyết định mặc định](#21-quy-tắc-quyết-định-mặc-định)
22. [Giấy phép](#22-giấy-phép)

---

## Tổng quan

CombatKeepInventory (CKI) là hệ thống gồm 3 module Maven:

| Module | Artifact | Vai trò |
|---|---|---|
| `cki-core` | `com.votri:cki-core` | Hợp đồng (interface/enum) và logic **không phụ thuộc nền tảng** — CombatTag, đánh giá cái chết, chính sách giữ/rơi đồ, phát hiện nền tảng. |
| `cki-bukkit` | `com.votri:cki-bukkit` | Plugin Paper/Spigot — nơi trạng thái giao chiến thực sự được tạo ra, lắng nghe sự kiện Bukkit, tích hợp WorldGuard, gửi trạng thái sang Velocity. |
| `cki-velocity` | `com.votri:cki-velocity` | Plugin Velocity — nhận và **phản chiếu** (mirror) trạng thái CombatTag từ Bukkit, theo dõi chuyển server/thoát cụm, cung cấp dịch vụ trừng phạt thủ công. |

Chức năng chính:

- **CombatTag**: đánh dấu người chơi đang trong giao chiến trong một khoảng thời gian có thể cấu hình (`combat.duration-seconds`, mặc định 10 giây), tự động refresh khi có thêm đòn đánh.
- **Chính sách khi chết**: quyết định giữ hay rơi túi đồ/kinh nghiệm dựa trên nguyên nhân chết và trạng thái CombatTag (xem [mục 21](#21-quy-tắc-quyết-định-mặc-định)).
- **Tích hợp WorldGuard** (tùy chọn, mềm dẻo qua reflection): giới hạn vùng cho phép PvP.
- **Cầu nối Bukkit → Velocity**: gửi trạng thái CombatTag qua kênh Plugin Messaging để Velocity biết người chơi có đang giao chiến hay không, kể cả khi họ đổi server hoặc rời cụm.
- **Dịch vụ trừng phạt phía Velocity**: `CombatPunishmentService` cho phép ngắt kết nối hoặc chạy lệnh console tùy chỉnh — hiện được **expose qua API để bên thứ ba tự gọi**, xem lưu ý ở [mục 6](#những-điều-cần-biết-trước-khi-tích-hợp).

Không tìm thấy file `README.md` nào trong mã nguồn tại thời điểm viết tài liệu này — đây là tài liệu tham khảo API độc lập, có thể đặt tại `API.md` hoặc `docs/API.md` trong repository.

---

## Cấu trúc module & Yêu cầu hệ thống

```
CombatKeepInventory (pom, groupId com.votri, version 1.2.0)
├── cki-core      → artifactId cki-core
├── cki-bukkit    → artifactId cki-bukkit   (phụ thuộc cki-core)
└── cki-velocity  → artifactId cki-velocity (phụ thuộc cki-core)
```

| Yêu cầu | Giá trị (theo `pom.xml` / `plugin.yml`) |
|---|---|
| Java | 21 (`maven.compiler.release=21`) |
| Paper API (biên dịch `cki-bukkit`) | `1.21.11-R0.1-SNAPSHOT` |
| `plugin.yml` `api-version` | `1.21` |
| Velocity API (biên dịch `cki-velocity`) | `3.4.0-SNAPSHOT` |
| WorldGuard (tùy chọn, `provided`) | `7.0.16` |
| SnakeYAML (chỉ `cki-velocity`) | `2.4` |

> **Lưu ý phiên bản:** file `cki-velocity/src/main/resources/velocity-plugin.json` hiện khai báo `"version": "1.1.0"`, trong khi annotation `@Plugin(version = "1.2.0")` trong `CombatKeepInventoryVelocity.java` và `pom.xml` gốc đều là `1.2.0`. Nếu bạn kiểm tra phiên bản qua lệnh `/plugins` hoặc metadata proxy, hai nguồn này có thể không khớp nhau.

---

## Thêm CombatKeepInventory làm dependency

Mã nguồn không có cấu hình `<distributionManagement>` hay repository công khai nào để publish 3 artifact này — CI (`.github/workflows/build.yml`) chỉ build và upload jar làm artifact của GitHub Actions, không publish lên Maven Central/JitPack/GitHub Packages. Vì vậy, cách thực tế để biên dịch chống lại API này:

1. Clone repo, chạy `mvn -B clean install` để cài `com.votri:cki-core:1.2.0` (và `cki-bukkit`/`cki-velocity` nếu cần) vào kho Maven local (`~/.m2`).
2. Trong plugin của bạn, khai báo dependency scope `provided` (Bukkit) tới `cki-bukkit`, hoặc chỉ `cki-core` nếu bạn chỉ cần các kiểu dữ liệu không phụ thuộc nền tảng:

```xml
<dependency>
    <groupId>com.votri</groupId>
    <artifactId>cki-bukkit</artifactId>
    <version>1.2.0</version>
    <scope>provided</scope>
</dependency>
```

3. Khai báo CombatKeepInventory là `softdepend`/`depend` trong `plugin.yml` của bạn (Bukkit) hoặc trong danh sách phụ thuộc plugin Velocity, để đảm bảo nó được nạp trước plugin của bạn.

---

## Mô hình kiến trúc

**Bukkit luôn là nguồn sự thật (authoritative)**; Velocity **không bao giờ tự tạo** CombatTag — nó chỉ lưu một bản phản chiếu (mirror) chỉ-đọc, nhận qua Plugin Messaging từ `CombatStateBridge` phía Bukkit. Điều này được chính mã nguồn ghi rõ trong Javadoc của `ProxyCombatState`, `ProxyCombatStateManager` và `VelocitySessionListener`.

```
Người chơi A đánh B trên server Bukkit "survival-1"
        │
        ▼
CombatListener (Bukkit) → CombatManager.tag(A, B) → CombatSessionManager (core)
        │
        ▼
CombatStateBridge.publishStart(...)  ──[Plugin Messaging: kênh "votri:combat"]──▶  VelocitySessionListener (Velocity)
                                                                                          │
                                                                                          ▼
                                                                          ProxyCombatStateManager.start(...)
                                                                          (chỉ lưu bản sao, có timeout riêng)
```

---

## Bắt đầu nhanh

### Trên Bukkit / Paper

Điểm vào duy nhất dành cho plugin bên thứ ba là **`BukkitCombatKeepInventoryAPI`** (package `com.votri.combatkeepinv.bukkit.api`). Không tự khởi tạo hay gọi `register()`/`unregister()` — hai hàm đó dành cho vòng đời nội bộ của chính plugin CombatKeepInventory (`onEnable`/`onDisable`).

```java
import com.votri.combatkeepinv.bukkit.api.BukkitCombatKeepInventoryAPI;
import com.votri.combatkeepinv.core.api.CombatKeepInventoryAPI;

Optional<CombatKeepInventoryAPI> apiOpt = BukkitCombatKeepInventoryAPI.getRegistered();

if (apiOpt.isPresent()) {
    CombatKeepInventoryAPI api = apiOpt.get();
    boolean inCombat = api.getCombat().isInCombat(player.getUniqueId());
}

// Hoặc, nếu bạn chắc chắn CKI đã bật và muốn ném lỗi khi chưa sẵn sàng:
CombatKeepInventoryAPI api = BukkitCombatKeepInventoryAPI.requireRegistered();
```

### Trên Velocity

Điểm vào là **`VelocityCombatAPI.get()`** (package `com.votri.combatkeepinv.velocity.api`), ném `IllegalStateException` nếu CKI Velocity chưa khởi tạo xong hoặc bị tắt qua cấu hình (`api.enabled` / `api.expose-api`).

```java
import com.votri.combatkeepinv.velocity.api.VelocityCombatAPI;

VelocityCombatAPI api = VelocityCombatAPI.get();

if (api.isInCombat(player.getUniqueId())) {
    long remaining = api.getRemainingCombatSeconds(player.getUniqueId());
}
```

---

## ⚠️ Những điều cần biết trước khi tích hợp

1. **Trùng tên class giữa các package — luôn kiểm tra import đầy đủ.** Có 3 cặp tên trùng nhau nhưng mang ý nghĩa khác nhau, và ngay trong chính mã nguồn CKI cũng có chỗ phải dùng tên đầy đủ (fully-qualified name) để tránh nhầm lẫn:

   | Tên | Biến thể 1 | Biến thể 2 |
   |---|---|---|
   | `CombatState` | `core.api.CombatState` — enum `SAFE / IN_COMBAT / ENDING`, dùng bởi `CombatService.getCombatState()` | `core.combat.CombatState` — enum `NONE / ACTIVE / EXPIRED / ENDED / FORCED_END`, dùng bởi `CombatSession.getState()` |
   | `DeathContext` | `core.api.DeathContext` — **enum** đơn giản `PLAYER / PROJECTILE / MOB / ENVIRONMENT / VOID / UNKNOWN` (đường dẫn "legacy", dùng bởi `CombatService.evaluateDeath(...)`, `CombatHook.onDeath(...)`) | `core.death.DeathContext` — **interface** đầy đủ thông tin (killer, combat session, phân loại PvP...), dùng bởi `DeathService`/`DeathAPI.createContext(...)` |
   | `InventoryPolicy` | `core.api.InventoryPolicy` — enum `KEEP / DROP / DEFAULT`, dùng trong `DeathResult` | `core.inventory.InventoryPolicy` — interface chi tiết (`keepArmor()`, `keepHotbar()`...), dùng bởi `InventoryAPI.getDefaultPolicy()` |

2. **`CombatAPI` ≠ `CombatService`.** Cả hai cùng nằm trong package `core.api` nhưng là hai interface độc lập, không kế thừa nhau. `CombatAPI` là mặt tiền công khai (lấy qua `CombatKeepInventoryAPI.getCombat()`), làm việc trực tiếp với `CombatSession`/`CombatSessionManager`. `CombatService` là interface ở mức thấp hơn mà `BukkitCombatService` triển khai, chỉ lấy được qua `CombatKeepInventory#getCombatService()` (tức là phải ép kiểu plugin chính, không đi qua `CombatKeepInventoryAPI`).

3. **`core.event.*` và `core.hook.CombatHook` hiện KHÔNG được kích hoạt.** Đã kiểm tra toàn bộ mã nguồn: không có nơi nào khởi tạo `CombatStartEvent`, `CombatEndEvent`, `CombatRefreshEvent`, `CombatDeathEvent`, `DeathEvaluateEvent`, `InventoryPolicyEvent`, cũng không có nơi nào gọi bất kỳ phương thức nào của `CombatHook`. Plugin cũng không tự phát (fire) bất kỳ Bukkit `Event` tùy chỉnh nào — `CombatListener` chỉ **lắng nghe** hai sự kiện vanilla của Bukkit (`EntityDamageByEntityEvent`, `PlayerDeathEvent`). `CombatPlayer` (interface) cũng không có bất kỳ lớp nào triển khai nó. Nói cách khác, các lớp này tồn tại như hợp đồng dự phòng cho tương lai; **để tích hợp ngay bây giờ, hãy gọi trực tiếp các API ở mục 7 (poll), không có cơ chế lắng nghe sự kiện/hook của riêng CKI.**

4. **Cấu hình trừng phạt tự động phía Velocity chưa được nối dây.** `VelocityConfig` có các khóa `punishment.on-server-switch.*` và `punishment.on-cluster-exit.*`, và `CombatPunishmentService.executeCommands(...)` tồn tại sẵn sàng để dùng — nhưng phương thức `handleTransition(...)` trong `CombatKeepInventoryVelocity` (nơi xử lý sự kiện chuyển server/rời cụm) hiện **không gọi** `punishmentService` ở bất kỳ đâu. Nếu bạn muốn tính năng "tự động trừng phạt khi combat-log", bạn cần tự lắng nghe (ví dụ qua sự kiện Velocity của riêng bạn, kết hợp `VelocityCombatAPI.get().isInCombat(...)`) và gọi `VelocityCombatAPI.get().punishments().executeCommands(...)` thủ công.

5. **`core.exception.*` chưa từng được `throw`** ở bất kỳ đâu trong mã nguồn hiện tại — không cần `catch` đặc biệt cho `CombatKeepInventoryException`/`InvalidCombatSessionException` khi gọi các API hiện có.

6. **`PlatformInfo.supports(PlatformCapability)` phía Velocity luôn trả về `false`** bất kể tham số truyền vào là gì (`VelocityPlatformDetector.detect(...)` hard-code `return false;`). Phía Bukkit, `BukkitPlatformDetector` khai báo hỗ trợ `COMBAT, DEATH, INVENTORY, DAMAGE_ATTRIBUTION, PLUGIN_MESSAGING, EVENTS`.

7. **`core.platform.PlatformDetector` (interface) không có lớp nào triển khai nó** — `BukkitPlatformDetector`/`VelocityPlatformDetector` là hai lớp tiện ích tĩnh (`static PlatformInfo detect(...)`) độc lập, không implement interface này (chữ ký tham số cũng khác nhau).

---

## 7. Core API — package `core.api`

Đây là bề mặt API chính, không phụ thuộc nền tảng. Package đầy đủ: `com.votri.combatkeepinv.core.api`.

### `CombatKeepInventoryAPI`

Điểm gộp (facade) chính — mọi thứ khác đi ra từ đây.

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

`CombatSession`, `CombatSessionManager`, `CombatReason` thuộc package `core.combat` — xem [mục 8](#8-package-corecombat).

### `CombatService`

Interface ở mức thấp hơn, được `BukkitCombatService` triển khai (xem [mục 17](#17-module-bukkit-cki-bukkit)). Không expose qua `CombatKeepInventoryAPI` — chỉ lấy qua `CombatKeepInventory#getCombatService()` phía Bukkit.

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

> Lưu ý: dù cùng nằm trong `core.api`, `createContext(...)` ở đây trả về `com.votri.combatkeepinv.core.death.DeathContext` (interface giàu dữ liệu), **khác** với `DeathContext` enum dùng trong `CombatService.evaluateDeath(...)` — xem [mục 6, điểm 1](#những-điều-cần-biết-trước-khi-tích-hợp).

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

> Không có lớp nào trong codebase triển khai interface này (xem [mục 6, điểm 3](#những-điều-cần-biết-trước-khi-tích-hợp)).

### `CombatTag`

Đại diện chỉ-đọc cho một CombatTag đang hoạt động.

```java
public interface CombatTag {
    UUID getPlayerId();
    boolean isActive();
    long getRemainingMillis();
    default long getRemainingSeconds();   // làm tròn lên tới giây
    long getExpiresAt();                  // mốc epoch millis
    UUID getLastOpponent();               // null nếu không có
    default boolean hasOpponent();
}
```

Có 2 lớp triển khai `CombatTag`: `CombatManager.CoreCombatTagView` (nội bộ, Bukkit) và `ProxyCombatState` (Velocity, xem [mục 18](#18-module-velocity-cki-velocity)).

### `DeathResult`

Lớp dữ liệu bất biến (immutable), kết quả cuối cùng cho một lần chết.

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

| Giá trị | Ý nghĩa |
|---|---|
| `SUCCESS` | Thao tác thành công. |
| `ALREADY_IN_COMBAT` | CombatTag đã tồn tại từ trước, không cần tạo mới. |
| `NOT_IN_COMBAT` | Người chơi hiện không có CombatTag. |
| `PLAYER_NOT_FOUND` | Không phân giải được UUID người chơi. |
| `INVALID_ARGUMENT` | Tham số truyền vào không hợp lệ. |
| `DISABLED` | Hệ thống combat đang bị tắt. |
| `WORLD_DISABLED` | Combat bị tắt ở world liên quan. |
| `IMMUNE` | Người chơi miễn nhiễm combat-tag. |
| `FAILED` | Thất bại vì lý do riêng của tầng triển khai. |

### Enum: `CombatState` (package `core.api`)

| Giá trị | Ý nghĩa |
|---|---|
| `SAFE` | Không trong giao chiến. |
| `IN_COMBAT` | Đang giao chiến. |
| `ENDING` | Đang kết thúc giao chiến. |

### Enum: `DeathContext` (package `core.api`)

Phiên bản đơn giản hóa ("legacy") của nguyên nhân chết, dùng bởi `CombatService.evaluateDeath(...)` và `CombatHook.onDeath(...)`.

| Giá trị | Ý nghĩa |
|---|---|
| `PLAYER` | Chết do người chơi khác. |
| `PROJECTILE` | Chết do đạn (do người chơi bắn). |
| `MOB` | Chết do quái vật. |
| `ENVIRONMENT` | Chết do môi trường. |
| `VOID` | Rơi vào hư không. |
| `UNKNOWN` | Không xác định. |

### Enum: `InventoryPolicy` (package `core.api`)

| Giá trị | Ý nghĩa |
|---|---|
| `KEEP` | Giữ túi đồ. |
| `DROP` | Làm rơi túi đồ. |
| `DEFAULT` | *(định nghĩa sẵn nhưng hiện không được logic nội bộ nào sử dụng — `DefaultDeathService` luôn dựng tường minh `KEEP` hoặc `DROP`.)* |

---

## 8. Package `core.combat`

### `CombatSession`

```java
public interface CombatSession {
    UUID getPlayerId();
    CombatState getState();                 // core.combat.CombatState
    long getStartedAt();
    long getLastActivityAt();
    long getExpiresAt();
    long getRemainingMillis();
    boolean isActive();
    boolean isExpired();
    Optional<UUID> getOpponentId();
    Optional<UUID> getLastAttackerId();
    Optional<UUID> getLastVictimId();
    Optional<CombatReason> getEndReason();
}
```

### `CombatSessionManager`

```java
public interface CombatSessionManager {
    Optional<CombatSession> getSession(UUID playerId);
    boolean isInCombat(UUID playerId);
    CombatSession startCombat(UUID playerId);
    CombatSession startCombat(UUID playerId, UUID opponentId);
    CombatSession refreshCombat(UUID playerId);
    CombatSession refreshCombat(UUID playerId, UUID opponentId);
    boolean endCombat(UUID playerId, CombatReason reason);
    boolean forceEndCombat(UUID playerId, CombatReason reason);
    void removeSession(UUID playerId);
    Collection<CombatSession> getActiveSessions();
    void cleanupExpiredSessions(long currentTimeMillis);
}
```

Triển khai mặc định: `DefaultCombatSessionManager` (package `core.internal`) — dùng `ConcurrentHashMap`, `durationMillis` phải `> 0`, `startCombat(...)` sẽ tái sử dụng session cũ nếu còn `isActive()`, ngược lại tạo mới.

### `CombatParticipant`

```java
public interface CombatParticipant {
    UUID getPlayerId();
    long getFirstHitAt();
    long getLastHitAt();
    long getExpiresAt();
    default boolean isExpired(long currentTimeMillis);
}
```

### Enum: `CombatReason`

| Giá trị | Ý nghĩa |
|---|---|
| `PLAYER_DAMAGE` | Do sát thương từ người chơi. |
| `PLAYER_ATTACKED` | Do bị người chơi tấn công. |
| `TIMEOUT` | Hết thời gian CombatTag. |
| `PLAYER_DEATH` | Do người chơi chết. |
| `PLAYER_QUIT` | Do người chơi thoát. |
| `SERVER_SWITCH` | Do chuyển server (Velocity). |
| `CLUSTER_EXIT` | Do rời khỏi cụm proxy. |
| `ADMIN` | Do quản trị viên can thiệp. |
| `SYSTEM` | Do hệ thống (ví dụ: `CombatManager.clear()` khi tắt plugin). |
| `UNKNOWN` | Không xác định. |

### Enum: `CombatState` (package `core.combat`)

| Giá trị | Ý nghĩa |
|---|---|
| `NONE` | Chưa từng có session. |
| `ACTIVE` | Đang hoạt động. |
| `EXPIRED` | Hết hạn tự nhiên (timeout). |
| `ENDED` | Kết thúc bình thường. |
| `FORCED_END` | Bị buộc kết thúc (ví dụ: admin, shutdown). |

---

## 9. Package `core.damage`

### `DamageSource`

```java
public interface DamageSource {
    DamageCauseType getCauseType();
    Optional<UUID> getDirectAttackerId();      // người/thực thể trực tiếp gây sát thương (có thể là đạn)
    Optional<UUID> getResponsiblePlayerId();   // người chơi chịu trách nhiệm cuối cùng (bắn tên → là người bắn)
    Optional<UUID> getVictimId();
    boolean isPlayerCaused();
    boolean isIndirect();                      // true nếu qua trung gian (đạn, TNT...)
    long getTimestamp();
}
```

### `DamageAttributionResult`

```java
public interface DamageAttributionResult {
    UUID getVictimId();
    DamageSource getDamageSource();
    Optional<UUID> getKillerId();
    boolean isPlayerCaused();
    boolean isDirectPvP();     // playerCaused && causeType == PLAYER (cận chiến)
    boolean isIndirectPvP();   // playerCaused && !isDirectPvP() (đạn, nổ do người chơi...)
}
```

### `DamageAttributionService`

```java
public interface DamageAttributionService {
    DamageAttributionResult resolve(UUID victimId, DamageSource rawSource);
    Optional<DamageSource> getLastDamage(UUID victimId);
    void recordDamage(DamageSource source);
    void clearDamageHistory(UUID victimId);
    void clearAll();
}
```

Triển khai mặc định `DefaultDamageAttributionService` lưu "sát thương cuối cùng" theo `victimId` trong `ConcurrentHashMap` (không giới hạn thời gian sống, chỉ bị xoá khi `clearDamageHistory`/`clearAll` được gọi tường minh).

### Enum: `DamageCauseType`

`PLAYER`, `PROJECTILE`, `TNT`, `EXPLOSION`, `FIRE`, `FIRE_TICK`, `FALL`, `VOID`, `LAVA`, `DROWNING`, `ENTITY`, `MAGIC`, `POISON`, `WITHER`, `STARVATION`, `SUFFOCATION`, `LIGHTNING`, `UNKNOWN`.

---

## 10. Package `core.death`

### `DeathContext` (interface — khác với enum cùng tên ở `core.api`)

```java
public interface DeathContext {
    UUID getPlayerId();
    long getTimestamp();
    DamageSource getDamageSource();
    DeathReason getDeathReason();
    Optional<UUID> getKillerId();
    Optional<CombatSession> getCombatSession();
    boolean wasInCombat();
    boolean wasPlayerCaused();
    boolean isDirectPvP();
    boolean isIndirectPvP();
    PlatformInfo getPlatform();
}
```

### `DeathDecision`

```java
public interface DeathDecision {
    boolean shouldKeepInventory();
    InventoryPolicy getInventoryPolicy();   // core.inventory.InventoryPolicy (interface)
    boolean shouldKeepExperience();
    boolean shouldDropExperience();
    DeathReason getReason();
    String getRuleId();                     // "pvp-combat" hoặc "default" — xem mục 21
}
```

### `DeathService`

```java
public interface DeathService {
    DeathContext createContext(UUID playerId, DamageSource damageSource);
    DeathDecision evaluate(DeathContext context);
    DeathOutcome process(DeathContext context);
}
```

Quy tắc quyết định mặc định (`DefaultDeathService`) được trình bày chi tiết ở [mục 21](#21-quy-tắc-quyết-định-mặc-định).

### Enum: `DeathOutcome`

| Giá trị | Ý nghĩa |
|---|---|
| `INVENTORY_KEPT` | Giữ nguyên toàn bộ túi đồ. |
| `INVENTORY_DROPPED` | Làm rơi túi đồ. |
| `INVENTORY_PARTIALLY_KEPT` | Giữ một phần. |
| `NO_ACTION` | *(định nghĩa sẵn, `DefaultDeathService.process()` hiện không bao giờ trả về giá trị này.)* |
| `CANCELLED` | *(định nghĩa sẵn, hiện không được trả về bởi triển khai mặc định.)* |

### Enum: `DeathReason`

`PLAYER`, `PLAYER_PROJECTILE`, `PLAYER_EXPLOSION`, `ENTITY`, `ENVIRONMENT`, `VOID`, `FIRE`, `LAVA`, `UNKNOWN`.

---

## 11. Package `core.inventory`

### `InventoryPolicy` (interface — khác với enum cùng tên ở `core.api`)

```java
public interface InventoryPolicy {
    InventoryAction getAction();
    boolean keepMainInventory();
    boolean keepArmor();
    boolean keepOffhand();
    boolean keepHotbar();
    boolean keepExperience();
    boolean keepLevels();
}
```

### `InventoryDecision`

```java
public interface InventoryDecision {
    InventoryAction getAction();
    InventoryPolicy getPolicy();
    boolean keepMainInventory();
    boolean keepArmor();
    boolean keepOffhand();
    boolean keepExperience();
}
```

### `InventorySnapshot`

```java
public interface InventorySnapshot {
    UUID getPlayerId();
    int getMainInventorySize();
    int getArmorSize();
    boolean hasOffhand();
    boolean hasExperience();
    long getExperience();
    int getExperienceLevel();
}
```

### Enum: `InventoryAction`

| Giá trị | Ý nghĩa |
|---|---|
| `KEEP` | Giữ. |
| `DROP` | Rơi. |
| `PARTIAL` | *(định nghĩa sẵn, hiện không được `DefaultDeathService` dựng ra.)* |
| `VANILLA` | *(định nghĩa sẵn, hiện không được `DefaultDeathService` dựng ra.)* |

---

## 12. Package `core.platform`

### `PlatformInfo`

```java
public interface PlatformInfo {
    PlatformType getType();
    String getImplementationName();
    String getImplementationVersion();
    String getMinecraftVersion();
    String getApiVersion();
    boolean isProxy();
    boolean isBackend();
    boolean supports(PlatformCapability capability);
}
```

### `PlatformDetector`

```java
public interface PlatformDetector {
    PlatformInfo detect();
}
```

> Không có lớp nào implement interface này — xem [mục 6, điểm 7](#những-điều-cần-biết-trước-khi-tích-hợp).

### Enum: `PlatformCapability`

`COMBAT`, `DEATH`, `INVENTORY`, `DAMAGE_ATTRIBUTION`, `PROXY_SESSION`, `SERVER_SWITCH`, `CLUSTER_EXIT`, `PLUGIN_MESSAGING`, `EVENTS`.

### Enum: `PlatformType`

`BUKKIT`, `SPIGOT`, `PAPER`, `PURPUR`, `VELOCITY`, `UNKNOWN`.

---

## 13. Package `core.event` (chưa được kích hoạt)

> **Xem cảnh báo ở [mục 6, điểm 3](#những-điều-cần-biết-trước-khi-tích-hợp) trước khi dùng phần này** — các lớp dưới đây không được plugin khởi tạo hay phát ra ở bất kỳ đâu trong mã nguồn hiện tại.

| Lớp | Trường dữ liệu |
|---|---|
| `CombatStartEvent` | `CombatSession getSession()` |
| `CombatRefreshEvent` | `CombatSession getSession()` |
| `CombatEndEvent` | `CombatSession getSession()`, `CombatReason getReason()` |
| `CombatDeathEvent` | `CombatPlayer getPlayer()`, `DeathContext getContext()` *(enum `core.api.DeathContext`)*, `DeathResult getResult()` |
| `DeathEvaluateEvent` | `DeathContext getContext()` *(interface `core.death.DeathContext`)*, `DeathDecision getDecision()` (có `setDecision(...)` — mẫu thiết kế cho phép plugin khác ghi đè quyết định, nếu sự kiện này từng được phát ra) |
| `InventoryPolicyEvent` | `InventoryPolicy getPolicy()` *(interface `core.inventory.InventoryPolicy`)* |

Tất cả đều là POJO `final class` thông thường (không kế thừa `org.bukkit.event.Event`), validate tham số khác `null` qua `Objects.requireNonNull`.

---

## 14. Package `core.hook`

### `CombatHook`

```java
public interface CombatHook {
    default void onCombatStart(CombatPlayer attacker, CombatPlayer victim, CombatTag attackerTag, CombatTag victimTag) {}
    default void onCombatRefresh(CombatPlayer attacker, CombatPlayer victim, CombatTag attackerTag, CombatTag victimTag) {}
    default void onCombatEnd(CombatPlayer player) {}
    default void onDeath(CombatPlayer player, DeathContext context) {}  // core.api.DeathContext (enum)
}
```

Tất cả phương thức có triển khai mặc định rỗng (`default {}`). Không có cơ chế đăng ký (register) `CombatHook` nào được cung cấp bởi plugin — xem cảnh báo [mục 6, điểm 3](#những-điều-cần-biết-trước-khi-tích-hợp).

---

## 15. Package `core.exception`

```java
public class CombatKeepInventoryException extends RuntimeException {
    public CombatKeepInventoryException(String message);
    public CombatKeepInventoryException(String message, Throwable cause);
}

public class InvalidCombatSessionException extends CombatKeepInventoryException {
    public InvalidCombatSessionException(String message);
}
```

Chưa được `throw` ở đâu trong mã nguồn hiện tại (xem [mục 6, điểm 5](#những-điều-cần-biết-trước-khi-tích-hợp)).

---

## 16. Package `core.internal` (triển khai mặc định)

Các lớp `Default*` là **chi tiết triển khai nội bộ** — không nên khởi tạo trực tiếp từ plugin bên thứ ba; hãy lấy instance qua các API ở mục 7. Liệt kê để tham khảo đầy đủ:

| Lớp | Triển khai interface | Ghi chú hành vi đáng chú ý |
|---|---|---|
| `DefaultCombatKeepInventoryAPI` | `CombatKeepInventoryAPI` | Gộp 5 API con qua constructor. |
| `DefaultCombatAPI` | `CombatAPI` | Bọc quanh một `CombatSessionManager`. |
| `DefaultCombatSessionManager` | `CombatSessionManager` | Xem [mục 8](#8-package-corecombat). |
| `DefaultCombatSession` | `CombatSession` | Đối tượng mutable nội bộ, cho phép `refresh(...)`/`end(...)`. |
| `DefaultDamageAPI` | `DamageAPI` | Bọc quanh `DamageAttributionService`. |
| `DefaultDamageAttributionService` | `DamageAttributionService` | Xem [mục 9](#9-package-coredamage). |
| `DefaultDamageAttributionResult` | `DamageAttributionResult` | POJO dữ liệu thuần. |
| `DefaultDeathAPI` | `DeathAPI` | Bọc quanh `DeathService`. |
| `DefaultDeathService` | `DeathService` | Chứa **quy tắc quyết định mặc định** — xem [mục 21](#21-quy-tắc-quyết-định-mặc-định). |
| `DefaultDeathContext` | `core.death.DeathContext` | POJO dữ liệu thuần. |
| `DefaultDeathDecision` | `DeathDecision` | POJO dữ liệu thuần. |
| `DefaultInventoryAPI` | `InventoryAPI` | Bọc quanh một `InventoryPolicy` mặc định. |
| `DefaultInventoryPolicy` | `core.inventory.InventoryPolicy` | POJO 7 trường boolean/enum, không có logic. |
| `DefaultInventoryDecision` | `InventoryDecision` | POJO dữ liệu thuần. |
| `DefaultPlatformAPI` | `PlatformAPI` | Bọc quanh một `PlatformInfo`. |
| `DefaultPlatformInfo` | `PlatformInfo` | POJO dữ liệu thuần. |

### `CombatKeepInventoryCore` (factory, ở gốc package `core`)

```java
public final class CombatKeepInventoryCore {
    public static CombatKeepInventoryAPI create(long combatDurationMillis, PlatformInfo platform);
    public static PlatformInfo createDefaultPlatform();
}
```

Đây là cách "lắp ráp" một `CombatKeepInventoryAPI` hoàn chỉnh từ các mảnh `core.internal` — dùng nội bộ khi cần một instance API không phụ thuộc Bukkit/Velocity (ví dụ để test). `cki-bukkit` hiện **không dùng** factory này (nó tự lắp `DefaultCombatAPI`/`DefaultDamageAPI`/... thủ công trong `CombatKeepInventory#initializeCoreAPIFacades()`), nhưng factory vẫn hoạt động độc lập nếu bạn muốn dùng phần lõi `core` mà không cần Bukkit/Velocity.

---

## 17. Module Bukkit (`cki-bukkit`)

Package gốc: `com.votri.combatkeepinv.bukkit`.

### `BukkitCombatKeepInventoryAPI` — *điểm vào chính*

*(package `bukkit.api`)*

```java
public final class BukkitCombatKeepInventoryAPI implements CombatKeepInventoryAPI {
    public BukkitCombatKeepInventoryAPI(CombatKeepInventory plugin);

    public static Optional<CombatKeepInventoryAPI> getRegistered();
    public static CombatKeepInventoryAPI requireRegistered();  // ném IllegalStateException nếu chưa đăng ký

    public static synchronized void register(BukkitCombatKeepInventoryAPI api);    // dành cho CKI tự gọi khi onEnable
    public static synchronized void unregister(BukkitCombatKeepInventoryAPI api);  // dành cho CKI tự gọi khi onDisable

    // implements CombatKeepInventoryAPI — ủy quyền (delegate) tới plugin.getXxxAPI()
    public CombatKeepInventory getPlugin();
}
```

### `CombatKeepInventory` (lớp plugin chính, `extends JavaPlugin`)

Các accessor công khai hữu ích cho tích hợp/reflection (đầy đủ vòng đời `onEnable`/`onDisable`/`reloadPlugin()` không liệt kê ở đây vì không phải API dành cho bên ngoài):

| Phương thức | Trả về |
|---|---|
| `getCombatAPI()` / `getDamageAPI()` / `getDeathAPI()` / `getInventoryAPI()` / `getPlatformAPI()` | 5 API con (giống hệt những gì `BukkitCombatKeepInventoryAPI` expose) |
| `getCombatService()` | `CombatService` (ném `IllegalStateException` nếu chưa init) |
| `getCombatManager()` | `CombatManager` |
| `getCombatStateBridge()` | `CombatStateBridge` (ném `IllegalStateException` nếu chưa init) |
| `getWorldGuardHook()` | `WorldGuardHook` |
| `getPvPManagerDetector()` | `PvPManagerDetector` |
| `getPlatform()` | `PlatformInfo` |
| `getMessages()` | `FileConfiguration` (nội dung `message.yml`) |
| `isCombatEnabled()` / `isDebugCombat()` / `isDebugDamage()` / `isDebugDeath()` / `isDebugBridge()` | `boolean`, đọc từ `config.yml` |
| `isWorldDisabled(World world)` | `boolean` |
| `getCombatDurationSeconds()` | `long`, tối thiểu 1 |
| `getListenerPriority()` | `org.bukkit.event.EventPriority` (mặc định `HIGHEST`) |
| `shouldKeepDeathExperience()` / `shouldKeepCombatDeathExperience()` | `boolean` |
| `reloadPlugin()` | nạp lại config, message, rebuild toàn bộ facade API |
| `getMessage(String path, String fallback)`, `getMessageList(String path)`, `color(String text)` | tiện ích đọc `message.yml` + dịch mã màu `&` |

`PLUGIN_VERSION` là hằng số `public static final String = "1.2.0"`.

### `BukkitCombatService implements CombatService`

Triển khai cụ thể của `CombatService` (mục 7) dành riêng cho Bukkit. Điểm đáng chú ý:

- `evaluateDeath(UUID player, DeathContext context)` **ưu tiên tuyệt đối** cho CombatTag đang hoạt động: nếu `combatManager.isInCombat(player)` → luôn trả `DeathResult(DROP, keepExperience-theo-config, wasCombatDeath=true)`, **bỏ qua** tham số `context` truyền vào.
- Nếu không trong combat, nó chuyển `context` (enum legacy) thành `DamageSource` qua `BukkitDamageSource.fromLegacyContext(...)`, rồi chạy qua `DeathService` core như bình thường.
- `getDefaultInventoryPolicy()` dựng `DefaultInventoryPolicy` từ các khoá `config.yml`: `inventory.keep-main`, `inventory.keep-armor`, `inventory.keep-offhand`, `inventory.keep-hotbar`, `inventory.keep-levels` (mặc định bằng `death.keep-experience` nếu không đặt riêng), và `death.keep-experience`.

### `CombatManager`

Lớp trung gian tương thích Bukkit, "sở hữu" một `CombatSessionManager` (core) và tự động publish sang `CombatStateBridge` mỗi khi trạng thái thay đổi.

```java
public final class CombatManager {
    public CombatManager(long durationMillis, CombatStateBridge stateBridge);
    public synchronized void setDurationMillis(long durationMillis); // dựng lại toàn bộ sessionManager, KHÔNG copy session cũ
    public long getDurationMillis();
    public CombatSessionManager getSessionManager();
    public void start(UUID attacker, UUID victim);
    public void refresh(UUID attacker, UUID victim);
    public void tag(UUID attacker, UUID victim);         // tự chọn start hoặc refresh
    public boolean remove(UUID player);                  // CombatReason.PLAYER_DEATH
    public boolean forceRemove(UUID player);              // CombatReason.ADMIN
    public boolean isInCombat(UUID player);
    public CombatTag getCombatTag(UUID player);           // null nếu không active
    public long getRemainingMillis(UUID player);
    public long getRemainingSeconds(UUID player);
    public void cleanupExpired();
    public int size();
    public void clear();                                   // publish FORCE_END cho tất cả, dùng khi onDisable
}
```

> ⚠️ `setDurationMillis(...)` (gọi khi reload config) **tạo `CombatSessionManager` hoàn toàn mới** — mọi CombatTag đang hoạt động bị mất khi admin chạy `/cki reload` với thời lượng combat thay đổi (đây là chủ đích, có ghi chú trong code: "Combat state should not survive a configuration reload").

### `CombatStateBridge`

Gửi trạng thái CombatTag từ Bukkit sang Velocity qua Plugin Messaging (kênh cố định `votri:combat`, xem [mục 19](#19-giao-thức-mạng-liên-server--kênh-votricombat)).

```java
public final class CombatStateBridge {
    public static final String CHANNEL = "votri:combat";
    public static final int PROTOCOL_VERSION = 1;
    public static final byte START = 1, REFRESH = 2, END = 3, FORCE_END = 4;

    public CombatStateBridge(CombatKeepInventory plugin);
    public boolean publishStart(UUID attacker, UUID victim, long expiresAt);
    public boolean publishRefresh(UUID attacker, UUID victim, long expiresAt);
    public boolean publishEnd(UUID player);
    public boolean publishForceEnd(UUID player);
    public void shutdown();
}
```

Gói tin được gửi thông qua một người chơi đang online làm "kênh vận chuyển" (Bukkit Plugin Messaging yêu cầu vậy) — ưu tiên `attacker`, nếu offline thì thử `victim`; nếu cả hai đều offline, gói tin **không được gửi** (trả về `false`, chỉ log nếu bật `debug.bridge`).

### `BukkitDamageSource implements DamageSource`

Factory tĩnh để tạo `DamageSource` từ dữ liệu Bukkit:

```java
public static BukkitDamageSource environment(UUID victimId, EntityDamageEvent event);
public static BukkitDamageSource fromLegacyContext(UUID victimId, com.votri.combatkeepinv.core.api.DeathContext context); // enum legacy
public static BukkitDamageSource playerAttack(UUID attackerId, UUID victimId);
public static BukkitDamageSource playerProjectile(UUID attackerId, UUID projectileId, UUID victimId);
```

`environment(...)` cũng là fallback khi `EntityDamageByEntityEvent` có damager không phải người chơi và không phải đạn do người chơi bắn (ví dụ quái vật cận chiến) — trong trường hợp đó, `DamageCause` gốc của Bukkit (ví dụ `ENTITY_ATTACK`) được ánh xạ qua cùng bảng dùng cho sát thương môi trường thuần túy.

### `WorldGuardHook`

Tích hợp WorldGuard hoàn toàn qua **reflection** (không hard-dependency biên dịch bắt buộc lúc runtime nếu WorldGuard không có mặt).

```java
public final class WorldGuardHook {
    public WorldGuardHook(CombatKeepInventory plugin);
    public boolean isAvailable();
    public boolean canPvP(Player attacker, Player victim);
}
```

Đọc các khoá cấu hình: `worldguard.enabled`, `worldguard.fail-open`, `worldguard.require-both-players-in-region`, `worldguard.restrict-to-enabled-regions`, `worldguard.enabled-regions`, `worldguard.excluded-regions`, `debug.worldguard`. **Lưu ý:** các khoá này hiện **không xuất hiện** trong `config.yml` mặc định đi kèm plugin — chúng chỉ tồn tại dưới dạng giá trị fallback trong code (ví dụ `worldguard.enabled` fallback `true`, `worldguard.fail-open` fallback `true`); muốn tuỳ chỉnh, bạn phải tự thêm các khoá này vào `config.yml`.

### `PvPManagerDetector`

Chỉ để log cảnh báo nếu phát hiện plugin "PvPManager" khác cùng cài — **không hook** vào PvPManager theo bất kỳ cách nào, chỉ cảnh báo admin về khả năng xung đột.

### `BukkitPlatformDetector`

```java
public static PlatformInfo detect(); // đọc Bukkit.getServer(), phân loại PURPUR > PAPER > SPIGOT > BUKKIT theo tên/version string
```

### `CombatListener implements Listener`

Lắng nghe `EntityDamageByEntityEvent` (đăng ký thủ công qua `PluginManager.registerEvent(...)`, không dùng `@EventHandler` tự động-quét, với `EventPriority` cấu hình được — mặc định `HIGHEST`) và `PlayerDeathEvent`. Đây là nơi duy nhất trong plugin chuyển đổi sự kiện Bukkit thành lệnh gọi vào `CombatService`/`CombatManager`. Quyền bypass: `combatkeepinventory.bypass`.

### `CombatCommand implements CommandExecutor, TabCompleter`

Xử lý lệnh `/cki` — xem [mục 20](#20-lệnh--quyền).

---

## 18. Module Velocity (`cki-velocity`)

Package gốc: `com.votri.combatkeepinv.velocity`.

### `VelocityCombatAPI` — *điểm vào chính*

*(package `velocity.api`)*

```java
public interface VelocityCombatAPI {
    static VelocityCombatAPI get();  // == Provider.get(), ném IllegalStateException nếu chưa init

    boolean isInCombat(UUID player);
    ProxyCombatState getCombatState(UUID player);   // null nếu không có
    CombatTag getCombatTag(UUID player);            // == getCombatState(player), kiểu trả về core.api.CombatTag
    long getRemainingCombatMillis(UUID player);
    default long getRemainingCombatSeconds(UUID player);
    UUID getOpponent(UUID player);
    String getBackendServer(UUID player);
    CombatPunishmentService punishments();
    boolean isEnabled();

    final class Provider {
        static VelocityCombatAPI get();                          // ném IllegalStateException nếu chưa đăng ký
        static void register(VelocityCombatAPI api);              // dành cho CKI tự gọi khi ProxyInitializeEvent
        static void unregister(VelocityCombatAPI api);            // dành cho CKI tự gọi khi ProxyShutdownEvent
    }
}
```

### `VelocityCombatAPIImpl implements VelocityCombatAPI`

Triển khai cụ thể — mọi phương thức đọc dữ liệu đều trả về giá trị "rỗng" (`false`/`null`/`0`) nếu `enabled == false` (tức `config.isApiEnabled() && config.exposeApi()` tại thời điểm khởi tạo, có thể đổi runtime qua `setEnabled(boolean)`).

### `ProxyCombatState implements CombatTag`

Bản sao bất biến (immutable) phía Velocity của một CombatTag phía Bukkit — **không bao giờ tự tạo trạng thái, chỉ phản chiếu** dữ liệu nhận từ `CombatStateBridge`.

```java
public final class ProxyCombatState implements CombatTag {
    public ProxyCombatState(UUID playerId, UUID opponentId, String backendServer, long expiresAt); // legacy ctor, updatedAt = now
    public ProxyCombatState(UUID playerId, UUID opponentId, String backendServer, long expiresAt, long updatedAt);
    public String getBackendServer();
    public long getUpdatedAt();
    public ProxyCombatState withBackendServer(String backendServer); // giữ nguyên expiresAt/updatedAt
    // + toàn bộ phương thức của CombatTag (isActive(), getRemainingMillis()...)
}
```

### `ProxyCombatStateManager`

Lưu trữ `Map<UUID, ProxyCombatState>` (`ConcurrentHashMap`), có **2 tầng hết hạn độc lập**: (1) `expiresAt` của chính CombatTag, và (2) "bridge-state timeout" (`combat.state-timeout-seconds`, mặc định 15s trong `VelocityConfig`) — nếu Bukkit ngừng gửi thông điệp đồng bộ lâu hơn khoảng này, trạng thái bị coi là "stale" và tự xoá dù `expiresAt` chưa tới.

```java
public final class ProxyCombatStateManager {
    public ProxyCombatStateManager(VelocityConfig config);
    public void start(UUID attacker, UUID victim, String backendServer, long expiresAt);
    public void refresh(UUID attacker, UUID victim, String backendServer, long expiresAt); // == start(...)
    public void end(UUID player);
    public void forceEnd(UUID player);
    public boolean isInCombat(UUID player);
    public ProxyCombatState getCombatState(UUID player);
    public long getRemainingMillis(UUID player);
    public UUID getOpponent(UUID player);
    public String getBackendServer(UUID player);
    public void updateBackendServer(UUID player, String backendServer); // không đổi timer CombatTag
    public void cleanupExpired();
    public void clear();
    public int size();
}
```

### `CombatPunishmentService`

Thực thi các hành động trừng phạt thô — **bản thân service này không tự động quyết định KHI NÀO trừng phạt**; đó là trách nhiệm của bên gọi nó (xem cảnh báo [mục 6, điểm 4](#những-điều-cần-biết-trước-khi-tích-hợp)).

```java
public final class CombatPunishmentService {
    public CombatPunishmentService(ProxyServer proxy, VelocityConfig config);
    public boolean disconnect(UUID playerId, Component reason);
    public boolean disconnect(UUID playerId, String reason);
    public Optional<Player> getPlayer(UUID playerId);
    public boolean executeCommands(UUID playerId, List<String> commands, String reason);
    // hỗ trợ placeholder trong `commands`: {player}, {uuid}, {reason}
    // trả về false ngay nếu !config.isPunishmentEnabled()
}
```

### `CombatStateProtocol`

Hằng số giao thức dùng bởi `VelocitySessionListener` (xem [mục 19](#19-giao-thức-mạng-liên-server--kênh-votricombat)):

```java
public final class CombatStateProtocol {
    public static final String CHANNEL = "votri:combat";
    public static final int VERSION = 1;
    public static final byte START = 1, REFRESH = 2, END = 3, FORCE_END = 4;
}
```

### `VelocityConfig`

Bọc quanh `config.yml` phía Velocity (parse bằng SnakeYAML thủ công, không dùng thư viện config annotation). Các nhóm getter chính (tất cả đều có giá trị fallback cứng trong code nếu khoá thiếu trong file):

| Nhóm | Getter tiêu biểu |
|---|---|
| Chung | `isEnabled()`, `isDebug()`, `getConfigVersion()` |
| Cầu nối (bridge) | `isBridgeEnabled()`, `getBridgeChannel()` (mặc định `votri:combat`), `verifyBridgeSourceServer()`, `getAllowedBridgeServers()` |
| Combat | `getStateTimeoutSeconds()` (≥1, mặc định 15), `clearOnDisconnect()`, `clearOnServerSwitch()` |
| Chuyển server | `isServerSwitchEnabled()`, `trackServer()`, `checkCombatOnServerSwitch()`, `requireActiveCombatOnServerSwitch()` |
| Rời cụm | `isClusterExitEnabled()`, `checkCombatOnClusterExit()`, `requireActiveCombatOnClusterExit()` |
| Trừng phạt | `isPunishmentEnabled()`, `isSwitchPunishmentEnabled()`, `getSwitchPunishmentCommands()`, `isClusterExitPunishmentEnabled()`, `getClusterExitPunishmentCommands()` *(4 getter cuối hiện không được đọc bởi bất kỳ đâu khác ngoài chính lớp này — xem mục 6, điểm 4)* |
| Điều kiện | `requireCombatTag()`, `getBypassPermission()` (mặc định `combatkeepinventory.bypass`), `requireUnexpiredTag()` |
| API | `isApiEnabled()`, `exposeApi()` |
| Log | `logStart()`, `logRefresh()`, `logEnd()`, `logForceEnd()`, `logServerSwitch()`, `logClusterExit()`, `logPunishment()`, `logInvalidMessage()` |
| Truy cập chung | `get(String path)`, `getString/Boolean/Long/Int(String path, fallback)`, `getStringList(String path)` — hỗ trợ path dạng `a.b.c` |

### `VelocitySessionListener`

Lắng nghe 3 sự kiện Velocity: `ServerConnectedEvent`, `DisconnectEvent`, `PluginMessageEvent` (giải mã giao thức `votri:combat`). Có xác thực nguồn gói tin: chỉ chấp nhận nếu `event.getSource()` là `ServerConnection` (tức backend server thật, không phải client) — nếu bật `bridge.verify-source-server`, chỉ chấp nhận từ danh sách `bridge.allowed-servers`.

### `PlayerSession`, `PlayerSessionManager`, `SessionTransition`

Theo dõi phiên người chơi trong phạm vi cụm Velocity (không liên quan CombatTag trực tiếp — dùng để phát hiện chuyển server/rời cụm).

```java
public final class SessionTransition {
    public enum Type { CONNECT, SERVER_SWITCH, CLUSTER_EXIT }
    // factory: connect(playerId, server) / serverSwitch(playerId, from, to) / clusterExit(playerId, from)
    public UUID getPlayerId();
    public Type getType();
    public String getFromServer();
    public String getToServer();
    public long getTimestamp();
    public boolean isServerSwitch();
    public boolean isClusterExit();
}
```

`PlayerSessionManager.serverSwitch(UUID, String)` và `.clusterExit(UUID)` là hai điểm tạo ra `SessionTransition`, được `CombatKeepInventoryVelocity` tiêu thụ trong `handleTransition(...)` để cập nhật `backendServer` trên `ProxyCombatStateManager` và (tuỳ chọn `clearOnServerSwitch`/`clearOnDisconnect`) buộc kết thúc CombatTag phản chiếu.

### `VelocityPlatformDetector`

```java
public static PlatformInfo detect(ProxyServer proxy); // type=VELOCITY, isProxy()=true, isBackend()=false, supports(...) luôn false
```

### `CombatKeepInventoryVelocity` (lớp plugin chính, annotation `@Plugin(id="combatkeepinventory", version="1.2.0")`)

| Phương thức | Trả về |
|---|---|
| `getProxy()` / `getLogger()` / `getDataDirectory()` | Dependency Velocity chuẩn (inject qua `@Inject`) |
| `getConfig()` | `VelocityConfig` |
| `getPlatform()` | `PlatformInfo` (ném `IllegalStateException` nếu chưa init) |
| `getSessionManager()` | `PlayerSessionManager` |
| `getCombatStateManager()` | `ProxyCombatStateManager` |
| `getPunishmentService()` | `CombatPunishmentService` |
| `getChannelIdentifier()` | `MinecraftChannelIdentifier` (ném `IllegalStateException` nếu chưa init) |
| `getCombatAPI()` | `VelocityCombatAPI` (ném `IllegalStateException` nếu chưa init) |

Hằng số: `CHANNEL = "votri:combat"`, `PROTOCOL_VERSION = 1`, `CHANNEL_IDENTIFIER` (mặc định, kênh thực tế lúc chạy lấy từ `config.getBridgeChannel()`).

---

## 19. Giao thức mạng liên server — kênh `votri:combat`

Plugin Messaging channel cố định `votri:combat` (có thể đổi qua `bridge.channel` phía Velocity, nhưng phía Bukkit hiện **hard-code**, không đọc từ config). Được định nghĩa trùng lặp ở 3 nơi với cùng giá trị: `bukkit.bridge.CombatStateBridge`, `velocity.bridge.CombatStateProtocol`, và hằng số riêng bên trong `velocity.listener.VelocitySessionListener`.

**Header chung mọi gói tin** (ghi bằng `java.io.DataOutputStream`, đọc bằng `DataInputStream`):

| Byte | Kiểu | Ý nghĩa |
|---|---|---|
| 0–3 | `int` | `PROTOCOL_VERSION` — hiện luôn `1`. Gói với version khác bị từ chối, chỉ log nếu `logging.invalid-message`. |
| 4 | `byte` | Mã thao tác — xem bảng dưới. |

**Mã thao tác (`operation`):**

| Giá trị | Tên | Payload theo sau header | Tổng kích thước gói (byte) |
|---|---|---|---|
| `1` | `START` | `attackerUUID` (16) + `victimUUID` (16) + `expiresAt: long` (8) | 45 |
| `2` | `REFRESH` | giống `START` | 45 |
| `3` | `END` | `playerUUID` (16) | 21 |
| `4` | `FORCE_END` | giống `END` | 21 |

Mỗi `UUID` được ghi dưới dạng 2 `long` liên tiếp: `mostSignificantBits` rồi `leastSignificantBits` (16 byte).

Phía gửi (Bukkit `CombatStateBridge`) chọn một người chơi đang online (`attacker`, dự phòng `victim`) làm "người mang" gói tin plugin-message (yêu cầu bắt buộc của Bukkit API). Nếu cả hai đều offline, gói tin **không gửi được** và bị bỏ qua âm thầm (trừ khi bật `debug.bridge`).

Phía nhận (Velocity `VelocitySessionListener.onPluginMessage`) chỉ xử lý gói tin nếu nguồn là `ServerConnection` thật (chặn mọi gói giả mạo từ client) và (tuỳ chọn) nằm trong `bridge.allowed-servers`.

---

## 20. Lệnh & Quyền

### Bukkit (`plugin.yml`)

| Lệnh | Cú pháp | Quyền |
|---|---|---|
| `/cki reload` | `/cki reload` | `combatkeepinventory.admin` |
| `/cki info` | `/cki info` | `combatkeepinventory.admin` |

| Quyền | Mặc định | Ý nghĩa |
|---|---|---|
| `combatkeepinventory.admin` | `op` | Cho phép `/cki reload` và `/cki info`. |
| `combatkeepinventory.bypass` | `false` | Người có quyền này được **bỏ qua hoàn toàn** xử lý combat/death của CKI (không bị tag, luôn giữ đồ khi chết). |

`plugin.yml` khai báo `softdepend: [WorldGuard, PvPManager]` — cả hai đều tùy chọn, không bắt buộc phải cài.

### Velocity

Không có lệnh nào được đăng ký phía Velocity trong mã nguồn hiện tại — chỉ có cấu hình qua `config.yml` (`conditions.bypass-permission`, mặc định cũng là `combatkeepinventory.bypass`, nhưng đây là một khoá cấu hình đọc qua `VelocityConfig`, không phải permission node được Velocity kiểm tra tự động ở bất kỳ đâu trong mã nguồn hiện tại).

---

## 21. Quy tắc quyết định mặc định

Logic nằm trong `DefaultDeathService.evaluate(DeathContext context)` (package `core.internal`), áp dụng khi đi qua đường "core mới" (tức là khi **không** có CombatTag đang hoạt động — trường hợp có CombatTag hoạt động được `BukkitCombatService.evaluateDeath(...)` chặn từ trước, xem mục 17):

```
giữ_túi_đồ = (KHÔNG phải người chơi gây ra cái chết) VÀ (KHÔNG đang trong combat)
```

| Trường hợp | `wasPlayerCaused()` | `wasInCombat()` | Kết quả | `getRuleId()` |
|---|---|---|---|---|
| Chết do người chơi khác (PvP) | `true` | bất kỳ | **DROP** toàn bộ (không giữ giáp/offhand/hotbar) | `"pvp-combat"` |
| Chết trong lúc có CombatTag, do mob/môi trường | `false` | `true` | **DROP** toàn bộ | `"default"` |
| Chết do mob/môi trường, không có CombatTag | `false` | `false` | **KEEP** theo `InventoryPolicy` mặc định đã cấu hình | `"default"` |

Khi kết quả là DROP (nhánh không giữ đồ), plugin dựng một `InventoryPolicy` "trắng" tường minh: `action=DROP`, toàn bộ `keepMainInventory/keepArmor/keepOffhand/keepHotbar/keepExperience/keepLevels = false` — **bỏ qua** mọi tuỳ chỉnh giữ-một-phần trong config ở nhánh này.

`DeathOutcome.process(...)` ánh xạ: giữ toàn bộ (`action == KEEP`) → `INVENTORY_KEPT`; giữ nhưng `action` khác `KEEP` → `INVENTORY_PARTIALLY_KEPT`; không giữ → `INVENTORY_DROPPED`.

`DeathReason` được suy ra từ `DamageAttributionResult` + `DamageCauseType`: PvP trực tiếp → `PLAYER`; PvP gián tiếp (đạn) → `PLAYER_PROJECTILE`; PvP gián tiếp (TNT/nổ) → `PLAYER_EXPLOSION`; còn lại map trực tiếp theo `DamageCauseType` (`FIRE`/`FIRE_TICK`→`FIRE`, `LAVA`→`LAVA`, `VOID`→`VOID`, `ENTITY`→`ENTITY`, mặc định →`ENVIRONMENT`).

---

## 22. Giấy phép

Apache License 2.0 — xem file `LICENSE` trong repository.

---

*Tài liệu này được biên soạn từ việc đọc trực tiếp toàn bộ mã nguồn trong `CombatKeepInventory-main.zip` (3 module, ~86 file Java). Nếu mã nguồn thay đổi (đặc biệt là các điểm được đánh dấu "chưa được kích hoạt"/"chưa nối dây" ở mục 6), hãy cập nhật lại tài liệu tương ứng.*
