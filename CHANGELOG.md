# Changelog

本项目的更改记录在此文件。

All notable changes to this project are documented in this file.

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)；
版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] - 2026-09-12

> **注意**：本节 1.2.0 的发布资产已于同日**重新构建并覆盖**：初版存在下面最后一条「基岩版白名单」的缺陷——`fwhitelist` 名字解析失败时只在控制台留一行错、不抛异常也不改白名单，而绑定被当成成功上报，导致基岩玩家永远进不来。版本号不升，资产以最新构建为准。

### 新增
- 一个外部账号可**同时**持有 Java 版与基岩版两条绑定：两条白名单条目并存、互不覆盖。此前第二次绑定会撤掉前一条，导致同一个人的电脑版与手机版只能进一个。
- 绑定 API 增加 `kind`（`java` / `geyser`）维度：`lookup` 返回 `bindings[]` 完整视图与 `javaBound`/`geyserBound`；`unbind` 支持按 `kind` 定向解除，省略即清空该账号全部绑定。
- 老 `bindings.json` 自动兼容：缺少 `kind` 的记录按原 `floodgate` 标记推断归类，升级不丢绑定。

### 变更
- 改绑的回收范围收窄为**同类**：改 Java 版名字不再影响基岩版那条（反之亦然）。
- `doc/protocol.md` 第 5 节补 `kind`、`bindings[]`、`removed[]` 说明，并注明「只绑基岩版时 `bound` 为 false 但请求成功」。

### 修复
- **修复离线模式下白名单写入错误 UUID、玩家永远进不来的问题**：此前绑定走 `whitelist add <游戏ID>`，服务端在 usercache 未命中该名字时会向 Mojang 名字 API 查询，把查到的**正版 UUID** 写进白名单；而离线客户端登录用的是本地推导的离线 UUID（`OfflinePlayer:<名字>` 的 v3 UUID），白名单按 UUID 匹配，两者不符即被拒绝（`You are not white-listed on this server!`）。名字越新、越没进过服的玩家越容易踩中。
  现在 `online-mode=false` 时改为直接读写服务器工作目录的 `whitelist.json`（顶层数组，每项只有 `uuid` 与 `name`）并执行一次 `/whitelist reload` 同步内存名单；同名但 UUID 不符的旧条目会被就地改写成正确的离线 UUID，**已中招的玩家重新 `/mc bind` 一次即可修好**。`online-mode=true` 时仍走 `whitelist add`，行为不变。
- 离线模式下解绑与改绑同样按 UUID 直接改白名单文件：`whitelist remove <名字>` 会解析出同一个错误 UUID，删不掉正确条目。
- 基岩版白名单改用 Floodgate 的 `fwhitelist add <名字>`（本版 Floodgate 不接受 UUID 参数，只能用不带 Geyser 前缀的用户名）；未检测到 Floodgate 时退回原指令并在日志中明确警告。
  **该路径改为写后回读校验**：`fwhitelist` 拿名字去查 XUID，公共 API 缓存未命中时只在控制台留一行 `Unable to find user in our cache`、**不抛异常也不改白名单**，而 `executeCommand` 恒返回 `true`（Cloud 指令框架自己吞掉错误），早期版本因此会谎报「已加入白名单」，实际白名单里还是旧的错误条目（例如按名字推导的离线 UUID），基岩玩家依旧被拒绝且重试无效。现在只在 `whitelist.json` 里确实存在该名字、且 UUID 是 Floodgate 生成的（高 64 位为 0 且非零）时才判成功，否则明确判失败并提示「让该玩家先用基岩版登录一次，或在玩家在线时重新绑定」；玩家**在线**时直接用其真实 Floodgate UUID 直写白名单文件，不再依赖那个 API。
- 白名单归属判定收紧：名字已在白名单且 UUID 正确时视为管理员手工添加（`whitelistAdded=false`），绑定照常成功但解绑不会误删该条目。

> **Note**: The release assets of this 1.2.0 entry were **rebuilt and overwritten** on the same day: the initial build carried the defect of the last item below, "Bedrock Edition whitelist" — when `fwhitelist` failed to resolve a name, it left a single error line on the console without throwing an exception or changing the whitelist, yet the binding was reported as successful, so Bedrock players could never get in. The version number is not bumped, and the latest build is authoritative for the assets.

### Added

- A single external account can now hold a Java Edition and a Bedrock Edition binding **at the same time**: both whitelist entries coexist and neither overwrites the other. Previously a second binding revoked the first one, so the same person could only join with either the desktop or the mobile version.
- The binding API gained a `kind` (`java` / `geyser`) dimension: `lookup` returns the full `bindings[]` view together with `javaBound`/`geyserBound`; `unbind` can remove bindings by `kind`, and omitting `kind` clears all bindings of that account.
- Old `bindings.json` files are compatible automatically: a record without `kind` is classified by inference from its original `floodgate` flag, so no binding is lost on upgrade.

### Changed

- The scope revoked on rebinding is narrowed to the **same kind**: changing the Java Edition name no longer affects the Bedrock Edition entry (and vice versa).
- Section 5 of `doc/protocol.md` now documents `kind`, `bindings[]` and `removed[]`, and notes that "when only the Bedrock Edition is bound, `bound` is false while the request still succeeds".

### Fixed

- **Fixed the wrong UUID being written to the whitelist in offline mode, which kept players from ever getting in**: binding previously went through `whitelist add <game ID>`, and when the server found no such name in usercache it queried the Mojang name API and wrote the **premium UUID** it received into the whitelist; an offline client, however, logs in with the locally derived offline UUID (the v3 UUID of `OfflinePlayer:<name>`), and since the whitelist matches by UUID, the mismatch got the player rejected (`You are not white-listed on this server!`). The newer the name and the less the player had ever joined the server, the easier it was to hit.
  With `online-mode=false`, the code now reads and writes `whitelist.json` in the server working directory directly (a top-level array whose entries hold only `uuid` and `name`) and runs `/whitelist reload` once to sync the in-memory list; an old entry with the same name but a different UUID is rewritten in place to the correct offline UUID, so **a player already affected only needs to run `/mc bind` once more to be fixed**. With `online-mode=true` the code still runs `whitelist add` and behaves as before.
- In offline mode, unbinding and rebinding likewise edit the whitelist file directly by UUID: `whitelist remove <name>` resolves the same wrong UUID and cannot delete the correct entry.
- Bedrock Edition whitelisting now uses Floodgate's `fwhitelist add <name>` (this Floodgate version accepts no UUID argument, only a username without the Geyser prefix); when Floodgate is not detected, the code falls back to the original command and logs an explicit warning.
  **That path now verifies success by reading back after writing**: `fwhitelist` looks the name up to find the XUID, and on a public API cache miss it leaves only a single `Unable to find user in our cache` line on the console, **throwing no exception and changing no whitelist entry**, while `executeCommand` always returns `true` (the Cloud command framework swallows the error itself); earlier versions therefore falsely reported "added to the whitelist" while the whitelist still held the old wrong entry (for example an offline UUID derived from the name), and Bedrock players remained rejected with retries being useless. Success is now judged only when the name really exists in `whitelist.json` and its UUID was generated by Floodgate (the high 64 bits are zero and the value is non-zero); otherwise the code reports failure explicitly and tells the user to "have that player log in once with Bedrock Edition first, or bind again while the player is online"; when the player **is** online, the code writes the whitelist file directly with their real Floodgate UUID and no longer depends on that API.
- Whitelist ownership detection is tightened: a name already on the whitelist with the correct UUID is treated as added manually by an administrator (`whitelistAdded=false`), so binding still succeeds but unbinding does not delete that entry by mistake.

---

## [1.1.0] - 2026-09-11

### 新增
- 群友绑定：群友可将 QQ 账号与游戏 ID 自助绑定，绑定后自动写入服务器白名单（Java 版 `/mc bind`，基岩版 `/mc geyserbind`）。
- 基岩版兼容：兼容 Geyser + Floodgate，玩家名带/不带前缀两种写法都可正确绑定；前缀与是否补齐可在 `binding.geyser` 中配置。
- 解绑：`/mc unbind` 只移除由绑定写入的白名单条目，不会误删管理员手工添加的同名条目。
- 绑定 API：新增 `POST /api/v1/bindings`、`POST /api/v1/bindings/unbind`、`GET /api/v1/bindings/lookup`、`GET /api/v1/bindings`，并新增能力位 `binding.v1` 供客户端探测。

### 变更
- **加载器改为 NeoForge 26.2**：本仓库现在只提供 NeoForge 26.2 版本（Minecraft 26.2，需 JDK 25）；Forge 1.20.1 版本保留在 [AstrBotAdapter_Forge](https://github.com/OMSociety/AstrBotAdapter_Forge)。Mod ID 与 Java 包名不变，配置文件键名与格式一致，可从 Forge 版直接迁移。
- 仓库结构：平台无关代码集中在 `common/`，NeoForge 专属层在 `src/`，构建时一起编译，共用代码保持单一副本。
- 老版本配置文件在加载时会自动补齐新增的 `binding` 配置节（保持原有设置不变）。

### 修复
- 修复配置文件缺少新增配置节时新功能静默不可用的问题：现在会按默认模板补齐缺失的顶层配置节并写回。
