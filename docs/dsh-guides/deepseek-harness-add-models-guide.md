# DeepSeek Harness 添加新模型操作指南

## 1. 目标

本文说明如何让新增模型同时出现在：

1. “设置 → 模型”的提供商配置中。
2. 新会话输入框下方的会话模型选择器中。

会话模型选择器是实际发起对话时使用的模型菜单。模型只出现在设置页，不代表它已经进入该菜单。

本文以 `opencode` 网关为例。当前环境有两条路由：

- `opencode`：直接服务应用内置的 pi-ai 模型目录（86 个模型），不写模型列表。
- `opencode-latest`：目录之外的 6 个模型，显式声明协议与地址：
  `gpt-6.2-astra`、`gpt-6.2-sol`、`claude-fable-5-2`、`gemini-4-carbon`、`gemini-4-argon`、`gemini-4-hydrogen`。

## 2. 先确定实际生效的配置文件

DeepSeek Harness 使用多层补丁配置。配置档（profile）是应用使用的一套配置环境。加载顺序为：

```text
内置组合包 → 配置档补丁 → 全局补丁 → 命令行覆盖层
```

涉及两层文件：

- 配置档补丁：`~/.dsh/profiles/desktop/cordis.patch.yml`
- 全局补丁（home patch）：`~/.dsh/cordis.patch.yml`

**当前环境：`llm-pi-ai` 只写在配置档补丁里，全局补丁不包含它。**

这一条是关键，原因有三：

1. “设置 → 模型”页写的就是配置档补丁。
2. 全局补丁若也写了 `llm-pi-ai`，它会覆盖配置档补丁；此时在设置页保存任何提供商都会失败并报：

   ```text
   Configuration for "llm-pi-ai" is overridden by a home patch or command-line overlay
   ```

   “添加模型服务商”按钮也会一起置灰。
3. 全局补丁里的 `llm-pi-ai` 一旦写错，会连同配置档补丁里的模型一起失效，等于整套提供商报废。

### 判断方法

```bash
grep -n "llm-pi-ai" ~/.dsh/cordis.patch.yml ~/.dsh/profiles/desktop/cordis.patch.yml
```

- 只有配置档补丁命中：正确，改配置档补丁（或直接用设置页）。
- 全局补丁也命中：先删掉全局补丁里的 `llm-pi-ai` 整段，再改配置档补丁。

## 3. 备份实际生效的文件

```bash
cp "$HOME/.dsh/profiles/desktop/cordis.patch.yml" "$HOME/.dsh/profiles/desktop/cordis.patch.yml.bak-$(date +%Y%m%d-%H%M%S)"
```

不要省略备份。配置错误会导致提供商加载失败。

## 4. 新增独立模型路由

模型路由是「提供商 ID + 通信协议 + 接口地址 + 模型列表」的组合。

### 4.1 内置路由不要写模型列表

`opencode` 路由依赖已安装的模型目录。配置里一旦写 `models`，就等于用自己列的清单**替换**目录；目录里没有的 `id` 不会自动补上协议：

```yaml
      opencode:
        apiKeyEnv: OPENCODE_API_KEY
```

保持这样即可，目录里的模型会全部出现。**不要**为了加一两个新模型去改写内置路由的 `models`。

### 4.2 新模型放到独立路由

在同一个 `llm-pi-ai.config.providers` 下新增一条路由，同时写 `api` 和 `baseURL`：

```yaml
- id: llm-pi-ai
  name: "@deepseek-ai/dsh-llm-pi-ai"
  config:
    providers:
      opencode:
        apiKeyEnv: OPENCODE_API_KEY

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

`opencode-latest` 与 `opencode` 必须处于同一 YAML 层级（都在 `providers` 下）。

`id` 是发送给服务端的模型标识，`name` 是界面显示名称。修改显示名称不会修改服务端模型标识。

目录里没有的免费模型同理，例如 `ling-3.0-flash-fin-free`：它不是内置目录的一部分，必须独占一条路由。当前配置里留了一段注释掉的模板，协议按同族 free 模型填的是 `openai-completions` + `https://opencode.ai/inference/v1`；启用前先确认该模型实际使用的协议，否则请求阶段会报错。

### 4.3 `api` 只能取四个值

| 值 | 含义 |
|---|---|
| `openai-completions` | OpenAI Chat Completions |
| `openai-responses` | OpenAI Responses |
| `anthropic-messages` | Anthropic Messages |
| `google-generative-ai` | Google Generative AI |

写成 `openai-compatible`、`openai`、`openai-chat` 之类的值**不是**可选写法：整个 `llm-pi-ai` 配置段会在校验阶段被判无效，所有路由（包括内置 `opencode`）一起从模型菜单消失。

## 5. 检查凭据引用

配置里只引用凭据，不保存明文：

```yaml
apiKeyEnv: OPENCODE_API_KEY
```

凭据解析失败时，该路由的模型列表返回空数组，整组不会出现在会话模型选择器中。

当前环境的凭据值保存在 `~/.dsh/.credentials.yaml` 的 `refs` 段（明文，权限 600）。不要把密钥写进 `cordis.patch.yml`。

## 6. 优先用设置页

`llm-pi-ai` 没有被全局补丁或命令行覆盖层遮住时，“设置 → 模型 → 添加模型服务商”可以直接增删提供商，写回的也是配置档补丁——效果与手改 YAML 相同，而且非法值会被工具当场拦下。

只有在界面提示覆盖、或需要写注释、批量整理时，才手动编辑 `~/.dsh/profiles/desktop/cordis.patch.yml`。

## 7. 重启并验证

配置在启动时挂载。保存后**完全退出 DeepSeek Harness 再打开**（⌘Q 或 Dock 右键退出，不能只关窗口；热重载不覆盖已挂载的 `llm-pi-ai`）。

打开新会话的模型菜单，分别搜索：

```text
GPT-6.2
Fable 5.2
Gemini 4
```

预期结果：

- 菜单出现 `opencode` 分组（内置目录的 86 个模型）。
- 菜单出现 `OpenCode Latest` 分组（6 个模型）。
- `GPT-6.2` 返回 Astra 和 Sol。

界面出现模型只证明模型目录已加载。最后分别发送一次短请求，检查服务端是否接受模型 ID、协议和凭据。

## 8. 故障排查

| 现象 | 检查项 | 处理 |
|---|---|---|
| 所有第三方模型一起消失 | `api` 是否写成了四个合法值之外的值 | 改成 `openai-completions` / `openai-responses` / `anthropic-messages` / `google-generative-ai` |
| 某条路由整组消失，报 `model "xxx" needs an api` | 该路由的 `models` 里是否有目录不认识的 id | 把该 id 移到独立路由，并给路由写 `api` + `baseURL` |
| 设置页保存报 `overridden by a home patch or command-line overlay` | 全局补丁或 `--patch` 覆盖层是否也有 `llm-pi-ai` | 删掉覆盖层里的 `llm-pi-ai` 整段 |
| 整组 `OpenCode Latest` 缺失 | `OPENCODE_API_KEY` 能否解析 | 在模型设置页检查凭据状态 |
| 模型出现在菜单中，但请求失败 | 模型 ID、协议、接口地址、账户权限 | 用短请求逐项检查服务端错误 |
| 修改配置后应用加载失败 | YAML 缩进与最近备份 | 恢复备份，再重新应用最小修改 |

排错顺序：先确认哪个文件生效（第 2 节），再看 `api` 取值（第 4.3 节），最后才查模型 id 与凭据。

## 9. 恢复备份

```bash
cp "$HOME/.dsh/profiles/desktop/cordis.patch.yml.bak-YYYYMMDD-HHMMSS" "$HOME/.dsh/profiles/desktop/cordis.patch.yml"
```

恢复后完全重启 DeepSeek Harness。

## 10. 核心原则

1. `llm-pi-ai` 只写在配置档补丁里；全局补丁写同名条目会锁死设置页。
2. 内置路由不写 `models`，新模型走独立路由。
3. `api` 只接受四个协议值；写错不是单条模型失效，而是整段配置失效。
4. 一条路由里只要有一个模型缺 `api`/`baseURL`，整条路由都不会加载。
5. 密钥使用凭据引用，不写入 YAML。
6. 修改后完全重启，再同时验证设置页、会话菜单和真实短请求。