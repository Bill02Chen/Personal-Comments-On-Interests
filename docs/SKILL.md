---
name: handoff-new-chat
description: |
  总结当前 Kimi 对话内容，并在 Kimi 桌面客户端（macOS）打开新对话，自动发送由「对话总结 + 用户对新对话的需求」组成的 handoff prompt。当用户要求总结当前对话并开启新对话继续工作、把对话上下文交接给新对话、或说"开个新对话做 X / 新建对话继续"时使用。Summarize the current conversation and open a new Kimi desktop chat with an auto-sent handoff prompt combining the summary and the user's next task.
---

# handoff-new-chat

把当前对话的上下文交接给 Kimi 桌面客户端中的一个新对话。

## 流程（按顺序执行）

1. **明确新对话任务**：从用户请求中提取"新对话要做什么"。若请求中没有或表述不明确，先向用户提问，不要猜。
2. **生成对话总结**：仅基于当前对话上下文（对话历史、本轮已生成的文件）。格式固定为：

   ```
   - 主题：<本次对话主题，一句话>
   - 关键结论：<编号列表>
   - 产物：<文件绝对路径或可访问链接；无则写"无">
   - 未决事项：<列表；无则写"无">
   ```

   约束：全文 ≤ 600 字；只写对新对话有用的事实与结论，不写过程描述；不确定的内容标注"不确定"。
3. **组装 handoff prompt**：按以下模板写入 workspace 内的临时文件（如 `handoff_prompt.txt`）：

   ```
   【对话交接摘要】
   <第 2 步的总结>

   【本对话任务】
   <第 1 步明确的需求>

   【注意事项】
   <可选：约束、偏好、相关路径；无则省略整节>
   ```
4. **发送**：先读 `references/kimi-cu-playbook.md`，按其步骤在 Kimi 桌面客户端点击「新对话」、聚焦输入框、输入 handoff prompt 并发送。
5. **回报**：最终回复中展示 handoff prompt 全文，并说明发送是否成功（以最后一次 `get_app_state` 的验证结果为准）。

## 授权与边界

- 自动发送已在创建本 skill 时获得用户授权，执行前无需再次确认。若用户在当次请求中明确要求不发送，则不发送，改为把 handoff prompt 文件路径交给用户手动复制。
- 发送重试最多 2 次；仍失败则保留 handoff prompt 文件并告知用户失败原因，不再继续点界面。
- 只操作 Kimi 桌面客户端中与任务相关的元素；不读取、不复述界面上的无关信息。
- 若 Kimi 桌面客户端未运行，提示用户启动后重试；不要自行安装或修改 KimiCU 服务。

## 资源

- `scripts/copy_to_clipboard.py` — 把 handoff prompt 写入 macOS 系统剪贴板（长文本输入失败时的备选路径）。用法见文件头注释。
- `references/kimi-cu-playbook.md` — 用 kimi-cu 操作 Kimi 桌面客户端的完整步骤、长文本策略与排障方法。执行第 4 步前必读。
