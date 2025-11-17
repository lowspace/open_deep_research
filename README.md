# 🔬 Open Deep Research

**项目简介**：Langchain 团队使用 langgraph 实现的 deep research 项目

- **August 2, 2025**: Achieved #6 ranking on the [Deep Research Bench Leaderboard](https://huggingface.co/spaces/Ayanami0730/DeepResearch-Leaderboard) with an overall score of 0.4344 using GPT-4.1.
- **Nov 15, 2025**: Achieved #2 ranking on the [Deep Research Bench Leaderboard](https://huggingface.co/spaces/Ayanami0730/DeepResearch-Leaderboard) with an overall score of 0.506 enhanced with [Gensee Search](https://github.com/GenseeAI/open_deep_research) using GPT-5.

**整体架构**：

<img width="1388" height="298" alt="full_diagram" src="https://github.com/user-attachments/assets/12a2371b-8be2-4219-9b48-90503eb43c69" />

通过三个阶段实现 deep research：

- scope（调查）：通过与用户互动，搞懂研究问题 + 根据互动结果生成研究主题；
- research（研究）：使用 Orchestrator-workers （supervisor + workers）的 workflow 进行研究任务的分解、分发、推进；
- write（写作）：根据 research 获得的结果，一次性写出研究报告。

**Agent 架构**：

<img width="817" height="666" alt="Screenshot 2025-07-13 at 11 21 12 PM" src="https://github.com/user-attachments/assets/052f2ed3-c664-4a4f-8ec2-074349dcaa3f" />

- clariy_with_user:
  - input: user query
  - output: agent 认为 query 中模糊不清的，或需要进一步明确的问题
- write_reseach_brief:
  - input: clariy_with_user 的所有交互过程
  - output: 一个综合了用户所有要求的研究任务
- supervisor:
  - input: write_reseach_brief 返回的研究任务 OR 每一次 supervisor_tools 返回的所有内容（每一个 supervisor_tool 都会调用一个 researcher 来完成子任务）
  - output: 下一步的研究计划
- researcher:
  - input: supervisor 分发的某一个子研究任务 OR 当前搜索的搜索进度
  - output: 下一步的搜索内容
- summarize_webpage:
  - input: supervisor 搜索结果中的某一个 webpage 的内容
  - output: 根据 webpage 类型进行结构化压缩后的 JSON。
- compress_research:
  - input: 某一个 researcher 的所有交互过程
  - output: 该子研究任务的报告
- final_report_generation:
  - input: write_reseach_brief 的结果 + clariy_with_user 的所有交互过程 + researcher 里的所有推理过程和研究报告
  - output: 最终的研究报告

### Findings

- https://github.com/lowspace/open_deep_research/issues/8 在 supervisor 分发任务和 researcher 进行搜索前，都会调用一个 `think_tool` 进行针对性的推理。think_tool 在这里的角色有两个，一是 agent 能带着所有的推理过程进行推理，extended thinking 的内容不进入上下文，而 tool resutls 通常都会进入上下文；二是相比 extedned thinking 更聚焦于任务的推理内容，让模型记住自己的目标，防止 lost-in-the-middle，类似于 manus 里面的 to-do list。
- https://github.com/lowspace/open_deep_research/issues/14 https://github.com/lowspace/open_deep_research/issues/17 deep research 里的终止条件结合了 model-aware 和 hard-code 进行软硬限制，前者是 agent 在执行任务过程中认为任务结束，call complete tool；后者是迭代到指定次数之后，自动抛出。在计数的时候，supervisor 是按照「agent 使用次数」进行计数；reseracher 是按照「tool 的总调用次数」进行计数，因为 researcher 中 think_tool - search tool 是一个 pair，每调用一次就计数一次。
- https://github.com/lowspace/open_deep_research/issues/28 平行的 multi-agent 有助于 context isolation。
- https://github.com/lowspace/open_deep_research/issues/27 https://github.com/lowspace/node-DeepResearch/issues/5 https://github.com/lowspace/node-DeepResearch/issues/3 为什么要「一次性写完报告」，而不是「分章节写，然后综合」？Jina AI 在 deep research 的实现中使用的是「分章节写，然后使用额外的 agent 专门解决 coherence 问题」，langchain 中直接使用 one-shot 写 report 则没有这个问题。
- https://github.com/lowspace/open_deep_research/discussions/24#discussioncomment-14404445 设计的架构时候确认 inductive bias => 方便替代和更新。
- https://github.com/lowspace/open_deep_research/discussions/1 https://github.com/lowspace/open_deep_research/discussions/23 prompt 写作的时候逐层进行展开，引入新指令的时候尽量遵循「就近原则」；使用 key-attribution 的描述定义 JSON schema，而不是直接给 few-shots 或者是完整的 JSON 示例。
