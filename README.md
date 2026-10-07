# dz-ue-mcp

DSH agent skill：通过 UE 5.8 官方 MCP（Unreal MCP 插件）驱动虚幻编辑器——查询资产、检查 GAS 状态、运行 PIE 验证、执行编辑器写操作。内含 DZProject 的使用规范与安全边界。

## 这是什么

- 一个 **DSH agent skill**（`SKILL.md`），教会 AI 正确使用 UE 官方编辑器 MCP
- 安装后，AI 会按固定纪律工作：自检连接 → 发现工具集 → 调用工具，并在 **plan 批准前保持只读**

## 前提条件

### UE 编辑器端（一次性）

1. **Edit → Plugins**：启用 **Unreal MCP** 与 **All Toolsets**（GAS 项目强烈推荐再启用 **GASToolsets**）
2. **Editor Preferences → General → Model Context Protocol**：勾选 **Auto Start Server**
   （或控制台执行 `ModelContextProtocol.StartServer`），默认绑定 `http://127.0.0.1:8000/mcp`
3. 注意：UE 官方 MCP 为**实验性**功能，工具集与 API 可能随版本变化

### DSH 端（一次性）

在 `<dsh-home>/profiles/web/cordis.patch.yml` 追加：

```yaml
- id: mcp-unreal
  name: "@deepseek-ai/dsh-mcp-client"
  config:
    serverName: unreal
    transport: streamable-http
    url: http://127.0.0.1:8000/mcp
```

- DSH 支持 patch 热加载，改完即生效，无需重启
- 编辑器未运行时 harness 照常启动（`failOnStartupError` 默认 false），工具调用会失败直至服务器可达

## 安装技能

```bash
git clone https://github.com/duskerCJ/dz-ue-mcp.git ~/.agents/skills/dz-ue-mcp
```

Windows 下即 `C:\Users\<你>\.agents\skills\dz-ue-mcp`；也可用支持 `github:` 引用或技能安装机制的 harness 直接安装（与 [dsh-gas-plan](https://github.com/duskerCJ/dsh-gas-plan) 同组织生态）。

## 使用要点（详见 [SKILL.md](SKILL.md)）

- 工具搜索模式只暴露 **3 个元工具**：`list_toolsets` → `describe_toolset` → `call_tool`
- 工具调用在**游戏线程串行执行**，不要并发重叠调用；单次超时 60s
- **Plan 批准前只读**；写操作执行前先向用户复述工具集、工具名与参数
- UE MCP 无鉴权、仅回环：不得把 8000 端口暴露到局域网/公网

### 已验证工具集速查（2026-10-07）

| 工具集 | 用途 |
|---|---|
| `EditorToolset.EditorAppToolset` | 控制台变量、选择、视口相机、Content Browser、**PIE 会话控制** |
| `EditorToolset.LogsToolset` | 读输出日志、调日志级别 |
| `GASToolsets.AttributeSetToolset` | AttributeSet 类与属性发现 |
| `GASToolsets.AbilitySystemInspectorToolset` | 运行时 ASC 状态检查 |
| `GASToolsets.GameplayCueToolset` | GameplayCue 及 notify 资产检查与执行 |
| `GameplayTagsToolset.GameplayTagsToolset` | 项目 GameplayTag 读取/管理 |
| `AutomationTestToolset.AutomationTestToolset` | 自动化测试发现 → 运行 → 取结果 |
| `ConfigSettingsToolset` / `GameFeaturesToolset` 等 | Config 检查、Game Feature 管理 |

常用配方：**PIE 验证** = `EditorAppToolset` + `LogsToolset`；**GAS 运行时验证** = `AbilitySystemInspectorToolset`；**Tag 核查** = `GameplayTagsToolset`；**回归** = `AutomationTestToolset`。

## 故障排查

1. 调用失败先确认：编辑器是否在跑？8000 端口是否监听？
2. 编辑器控制台 `LogModelContextProtocol Verbose` 看协议日志
3. DSH 侧重载配置或重启 harness（dsh-mcp-client 连续失败 10 次会停止重连）
4. 协议级调试：`npx @modelcontextprotocol/inspector`，Streamable HTTP 指向 `http://127.0.0.1:8000/mcp`
