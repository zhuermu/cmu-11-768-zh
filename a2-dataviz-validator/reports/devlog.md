# 作业 2（Data-Visualization Validator）开发日志

仓库：cmu-agents/assignment-2 @ f609e76（2026-09-20），本地 `assignments/a2-dataviz-eval`。

## 2026-11-07 决策

- 作业 3（RL 训练，全 H100，估 $180-250）不做；作业 2 走 Modal 免费额度（方案 A），每阶段查 `modal billing report`，累计 $25 停。
- 视频里不说"抢不到 GPU"：真实原因是没有 CMU 发给选课学生的额度、自费成本高。
- 开工前 Modal 本月用量：$0.05（作业 1 的 swe-rex 沙箱）。

## 坑

1. **镜像构建器版本**（和作业 1 同一个坑）：新账号默认 2026.10，作业锁定的 modal 1.5.5 只支持到 2025.06。
   处理：`MODAL_IMAGE_BUILDER_VERSION=2025.06`，不升级依赖。
2. **代理**：照旧 unset `ALL_PROXY/HTTPS_PROXY/HTTP_PROXY` 再跑 modal。
3. **免费账户不能用 L40S**：镜像构建成功（178 个包，vllm/torch），部署时报
   `Please add a payment method to use L40S GPU functions.`
   免费额度 $30 还在，但 Modal 要求先绑支付方式才能起 GPU 函数。
4. **两种 token 不是一回事**：作业 1 里 `modal token set` 配的是 CLI 的账户 token（ak-/as-），用来部署；调 endpoint 要另一种 Proxy Auth Token（wk-/ws-），在 Modal 后台 Settings 里建，
   写进 `.env` 的 `VALIDATOR_API_KEY`，客户端当 OpenAI api_key 发出去。

## 2026-11-07 22:00-22:45 换掉 Modal：实验机 + g6e + Bedrock

- Modal 绑卡失败（L40S 要求有支付方式，免费额度也不行）。先试了我自己的模型代理：OpenAI 兼容接口全部 401，Anthropic 接口只有 Claude 能用，代理里没有 Qwen。放弃。
- 最终架构（讲师定）：
  - 实验机：一台 ARM 云主机（ARM64 8 核 30 GB，Docker），作业提交 f609e76。
  - 裁判：g6e.xlarge（1x L40S 46 GB，$2.70/h），vLLM 0.13.0 原样部署 Qwen/Qwen3-VL-30B-A3B-Instruct-FP8，参数照抄 `infrastructure/validator_model.py`。无公网 vLLM 访问，安全组只放行实验机访问 8000。空闲 60 分钟自动关机（shutdown 行为 = stop，权重保留在盘上）。开机脚本 `assignments/a2-infra/g6e-user-data.sh`。
  - agent：Bedrock。Qwen2.5-Coder-3B -> `mistral.ministral-3-3b-instruct`（体量最接近的弱模型）；Ministral-3-14B -> `mistral.ministral-3-14b-instruct`（同一模型）；GLM-4.7-Flash -> `zai.glm-4.7-flash`（同一模型）。三个都在 bedrock-mantle 和 bedrock-runtime/openai/v1 两个 OpenAI 兼容端点上测过工具调用，正常。
  - Bedrock key：方案 A，用 SSO 会话生成短期 key（最长 12 小时），通过管道直接写到实验机上的 key 文件（权限 600），不落本机、不打印。
- 坑：实验机的 ssh 配置带 `RemoteCommand tmux new-session -A -s main`，非交互执行命令要加 `-o RemoteCommand=none -o RequestTTY=no`，否则报 `Cannot execute command-line and remote command`。
- 坑：g6e 的 key pair 和实验机用的不是同一把，用 `ec2-instance-connect send-ssh-public-key` 推临时公钥再走 EIC 隧道。
- 观察：Bedrock 上 qwen3-vl-235b 看一张作业里的图要约 1.2 万 token，ministral-3-14b 只要约 2 千，不同模型的图像 token 计算差别很大。

## 2026-11-07 22:55-23:15 Part 1 baseline（原版裁判，g6e）

- vLLM 冷启动：装 vLLM 约 2 分钟，下载 30 GB 权重 216 秒，从 gp3 读进显存约 6 分钟（gp3 默认 125 MB/s 是瓶颈），开机到 /health 200 共约 20 分钟。
- 第一次看图请求 15.9 秒，作业里那张鱼骨图的 prompt 是 12,579 token。
- baseline 跑 110 个种子 run 用了约 15 分钟（平均约 8 秒/个），日志里没有 warn/error/retry，没有触发上下文截断。
- 结果（`workflow score`）：
  - execution_failure MCC=-0.0022（TP4 TN65 FP33 FN8）
  - wrong_data MCC=0.0000（TP0 FN25）
  - wrong_chart MCC=0.1096（TP3 FN68）
  - hard_to_read MCC=0.0000（TP0 FN39）
  - macro-MCC = 0.0268，基本等于瞎猜。

## 2026-11-07 23:05-23:20 Part 1 误判分析

- 23:07 讲师问后面还用不用 GPU：Part 1 分析不用，所以主动 stop 了 g6e（跑了约 38 分钟，约 $1.7）。磁盘和权重保留，Part 2 跑新 validator 时再开。
- 误判类型 1：execution_failure 的 33 个误报全部有 figure.png，32 个模型自己写的理由在说 agent 成功了，开头却是 YES（问题带否定，正反答反了）。validate() 判出 execution_failure 后提前返回，盖住了 25 个真实标签。8 个漏判全部没有 figure.png。种子集上"没图 ⇔ execution_failure"MCC=1.000。
- 误判类型 2：65 个"判没问题但其实有错"，漏了很多写死在任务说明里的参数（30 度刻度旋转 009/038，120 dpi 060/130，禁用 jet 051/106/130，缺图例 016/028）。
- 详细见 `a2-dataviz-eval/notes/part1-error-analysis.md`。

## 【汇总】我们的实验环境和原作业不一样的地方（做视频时要逐条说明）

| 项目 | 原作业 | 我们 | 原因 / 影响 |
|---|---|---|---|
| 算力平台 | Modal（CMU 发 credit） | AWS：一台 ARM 云主机当实验机，g6e.xlarge 跑裁判 | Modal 绑不上卡；L40S 要求有支付方式，免费额度也用不了 |
| 裁判模型 | Modal L40S 上的 Qwen3-VL-30B-A3B-FP8，vLLM 0.13.0 | **同一个模型、同一个 vLLM 版本、同样的参数**，部署在 g6e 的 L40S 上 | 无差别。只是端点不开鉴权，改用安全组限制访问 |
| Agent 1 | Qwen2.5-Coder-3B（L4 自部署） | Bedrock `mistral.ministral-3-3b-instruct` | Bedrock 上没有 Qwen2.5-Coder-3B；选体量最接近的弱模型，作用仍是"最弱的那个 agent" |
| Agent 2 | Ministral-3-14B-2512（L40S 自部署） | Bedrock `mistral.ministral-3-14b-instruct` | 同一个模型，换成托管推理，量化和推理栈可能不同 |
| Agent 3 | GLM-4.7-Flash（H100 自部署） | Bedrock `zai.glm-4.7-flash` | 同上 |
| agent 运行方式 | Modal 容器里跑 mini-swe-agent 2.4.5，模型调用经 Modal gateway | 实验机上的 Docker 容器，依赖版本照抄 `infrastructure/agent_runner.py`；模型调用经本机 gateway 转发到 Bedrock，key 不进容器 | Modal 不可用 |
| 提交检查 | `check-submission` 要求 agent 的 key 和 model 都是那三个固定模型 | model 名不同，这一条会过不了 | 我们不交作业，可以接受 |
| 私有测试集 | 30% 分数看私有集 macro-MCC | 拿不到，只报种子集和自建集 | |
| 机器架构 | Modal x86_64 | 实验机是 ARM64（Graviton） | 重放 agent 代码的 Docker 镜像是 arm64 版，Python 包版本相同 |

## 2026-11-07 23:15-23:45 Part 2 实现

- 改进 1（确定性 execution）：没图、PNG 打不开、图太小、整张图一个颜色，都判 execution_failure 并直接返回，因为 ground truth 里这一类是终结性的（ASSIGNMENT："execution_failure is terminal"，其他三类在这些 run 上不计分）。有图时其他三类一律都查。
- 改进 2（重放 + 测量）：把 trajectory 里 agent 发的每条 bash 命令按顺序在 Docker 沙箱里重放（`--network none`、2 GB 内存、非 root），`PYTHONPATH` 指向一个 `sitecustomize.py`，劫持 `Figure.savefig`，用 harbor 示例自带的 `figure_manifest.py` 把 artist 树存成 JSON。另外补记了原 manifest 没有的字段：savefig 实际用的 dpi、刻度旋转角度、spine 是否可见、每段文字的像素框和颜色。
  - 规则检查：dpi、英寸尺寸、禁用 colormap、刻度旋转、图例/网格/色条是否存在、y 轴从 0 开始、横竖柱、3D、极坐标、隐藏 spine。
  - 模型检查：不给图，只给"请求 + 测量出的事实 JSON"，让裁判逐条要求输出 met/violated/unknown，只收 violated。
  - Part 2 的 readability 保持 baseline 原样，免得说不清提升来自哪一项。
- 离线探针（`notes/offline_probe.py`，不调模型，1 分 45 秒）：98 个有图的 run 重放成功 97 个，重放出的图尺寸和原 figure.png 全部一致。只用规则检查，wrong_chart TP23 FP0 FN48，MCC 0.341（baseline 0.110）；wrong_data 规则一条都没有，TP0。
- 坑：
  - **dpi 要看 savefig 的参数**。`fig.get_dpi()` 是画布 dpi，`savefig(dpi=120)` 会覆盖它；harbor 自带的 manifest 只记了前者，要在 hook 里另记 savefig 的 kwargs。
  - **agent 会用绝对路径**。run 173 里写的是 `cd /tmp/agent-eval-3s2zmpic && python plot_script.py`，重放时把工作目录同时挂载到这个原始路径才跑通。
  - **重放不能联网**。run 191 中途 `pip install kaleido`，沙箱断网装不上，这一个重放失败，回退到看图判断。安全上宁可失败也不放开网络。
  - **重放的每条命令都要单独开 shell**。mini-swe-agent 每条命令都在新 shell 里执行，命令之间只有文件留存，`cd` 不留存，重放要照这个方式。
  - 不能在宿主机上直接执行 agent 写的代码，所以没有 Docker 时 validator 直接放弃重放，不退回本机执行。

## 2026-11-07 23:45 - 11-08 01:00 Part 3、Part 4、报告

### 分数总表（同一个原版裁判）
| | baseline | Part 2 | Part 4 | Part 4 + 刻度角色修复 |
|---|---:|---:|---:|---:|
| 种子集 macro-MCC | 0.027 | 0.332 | 0.500 | **0.511** |
| 自建集 macro-MCC | - | -0.064 | 0.398 | **0.435** |
- 种子集 hard_to_read：0.000 -> -0.242 -> 0.481 -> 0.523（最终 15 TP、0 FP）。
- 回退：种子集 wrong_data 从 0.157 掉到 0.041。原因是 Part 4 让裁判"不要自己算数"，误报少了，命中也跟着少了。
- 最后一列只替换了 hard_to_read 的判断。这一项是纯规则、不调模型，离线重算后合成，报告里写明了。

### Part 3
- 两类弱点：derived-aggregation（筛选、排序、累计、滚动窗口）和 dense-text-layout（长标签、密集标注）。共 5 道题，数据用固定随机种子合成。
- 15 个 run（5 题 x 3 agent）全部出图。Bedrock 用量：3 个模型共 130 次调用，约 33 万输入 token、1.5 万输出 token，费用不到 $0.1。
- 有意思的发现（适合做视频）：
  - **三个 agent 犯同一个错**：营收占比题要求"最大的在最上面"，三个都按降序排，但 `barh` 的第一根柱子画在最底下，结果最大的跑到了最下面，数值本身全对。
  - **min_periods=1**：14B 和 GLM 的 7 日均线从第 1 天就开始画，题目要求前 6 行没有值。
  - **3B 模型的空图**：累计面积图把列赋值到错位的 index 上，全变成 NaN，画出来只剩图例，还顺手 `invert_yaxis()`，注释写"Stacking is inverted in matplotlib"，这是它自己编的说法。
  - **标签重叠**：三个散点图全都有城市名叠在一起。
- 迭代前 validator 在自建集上是 -0.064：13/15 个 run 被误判 wrong_chart。原因是 validator 自己的问题：facts JSON 把标注截断到 10 条，裁判就报"只有 10 个标注"；灰色散点的 colormap 字段是默认的 viridis，裁判就报"点不是灰色"。
- 标注是我（AI）写的初稿，依据是看图、读最终脚本、用代码从输入重算的正确值，**待讲师复核**。

### Harbor
- 示例：oracle 1、nop 0、illegible 变体 1，和说明一致。5 道自建题 oracle 全部 1.0，nop 为 0。
- 两个变体都取自 agent 真实犯的错：
  - `largest_at_bottom`（营收占比题，最大的在底部）：reward 0，被 `test_s2_largest_share_at_the_top` 拦下。
  - `label_pileup`（散点图，城市名重叠）：reward 1.0。数值、标签、尺寸全对，但没有任何断言检查文字是否互相覆盖。这正好说明确定性 verifier 的边界，也就是为什么还需要 VLM 裁判。
- `check-submission` 唯一不过的一条是 agent 的 model 名不是那三个固定模型（我们换成了 Bedrock）。在临时副本里把 model 名改回去再跑，输出 `submission valid`，其余各项包括 Harbor 实跑全部通过。

### 坑
- **Amazon Linux 的 Docker 没有 compose v2**，Harbor 报 `unknown flag: --project-name`。在 `~/.docker/cli-plugins` 装了官方 compose v2.40.3（校验 sha256，不用 sudo）。之后又报 `compose build requires buildx 0.17 or later`，同样方法装了 buildx v0.29.1。
- **Matplotlib 3.10 里刻度标签的 `.axes` 是 None**，用 `t.axes` 判断角色会把所有刻度标签当成普通文字，导致 lyon__qwen 那 86 个重叠的日期刻度漏判。改成从每个 axes 的 `get_xticklabels()` 反查。
- **旋转刻度标签的外接框会互相重叠**，45 度的长标签即使字形没碰到，它们的轴对齐外接框也会重叠。改为用刻度锚点间距乘以 sin(角度)，再和字高比较。
- **3D 坐标轴每个刻度标签会画两次**，投影后的文字框不能直接比较，所以 3D 坐标轴排除在测量检查之外。
- **数字刻度不算难读**：种子集的人工标签从来不把"数字刻度贴边/轻微重叠"算成 hard_to_read。不排除的话，种子集上会多出 20 个误报。
- **heredoc 嵌套**：ssh 的 heredoc 里再嵌 python 的 heredoc，引号冲突，改成在本机写好文件再 scp 上去。
- **空目录导致变体复制错位**：第一次失败留下空的 `mutants/<name>`，`cp -r` 就把文件复制到了下一层。那次跑出来的两个 1.0 其实跑的是参考解，差点当成结果。重跑前确认了 solve.sh 里有 MUTANT 字样。

### 费用
- g6e.xlarge 共开机约 2 小时 6 分钟（14:29-15:07、15:18-16:46 UTC），按 $2.70/h 计约 **$5.7**。已 stop，200 GB 盘每天约 $0.6，直到删除为止。
- Bedrock：探测和 agent 调用加起来不到 $0.1。
- 实验机是原有机器，不单独计费。
