# EatingAgent

Minimal runnable source distribution. Requires Python 3.11 or newer; verified on Windows with Python 3.12.7 in `.venv`.

## Setup (PowerShell)

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Fill in `.env` with your own DEEPSEEK_API_KEY, AMAP_WEBSERVICE_KEY, API_KEY, and CONSENT_GRANT_ID. The last two values authenticate the Clawdot takeout gateway and authorize the user account. Retain or adjust the endpoint and model settings as appropriate for your accounts.

## Run

```powershell
.\.venv\Scripts\python.exe agent.py
```

Type `exit` to stop. Optional arguments are listed by `python agent.py --help`. Voice input uses `--voice` and requires Windows PowerShell, a microphone, and the corresponding Windows speech recognition language.

The program generates `agent_config.json` and `agent_saved_addresses.json` as needed. When no configuration exists, the source defaults to HKUST (Guangzhou) as the initial location. You can change the delivery location in the conversation or configure `default_address_text` in `agent_config.json` before first use.

This package contains the entry point, implementation modules, a blank environment template, the sole third-party dependency declaration, and these setup instructions. External APIs and valid account credentials are required for live use.

## 离线世界模式（PowerShell）

公开仓库不包含体积约 687 MB 的 `evaluation/delivery_snapshot/worlds.jsonl` 及排除快照；请按 `evaluation/delivery_snapshot/README.md` 使用你自己的凭据采集后再运行本节命令。冻结任务和历史评测结果仅用于复现实验结构与审计，不代替原始世界快照。

```powershell
cd D:\projects\EatingAgent
.\.venv\Scripts\python.exe agent.py --world world_001
```

`--world` 按编号读取 `evaluation/delivery_snapshot/worlds.jsonl`，加载时提取精简视图：地点、商家、完整商品与规格，以及少量版本、时间和覆盖信息。运行视图不包含 `raw_calls`、图片、logo、定位候选或采集日志；原始快照继续保留用于追溯。不额外生成精简数据文件。

世界数保持不变；每个世界按快照顺序最多保留 100 家具有有效对应菜单的商家，保留入选商家的全部商品和规格。当前 49 个正式世界各有 40–49 家有效商家，因此该上限不会裁剪当前数据。

配送地点和坐标固定为所选世界，跳过高德查询及个人配置、地址文件的读写。对话中提到其他地点时仍使用当前世界；换世界需要重新运行命令并调整 `--world`。

商家和菜单离线读取，需求理解和对话仍调用 DeepSeek，因此需要 `.env` 中的 `DEEPSEEK_API_KEY` 与模型配置。该模式无需高德 Key、Clawdot API Key 或授权 ID。

关键词搜索在已采集商家名称、品牌、商品名称、描述和分类中进行文字包含匹配，范围限于快照；不复现平台实时搜索排序。菜单摘要、分类查询、单品详情、SKU/属性转换与分→元转换复用现有 `TakeoutAdapter`。

离线模式呈现商品、规格、数量、世界地点和已知金额，并明确未知费用。订单保存在当前会话的 `state.presented_order`，呈现后设置 `state.order_presented`。手动对话可继续修改或加购同店商品；不调用地址、平台订单预览、下单或订单状态接口。

`world_024` 已从正式世界列表排除，其原始记录保留在 `evaluation/delivery_snapshot/excluded/world_024.jsonl`；正式世界编号保留缺口。

验证命令：

```powershell
.\.venv\Scripts\python.exe -X utf8 -m unittest evaluation.delivery_snapshot.test_offline -v
```

## 偏好评测（PowerShell）

当前流程：读取模拟用户 → 生成一次合成偏好 → 按历史品类次数抽样并生成一次需求 → 冻结任务 → 同一任务执行四次独立会话 → 记录评分和 CSV 指标。

实际环境：`D:\projects\EatingAgent\.venv\Scripts\python.exe`，Python 3.12.7。程序结构见 [evaluation/README.md](evaluation/README.md)。

先做只读校验和离线回归，不调用外部模型：

```powershell
cd D:\projects\EatingAgent
.\.venv\Scripts\python.exe -X utf8 -B -m evaluation.dataset check
.\.venv\Scripts\python.exe -X utf8 -B -m unittest evaluation.test_agent evaluation.test_run evaluation.test_rubric evaluation.ablation_user_preferences.test_run evaluation.delivery_snapshot.test_offline -q
```

准备一个新任务（会调用 DeepSeek Flash）：

```powershell
.\.venv\Scripts\python.exe -X utf8 -B -m evaluation.dataset prepare --pair-index 0 --seed 20261015 --output evaluation/user_preferences/user_045_new_task.json
```

`--pair-index` 是开发集用户与世界的配对下标，从 0 开始。默认根据模拟用户档案生成合成偏好，再生成一次初始需求，通常共两次逻辑模型调用；加 `--preference evaluation/user_preferences/user_045.json` 可复用已确认偏好，仅生成需求。模型请求可能因可恢复服务错误重试一次。seed 只固定品类抽样，模型输出以冻结文件为准。

冻结文件包含用户、世界、偏好、需求及准备记录；已有文件会拒绝覆盖。合成历史中的次数、价位、商家与商品不表示真实消费或当前可售菜单。

运行冻结任务（会调用 DeepSeek Flash）：

```powershell
.\.venv\Scripts\python.exe -X utf8 -B -m evaluation.run --tasks evaluation/user_preferences/user_045_new_task.json --limit 1
```

当前评测中的偏好生成、需求生成、Agent、模拟用户和评分器统一使用 Flash 模型；实际模型名称写入 `run.json`。

已有任务可直接执行，例如 `--tasks evaluation/user_preferences/user_044_task.json`。执行器支持冻结 JSON 或 JSONL；每个任务始终运行四次，复用需求、偏好与世界快照，每次创建独立 Agent 状态，不更新偏好。默认结果写到 `results/preference-时间戳/`，通过 `--stage baseline` 或 `--stage calibration` 可单独记录对照或校准。

每次基础上限为 6 轮、600 秒；到达轮次边界时，模拟用户若仍能产生合法下一步，则增加 2 轮，整个会话最多 10 轮。首次 `order_presented` 后停止，不进行真实下单。单次请求默认总时限 45 秒；超时不重试，可恢复连接或 HTTP 错误最多重试一次。模型服务失败与评分格式错误单独记录原因。Time Cost 包含 Agent 和模拟用户交互，不包含准备、世界加载、初始化及事后评分。

结果只有三个文件：

- `run.json`：冻结任务、模型与源码版本、运行配置。
- `trials.jsonl`：每次交互、结构化订单、评分、请求统计与错误证据。
- `summary.csv`：四次成功/耗时/轮数，以及 `Avg@4`、`Pass@4`、`Pass^4` 和平均耗时、轮数。

评分器 v6 按最新有效需求与订单分别给品类、数量、送达时限、适用忌口四项 0/1 分，程序相乘得到 success。历史只提供品类名称参考；未提时限记 1，提出时限但缺少预计时间记无法评分，该任务 @4 留空。轨迹保存分数与理由，CSV 保存分项分数和状态。详见 [评分约定](evaluation/grading.md)。

偏好注入与排序沿用已有机制：每次 Agent 模型调用携带完整历史偏好，本轮明确要求优先；历史价位只辅助排序，不转为预算。模拟用户选第一个符合需求的候选，规格先遵循本轮要求，再参考历史倾向。

Agent 先结构化提取 `order_type`、`category`、分类型 `search_terms` 与 `constraints`。缺类型先问饮品还是食品，缺品类再问奶茶、面食、米饭简餐、烘焙甜点等，补齐前不搜索。检索仅使用当前类型的正向词，食品和饮品的要求分别处理；历史偏好不替用户补出目标。候选审核另返回 food/drink 类型，程序排除类型不符的候选。类型标签的语义仍由 LLM 判断。

模拟用户版本为 `clarify-and-preserve-quantity-v6.2-no-candidate-replan`：支持回答具体品类，并在重申需求时保留已澄清品类；未要求多份时保持一份，起送不足请求换店，程序拒绝额外数量动作。Agent 继续核验起送额、最低购买量和库存，允许用户自主明确改数量，不自动凑单。换店后排除当前已选店，选规格期间同样支持换店或主动改目标。助手明确表示没有符合候选且任务含历史偏好时，模拟用户只允许根据历史偏好生成一条不同的新需求；该需求只存在于当前对话，不写回冻结任务。候选是否符合由模拟用户 LLM 判断；程序不会因重试而强制代选。程序按当前阶段限制允许动作，连续两次相同无效动作立即退出。无历史偏好且没有更多候选时结束会话，重复相同回复时执行器以 `repeated_interaction` 退出。第7～11对续跑结果见[续跑报告](evaluation/results/continuation-20261016-080700/report.md)。

新偏好生成只使用实际可检索品类，“原味、单品、椒香”等不能作为有历史次数的分类键；需求生成按当前食品/饮品类型表达相关要求。原有冻结任务与历史结果不重新生成，旧任务若含糊，会走新增澄清流程。9 次 Flash 定点校准有 8 项符合预期，暴露了凑起送多份仍被判成功的缺陷；当前 v5.1 已拆分主体与数量判定，新增真实验证仍待执行。历史 7/9 保持原样，详见 [校准结果与限制](evaluation/grading.md#本次修改的验证状态)。

规格录入只让模型提取 `notes` 与 `quantity_by_option_id`；规格匹配仅传商品名、要求和“编号/组名/选项名”，模型返回编号，程序还原真实选项对象。包装规格与购买份数分开，例如“8个装，买两份”对应规格 8个、数量 2。真实 ID、价格及合法性校验仍在程序内部保留。

原始用户、世界、固定切分及历史结果保持独立；新评测不会覆盖历史 baseline，不将准备记录、训练输出或人类反馈混入四次评测轨迹。执行真实评测时，模拟用户档案和偏好用于准备，请求、对话及相关菜单候选发送给 DeepSeek；整轮调用次数取决于交互，不是四次 API 请求。
