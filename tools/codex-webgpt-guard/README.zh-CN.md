# Codex WebGPT Guard

这是一个运行在 Windows 用户态的 **Codex 路由所有权协调器 / 配置修复守护进程**。

它最初用于解决 Cockpit Tools 与 `codex-chatgpt-web` 同时使用时的冲突，但真正解决的问题更深：**多个工具同时修改同一份 Codex 配置，却没有统一的字段所有权和事务边界。**

## 它解决的不是一个按钮问题，而是一类系统问题

Codex 生态里可能同时存在：

- 账号 / OAuth 切换工具；
- 本地 Responses API 路由；
- 自定义 provider；
- 模型目录注入器；
- 桌面启动器 / wrapper；
- 原生 Codex。

这些工具最终都可能修改：

```text
~/.codex/config.toml
```

于是会出现典型的“最后写入者获胜”问题：

- 账号切换成功了，但把本地 route 删掉了；
- route 服务仍然健康，但 Codex 已经退回 native；
- model catalog 被另一个工具覆盖，模型选择器突然变化；
- journal 认为 route 仍然 active，但实际配置已经被别人改掉；
- config 最终修好了，但 Codex 已经在几毫秒前按错误配置启动并缓存了旧状态。

Guard 的核心不是“强制把配置改成我的”，而是：

> **能证明归属的字段才修复；归属不明的字段拒绝覆盖。**

## 当前实现做什么

- 从 `codex-chatgpt-web` 自己的 integration journal 获取 route 所有权，不猜端口；
- Web / Native 切换优先调用官方 `route connect` / `route disconnect` 生命周期；
- 账号切换工具误删 Web route 时自动恢复；
- 只移除明确识别为 Cockpit 管理、且会干扰 Web GPT 的 model catalog；
- 遇到未知 `model_provider`、未知 catalog、其他自定义 `openai_base_url` 时不抢配置；
- 处理 Codex 启动竞态：只允许重载与本次配置写入时间对应的“刚启动”同一应用族进程，不会批量结束旧会话；
- 原子写配置、单实例运行、当前用户开机自启，不需要管理员权限。

## 当前边界

它的架构可以扩展成通用的 Codex Route Guard，但当前版本有一个成熟的 route-owner adapter：`codex-chatgpt-web`。

因此它并不是“看到任何 Router 都强行接管”的万能工具。相反，它把**不认识的 Router 当作别人的所有权**，选择不动。

这也是为什么它适合长期运行：不会为了修一个兼容问题，再制造新的配置争夺。

详细设计见：

- [ARCHITECTURE.md](ARCHITECTURE.md)
- [SECURITY.md](SECURITY.md)
- [COMPATIBILITY.md](COMPATIBILITY.md)
