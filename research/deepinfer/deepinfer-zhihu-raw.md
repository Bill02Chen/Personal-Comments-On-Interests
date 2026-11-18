Title: DeepSeek V4.1 Flash 大规模推理部署实践

URL Source: https://zhuanlan.zhihu.com/p/2095560295930508953

Markdown Content:
大模型的能力边界由模型本身划定，而如何让这些能力以合理的成本服务海量用户，则是**[推理框架](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E6%8E%A8%E7%90%86%E6%A1%86%E6%9E%B6&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLmjqjnkIbmoYbmnrYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.0c-h6uKlO7DwgdaKoHNZJVNXKBQbB3IlPscjwMc4_eo&zhida_source=entity)**要解决的核心问题。

面对大规模的 AI 调用需求，推理框架已经从单机服务程序演变为跨集群协同的复杂系统。算子、通信、调度与缓存管理相互影响，共同决定[端到端性能](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E7%AB%AF%E5%88%B0%E7%AB%AF%E6%80%A7%E8%83%BD&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLnq6_liLDnq6_mgKfog70iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.KVdfFeOqQUsP0U0m1xoqXiyKDaab8-_3SLGcJ1fvkac&zhida_source=entity)。对于大规模在线服务，[性能优化](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLmgKfog73kvJjljJYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.ctNLxroO43FF3iF5Qmqw_8L-cJ2H_bqdnIrZMW25iT4&zhida_source=entity)需要落实到具体的部署条件中：使用什么芯片、如何组织硬件、如何分配计算，以及如何复用已有的计算结果。

[DeepSeek V4.1 Flash](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=DeepSeek+V4.1+Flash&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJEZWVwU2VlayBWNC4xIEZsYXNoIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6Mjg1NzM3MzU0LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.7EwAhcY2th65FOMQ5CA7gJVwpedl1w7OqRU0Fgse0UY&zhida_source=entity) 推理系统从设计之初，就确立了三个目标：

1.   **兼容多种厂商的[计算芯片](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E8%AE%A1%E7%AE%97%E8%8A%AF%E7%89%87&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLorqHnrpfoiq_niYciLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.pWrfRD1XSTzTJhGYvOt7kRiD27z-s4MTYhlLyVKO5FI&zhida_source=entity)**：支持在不同硬件平台上部署，并充分发挥各平台的性能。
2.   **基于常见硬件组织 Prefill 和 [Decode](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=Decode&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJEZWNvZGUiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.KUcbFCipqAI7cfQVb_VFWWDijn9_RFT2buGLx1Ktd7U&zhida_source=entity)**：用常见的 8 卡机器做 Prefill，用常见的 64～128 卡[超节点](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E8%B6%85%E8%8A%82%E7%82%B9&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLotoXoioLngrkiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.cM86LvWfnFcQxQV8py1Nr5Dl6LuZH4klxYjpO7NSoHc&zhida_source=entity)做 Decode，围绕两个阶段的负载特征设计计算、通信与调度。
3.   **支持将 [KVCache](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=KVCache&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJLVkNhY2hlIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6Mjg1NzM3MzU0LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.xPrT0BGg8KAnGUUO_b1wyq-wjZMyLJ6P5w6PPWNDzJ4&zhida_source=entity) 保留 72 小时以上**：为上下文复用提供更长的时间窗口，在缓存命中时减少重复的 Prefill 计算，并将长时间保留缓存所需的容量与管理开销纳入[系统设计](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLns7vnu5_orr7orqEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.8M6p_Z-zvaOh2x0OvMPhn2rfHswNZRXRty4zHQV3wps&zhida_source=entity)。

围绕这些目标，我们持续优化计算、通信、调度与存储，在满足延迟要求的前提下提高系统的有效吞吐，用更少的算力与能源支撑更多请求，让智能服务以更低的价格触达更多用户。

此前，我们已开源[用于开发高性能算子的 TileLang 以及一系列推理算子库](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/X41mKH4Ds-VXUAnK6M8Eww)。**今天，我们正式开源大规模云服务推理框架**[DeepInfer](https://link.zhihu.com/?target=https%3A//github.com/deepseek-ai/DeepInfer)**。** 基于这一框架，我们已实现 DeepSeek V4.1 Flash 的高吞吐、低延迟大规模推理服务。目前单一集群的服务吞吐是 155 万亿 Tokens/天，整体收入是每卡每小时 38 美金（按谷价）。

DeepInfer 集成了高性能算子和深度优化的推理引擎，支持分布式推理、推测解码、[SWA Bounded Replay](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=SWA+Bounded+Replay&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJTV0EgQm91bmRlZCBSZXBsYXkiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.SiL5DtJMrj4GegIIOG-T_uWuVSfgWJIP2Ehn2Bv4ftE&zhida_source=entity)、结构化输出等能力，可在英伟达与[华为昇腾](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%8D%8E%E4%B8%BA%E6%98%87%E8%85%BE&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLljY7kuLrmmIfohb4iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.5rsBJrC7AmaYRCLvNU52UgFK0XIFqxNspNOXkjZA9uc&zhida_source=entity)平台上运行。同时，我们还开源了生产级 Token [编解码](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E7%BC%96%E8%A7%A3%E7%A0%81&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLnvJbop6PnoIEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.h7hDb1O9A4jqEuAlpQNu3K2Bhw5_AuxmRzFWIcI0KAE&zhida_source=entity)组件 [deepseek-recipe](https://link.zhihu.com/?target=https%3A//github.com/deepseek-ai/deepseek-recipe)，支持主流大模型 API 协议与模型 Tokens 之间高效、准确的编解码，处理逻辑与 DeepSeek 官方 API 完整对齐，以确保第三方部署能获得与官方部署一致的 API 体验。未来，DeepSeek 新模型的推理代码及后续性能优化也将在 DeepInfer 中持续开源。

推理系统中的 KVCache 管理与负载均衡组件因与内部基础设施耦合较深，相关代码目前尚未完整开源。本文将从上述设计目标出发，介绍 DeepSeek V4.1 Flash 推理系统的整体架构与关键实现，以及这两部分的设计细节。

* * *

## **推理系统设计概览**

**DeepSeek****V4.1 Flash 推理系统的优化目标是：更大的吞吐，更低的延迟。**  
为了实现吞吐和延迟的两个目标，我们使用：

*   **（超）节点内 MoE 部分[专家并行](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E4%B8%93%E5%AE%B6%E5%B9%B6%E8%A1%8C&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLkuJPlrrblubbooYwiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.Z3zguZqCZXrXJiJ9gIJRGviJsBlp4roL_Dw-GFLSrhw&zhida_source=entity)（EP）**：增大 batch size，提高 GPU [矩阵乘法](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E7%9F%A9%E9%98%B5%E4%B9%98%E6%B3%95&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLnn6npmLXkuZjms5UiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.a-ObZLBHaLFMG4os45I-xtgwRCszGvMU2UZdTZykfNM&zhida_source=entity)效率，从而提高吞吐。
*   **Vision Encoder / Prefill / Decode 分离部署**：解耦各阶段资源分配，独立控制并分别优化 TTFT 和 TBT。

继承 [DeepSeek-V3 / R1 推理系统](https://zhuanlan.zhihu.com/p/27181462601)，ATTN 部分使用[数据并行](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E6%95%B0%E6%8D%AE%E5%B9%B6%E8%A1%8C&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLmlbDmja7lubbooYwiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.wLV-4Lf0PET5VUvCuxeCQKTWpFFdzySlkYbEtvfFK-0&zhida_source=entity) DP （#DP=#EP），不同的 DP / EP 之间需要进行[负载均衡](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=2&q=%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLotJ_ovb3lnYfooaEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MiwiemRfdG9rZW4iOm51bGx9.6qM0g9lU0pAP5JgVBjAgnZc-L8Bu4TYmM0TykVoeUOg&zhida_source=entity)，为此我们开发了 **Prefill Load Balancer、Decode Load Balancer、Vision Encoder Load Balancer、Expert-[Parallel](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=Parallel&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJQYXJhbGxlbCIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI4NTczNzM1NCwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.m3EuWg1x-Jl6r5zrJcYjZe7cNk1U1lcUc8IskkMW650&zhida_source=entity) Load Balancer**；为了节省重复 Prefill 的成本，我们开发了 **KVCache Storage**。

在用户请求进入推理前后，需要进行 API 协议与 token 序列之间的转换，**DeepSeek [Recipe](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=Recipe&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJSZWNpcGUiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.fwC09Aqbt7eZ4zsVTddlu_mjn_zrUh6vn5Gpe1miQSc&zhida_source=entity)**支持主流大模型 API 协议与模型 token 序列之间高效、准确的编解码，通过 special token 分段和并行的策略，避免 special token 注入的同时，降低长文本 [tokenize](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=tokenize&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJ0b2tlbml6ZSIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI4NTczNzM1NCwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.O8OPlqMx8Ey0gi-GN1K1DhE_vmDWwNdQYL8fqkbCZms&zhida_source=entity) 的处理[时延](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E6%97%B6%E5%BB%B6&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLml7blu7YiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.kEX87qNggbh5EIzIYnU3sJIlOg0h7kRFQ9AsL2vX1TE&zhida_source=entity)。

可参考以下架构示意图：

![Image 1](https://picx.zhimg.com/v2-9dcb32fbcfa3caa70540e88955eafd81_1440w.jpg)

**[并行策略](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%B9%B6%E8%A1%8C%E7%AD%96%E7%95%A5&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLlubbooYznrZbnlaUiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.m1SVr_lOpM0PXKC59eY6KvBBJEhpo-IhG3DKoZy6Njk&zhida_source=entity)：**

*   **Prefill**: EP2～8/DP2～8; 在显存足够的情况下用尽可能小的 EP/DP
*   **Decode**: EP64~128/DP64~128 (Wide-EP); 在[显卡](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E6%98%BE%E5%8D%A1&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLmmL7ljaEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.NstvSaKDcqAVEK4DoxnTQubRlp09WRG7YiuX8_RRVQg&zhida_source=entity)可用的情况下用尽可能大的 EP/DP
*   **Vision Encoder**: 一般[单卡部署](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%8D%95%E5%8D%A1%E9%83%A8%E7%BD%B2&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLljZXljaHpg6jnvbIiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.Y49G4yRIWIa2reyXnwru18kZ-iY9P5LzplKYFxgRIio&zhida_source=entity)就可以，建议用可用范围内最便宜的卡部署。

### DeepSeek-V4.1-Flash 线上系统的实际统计数据

![Image 2](https://pic2.zhimg.com/v2-7dd6fca82884c12b56d4d8b40043c979_1440w.jpg)

### **离线性能基准测试数据**

![Image 3](https://pic1.zhimg.com/v2-bff36102d7ffd58ffca11ad64738012a_1440w.jpg)

![Image 4](https://picx.zhimg.com/v2-c1c07ce013c2dc0b0a3864c566bad801_1440w.jpg)

## DeepSeek Recipe

### API 多协议适配

DeepSeek Recipe 将 Chat Completions API、[Responses API](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=Responses+API&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJSZXNwb25zZXMgQVBJIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6Mjg1NzM3MzU0LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.wtbuhnI5CttSnSFWXH3-Ym7Rlvhr-OFQpD568GHyHR0&zhida_source=entity) 和 Messages API 的请求转成同一种对话格式，共用模型模板和编码逻辑，按模型版本生成 prompt 和 token 序列。

模型输出经解析后，形成正文、思考内容和工具调用等事件，再转换为各协议的[流式响应](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E6%B5%81%E5%BC%8F%E5%93%8D%E5%BA%94&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLmtYHlvI_lk43lupQiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.pxEroEAAHrC236qx6vEvEFfZ0BXb0N6XpXTVLzwmMsE&zhida_source=entity)。请求转换、prompt 编码和输出解析都由 Rust 实现，以实现高效处理，同时支持 Python 绑定调用，以提升易用性。

### Tokenizer 处理与加速

实践中，Tokenizer 面临两个比较棘手的问题：（1）随着模型上下文变长，线性 tokenize 的方法会引入较大的延时，影响 TTFT，且显著增加了服务的 CPU 开销；（2）special token 用于标记消息角色、思考边界和工具调用等结构，但这些标记也可能作为普通文本出现在对话当中，在处理请求时，需避免用户输入中携带的 special token 文本对模型推理效果产生不良影响。  
针对这两个问题，我们在 DeepSeek Recipe 中实现了基于 special token 边界匹配的解决方案。

**输入编码**：构建 prompt 时记录显式插入的 special token 及其位置，编码时直接映射到对应的 token ID，普通文本按基础词表编码。用户输入、工具返回和工具调用参数中的 `<｜User｜>` 和 `<｜image｜>` 等字符串会被编码为相应的普通 token 序列，防止通过文本注入控制 token 影响模型行为。  
**输出解析**：在处理响应 token 流时，我们根据 token ID 识别思考结束标记 `</think>` 和 DSML 标签中的 `｜DSML｜` special token。模型用普通 token 拼出的相同文本会被正确保留为原始内容，不会获得 special token 触发思考结束或工具调用的能力。  
**分段加速**：构建 prompt 时，已经能够确定 special token 的位置和边界。根据 BPE tokenizer 的[编码原理](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E7%BC%96%E7%A0%81%E5%8E%9F%E7%90%86&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLnvJbnoIHljp_nkIYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9._SLzt6V_Hzahkn3HShiN8YO7Pc8y6fe-6zAY6vo99fM&zhida_source=entity)，可以发现 special token 之间的各段普通文本可独立编码。因此，我们在编码时首先跳过普通文本当中 special token 的匹配与切分，然后并行地编码各段普通文本，最后再按原顺序拼接结果。这样能降低超长上下文请求的分词耗时，优化请求延迟。

### 流式输出解析

推理后端按 token [粒度](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E7%B2%92%E5%BA%A6&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLnspLluqYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.3dZey2WayYRudVCi9f9Z2R7GE9YG-dq4c5wF4JZnmPA&zhida_source=entity)返回 [chunk](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=chunk&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJjaHVuayIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI4NTczNzM1NCwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.CxY_vz7tU_JEf0ALKzyaPvZDyXftQ-Xgd6IRuSF2Lmw&zhida_source=entity)，一个 [UTF](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=UTF&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJVVEYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.ODAUyipFuBDvcAEJZIcVdrbtosyiZv87tIyRTln0qBE&zhida_source=entity)-8 码点、DSML 标签或工具调用参数都可能被拆分到多个 chunk 中。为了正确 decode 模型的输出，并尽早解析出其中的结构以降低返回时延，我们需要实现一套跨 chunk [流式解析](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E6%B5%81%E5%BC%8F%E8%A7%A3%E6%9E%90&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLmtYHlvI_op6PmnpAiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.7dMUsCDP5u8bzzaUao4IJeyFm6LOScGjXALJbEAQAJA&zhida_source=entity)的状态机，在内存中保留尚未完成的处理进度和状态，并及时返回已经确定的内容。

**跨 chunk 解析**：解码遇到不完整的 UTF-8 码点时，状态机会先缓冲相关 token，待后续 token 到达、能够补齐状态后，再交给解析器。解析器按思考、正文和工具参数等阶段维护状态，使用 KMP [算法](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E7%AE%97%E6%B3%95&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLnrpfms5UiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.29GIBlvA4EDQw6j5EexC2LeNyCAD-IXyEashCw54t-8&zhida_source=entity)增量地匹配分隔标记。若发现 chunk 末尾只有标记的一部分，就暂时保留匹配进度，在下一个 chunk 到达时继续增量识别；已确定的内容则随解析进度输出，无需反复扫描此前的文本。  
**工具参数转换**：工具名解析完成后即可返回，后续根据 DSML 参数边界直接生成 JSON 片段。字符串值逐段做 JSON 转义，数组、对象等非字符串值按原始 JSON 文本输出，无需构造完整参数对象，省去反复解析和序列化的开销。即使参数较长，也可以在生成过程中逐段返回，缩短首个参数片段的等待时间。

## 负载均衡

数据并行和专家并行都需要尽可能地为每张卡分配均衡的负载，从而降低卡间不同步造成的空转和等待开销。其中 Prefill/Decode Load Balancer 通过请求[路由](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E8%B7%AF%E7%94%B1&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLot6_nlLEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.qsAdNC7YYPSx2td6UZabB0nwjIc2Rney37YyFv-pV_k&zhida_source=entity)和调度来分别控制 prefill 和 decode 阶段不同卡上注意[力子](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%8A%9B%E5%AD%90&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLlipvlrZAiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.y6s5OI12WkXvfVosOmpJbBiRqb6yB-CXUnT8lXnZ4yo&zhida_source=entity)层的负载均衡，Vision Encoder Load Balancer 控制不同 vision encoder 之间的负载均衡，Expert-Parallel Load Balancer 通过冗余专家和动态调整来控制 MoE 子层的负载均衡。

### Prefill Load Balancer

Prefill Load Balancer 的实现机制可分为 4 层，思路是围绕拟合的注意力子层计算量（记为 compute-quota，是预先拟合的二次代价模型`cost(q, c)`，`q` = 序列中未命中的 token 数，`c` = 这条序列已经算完的前缀长度）来进行请求的路由和分布调度。

**分桶：**按输入总长分桶为 4 档：<16K / 16-64K / 64-256K / ≥256K，每档分到的引擎台数按该档上一轮的 compute-quota 需求占比动态重分；  
**DP 组间：**按目前剩余的 compute-quota 拉取，由于任何引擎都能命中任何前缀，不做前缀亲和路由；  
**DP 组内：**一批请求给到引擎后，每收到一个，就分发到当前 compute-quota 负载最低的 DP rank；  
**单卡内：**请求 FIFO 出队，chunk size 不固定，由预算反解。每步有一个时间预算（单位是纳秒），调度时用拟合的二次代价模型 `cost(q, c)` 预测这一步要花多久，对 `q` 反解这个[二次方程](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E4%BA%8C%E6%AC%A1%E6%96%B9%E7%A8%8B&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLkuozmrKHmlrnnqIsiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.pvg_-LG-_mEVZVb2uLFjsHCYVSgmRHR_g-Skx33rU64&zhida_source=entity)，取正好满足剩余时间预算的最大 `q`。

### Decode Load Balancer

Decode Load Balancer 对所有decode实例的所有卡进行统一路由，原则是同时均衡请求数和 KVCache 总长度两个维度，前者决定每步批大小，后者决定注意力部分的代价。Decode Load Balancer 启发式地将短请求发给并发较小且 KVCache 总长偏大的卡，长请求发给 KVCache 总长较小的卡，具体可参考DualPath的组内调度算法 [[1]](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_1)。

### Vision Encoder Load Balancer

对于图片的路由以单台机器为单位，由 prefill 机器随机选中一台部署有卡数个 vision encoder 的机器，在这台机器上，将图片交由当前处理图片 patch 数目（由宽高直接算出，`(width/patch_size) × (height/patch_size)`）最少的卡的处理生成 embedding。

### Expert-Parallel Load Balancer

专家并行负载均衡的实现方法可概括为：每卡在常驻专家之外多备若干冗余专家，用来放热门专家的副本。通过统计最近 N 步各专家的负载，每 M 步根据统计结果重新分配专家到卡实现动态调整，其中专家权重在 scale-up network 里经 symm_mem 在[并行组](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%B9%B6%E8%A1%8C%E7%BB%84&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLlubbooYznu4QiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.0deEoELmFt_5bElRVVfnSJW1iI8MMd8UHhJykGf7Cvw&zhida_source=entity)内转移。

**Prefill 的策略**：每卡 4 个冗余专家起步（向上填充至总专家数为 ep_size 的倍数）；统计窗 100 步，每 1000 步重排，线上数据实测 moe_lb（最慢 rank / 均值）= 1.0320（EP2/DP2）/ 1.0618（EP4/DP4）。  
**Decode 的策略**：每卡 1 个[冗余专家](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=4&q=%E5%86%97%E4%BD%99%E4%B8%93%E5%AE%B6&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLlhpfkvZnkuJPlrrYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6NCwiemRfdG9rZW4iOm51bGx9.4SRm9MIMengPkYCtBYsRUbQcHNRE8zoRYN0kAE6eCl4&zhida_source=entity)起步（向上填充至总专家数为 ep_size 的倍数），例如 EP64 取整后每卡共 7 个专家（不包括[共享专家](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%85%B1%E4%BA%AB%E4%B8%93%E5%AE%B6&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLlhbHkuqvkuJPlrrYiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9._SGlt-yekZPB3BPjEJ7fDrfXgO9C2u00gkPcohOkeB8&zhida_source=entity)）；统计窗 1000 步，每 10000 步重排一次，线上数据实测 moe_lb = 1.090。

### 自动扩缩容

PD 分离后两者的资源需求比例随流量动态发生变化，静态配比下的资源错配导致排队和空转的发生，因此需要通过自动[扩缩容](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=2&q=%E6%89%A9%E7%BC%A9%E5%AE%B9&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLmiannvKnlrrkiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MiwiemRfdG9rZW4iOm51bGx9.W0AYezHWyRfKEJA8PYD7s4hdfJSdZZIXcGC2U9kZlEI&zhida_source=entity)在集群层面跟随流量调整 Prefill/Decode 实例的比例。

**实现方式**：Prefill 通过统计未命中 token 数判断所需实例数目，decode 通过在计算的请求数判断所需实例数目，调整有限速和上下界。其中，针对大 EP 部署时[长尾](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E9%95%BF%E5%B0%BE&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLplb_lsL4iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.x07-AQcBh0dIuQlc21SEQoofIhBfPuZ-xho89p6r-64&zhida_source=entity)请求导致资源浪费的问题，decode 实例收到下线通知后进入[宽限期](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%AE%BD%E9%99%90%E6%9C%9F&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLlrr3pmZDmnJ8iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.bm4H0CvJ_f3b_u7J8hOUSDTsq0vMhueqCIJ1sxyo_qE&zhida_source=entity)（1 min），宽限期结束后仍未完成的请求及其 KVCache 一次性全部转发给其他实例继续 decode 计算。  
**冗余[配额](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E9%85%8D%E9%A2%9D&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLphY3pop0iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.B2_Aza65TQYwrWIoiq_wLWv1y5vvrOUV8q_aIJrDHZ0&zhida_source=entity)**：部署时留冗余配额来应对流量变化引发的冲击，包括两方面：一是池子里留有尚未被占用的 ready 机器作为扩容库存；二是永远比算出来的需求多的实例在线，具体来说，配额占用率高于 0.8 时禁止缩容。  
**容错机制**：错误重启在分钟量级，容错依赖配额冗余 + 请求级重新路由的组合机制，而不是靠缩短实例重启时间。大 EP 部署下，一个实例的全部卡构成单一[通信域](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E9%80%9A%E4%BF%A1%E5%9F%9F&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLpgJrkv6Hln58iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.We5QU7Q1JrFc0si5NX3tHTGsWZ9pRbBX4EBMZ3nfXTs&zhida_source=entity)，任意一张卡故障即导致整个实例不可用。重启时为了把该实例其余健康的卡继续用起来，平台按当前健康的所有卡重新确定EP大小，实例以实际拿到的卡数作为 EP 规模。

## KVCache Storage

随着对话上下文的增长和 agent 的广泛使用，KVCache 前缀缓存复用能够极大减少 prefill 重算，从而显著降低成本。DeepSeek-V4.1-Flash 模型的 KVCache 存储分为两个部分：Block KVCache 和 State KVCache，分别存储于 3FS 集群和本机内存中。  
**Block KVCache**：以 128 tokens 为一个 block 进行分页式管理，根据模型架构设计，存储 4 层的 main KV 和 indexer K。  
**SWA KVCache**：前 20 层各一个 134 token 的 ring（窗口长度=128 + 投机回退=6），不存储压缩中间态。

| 内容 | 介质 | 副本 | 管理 |
| --- | --- | --- | --- |
| 运行中的Block KVCache + State KVCache | 推理卡显存 | — | prefill/decode实例 |
| SWA KVCache | 产出的 prefill/decode机器的内存 | 单副本，3FS 上没有 | prefill/decode实例；跨机命中使用400G VLAN 拉取 |
| 分页 Block KVCache | 独立 3FS 存储集群 | 应用层无副本，靠 3FS 自身冗余 | kvcache-store |

### Block KVCache

Block KVCache 落盘于独立的 3FS 存储集群，不使用本机内存存储。相比多级存储系统，独立的存储集群使得所有 prefill 机器能命中任何前缀，避免前缀亲和和负载均衡路由策略冲突，也使得存储和[计算节点](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E8%AE%A1%E7%AE%97%E8%8A%82%E7%82%B9&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLorqHnrpfoioLngrkiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.wYg44HNnqiFi-LBh9mkda4DhYQS2ZB98HQPIVfyCkq0&zhida_source=entity)自动扩缩容解耦、KVCache 缓存不受计算节点故障的影响（计算节点故障率比存储高数量级以上）。KVCache 系统设计考虑两方面因素：扩大存储容量来提升命中率，提升[带宽](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%B8%A6%E5%AE%BD&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLluKblrr0iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.Utjhy0VwPvIulHtKCKXfmoZ4DnsQk0WgDug1V0FBkX8&zhida_source=entity)来降低读取延时。

容量方面，[3FS 分布式文件系统](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=3FS+%E5%88%86%E5%B8%83%E5%BC%8F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiIzRlMg5YiG5biD5byP5paH5Lu257O757ufIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6Mjg1NzM3MzU0LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.GjZqqxrnY1x2fywO1EkeRLoprEF442IQhxSKtjTZQ5Q&zhida_source=entity)[[2]](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_2)为 KVCache 提供约 100 PiB 的 SSD 存储，agent 流量 KVCache 的 [TTL](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=TTL&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJUVEwiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.RUH5Vaspluc62-FlPThG84lTrYTbRIgnTOJWULGbcL4&zhida_source=entity)（Time-To-Live）达到 3 天，chatbot 流量达到 7 天。对于 chatbot，其长尾命中更严重，少量 KVCache 会极多次读取，缺失后对整体命中率损失较大。根据经验，3 天左右的 TTL 会造成 chatbot 流量的 KVCache 命中率下降 0.5%-1%；带宽方面，3FS 存储集群针对 KVCache block 粒度（约 112 KiB）的随机读写进行优化，峰值聚合读带宽达到 10 TiB/s，峰值聚合写带宽约 850 GiB/s。

推理框架层面，Block KVCache 由 decode 实例落盘存储，使得 prefill 和 decode 输出均被复用。对于 KVCache 命中开销，命中路径分为三段，只有第一段在关键路径上：**匹配**由 Entrypoint 在本机 CHECK 角色的 kvcache-store 上做[哈希链](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%93%88%E5%B8%8C%E9%93%BE&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLlk4jluIzpk74iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.Vx6b4YgFHKjOZGHDgC3DZdMDiVU4x42QrnzaL-2gQpY&zhida_source=entity)前缀匹配，只查元数据，但命中长度需在路由前进行计算，无法掩盖；**取 KV** 在独立进程里[异步](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E5%BC%82%E6%AD%A5&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLlvILmraUiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.vK30KgwYfD-mXkoO_3B3wi9nUvUzNhaUWy68a4hyxAw&zhida_source=entity)做，把 KVCache 从 3FS 搬进宿主机 shm；**换入**从 shm 到 HBM 在 engine 进程内的线程里做，目标是实现逐层 H2D （Host-to-Device） 传输与前向计算重叠，由于 CUDA copy engine 对于 KVCache Block 细粒度的拷贝性能不佳，换入使用 GDR 实现，用 RDMA 网卡就能打满 PCIe 带宽，并且不占 SM。

### SWA KVCache

SWA KV 不进持久化缓存，而是留在产生它的那台机器的内存里，由一个逻辑上统一、实际按 key 分开存放在各台 Entrypoint 上的内存目录记录在哪台机器，命中时本机没有就跨机取回。由于 SWA KV 只在会话内的分钟级窗口里被复用，池子总容量虽小但周转快，足以覆盖同时活跃的会话，线上数据显示，其TTL约 180 秒，命中率在 98% 以上。被驱逐造成的未命中由上述的 Encoder SWA Bounded Replay[[3]](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_3) 重算。

SWA KVCache 只在 128 tokens 的整数倍位置上存储。Prefill 阶段结束后存储：位置取序列末尾留出一个 token 后向下对齐到 128 所得的位置，不足一个 128 tokens 则本轮不存储；chunked prefill 中途不存储，只有最后一个 chunk 才存储。Decode 阶段在显存中另开一块副本区，每跨过一个 128 tokens 边界就把该请求当前的窗口在显存内拷入副本区，请求结束时从副本区中最近一次完成拷贝的位置存储，附着点向下对齐到 128 边界，拷贝位置最多越过边界 5 个 token，由 ring 比窗口多出的 6 个位置吸收。

### 命中策略

命中的原则是：命中长度由 Block KVCache 决定，最后一个 block 是否需要重算由 SWA KVCache 决定，若 SWA KVCache 未命中，就退一块重算。具体地，根据YOCO截断要求，对于输入长度 L，最大的命中长度最多为 L - 134（窗口 128 + 投机回退 6）。在这个上限下，首先查询 Block KVCache 命中情况，假设命中长度为 L1（L1为block size=128的整数倍），再查询SWA KVCache命中情况。假如在 L1 - 128 至 L1 长度范围内，存在 SWA KVCache 则命中长度为 L1，否则命中长度为 L1 - 128，其中 128 为重算窗口。

### 3FS 分布式文件系统

如上所述，Block KVCache 数据量巨大，依赖 3FS 分布式文件系统存储。为应对节点宕机或网络抖动，需要兼顾高可用与高性能。

*   数据以[纠删码](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E7%BA%A0%E5%88%A0%E7%A0%81&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLnuqDliKDnoIEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.F4lCEEP5X7SQH-787Vz18G8lNeygwkthg4vXAeKWD_I&zhida_source=entity)（10+2 Erasure Coding）存放在 3FS 上，每个 EC 组可容忍 2 节点同时故障。
*   读取时同时向组内 11 台 Storage 节点发起请求，任意 10 个响应即可解码返回。
*   GC Worker 组件负责清理过期缓存：每个 Storage 节点部署一个实例，按 LRU 策略周期性淘汰最久未访问的 KVCache，将磁盘占用控制在水位线以下。

## SWA Bounded Replay

SWA KVCache 不进行持久化存储，缺失时以重算恢复，这是 V4.1 相对 V4 在 KVCache 管理上的主要变化，具体可参考 V4.1 [技术报告](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E6%8A%80%E6%9C%AF%E6%8A%A5%E5%91%8A&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLmioDmnK_miqXlkYoiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.7X0sp34XcAc1echicu0zZEurww5cqoZpzm5T9EiDC6o&zhida_source=entity)[[2]](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_2)。V4 的做法是落盘加精确重算：SWA KVCache 在两个位置写入持久化存储，未命中时重算“层数 × 窗口”个 token 做完整前向，逐层精确恢复窗口状态。这条路径代价很高，必须用足够大的 SSD 容量把未命中率压到极低，代价是 SWA KVCache 占去持久 KVCache 近一半容量，且未能带来相应的命中收益：Block KVCache 呈长尾复用，一段前缀可在数天内被反复命中；SWA KVCache 只在活跃会话内被分钟级复用，会话结束或进入下一轮即失效。以数天留存的 SSD 容量承载分钟级生命期的数据，在多轮短对话场景下尤其不划算。因此，V4 技术报告提出的 [Zero SWA Caching](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=Zero+SWA+Caching&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJaZXJvIFNXQSBDYWNoaW5nIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6Mjg1NzM3MzU0LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.v1BY0vRBZLJkmUv3RpzSY6qjEpgHkVcPP3lDdsPh7QM&zhida_source=entity)（即完全不落盘、仅依赖精确重算）因其代价在生产部署中过高，在V4.1推理系统中未实际采用。

V4.1推理系统采用 SWA Bounded Replay，只重算最近 128 个 token，并把[注意力窗口](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E6%B3%A8%E6%84%8F%E5%8A%9B%E7%AA%97%E5%8F%A3&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLms6jmhI_lipvnqpflj6MiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.Ek1VL-bg491jQarlwxEzReMPACnlpAJhA4BJAw-6q-c&zhida_source=entity)截断在重算段内：从位置 s 起回放，位置 i 的 query 只和 max(s, i − 127) 到 i 区间的 key 计算，重算得到近似的 SWA KVCache。重算代价由此降为与上下文长度和层数无关的常数。该机制用在两处：

*   **Encoder SWA Bounded Replay**：前缀缓存命中长度从此只依赖 Block KVCache。SWA KVCache 缺失时，重算命中前缀的最后 128 个 token，与未命中后缀一同前向；重算段只重建 SWA KVCache，Block KVCache 原样复用、不重算不覆写。
*   **Decoder SWA Bounded Replay**：prefill 计算仅在 encoder 层进行。CED 架构下 [decoder](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=decoder&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJkZWNvZGVyIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6Mjg1NzM3MzU0LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.RYLYrXjw1lKX1I_aHHtVpm5gZc67jYmYRFwQXGN91Gs&zhida_source=entity) 层的 Block KVCache 由 encoder 末层 hidden state 投影得到，仅 decoder SWA KVCache 需各层自算，精确重建要 20 个 decoder 层跑最后 2560 个 token；改为每次 prefill 在 decoder 层只重算 prompt 末尾 128 个 token，产出仅供 decode、不进前缀缓存，总 prefill 计算量近乎减半。

## DSpark投机解码

DSpark 是 DeepSeek 提出的投机解码算法框架，相比 DFlash 在输出的 base logits 上叠加只依赖前一 token id 的低秩 logit bias，用极轻量的串行修正补回相邻 token 的局部条件化信息，提升 draft 准确率，具体可参考 DSpark 论文[[4]](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_4)。

在推理系统层面，DSpark 采用变长验证。由于不同请求、不同位置的 draft-token 接受概率不同，固定长度验证会浪费验证算力在低概率 token 上。DSpark 通过 Confidence Head 预测每个 draft token 的接受率，调度时把验证算力分配给边际收益最高的 token。线上部署时，draft 一次生成 5 个 token，根据 profile 出来的硬件吞吐曲线和每个请求的每个 draft-token 的接受率，把各 draft-token 按预期接受率排序，选取有效吞吐最大的前缀子集进行验证。经过对比，相比于固定验证 5 个 token，变长验证在线上部署场景吞吐提升 6%。

与 Asynchronous Scheduling 的配合：schedule（N+1） 执行时第 N 步的 confidence 尚未回传，因此 CPU 调度时只确定总预算，逐请求长度在 GPU 上由一个 tilelang kernel 重算（累乘、全局排序取 top-k，并融合 remap scatter），结果回写 pinned buffer。

## Asynchronous Scheduling (Zero-Overhead Batch Scheduler)

Decode 第 N 步流程为：schedule(N) → forward(N) → sample(N) → post-process(N)，其中 schedule 和 post process 是纯 CPU 逻辑，在 post-process(N) 之前，需要 GPU-CPU 同步拿到 sample(N) 的结果。这就导致 post-process(N) 和 schedule(N+1) 的时候，GPU 是空闲的状态。  
参考已有实现[[5]](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_5)[[6]](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_6)，我们通过 GPU 和 CPU 之间的异步调度，将流水线重叠：GPU 执行 forward(N) → sample(N)时，CPU 同步执行 schedule(N+1)→ launch_forward(N+1)→ post-process(N-1) → launch_sample(N+1)，GPU 全程无空闲。

## 结构化输出

结构化输出（Structured Output, SO）用于约束模型输出严格符合指定格式（JSON schema、正则表达式、[EBNF](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=EBNF&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiJFQk5GIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6Mjg1NzM3MzU0LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.M4RqDcHSC-rlVCtYU1cg-v8gb51thu4U2oj6AfNkLo4&zhida_source=entity) 等）。

**工作流程：**

*   **编译阶段（独立 SO 机器）：**将格式约束编译为语法[自动机](https://zhida.zhihu.com/search?content_id=285737354&content_type=Article&match_order=1&q=%E8%87%AA%E5%8A%A8%E6%9C%BA&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3OTI2NzQxNDYsInEiOiLoh6rliqjmnLoiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyODU3MzczNTQsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.ZGzsbNjGRWBuCsCJxrNgbubEvTR-LREJFxgfEbJNk_Q&zhida_source=entity)。
*   **运行时：**每步 decode 时，更新自动机，判断合法 token 集合，生成 mask 作用于采样逻辑，强制输出符合目标结构。

编译与推理解耦，SO 编译异步进行，请求在进入 engine 前取回编译结果。

## 参考

1.   [^](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_1_0)Y. Wu, S. Chen, R. Huang, Y. Tan, Y. Zhong, M. Zhang, X. Jin, and P. Huang. DualPath: Accelerating Agentic LLM Inference by Harvesting Disaggregated KV-Cache Storage I/O. In Proceedings of the ACM SIGCOMM 2026 Conference, SIGCOMM '26, Denver, CO, USA, 2026. ACM. ISBN 979-8-4007-2467-1.
2.   ^[a](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_2_0)[b](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_2_1)DeepSeek-AI. Deepseek-v4.1: DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression, 2026. URL https://arxiv.org/pdf/2609.19969.
3.   [^](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_3_0)DeepSeek-AI. A High-Performance Distributed File System Designed to Address the Challenges of AI Training and Inference Workloads, 2025. URL https://github.com/deepseek-ai/3FS. 
4.   [^](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_4_0)X. Cheng, X. Yu, C. Shao, J. Li, Y. Xiong, Y. Qian, J. Zhu, S. Ma, X. Zhang, J. Ye, et al. DSpark: Confidence-Scheduled Speculative Decoding with Semi-Autoregressive Generation. arXiv preprint arXiv:2607.05147, 2026a.
5.   [^](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_5_0)L. Zheng, L. Yin, Z. Xie, C. Sun, J. Huang, C. H. Yu, S. Cao, C. Kozyrakis, I. Stoica, J. E. Gonzalez, C. Barrett, and Y. Sheng. Sglang: Efficient Execution of Structured Language Model Programs. In Advances in Neural Information Processing Systems, volume 37, pages 62557–62583, 2024. doi: 10.52202/079017-2000.
6.   [^](https://zhuanlan.zhihu.com/p/2095560295930508953#ref_6_0)K. Zhu, Y. Gao, Y. Zhao, L. Zhao, G. Zuo, Y. Gu, D. Xie, T. Tang, Q. Xu, Z. Ye, K. Kamahori, C.-Y. Lin, Z. Wang, S. Wang, A. Krishnamurthy, and B. Kasikci. Nanoflow: Towards Optimal Large Language Model Serving Throughput. In Proceedings of the 19th USENIX Conference on Operating Systems Design and Implementation, OSDI ’25, USA, 2025. USENIX Association. ISBN 978-1-939133-47-2.
