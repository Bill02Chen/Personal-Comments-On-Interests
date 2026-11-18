# DeepSeek Harness 添加新模型操作指南

## 1. 目标

本文说明如何让新增模型同时出现在：

1. “设置 → 模型”的提供商配置中。
2. 新会话输入框下方的会话模型选择器中。

会话模型选择器是实际发起对话时使用的模型菜单。模型只出现在设置页，不代表它已经进入该菜单。

本例新增以下 6 个模型：

- `gpt-6.2-astra`：GPT-6.2 Astra
- `gpt-6.2-sol`：GPT-6.2 Sol
- `claude-fable-5-2`：Claude Fable 5.2
- `gemini-4-carbon`：Gemini 4 Carbon
- `gemini-4-argon`：Gemini 4 Argon
- `gemini-4-hydrogen`：Gemini 4 Hydrogen

## 2. 先确定实际生效的配置文件

DeepSeek Harness 使用多层补丁配置。配置档（profile）是应用使用的一套配置环境。

本次涉及两层文件：

- 配置档补丁：`~/.dsh/profiles/desktop/cordis.patch.yml`
- 全局补丁：`~/.dsh/cordis.patch.yml`

加载顺序为：

```text
内置组合包 → 配置档补丁 → 全局补丁 → 命令行覆盖层
```

本次环境中，全局补丁包含 `llm-pi-ai`，并覆盖了配置档补丁中的同名配置。因此，**必须修改 `~/.dsh/cordis.patch.yml`**。

不要只根据“设置页显示了修改内容”判断文件已经生效。会话模型选择器使用运行时模型目录，设置页可以显示另一层原始配置。

### 判断方法

1. 检查配置档补丁是否包含 `llm-pi-ai`。
2. 检查全局补丁是否也包含 `llm-pi-ai`。
3. 如果全局补丁包含该条目，修改全局补丁。
4. 如果启动命令还提供其他覆盖层，最后检查该覆盖层。

本次验证中，会话菜单包含 `GPT-6.1 Sol`。该模型只存在于全局补丁，不存在于已修改的配置档补丁。这证明会话菜单使用了全局补丁。

## 3. 备份实际生效的文件

```bash
cp "$HOME/.dsh/cordis.patch.yml" \
  "$HOME/.dsh/cordis.patch.yml.bak-$(date +%Y%m%d-%H%M%S)"
```

不要省略备份。配置错误会导致提供商加载失败。

## 4. 新增独立模型路由

模型路由是提供商 ID、通信协议、接口地址和模型列表的组合。

不要把未知模型直接追加到内置 `opencode` 路由。内置路由依赖已安装的模型目录。目录不认识的新模型会缺少 `api` 字段，并被会话模型选择器过滤。

保留现有 `opencode` 配置，在同一个 `llm-pi-ai.config.providers` 下新增 `opencode-latest`。

```yaml
- id: llm-pi-ai
  name: "@deepseek-ai/dsh-llm-pi-ai"
  config:
    providers:
      # 保留原有 opencode 配置，不要删除其中的模型。

      opencode-latest:
        displayName: OpenCode Latest
        apiKeyEnv: OPENCODE_API_KEY
        api: openai-responses
        baseURL: https://opencode.ai/inference/openai/v1
        models:
          - id: gpt-6.2-astra
            name: GPT-6.2 Astra
            contextWindow: 1050000
            maxTokens: 128000
            input: [text, image]
            reasoningEfforts:
              low: low
              medium: medium
              high: high
              xhigh: xhigh
              max: max

          - id: gpt-6.2-sol
            name: GPT-6.2 Sol
            contextWindow: 1050000
            maxTokens: 128000
            input: [text, image]
            reasoningEfforts:
              low: low
              medium: medium
              high: high
              xhigh: xhigh
              max: max

          - id: claude-fable-5-2
            name: Claude Fable 5.2
            contextWindow: 1000000
            maxTokens: 128000
            input: [text, image]
            reasoningEfforts:
              low: low
              medium: medium
              high: high
              xhigh: xhigh
              max: max

          - id: gemini-4-carbon
            name: Gemini 4 Carbon
            contextWindow: 1048576
            maxTokens: 65536
            input: [text, image]
            reasoningEfforts:
              low: low
              medium: medium
              high: high

          - id: gemini-4-argon
            name: Gemini 4 Argon
            contextWindow: 1048576
            maxTokens: 65536
            input: [text, image]
            reasoningEfforts:
              low: low
              medium: medium
              high: high

          - id: gemini-4-hydrogen
            name: Gemini 4 Hydrogen
            contextWindow: 1048576
            maxTokens: 65536
            input: [text, image]
            reasoningEfforts:
              low: low
              medium: medium
              high: high
              xhigh: xhigh
```

`opencode-latest` 与 `opencode` 必须处于同一 YAML 层级。

本例通过 OpenCode 接口统一使用 `openai-responses` 协议。Claude 和 Gemini 模型不在此处改用 Anthropic 或 Google 的原生协议。

`id` 是发送给服务端的模型标识。`name` 是界面显示名称。修改显示名称不会修改服务端模型标识。

## 5. 检查凭据引用

配置中的字段是：

```yaml
apiKeyEnv: OPENCODE_API_KEY
```

该字段引用凭据，不保存密钥明文。不要把真实 API 密钥写进 `cordis.patch.yml`。

在“设置 → 模型”中检查该提供商的 API 密钥状态。如果已有 `opencode` 使用同一凭据并且可以列出模型，先确认其凭据仍然有效。

凭据无法解析时，`opencode-latest` 的模型列表会返回空数组。结果是整个 `OpenCode Latest` 分组不出现在会话模型选择器中。

## 6. 不要通过 UI 重复创建提供商

存在全局补丁覆盖时，通过“自定义模型 API”创建提供商会返回：

```text
Configuration for "llm-pi-ai" is overridden by a home patch or command-line overlay
```

此时应修改实际生效的补丁文件。重复点击“创建提供商”不会绕过该限制。

## 7. 重启并验证

完全退出 DeepSeek Harness，再重新打开应用。

打开新会话的模型菜单，分别搜索：

```text
GPT-6.2
Fable 5.2
Gemini 4
```

预期结果：

- 菜单出现 `OpenCode Latest` 分组。
- `GPT-6.2` 返回 Astra 和 Sol。
- `Fable 5.2` 返回 Claude Fable 5.2。
- `Gemini 4` 返回 Carbon、Argon 和 Hydrogen。

本次修正后的配置包含原有 73 个 `opencode` 模型和 6 个新模型，共 79 个模型。路由检查结果为 79 个已配置、79 个可路由，缺失 0 个。

界面出现模型只证明模型目录已加载。最后分别发送一次短请求，检查服务端是否接受模型 ID、协议和凭据。

## 8. 故障排查

| 现象 | 检查项 | 处理 |
|---|---|---|
| 设置页有新模型，会话菜单没有 | 新模型是否直接放在内置 `opencode` 下 | 改为独立路由，并显式设置 `api`、`baseURL` 和 `models` |
| 修改并重启后仍显示旧模型 | 全局补丁是否覆盖配置档补丁 | 修改 `~/.dsh/cordis.patch.yml`，不要只修改配置档补丁 |
| 整个 `OpenCode Latest` 分组缺失 | `OPENCODE_API_KEY` 是否能解析 | 在模型设置页检查凭据状态 |
| UI 创建提供商时报覆盖错误 | 是否存在全局补丁或命令行覆盖层 | 编辑实际生效的配置文件 |
| 模型出现在菜单中，但请求失败 | 模型 ID、协议、接口地址、账户权限 | 用短请求逐项检查服务端错误 |
| 修改配置后应用加载失败 | YAML 缩进和最近备份 | 恢复备份，再重新应用最小修改 |

## 9. 恢复备份

将下列命令中的时间戳替换为实际备份文件名：

```bash
cp "$HOME/.dsh/cordis.patch.yml.bak-YYYYMMDD-HHMMSS" \
  "$HOME/.dsh/cordis.patch.yml"
```

恢复后完全重启 DeepSeek Harness。

## 10. 核心原则

1. 先确定最后一层有效配置，再编辑文件。
2. 设置页可见不等于会话菜单可选。
3. 未知模型需要显式路由，不能只添加模型名称。
4. 密钥使用凭据引用，不写入 YAML。
5. 修改后同时验证设置页、会话菜单和真实短请求。
