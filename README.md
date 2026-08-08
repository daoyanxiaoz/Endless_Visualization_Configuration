无尽通用框架 - 可视化布阵配置工具
技术文档（v2.0）
1. 项目概述
1.1 项目目标
开发一个基于 HTML/CSS/JavaScript 的可视化工具，用于生成 MaaFramework 的 interface.json 中的 option 配置。用户通过拖拽或点击在棋盘上布置植物操作，工具自动生成对应的开关和坐标配置，最终导出可直接放入 config/instances 的完整实例 JSON。

1.2 核心功能
双棋盘布阵：支持前期（Early）和后期（Late）两个独立 9×5 棋盘，可分别配置不同阶段的植物布局。

拖拽式操作：用户从左侧操作列表中拖拽或点击选中操作项，然后点击棋盘格子放置，支持批量放置模式。

智能开关联动：棋盘上的操作自动控制对应的 MaaFW Option 开关（如卡槽种植、守卫菇、喂豆、抛花等），未布置的操作开关自动关闭。

完整配置导出：导出 JSON 包含 CurrentTasks、TaskItems、option 树、AdbDevice 等完整字段，满足 MaaFramework 实例要求。

UI 辅助：提供快捷开关、卡槽备注、前期转后期输入框、选项树预览、悬停提示系统等。

1.3 适用场景
为 MaaFramework 的“无尽模式”任务生成自定义布阵配置。

需要精细化控制植物种植位置、喂豆、抛花、魔甘收尾等操作的场景。

适合需要快速调整布局并导出配置的无尽模式玩家。

2. 技术栈与依赖
HTML/CSS：纯原生，无外部依赖，自适应布局。

JavaScript：原生 ES6，无框架，所有逻辑集中在一个页面。

MaaFramework：配置输出格式符合 interface.json 的 Option 协议（v2.3.0+）。

浏览器：现代浏览器（Chrome/Firefox/Edge）均可运行。

3. 目录结构与核心模块
整个工具是一个单页面 HTML 文件，结构如下：

text
无尽配置.html
├── HTML 结构
│   ├── 工具栏 (实例名称、文件名、控制器、资源、入口、导出/默认/复制按钮)
│   ├── 左侧面板
│   │   ├── 卡槽备注 (8个输入框)
│   │   ├── 操作项列表 (前期/后期分组，含卡片、守卫菇、铲子、喂豆、抛花、魔甘等)
│   │   ├── 快捷开关与批量放置 (开关组 + 坐标输入)
│   │   └── 无尽选项树 (只读预览，显示完整 Option 层级)
│   ├── 右侧面板
│   │   ├── 双棋盘 (前期/后期切换)
│   │   ├── 棋盘控制 (列/行调整、重置)
│   │   ├── 前期转后期输入框 (显眼黄色区域)
│   │   └── 右侧快速控制 (常用开关)
│   └── 预览区 (只读 JSON 预览 + 状态消息)
├── CSS 样式 (内联)
└── JavaScript 逻辑
    ├── OPTION_DEFS (完整配置定义)
    ├── 操作项定义 (EARLY_*, LATE_*)
    ├── Tooltip 系统 (全局悬停提示)
    ├── 棋盘数据管理 (boardEarly/boardLate)
    ├── UI 渲染 (渲染操作项、棋盘、快捷开关、选项树)
    ├── 核心导出函数 (buildFullInstance)
    ├── 辅助功能 (复制、加载默认、重置)
    └── 事件绑定 (拖拽、点击、键盘导航、Tooltip)
4. 核心数据结构
4.1 OPTION_DEFS
这是整个工具的配置核心，描述了所有 MaaFW Option 的定义，包括类型、子选项和默认值。工具根据此定义生成选项树、校验用户输入和构建导出 JSON。

结构示例：

javascript
"是否抛花": {
    type: "switch",
    cases: [
        { name: "No" },
        { name: "Yes", option: ["小关是否抛花", "boss关是否抛花", "识别能量花卡槽位置"] }
    ]
}
注意事项：

switch 类型的 cases 顺序影响 UI 中的 index 映射，一般 No 在前（index:0），Yes 在后（index:1），但三个“识别”开关在导出时会反转（见后文）。

input 类型需定义 inputs 数组，每个对象有 name 和 default，导出时作为 data 对象。

select 类型用于枚举选择，其 cases 中的 option 数组表示子选项。

动态生成：代码中通过循环自动生成了大量子选项（如 卡X种植、卡X种第Y次、卡X种第Y次坐标），减少手工维护。

4.2 操作项定义 (ALL_OPS)
所有可拖拽/点击放置的操作项，包括前期和后期分类。每个操作项包含：

id：唯一标识

label：显示名称

type：用于标签样式和逻辑分类（如 card, guard, feed, flower, sweet 等）

phase：early 或 late

single：布尔值，true 表示只能放置一次（如喂豆），false 可放置多次

maxCount：最大放置次数（仅当 single=false 时有效）

slot（可选）：用于补植物操作，表示卡槽编号

扩展新操作：在对应的数组（如 EARLY_CARDS）中添加新对象即可。

4.3 棋盘数据 (boardEarly, boardLate)
每个棋盘是一个二维数组 [rows][cols]，每个格子是一个数组，存储放置在该位置的操作项对象。每个操作项包含：

id：操作项 ID

label：显示标签（可能包含序号）

type：操作类型

col：列坐标（1-based）

row：行坐标（1-based）

slotPos：同一操作的第几次放置（从1开始）

opData：指向 ALL_OPS 中对应对象的引用

5. 详细功能模块
5.1 工具栏
配置名称 (#configName)：导出 JSON 中的 InstanceName 字段。

文件名 (#fileName)：下载的文件名（不含 .json）。

控制器 (#controllerName)：对应 CurrentControllerName。

资源 (#resourceName)：对应 Resource 字段（当前强制为空字符串）。

入口 (#taskEntry)：对应任务的 entry 节点。

导出 JSON：调用 exportJSON() 下载文件。

默认选项：重置所有状态到默认值。

复制：复制预览区 JSON 到剪贴板。

所有输入框和按钮都带有悬停提示（Tooltip），方便了解用途。

5.2 左侧面板
卡槽备注
提供 8 个输入框，用于给卡槽（卡1~卡8）添加备注。备注会显示在对应卡槽操作项的名称旁（如 卡1(备注)），并影响导出时的显示。

操作项列表
按“前期”和“后期”分组显示，每个操作项可拖拽或点击选中。选中后点击棋盘格子即可放置（批量模式下连续放置，否则放置后取消选中）。操作项支持键盘导航（W/S 移动选择，A/D 切换前后期）。

每个操作项自带悬停提示，说明其用途和最大次数。

快捷开关与批量放置
批量放置 (#batchMode)：勾选后，放置操作不取消选中，可连续放置多个同种操作。

快捷开关：根据当前选中的 Tab（前期/后期）显示对应的开关（如小关喂豆、Boss喂豆、加速等），用于快速开启/关闭总开关，并自动联动依赖关系（如开启“识别能量花”会自动打开“是否抛花”）。

坐标输入：为喂豆位置等提供列-行输入框，依赖对应的开关是否启用（未启用时灰显）。

无尽选项树（只读预览）
基于 OPTION_DEFS 和 currentValues 构建的完整选项树，实时反映当前配置，便于用户预览结构。所有控件（radio、select、input、checkbox）都带有悬停提示。

5.3 右侧面板
布阵棋盘
前期/后期 Tab：切换显示不同棋盘，同时切换左侧操作项列表和快捷开关。

网格：默认 9×5，可调整尺寸（列 1~9，行 1~5）。每个格子可放置多个操作项，并显示彩色标签。

操作：点击放置（需选中操作项）、右键清空格子、拖拽操作项到格子。

魔音占位：若后期魔甘开启且格子 (9,3) 为空，自动显示“魔音(9-3)”占位标签，提示用户该位置会被魔音占据。

棋盘控制
列数/行数：调整棋盘尺寸，点击“应用”生效，已布置的操作会尽量保留在新尺寸内。

重置前期/后期：清空对应棋盘所有操作（需确认）。

前期转后期输入框（显眼黄色区域）
输入数字后自动设置 frame_wj_custom_回到后期 的 关卡 值，并强制开启 自定义布阵 父开关，使脚本在指定关卡退出自定义布阵，恢复普通流程。支持合法性校验（1~149且非5的倍数）。

右侧快速控制
显示常用开关（抛花、魔甘、加速、智能补给等），与左侧快捷开关联动，方便快速调整。

5.4 预览与导出
预览区显示当前构建的完整 JSON，只读，可手动复制或通过“复制”按钮复制。点击“导出 JSON”下载为 .json 文件。

5.5 悬停提示系统（Tooltip）
这是本工具的重要辅助功能，帮助用户快速了解每个控件的用途。

全局机制：所有带有 data-tooltip 属性的元素，在鼠标悬停一定时间后显示提示框。

延迟控制：默认延迟 500ms（可通过修改 handleMouseEnter 中的值调整），但可通过 data-tooltip-delay 属性单独指定延迟（单位毫秒）。例如棋盘格子使用了 data-tooltip-delay="1000" 实现 1 秒延迟。

动态绑定：对于动态生成的元素（如操作项、快捷开关、选项树控件），在创建时通过 setAttribute('data-tooltip', tip) 和 enableTooltip(el) 绑定事件。

提示内容：提供清晰、简洁的功能说明，包含依赖关系、注意事项等。

样式：深色半透明背景，白色文字，圆角边框，跟随鼠标移动，不遮挡操作。

6. 核心逻辑详解
6.1 放置逻辑 (placeOp)
javascript
function placeOp(board, opId, r, c) {
    // 1. 查找操作定义
    // 2. 如果是单次操作 (single=true)，移除旧位置
    // 3. 如果是多次操作，检查是否已达最大次数 (maxCount)
    // 4. 分配 slotPos（第几次放置），用于区分同一操作的多个实例
    // 5. 构造操作数据并推入棋盘格子
    // 6. 返回是否成功
}
6.2 导出逻辑 (buildFullInstance)
这是最重要的函数，负责将当前 UI 状态转换为 MaaFW 实例 JSON。流程如下：

备份用户开关：保存所有用户手动控制的开关（如抛花、魔甘、加速等），不包括识别开关（它们由 UI 直接控制）。

提取棋盘操作：遍历棋盘，提取所有有坐标的操作项，存储到 rawOps 数组。

重置所有棋盘驱动的开关：将所有可能由棋盘控制的开关（卡槽种植、守卫菇、铲子、补植物、铲除、喂豆、抛花、魔甘等）重置为关闭状态（index:0），并清空坐标数据。

填充数据：根据 rawOps 重新开启对应的开关，并填充坐标。

卡槽种植：按卡槽分组，开启 卡X种植 和各个 卡X种第Y次 及其坐标。

守卫菇：开启 frame_wj_custom_是否种守卫菇 和各个位置子开关。

铲子：区分前后两组（铲1~5 和 铲6~10）。

补植物：按卡槽和位置开启。

铲除：开启 铲除植物 及各个子项。

喂豆：分别处理普通喂豆和 Boss 喂豆，并填充位置。

抛花：根据是否存在操作，开启对应的子开关并填充坐标。

魔甘：根据是否存在甜薯、甘蓝、魔甘喂豆等操作，开启对应输入并填充坐标。

自定义布阵：若棋盘非空或用户输入了“前期转后期”，则开启。

恢复用户开关：将备份的用户开关（除识别开关外）恢复，确保用户手动设置不被覆盖。

构建选项树：递归遍历 OPTION_DEFS，根据 currentValues 生成树形结构，并应用识别开关的反转逻辑（勾选时导出 0，未勾选导出 1）。

组装最终实例：包含 CurrentControllerName、Resource（空）、CurrentTasks、TaskItems、AdbDevice 等。

6.3 识别开关反转逻辑
三个识别开关（识别能量花卡槽位置、识别魔甘卡槽位置、识别三叶草卡槽位置）在 OPTION_DEFS 中定义为 cases: [{name:"No"},{name:"Yes"}]，UI 中勾选对应 index===1。但 MaaFramework 的预期行为是：勾选表示“不识别”，未勾选表示“识别”，因此在导出时对这三个开关进行反转：

javascript
const isRecog = name === '识别能量花卡槽位置' || name === '识别魔甘卡槽位置' || name === '识别三叶草卡槽位置';
if (isRecog) {
    idx = idx === 0 ? 1 : 0;
}
6.4 前期转后期联动
用户在右侧输入框中输入数字后，会写入 currentValues['frame_wj_custom_回到后期'].data.关卡，并强制 自定义布阵 为 1，确保该子节点出现在导出 JSON 中。

6.5 Tooltip 系统实现
核心函数：

showTooltip(text, x, y)：显示提示框，定位在鼠标右下方。

hideTooltip()：隐藏提示框。

handleMouseEnter(e)：读取 data-tooltip 和 data-tooltip-delay，启动定时器。

handleMouseLeave(e)：取消定时器并隐藏。

handleMouseMove(e)：若提示已显示，跟随鼠标移动。

启用方法：调用 enableTooltip(el) 为元素绑定事件，或通过 enableTooltips(selector) 批量启用。

动态监听：使用 MutationObserver 监听新增节点，自动为带有 data-tooltip 的新元素启用提示。

7. 扩展指南
7.1 添加新的操作类型
在 ALL_OPS 对应的数组（如 EARLY_CARDS）中添加新对象，定义 id、label、type、phase、single、maxCount 等。

在 buildFullInstance 的填充部分（第 4 节）添加对应的处理逻辑，解析 rawOps 并设置对应的开关和坐标。

确保 OPTION_DEFS 中有对应的定义，且层级正确（父开关需包含子选项）。

7.2 添加新的开关
在 OPTION_DEFS 中添加新条目，指定 type（switch/select/input/checkbox）。

在 userControlledKeys 或 userOnlyKeys 中根据是否需要备份决定是否包含。

如需在 UI 中显示，添加到对应的 SWITCH_GROUPS（左侧快捷开关）或 RIGHT_SWITCHES（右侧快速控制）。

如果需要联动，在 DEPENDENCY_RULES 中添加依赖关系。

7.3 添加新的输入字段
在 OPTION_DEFS 中添加 type: "input" 的定义，指定 inputs 数组。

在 buildFullInstance 中确保该输入不会被重置（除非需要棋盘驱动）。

在对应面板中添加 UI 输入控件（如快捷开关的 coords 数组），并绑定事件更新 currentValues。

7.4 调整棋盘尺寸
修改 cols 和 rows 的初始值及 colN/rowN 的 max 属性，并确保网格渲染和操作逻辑兼容（在 resizeBoards 中已有处理）。

7.5 自定义 Tooltip 内容或延迟
修改全局默认延迟：在 handleMouseEnter 中将 500 改为其他值。

单独指定延迟：在元素上设置 data-tooltip-delay 属性（单位毫秒）。

修改提示内容：直接编辑 data-tooltip 属性，或在代码中生成时调整 tip 变量。

8. 常见问题与调试技巧
8.1 导出的 JSON 中某些开关没有按预期开启
检查 buildFullInstance 中的重置与填充逻辑：确保对应操作在 rawOps 中被正确识别，并且填充代码正确设置了 index 和 data。

检查 OPTION_DEFS：确保该开关存在，且层级路径正确。

使用 console.log 调试：在关键位置输出 currentValues 查看值的变化。

8.2 识别开关导出值相反
检查 buildCleanNode 中的反转逻辑是否正确应用了这三个开关。

确认 OPTION_DEFS 中它们的 cases 顺序为 [{name:"No"},{name:"Yes"}]，并且 UI 中 checked 对应 index===1。

8.3 前期转后期输入不生效
确认输入框的 id 为 customReturnLate，且事件监听已绑定。

检查 currentValues['frame_wj_custom_回到后期'] 是否正确更新。

确保 buildFullInstance 中没有重置该值（已移除重置）。

8.4 自定义布阵没有自动开启
检查棋盘是否有操作（rawOps.length > 0）或用户输入了“前期转后期”。

确认 currentValues['自定义布阵'] 被正确设置为 1。

8.5 导出 JSON 包含其他任务
已将 Resource 设为空字符串，并删除 ResourceOptionItems，避免 MaaFramework 自动补全。

8.6 Tooltip 不显示或延迟不准确
检查元素是否已正确添加 data-tooltip 属性和 data-tooltip-delay（若需要）。

确认 enableTooltip 已对该元素调用（动态元素需在生成后调用）。

查看浏览器控制台是否有 JavaScript 错误。

检查 CSS 中 .tooltip-box 的 z-index 是否被其他元素遮挡。

9. 维护与版本记录
日期	版本	变更内容
2026-08-07	v1.0	初始版本，包含双棋盘、所有操作类型、智能开关、导出功能。
2026-08-07	v1.1	修复识别开关反转逻辑，添加前期转后期联动，分离配置名与文件名。
2026-08-08	v2.0	新增全局悬停提示系统（Tooltip），支持自定义延迟，优化用户体验。完善文档。
10. 项目约定与编码规范
命名约定：变量名采用 camelCase，常量采用 UPPER_CASE。

代码风格：使用 4 空格缩进，注释使用中文。

UI 布局：采用 Flexbox 自适应，保持响应式。

数据流：所有状态存储在 currentValues 中，UI 渲染基于该对象，修改后调用 updatePreview() 更新预览。

Tooltip：统一使用 data-tooltip 属性存储提示文本，data-tooltip-delay 控制延迟（可选），通过 enableTooltip 绑定事件。

11. 附录：核心函数速查表
函数名	作用	关键参数
placeOp(board, opId, r, c)	在棋盘指定位置放置操作	board, opId, r, c
buildFullInstance()	构建完整导出 JSON	无
updatePreview()	刷新预览区 JSON 和快捷开关	无
renderBoard(gridId, boardData, isLate)	渲染棋盘	gridId, boardData, isLate
renderQuickSwitches(phase)	渲染左侧快捷开关	phase (early/late)
enableTooltip(el)	为元素启用悬停提示	el
handleMouseEnter(e)	鼠标进入时启动定时器	事件对象
handleMouseLeave(e)	鼠标离开时隐藏提示	事件对象
文档结束

如需进一步扩展或调整，欢迎参照上述指南进行修改。如有疑问，可查阅代码内注释或咨询维护者。