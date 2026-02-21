# Currency Tracker UI 设计（初版草案）

本设计文档定义 Accountant Classic 的 Currency Tracker 独立窗口 UI。目标是在不影响现有 Gold Tracker 的前提下，为货币追踪提供与原金钱页面一致的外观与交互，同时复用 CLI 逻辑，保证“预览/展示”和“UI 渲染”使用相同的数据来源与计算路径。

## 目标与原则
- 与 Gold Tracker 的窗口风格一致：标题栏、关闭按钮、边框、表格风格、底部时间范围切换条。
- 入口采用新命令：`/ct ui` 打开/聚焦/显示 Currency UI 独立窗口（后续也会在主面板中加入入口）。
- 由于 Currency Tracker 支持多种货币：在“服务器-角色”下拉框右侧新增“货币下拉框”。
- 默认打开时展示“所有货币”的汇总（等价于 CLI: `ct show-all this-session`）。
- 底部的时间范围切换与 Gold UI 相同：切换后展示相同 timeframe 的“所有货币汇总”；当选择了某个货币后，展示 `ct show <timeframe> <currencyId>` 的结果。
- 严格遵循“预览逻辑与应用逻辑共享”的规则：UI 仅渲染数据，不重新实现计算/查询。
- 不允许修改任何原有 `/ct` 命令使用到的实现函数；新增 UI 仅负责渲染与交互。凡涉及存储写入/状态变更（如追踪开关），一律委托现有 CLI 处理函数；展示数据读取走 `DataManager`/`Storage` 的只读接口。

## 窗口结构
- 标题：`Accountant Classic`（保持一致），副标题区域可显示当前 timeframe。

- 顶部栏：
  - 左侧头像与总览：显示文本总收入/总支出/净收益的本地化字符串。
  - 右侧：
    - 服务器下拉：列出有数据的服务器；
    - 角色下拉：根据服务器筛选角色；
    - 货币下拉：显示“已发现（discover）”的货币本地名称（`Storage:GetDiscoveredCurrencies()`）。默认项：`全部货币`；选择具体货币后为“单货币模式”。

- 中部表格区域：
  - 当“全部货币”时：
    - 每一行一条货币（本地名），列为：`货币ICON | 收入 | 支出 | 净额 | 总上限`，对应 CLI 的 `ct show-all <timeframe>` 输出（复用数据路径）。
    - 每一行最左边设计一个checkbox，用于设置“追踪/不追踪”状态。
  - 当“单货币模式”时：
    - 表格呈现 `ct show <timeframe> <currencyId>` 的结果：顶部显示汇总（收入/支出/净额/总上限（TotalMax））
    - 表格内容为“来源拆分”，列：`来源 | 收入 | 支出 | 净额`。

- 底部按钮条（时间范围切换）：
  - `本次/今天/昨天/本周/上周/本月/上月/今年/去年/总计`（与 Gold UI 文案一致，具体映射见下节）。
  - 右侧可保留“重置/选项/退出”等按钮占位（后续迭代）。

## 时间范围映射
UI 按钮与内部 timeframe 映射：
- 本次 -> `Session`（等价 CLI: `this-session`）
- 今天 -> `Day`（`today`）
- 昨天（Yesterday） -> `PrvDay`（`prv-day`）
- 本周 -> `Week`（`this-week`）
- 上周 -> `PrvWeek`（`prv-week`）
- 本月 -> `Month`（`this-month`）
- 上月 -> `PrvMonth`（`prv-month`）
- 今年 -> `Year`（`this-year`）
- 去年 -> `PrvYear`（`prv-year`）
- 总计 -> `Total`（`total`）

默认打开：`Session`。

## 页面逻辑与状态机
- 打开 UI：默认 `全部货币 + Session`，相当于执行 `ct show-all this-session` 并将结果渲染为表格。
- 切换 timeframe：
  - 处于“全部货币”时 -> 调用 `ct show-all <timeframe>` 数据路径。
  - 处于“单货币模式”时 -> 调用 `ct show <timeframe> <currencyId>` 数据路径。
- 选择货币：
  - 切换为“单货币模式”，即刻刷新为 `ct show <current_timeframe> <currencyId>` 的表格。
- 选择“全部货币”：
  - 退回“全部货币模式”，刷新为 `ct show-all <current_timeframe>`。
- 状态持久化（后续）：
  - 最近一次 timeframe、服务器、角色、货币选择可写入 `currencyOptions`，供下次打开时恢复。

## 数据来源与复用（不修改既有 CLI 函数）
- 不对任何现有 CLI 函数签名或内部实现做修改（包括 `Print*` / `Show*` 系列）。
- 读写约定：
  - 涉及 Store 数据“写入/状态变更”的操作（例如：追踪开关），UI 通过直接调用现有 CLI 处理函数复用逻辑，例如：
    - `CurrencyTracker:DiscoverTrack(sub)`（由 UI 传入等价参数，行为与 `/ct discover track` 一致）。
  - 仅“读取/展示”的数据，UI 直接使用既有只读数据访问层：
    - `CurrencyTracker.Storage` 与 `CurrencyTracker.DataManager` 的现有查询接口（例如：`GetDiscoveredCurrencies()`、`GetAvailableCurrencies()`、`GetCurrencyData(id, timeframe)`、`GetCurrencyInfo(id)` 等）。
  - 如需对展示数据做轻量变换（例如字段重命名、拼装 UI 视图模型），在 UI 内部完成，不改变 CLI 的输出路径与行为。
- `CurrencyTracker:ParseShowCommand()` 继续仅服务于 CLI 场景（UI 不依赖它来构造内部状态）。
- 货币下拉内容：
  - 使用 `Storage:GetDiscoveredCurrencies()`；显示 `meta.name`（若有本地化则取本地化）；按 `name/id` 排序。
  - 过滤规则：默认仅显示 `tracked ~= false` 的货币；可在 UI 中增加“包含未追踪”开关（后续迭代）。

## 技术实现拆解
- 新建 `CurrencyTracker` 的 UI 模块（文件清单）：
  - `CurrencyTracker/CurrencyFrame.lua`（必须，控制逻辑：状态、交互、渲染绑定）
  - `CurrencyTracker/CurrencyFrame.xml`（可选但推荐，静态布局与模板：主 Frame、行模板、timeframe 按钮）
  - 对外方法：
    - `CurrencyTracker.CurrencyFrame` 提供：`Initialize() / Enable() / Disable() / Toggle() / Show()`；
    - 暴露给 Core：`CurrencyTracker:OpenUI()`（内部调用 `CurrencyFrame:Show()`）。
  - 关于 Lua 与 XML 的分工：
    - XML：负责静态布局与模板声明，确保视觉风格与 Gold UI 一致。
      - 新增 `CurrencyFrame.xml`：
        - 定义主 Frame（尺寸、贴图、标题、关闭按钮）。
        - 定义顶部选择区的容器（服务器/角色/货币下拉的占位）。
        - 定义行模板：
          - `CurrencyRowTemplate`（全部货币模式：左侧货币 ICON + 追踪 checkbox + 收入/支出/净额/总上限列）。
          - `CurrencySourceRowTemplate`（单货币模式：来源 | 收入 | 支出 | 净额）。
        - 定义滚动区域与底部 timeframe 按钮条占位（可参考 `Core/Template.xml` 中的 `AccountantClassicRowTemplate`、`AccountantClassicTabTemplate`）。
      - XML 的好处：
        - 更容易复用 Gold 的皮肤与对齐方式，布局结构直观；
        - 行模板可复用与迭代，减少 Lua 代码体量与样式分散。
    - Lua：负责控制逻辑与数据绑定。
      - 在 `CurrencyFrame.lua` 中实现：
        - `Initialize()`：加载/绑定 XML 中的控件、注册事件/回调；
        - `Show()/Toggle()`：显示/隐藏窗口，设置默认状态（全部货币 + Session）；
        - 下拉与 timeframe 按钮的回调（变更状态 → `RefreshView()`）；
        - `RefreshView()`：根据当前模式与 timeframe 调用 Getter（`GetMultipleCurrenciesData / GetCurrencyDetailData`）并渲染行模板；
        - 追踪 checkbox 的处理逻辑与存储更新（写入 `discovered.tracked`）。
  - 资源与加载：
    - 贴图复用：`Images/AccountantClassicFrame-Left/Right` 等，与 Gold UI 统一。
    - 模板复用：参考 `Core/Template.xml` 与 `Core/Core.xml` 的结构与命名。
    - 加载顺序：在 `CurrencyTracker/CurrencyTracker.xml` 中 `<Script file="CurrencyFrame.xml"/>`（若使用 XML）与 `<Script file="CurrencyFrame.lua"/>`，保持在 `CurrencyCore.lua` 之前加载，以便 Core 能调用 `OpenUI()`。
- Slash 命令：
  - 在 `CurrencyTracker/CurrencyCore.lua` 的命令分发中新增 `^ui`：调用 `CurrencyTracker:OpenUI()`。
  - 同步更新 `ShowHelp()`（根据工程记忆：命令新增/修改必须更新帮助）。
- 视图层：
  - 采用与 Gold 相同的 Frame 构造与模板；底部 Tab 重用现有的按钮模板与定位（尽量复用贴图/样式）。
  - 表格行使用可复用的 RowTemplate，支持“全部货币行”和“来源行/交易行”两种渲染模式；
  - 大数据量滚动：使用 `HybridScrollFrame` 或现有的滚动模板。

## 交互细节
- 按钮/下拉联动均触发 `RefreshView()`；
- `RefreshView()` 根据当前状态（all vs single、timeframe）决定调用哪条数据路径，
  并将结构化数据渲染为表格；
- 合计行与总上限（TotalMax）展示：
  - 若 `C_CurrencyInfo.GetCurrencyInfo(currencyId).maxQuantity` 或 `totalMax` 可用，则在“全部货币模式”中以列的形式附加显示；
  - 在“单货币模式”中显示于标题下的摘要栏；
- 追踪 checkbox 行为：点击后立即写入 `Storage:GetDiscoveredCurrencies()[id].tracked`（true/false），并触发一次 `RefreshView()`；不要求 `/reload`。
- 错误与空态：
  - 无数据时显示“无货币数据”；
  - 未选择货币且处于“单货币模式”时回退为“全部货币模式”。

## 本地化
- 所有文案走 `AceLocale`；
- 货币名优先取 `DataManager:GetCurrencyInfo(id).name` 并映射到本地化表；
- 时间范围标签使用现有 `CT_GetTimeframeLabel()` 逻辑。

## 性能与兼容
- UI 打开时延迟拉取数据；切换时使用轻量刷新，避免反复扫描；
- 兼容没有 `C_CurrencyInfo` 的旧版本，TotalMax 显示为 `Unlimited`；
- 不修改 Gold 逻辑与数据结构，保持向后兼容。

## 开发里程碑（本 PR）
1) 新增文档（当前文件）。
2) 在 `CurrencyCore.lua` 中加入 `/ct ui` 命令入口与 `ShowHelp()` 更新（仅调用占位 `OpenUI()`）。
3) 搭建 UI 模块骨架（空窗体 + 下拉/Tab 占位）。
4) 绑定“打开默认展示为 `ct show-all this-session`”。
5) 绑定“选择货币后展示 `ct show <timeframe> <currencyId>`”。

## 后续迭代
- 与主面板（Gold UI）同窗 Tab 切换整合；
- 更多筛选（仅显示已追踪 / 包含未追踪）；
- 排序与列宽自适应；
- 近上限提醒（near-cap）在 UI 上的提示与设置入口；
- 永久化 UI 状态（服务器/角色/货币/时间范围）。

---

附注：记得在每次新增或修改 `/ct` 命令后，更新 `CurrencyTracker/CurrencyCore.lua` 的 `ShowHelp()` 输出以保持帮助文本同步（工程记忆要求）。
