# Harness 是 governance 工程

**Model + Harness = Agent**。这是这篇文章想讲清楚的一件事。

同一个 Claude 4.7 模型,被 Anthropic 自己塞进三个产品里。推理、规划、工具调用,这些底层能力完全一样。但三个产品长得完全不同:

| 产品 | 权限边界 | 可逆性 | 用户介入频率 |
|---|---|---|---|
| Claude chat | 仅 context | 100% 可逆 | 每轮 |
| Claude Code | 整个 repo + shell | 大部分可逆(git rollback) | 委派后异步 |
| Claude Cowork | 用户授权的文件夹 + 第三方连接器 | 多数不可逆 | 关键决策同步阻断 |

表里是各产品的 default governance 配置 — 连接器和工具集可调,但默认值反映了各场景的设计判断。

这张表底下藏着一对该被分清楚的概念:**orchestration** 和 **governance**。

Orchestration 是模型自己的认知部分,负责把任务拆步、调用什么工具、什么时候停下来。它写在模型权重里,Anthropic 把同一个 Claude 4.7 塞进三个产品,orchestration 能力完全一样。

Governance 是工程层的部分,负责这个 agent 允许动什么、能不能撤销、什么时候必须停下来等待用户确认。它主要落在工程层,跟场景有关 — 但这条边界是流动的。模型自己也带一部分 governance:RLHF 训的拒绝行为、safety alignment 都是嵌进权重的 governance,base model 厂正在把更多 governance 往权重里推。orchestration 和 governance 不是固定的两层,是一条正在被重画的边界,harness 工程就是在这条边界上工作。Claude chat 的用户是临时聊天用户,可逆性 100%,介入频率每轮;Cowork 的用户是不会 git rollback 的非开发者,可逆性低,关键步骤必须阻断。同一个模型,不同 governance。

三个产品之间的结构性差异,主要落在 governance 层。Cowork 不是把 Claude Code 砍掉一半的玩具,是同一个 agent 引擎换了工具集和宿主环境之后,必须重新设计的 governance 默认基线。

这是 harness 工程的本质问题。Prompt 工程在问"怎么让模型理解我想要什么",处理的是 orchestration 之内的事。Harness 工程在问一个更早、更难的问题:在什么权限边界内、以什么可逆性默认、以什么介入频率,让 agent 替我做一件事。前者是引导词,后者是治理。

---

## 一、三支柱不是清单,是 trade-off 三角

OpenAI 把他们 Codex harness 的核心总结成一句话:"Humans steer, Agents execute"。Anthropic 描述长期运行 agent 的时候用了另一句:"Imagine a software project staffed by engineers working in shifts, where each new engineer arrives with no memory of the previous shift"。

这两句加起来定义了 harness 工程要解决的真问题。Agent 不缺执行能力,缺的是跨轮班的纪律。harness 的工作不是让模型变聪明,是让一群无记忆的、轮班的、各自只看到局部的 agent,对外表现得像一个有连续意志的工程师团队。

落到工程上,这种纪律不是靠 prompt 灌进去的,是靠 governance 框架定下来的。governance 由三个变量决定:**权限边界**(允许动什么)、**可逆性默认**(动错了能撤吗)、**用户介入频率**(动之前用户管不管)。它们不是三条独立的清单,是一个 trade-off 三角。三者一起决定 governance 给整套交互施加的摩擦总量 — 我把这个关系暂时建模成乘积,任何一个收紧,另外两个就有放松空间;任何一个放开,另外两个必须补回去。具体的函数形式(乘积、加权求和、还是某种非线性组合)是开放问题,本文当一阶近似用。

这个三角解释了为什么 chat / Code / Cowork 三种 governance 都是合理的,即使三种看上去互相矛盾。

chat 走"权限收死 + 全可逆 + 高频介入"。模型本质上什么都改不了,介入频率高一点也无所谓 — 用户本来就在每轮发言。

Code 走"权限宽 + git 兜底 + 低介入"。因为 git rollback 这种工程级 backstop 存在,可逆性极强,所以可以放低介入频率,让 agent 异步跑长任务。

Cowork 走的是另一头:"权限授权式 + 多数不可逆 + 高频同步阻断"。用户文件夹和第三方连接器的写操作没法 rollback,可逆性低,只能用高频确认补回去。如果哪个 Cowork 流程跳过了同步阻断、又没办法事后撤销,这个 governance 就是漏的。

三家三种 trade-off,底层公式一样:可逆性低,就必须用更窄的权限或更频繁的介入补;权限放宽,就必须用更强的可逆性或更频繁的介入补。任何一个 harness 设计,如果三角里少一个角、又没有别的角补偿,就是漏的。

这个三角的边界条件值得标出来。chat / Code / Cowork 三种 governance 都在软件 harness 场景里,可逆性虽不同,但都有不同层级的工程级 backstop(context buffer / git / 用户授权确认)。当场景外推到非代码 — 医疗 LLM 开处方、金融 agent 下交易、自动驾驶 — 操作本身物理不可逆,而且终端用户没能力做关键决策同步阻断(普通病人没有医学专业判断)。这时三角里有一个角(可逆性)被物理世界锁死,另外两个角能不能补回来,本文没有答案。我目前的判断是:跨出软件 governance 边界后,需要的不是更紧的 harness,是 institutional governance — 人类组织流程(医师审核、合规审计、伦理委员会)接管 governance 的位置。这超出本文范围。

Harness 工程不是给模型加 prompt,是为这三个变量找一组互相补偿的默认配比。

换个公式说法:**Model + Harness = Agent**。Prompt 工程在 model 边界内调参,harness 工程是把 model 升级成 agent 的工程层。

---

## 二、真问题是漂移,不是出错

把 agent 当会犯错的开发者,会习惯性地加重试、加测试、加 review。这套思路在 harness 里只解决了一部分问题。

真正吃掉时间的不是错误,是漂移。三类:

**已验证状态被悄悄改回去。** Agent 改了 A 模块,测试通过。下一轮改 B,B 的实现里回滚了 A 的某个边界条件,因为它"看着不对"。下一次测试只跑 B 的部分,A 的退化不会被发现,直到一周后用户报 bug。

解法是把"已验证"做成不可逆状态。features.json 里的 passes 字段只允许 false → true 单向流动,任何反向修改被外层校验拒绝。这是个极小的工程约束,但把这一类漂移从可能变成不可能。Anthropic 那句话更狠:"It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality"。

**agent 给自己的工作打高分。** Anthropic 另一篇关于多 agent harness 的文章里有一句话直接得很:"When asked to evaluate work they've produced, agents tend to respond by confidently praising the work—even when, to a human observer, the quality is obviously mediocre"。我自己验证过若干次。

解法是 GAN-inspired 的生成 agent 和评估 agent 分离。两个 agent 在不同 context、用不同 prompt 跑,最好用不同模型跑。SWE-bench 早就是行业最佳实践,但工程实践里大量项目还在让同一个 agent 自评。

**换 session 之后丢上下文。** 经典的轮班交接问题。新 session 启动没人告诉它前 50 轮发生了什么,会自信地重新发明已经被否决过的方案。

解法是把状态写进磁盘。决策记录、未完成 TODO、已踩过的坑全部写到文件,每次 session 启动先读。Codex harness 里是 progress 文件,Claude Code 里是 CLAUDE.md 加 memory 索引。机制同构:不信任 context window,信任磁盘。

三类漂移同一个根:agent 没有轮班工程师那种严肃的交接纪律。harness 工程的第一性工作,是把交接纪律外化成磁盘上的协议,不是寄希望于 prompt 里加一句"请记得"。我最近做 harness 时,所有踩坑都属于这三类。改 prompt 一次都没救过任何一类。

---

## 三、多 agent 协作的真问题在元认知

generator/evaluator 解决了一类 agent 自评盲点,但只是一类。更深的问题是沉没成本。一个 agent 在一条路线上跑了 30 轮,每一轮局部优化都让它更确信这条路线对,无法从外部看到"路线本身可能选错"。这不是评估问题,是元认知问题。

多 agent 系统常见误用是把它当 N 个 agent 的算力加和。开几十个 agent 跑同一个问题、希望加和比单 agent 强,基本无效。它们共享同一个问题描述、同一套 context、同一个隐含假设,输出的不是几十个独立判断,是 1 个判断的几十个变体。

我做过一次决策 battle,开了几十个 agent。关键不是 agent 数量加和,是其中几个"反共识 generator"被强制生成完全不同的方向、互相不许参考。没有这几个 generator,剩下大部分 agent 会在我已选定的路线上反复优化细节,给我"我做了充分论证"的错觉。

另一次行业判断 battle,规模更大。关键不是工种角色 agent 的数量,是其中几个 META 元认知 agent。它们的任务不是给立场,是看完前面所有立场后,列出"我们这场讨论里被默认成立、没人质疑过的前提"。最有价值的产出不是任何一个 agent 的结论,是这几个 META 列出的一组未挑战前提。其中最尖锐的一条是:"benchmark 仍然有效"这件事可能本身就不成立 — 中国大陆用户体感 GLM-5.1 最强、Qwen 偏弱,benchmark 综合榜是 DeepSeek V4 第一。这个差异是否正在结构性扩大需要更大样本验证,但目前没人在主轴里讨论。

多 agent 协作的真正杠杆不在 N 的大小,在两件事:强制多视角生成、不许互相参照;元认知反身、一部分 agent 专门盘点假设。不带这两条,N 个 agent 加和跟自己想 N 次差不多。但要诚实说:META agent 本身也是手动设计的角色,它有自己的盲区。元认知的元认知谁来做,我没有答案 — 只能在每个 battle 之外靠人监督。这是这个方案最脆弱的一层。

我原本要去训练 RL。battle 之后才意识到我已经手里有一份独家失败数据。应该挖这个金矿,不是再开新矿。

---

## 四、模型升级,harness 不会消失

Anthropic 那篇文章结尾有一句话:"the space of interesting harness combinations doesn't shrink as models improve. Instead, it moves"。

这句话的推论比看上去更狠。它意味着 harness 工程不是给"模型暂时不够强"打的临时补丁。它是给"人和 agent 协作"打的长期接口层。

模型越强,能委派的任务越复杂,权限边界、可逆性、介入频率这三件事就越需要被认真设计。Claude chat 时代的 harness 是 prompt 模板。Claude Code 时代的 harness 是 features.json 加 git。Cowork 时代的 harness 是 MCP 权限协议加关键决策同步阻断。下一代 harness 长什么样我不确定,但确定它不是"消失",是"换形态"。

更现实的判断是:今天 harness 工程的默认值,未来几年可能沉淀为既定事实。git 在 2005 年也只是 Linux kernel 团队自己的工具。harness 工程现在处在"标准还没定"的窗口期,不是模型不够强。

2026 年 5 月这个判断有了产业级信号。阿里云发布千问云,把 480+ 模型加 60 多个云产品的能力全封装成 Skill 和 CLI,让 agent 不经过人类中介直接调用云资源。同期 Google I/O 把开发平台 Antigravity 升级成 agent-first 架构,官方文档直接用 "agent harness" 这个词描述 Gemini 执行真实任务的框架。两件事指向同一个拐点:agent 正在成为云基础设施的一等公民消费者。而 Google 用 "agent harness" 命名框架是个额外信号 — 这个词正在从学术讨论进入产业词汇表。

但云厂商解决的是"能不能调通",模型厂商解决的是"输出质量",中间这一层 — 这个 agent 该不该调、能花多少、调错了怎么回滚 — 没人管。这正是 governance。当 agent 能自主调云资源,governance 就从"可选的架构层"变成"大规模自主行动的前提"。可逆的(多花钱)可以 auto-execute,部分可逆的(数据外传)要 async 审批,不可逆的(删库、对外发布、签约)必须 sync 阻断。这不再是抽象分类,是 agent 调云资源时真实的成本-风险 trade-off。

云厂商造的是油门,不是刹车。卖调用量的人没有动力限制调用量。治理层判断的是 agent 该不该做,不只是能不能做,这一层必须独立于提供能力的一方。

---

## 五、本文可能错在哪里

1. base model 厂在 12 个月内把 harness 标配化。Claude Code 内嵌 features.json,Codex 内嵌 progress 文件,Cowork 内嵌 MCP 权限框架。已经在发生。独立 harness 方法学的可复用价值会降一档。
2. 开源社区做出 agent 协议标准。类似 HTTP 之于 web,governance 沉淀到协议层,harness 工程作为独立学科会被吸收。
3. AGI 5 年内出现。AGI 不需要外部 governance。整篇文章的前提作废。
4. 监管把 AI 工程纳入合规框架,harness 方法被官方化。沉淀窗口期消失。
5. 我自己的实战数据被证伪。我手里的失败信号识别如果重审之后发现是噪声,工程实践的零返工如果审计后不成立,本文的实证基础崩塌。

这五条我都不能 100% 排除。我做这个判断是基于现在能看到的迹象。

---

## 六、接下来(What comes next)

如果 harness 工程是一门正在被发明的学科,它的开放问题清单大概长这样:

- 多 agent 协作里,META 元认知角色的最优数量是多少。3 个够不够,更多会不会反而稀释。
- governance 默认基线怎么标准化。features.json 是一种 schema,MCP 是另一种协议,两者之间还有没有更基础的抽象。
- harness 工程跟 base model 厂的边界在哪里。哪些纪律必须内嵌到模型 RLHF 里,哪些必须留在外层 harness 里。
- agent 之间的"权限传递"协议怎么设计。父 agent 派子 agent 的时候,权限是同等继承、必须显式声明、还是默认收紧。
- 跨 session 交接的状态格式应该长什么样。纯文本够不够,要不要结构化 schema。
- 元认知 agent 自己也会犯错。元认知的元认知谁来做。

这是我目前能列出来的。每一条都没有共识答案。

---

## 经验(Lessons to carry forward)

Harness 工程不是 prompt 工程的扩展,是另一件事。Prompt 工程在模型边界内工作,假设模型理解了就会做对。Harness 工程在模型边界外工作,假设模型理解了之后还会漂移、还会自我夸大、还会忘记上一轮发生过什么,需要外部协议把这些行为约束住。

把这一切落到具体工程,核心要做的事不多:把模型该做的(orchestration)和工程要做的(governance)分清楚;把"已验证"做成不可逆;把"自评"换成第三方评;把"上下文"写到磁盘;在重大决策前强制多视角和元认知反身。

这一切的工作量,不会随着模型变强而减少。它会换形态。

---

## 我的判断(My conviction)

harness 工程的标准还没定。中文圈做这件事的人很少,工业级落地的更少。我手里最近的项目、PIVOT 决策、几次多 agent battle,提供了一些第一手数据。但我自己也清楚这些数据不够,需要被更多人复现、证伪、改写。

这篇文章应该被读作 working hypothesis,不是 validated conclusion。它的价值如果有,在框架本身能不能被拿去验证、被证伪、被改写,不在我个人经验是否覆盖了所有场景。

我打算把后续的工作做在公开 repo 上。

---

## References

- OpenAI · [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- Anthropic · [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Anthropic · [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)

---

<sub>**相关开源工作**:[NanoReasoner](https://github.com/toffee-desuwa/nanoreasoner) — 158x 压缩 KD pipeline,training 端工程实证 · [distillation-reward-audit](https://github.com/toffee-desuwa/distillation-reward-audit) — 双维度失败诊断工具(token-level gradient + sample-level pass@k),audit 端工程实证</sub>

<sub>Toffee · GitHub [@toffee-desuwa](https://github.com/toffee-desuwa) · 北京交通大学机电学院大二 · 2026-05-24</sub>
