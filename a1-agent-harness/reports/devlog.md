# A1 Agent Harness 开发日志（公开版）

> 去掉了本机路径、代理配置和第二个对比模型的接入细节；第二个模型统一写作"另一个模型"。不含答案代码。

作业仓库：https://github.com/cmu-agents/assignment-1 （commit 67498d8，chess_app 4aaabca）
本地代码：a1-agent-harness

## 环境决定

- 模型：DeepSeek-V4.1-Flash-1017，官方 API 模型名 `deepseek-flash`，OpenAI 兼容 base URL `https://api.deepseek.com`。
  作业默认 `deepseek/deepseek-v4-flash-0731`（课程给的代理商命名），我们换成官方直连。
- SDK：保持作业自带的 OpenAI SDK（chat.completions + tools）。`query_language_model` 和
  `compact_context` 里的调用段落是"不许改"的评分基础设施，trajectory 也按 OpenAI 消息格式记录，
  换 Anthropic SDK 等于改评分基础设施。

## 过程记录

### 2026-11-05 00:55 环境搭建
- `git clone --recurse-submodules` + `make setup` 一次成功（uv 0.9.12）。
- 坑 1：`make test` 报 `ImportError: Using SOCKS proxy, but the 'socksio' package is not installed`。
  原因：本机 shell 设了 `ALL_PROXY=socks5h://本机代理`，httpx 看到 socks 代理就要 socksio。
  这不是作业 TODO 的失败。处理：跑测试时 `env -u ALL_PROXY -u all_proxy`（HTTP(S)_PROXY 保留）。
- starter 基线：`11 failed, 4 passed, 4 deselected`，失败的全是学生 TODO，符合文档"starter 故意不过"。

### 2026-11-05 01:05 key 与模型名
- 坑 2：`make doctor` 报 401，但同一个 key 直接调 `GET /models` 返回 200。原因：shell 环境里已经有一个
  别处用的 `OPENAI_API_KEY`，`load_dotenv()` 默认不覆盖已有环境变量，`.env` 里的 key 根本没生效。
  处理：`assignments/run.sh` 包一层，`env -u OPENAI_API_KEY -u ALL_PROXY ...` 再跑命令。
- `.env` 里模型名改成官方 `deepseek-flash`（`deepseek-v4-flash-0731` 是课程代理商的名字）。
- 探测 deepseek-flash 的协议细节（3 次小请求）：
  - 支持 `reasoning_effort` 参数；assistant 消息带 `reasoning_content`。
  - 回传历史时带或不带 `reasoning_content` 都被接受 → 选择保留（作业说 prompt 里要有 reasoning）。
  - 不带 tools、`max_completion_tokens=1200` 的总结调用能正常返回 content（reasoning 只占几十 token）。

### 2026-11-05 01:10–01:40 离线实现（3 个 commit，+687/−118 行）
Part 1（commit 938eeed）
- 状态设计：`history` 只存 system/task 之后的交互；system 和 task 每次从属性重建 → 压缩永远动不到它们。
- 循环顺序：先查步数上限 → 再压缩 → 再请求。若先压缩，到上限那一步会多花一次总结调用，公开测试也会因多一次压缩失败。
- 纯文本回复：保留这条 assistant 消息，再追加一条 user 提示"你没调工具，什么都没发生"，避免原样重发同一个 prompt。
- `_linked_observations`：按位置（不是按 id）给每个 tool call 配一条 tool 消息，缺的补错误。
  provider 会重复用 `call_0`，按 id 全局匹配会配错；少一条 tool 消息 API 直接 400。
- `send_message` 之后同一回复里的其他调用一律不执行（skill 写了"提交后不能再做任何事"）。
- 没有 skill 时 prompt 不出现 patch.txt / 提交流程；有 skill 时只放 `name + description`。
Part 2（commit c91bf81）
- 按"步"切分：一步 = 一条 assistant + 它后面的 tool/user 消息，保证不把 tool_call 和结果拆开。
- 总结输入 = 任务 + 上一次的 working memory + 被删前缀的纯文本转写；多次压缩时记忆是滚动合并的。
- 总结为空时不删历史（宁可不压缩，也不能丢上下文）。
Part 3（commit dffbc6b）
- 一次回复里多个 `play_move`：只执行第一个成功的，其余返回 chess_error。`run_python` 也算"可能落子"。
- `run_python` 跑完不信任模型自己说的，重新 GET 一次棋盘并附在 observation 后面。
- 作业 TODO 3.1.b 的"missing or non-string fen"是从 simulate 抄过来的笔误，play_move 没有 fen。
- 坑 3：本地自测起 chess_server 报 `No module named 'chess'`。python-chess 只装在 Modal 测试镜像里，
  本地用 `uv run --with chess` 临时带上。

### 验证
- `make test`：15 passed（starter 是 11 failed）。
- 自写 `assignments/selfcheck.py`（不放进 tests/，作业规定不许改 tests）：起真实 chess_server，覆盖
  私有测试点名的情况——畸形参数、传输失败、并行落子、run_python 刷新棋盘、重复/坏 skill 等，41/41 通过。

## 2026-11-05 真跑

- Modal token 配好后 `modal token set` 报 `Could not connect to the Modal server`。原因：shell 里有 `HTTPS_PROXY=http://本机代理`，Modal 客户端的 gRPC 走这个代理连不上；直连 api.modal.com 是通的。`run.sh` 改为去掉全部代理变量。
- 第一次 `run-code-agent` 报 `VersionError: ... You are using '2026.10'`：新注册的 Modal 账号默认镜像构建器是 2026.10，作业锁定的 modal 客户端最高只认 2025.06。不升级依赖，在 `run.sh` 里设 `MODAL_IMAGE_BUILDER_VERSION=2025.06`。
- Part 1 第二次运行：沙箱启动 52.7s，agent 9 步完成（6 次 execute、1 次 invoke_skill、1 次 execute、send_message）。patch 只改 `game.py` 一处：`if self._board.outcome(claim_draw=True) is None:` 才让引擎回应。`check-part1`：FAIL_TO_PASS 1/1，PASS_TO_PASS 9/9，RESOLVED。
- Part 2：两次运行并行跑。开压缩 22 步，第 12、18 步各压缩一次（约 5,900 → 3,000 token）；不开压缩 16 步，prompt 从 1,701 涨到 9,752。两个 patch 完全一样，都是把 `isinstance(attr, Expression)` 改成 `hasattr(attr, 'resolve_expression')`，check-swebench 1/1 + 29/29。总 token 开压缩反而多约 5%（约 110k 对 105k），未命中缓存的输入约 20.3k 对 6.8k，是 3 倍：压缩改写了前缀，DeepSeek 的前缀缓存失效，总结调用本身也几乎全是冷输入。压缩后 agent 回头重读了 `query.py` 660-685 行和测试文件 1-30、190-225 行（第 13、15 步）。报告在 `artifacts/token-usage-analysis.md`。
- Part 3 python skill 那局：invoke_skill → 第 2 步直接 play_move e2e4 → 之后 11 次 run_python，每段代码 2-3 次 simulate_move + 1 次 play_move。第 13 步以和棋结束（status Draw）。注意 `assignment-play-chess` 不给 `--result` 时默认写 `artifacts/game-result.json`，会和主局抢同一个文件，已另存到 scratch。
- 对局速度：DeepSeek 每步约 30-40 秒（含推理），作业默认 `CHESS_TIMEOUT=1800` 秒的沙箱寿命大约只够 45-60 步，长局可能下不完。
- 观测实验 DeepSeek 无合法着法列表：70 步、70 次 play_move、0 次非法，白方（agent）在第 139 个半回合将杀获胜，约 30 分钟内下完，没撞上 1800s 沙箱超时。统计脚本 `assignments/obs_stats.py`。
- **两局撞上 Modal 沙箱 1800s 寿命**（08:48 左右，开局后约 30 分钟）：
  - 主局 `run-chess-agent`：前 45 步正常（44 次合法落子，第 8 步 1 次非法 e4d3），第 46 步起 chess server 断开，之后 125 次 play_move 全部返回 `Could not reach the chess server (RemoteProtocolError ...)`，中间 29 次模型只回文字（"I'm unable to continue the game because the chess server is repeatedly disconnecting"），一直耗到 200 步 StepLimitError。没写出 game-result.json（现在那个文件是 python skill 那局 08:19 写的）。
  - DeepSeek 带合法着法列表：前 31 步正常、0 次非法，第 32 步起 167 次连接错误，同样耗到 200 步。
  - 无列表那局 25 分钟下了 70 步，带列表那局 30 分钟只下了 31 步：每步 prompt 多了整张合法着法表，推理更慢。
  - harness 行为本身符合作业要求（网络错误转成可恢复的 observation，不崩），但没有"连续 N 次连接失败就停"的熔断，白白跑满 155 步。视频素材。
  - trajectory 体积：主局 175 MB、带列表那局 188 MB、无列表那局 23 MB，因为 `api_prompts` 每一步都存一份完整 prompt。超过 GitHub 单文件 100 MB 上限，不能直接 commit。
- **重跑（CHESS_TIMEOUT=3600）两局都下完了**：
  - 主局：52 次 play_move、0 次非法，103 个半回合白方胜，约 20 分钟。trajectory 11 MB。
  - DeepSeek 带合法着法列表：27 次 play_move、1 次非法（第 11 步 `d4d5`，d4 上是白马，模型当成兵往前推，而这一步根本不在给它的列表里）、1 次瞬时网络错误（第 8 步，下一步重发同一手 f1c4 成功）。49 个半回合白方胜。
  - 两种观测条件非法率都很低（0/70 对 1/27），样本各一局，不能说明列表有用；唯一的非法反而出在带列表那局。
  - 带列表每步 completion 约 4,400 token，不带约 3,400；prompt 每步多涨约 4.7k 对 3.7k。不带列表那局 70 步后 prompt 到 257k token，总计 7.6M prompt token：下棋不压缩，上下文一直涨。
  - 第一轮主局 30 分钟只走了 45 步、这轮 20 分钟走完 52 步，差别来自对局本身（引擎是确定性的，不同的白方着法导致不同棋局）。
  - `obs_stats.py` 第一版把网络错误也算成非法，已改成分开统计。
  - 报告：`artifacts/observation-experiment.md`。