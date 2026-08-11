MaaPvz 无尽通用框架 - 可视化配置工具 完整开发技能文档
版本：v2.0（最终版）
更新日期：2026-08-11
适用范围：为 MaaFramework 的无尽模式生成自定义布阵配置的 HTML 工具，支持前后期双棋盘、拖拽布阵、智能开关联动及完整 JSON 导出。

📌 项目背景与目标
原项目提供了一个单页 HTML 工具，用于通过可视化方式生成 MaaFramework 的 interface.json 中的 option 配置。但在实际使用中，发现以下功能缺失或交互不理想：

缺失部分 Option 定义（编队切换、自定义 Boss 喂豆间隔等）。

后期喂豆间隔设置不够显眼，用户不易找到。

前期的自定义 Boss 喂豆间隔没有独立显示，且与后期共用导致混淆。

编队切换控件 UI 简陋，不够美观且操作不便。

前后期 Boss 喂豆开关相互干扰——开启前期自定义 Boss 喂豆会导致后期的通用 Boss 喂豆也被自动开启。

本次开发任务即针对上述问题进行系统性修复和增强，最终得到一个功能完整、交互友好、符合 MaaFramework Option 协议的可视化配置工具。

📁 最终文件结构
index.html – 完整的单页面应用，包含 HTML + CSS + JavaScript。

无外部依赖，所有逻辑、样式、定义均内联。

🔧 修改内容逐项详解
1. 补充缺失的 OPTION_DEFS 定义
问题：frame_custom.json 中定义的 自定义布阵 的子项（编队切换、Boss 喂豆间隔）在 HTML 的 OPTION_DEFS 中缺失，导致选项树无法生成这些节点。

解决：在 window.OPTION_DEFS 对象末尾（紧接原有定义之后）添加以下代码：

javascript
// 1. 编队切换开关及输入
OPTION_DEFS['frame_wj_custom_前期使用编队'] = {
    type: 'switch',
    cases: [
        { name: 'No' },
        { name: 'Yes', option: ['frame_wj_custom_前期切换到第几编队'] }
    ]
};
OPTION_DEFS['frame_wj_custom_前期切换到第几编队'] = {
    type: 'input',
    inputs: [{ name: '编队', default: '' }]
};
OPTION_DEFS['frame_wj_custom_回到后期切换编队'] = {
    type: 'switch',
    cases: [
        { name: 'No' },
        { name: 'Yes', option: ['frame_wj_custom_切换到第几编队'] }
    ]
};
OPTION_DEFS['frame_wj_custom_切换到第几编队'] = {
    type: 'input',
    inputs: [{ name: '编队', default: '' }]
};

// 2. Boss 关喂豆间隔（自定义布阵下）
OPTION_DEFS['frame_wj_boss_custom_喂豆间隔'] = {
    type: 'input',
    inputs: [{ name: '秒数', default: '3' }]
};

// 3. 将上述子项添加到父级 Yes 分支的 option 数组中（确保层级正确）
if (OPTION_DEFS['自定义布阵']) {
    const cases = OPTION_DEFS['自定义布阵'].cases;
    for (let c of cases) {
        if (c.name === 'Yes') {
            if (!c.option) c.option = [];
            ['frame_wj_custom_前期使用编队', 'frame_wj_custom_回到后期切换编队'].forEach(item => {
                if (!c.option.includes(item)) c.option.push(item);
            });
            break;
        }
    }
}
if (OPTION_DEFS['frame_wj_custom_boss关是否喂豆']) {
    const cases = OPTION_DEFS['frame_wj_custom_boss关是否喂豆'].cases;
    for (let c of cases) {
        if (c.name === 'Yes') {
            if (!c.option) c.option = [];
            if (!c.option.includes('frame_wj_boss_custom_喂豆间隔')) {
                c.option.push('frame_wj_boss_custom_喂豆间隔');
            }
            break;
        }
    }
}
说明：这确保了 自定义布阵 的 Yes 分支下会出现 frame_wj_custom_前期使用编队 和 frame_wj_custom_回到后期切换编队 两个开关，并且各自的 Yes 分支会引出对应的编队编号输入框；同时，frame_wj_custom_boss关是否喂豆 的 Yes 分支会包含 frame_wj_boss_custom_喂豆间隔 输入。

2. 后期喂豆间隔动态显示（棋盘下方）
问题：后期喂豆间隔（小关和 Boss）只藏在选项树中，用户不易找到。希望在切换到后期 Tab 且棋盘上有相应喂豆操作时，在棋盘下方显眼显示。

解决：

HTML 新增（位于 <div class="tab-content" id="tabLate"> 内的 gridLate 之后）：
html
<div id="lateFeedIntervals" style="display:none; margin-top:10px; padding:8px; background:#f0fdf4; border-radius:6px; border:1px solid #bbf7d0;">
    <div style="font-size:13px; font-weight:600; margin-bottom:4px;">⏱️ 后期喂豆间隔设置</div>
    <div style="display:flex; flex-wrap:wrap; gap:12px;">
        <div id="lateSmallFeedInterval" style="display:flex; align-items:center; gap:6px;">
            <label style="font-size:12px;">小关喂豆间隔：</label>
            <select id="lateSmallFeedIntervalSelect" style="padding:3px 8px; border:1px solid #d0d7de; border-radius:4px; font-size:12px; background:#fff;">
            </select>
        </div>
        <div id="lateBossFeedInterval" style="display:flex; align-items:center; gap:6px;">
            <label style="font-size:12px;">Boss关喂豆间隔（秒）：</label>
            <input type="number" id="lateBossFeedIntervalInput" style="width:60px; padding:3px 4px; border:1px solid #d0d7de; border-radius:4px; font-size:12px;" min="1" max="10">
        </div>
    </div>
</div>
JavaScript 新增（位于 window.onload 内，在 loadDefaults(); 之前）：
javascript
// 后期喂豆间隔动态显示
function initLateFeedIntervals() {
    const sel = document.getElementById('lateSmallFeedIntervalSelect');
    const def = OPTION_DEFS['小关喂豆间隔'];
    if (def && def.type === 'select') {
        sel.innerHTML = '';
        def.cases.forEach((c, idx) => {
            const opt = document.createElement('option');
            opt.value = idx;
            opt.textContent = c.label || c.name;
            sel.appendChild(opt);
        });
        sel.value = currentValues['小关喂豆间隔']?.index ?? 0;
        sel.addEventListener('change', function() {
            currentValues['小关喂豆间隔'] = { index: parseInt(this.value) };
            updatePreview();
        });
    }
    const inp = document.getElementById('lateBossFeedIntervalInput');
    inp.value = currentValues['fw_boss关喂豆间隔']?.data?.秒 || '4';
    inp.addEventListener('input', function() {
        let val = this.value.trim();
        if (val === '') val = '4';
        if (!currentValues['fw_boss关喂豆间隔']) currentValues['fw_boss关喂豆间隔'] = { data: {} };
        currentValues['fw_boss关喂豆间隔'].data.秒 = val;
        updatePreview();
    });
}

function updateLateFeedIntervalsVisibility() {
    const container = document.getElementById('lateFeedIntervals');
    if (!container) return;
    const currentTab = document.querySelector('.tab.active')?.dataset.tab;
    if (currentTab !== 'late') {
        container.style.display = 'none';
        return;
    }
    let hasSmallFeed = false;
    let hasBossFeed = false;
    for (let r = 0; r < boardLate.length; r++) {
        for (let c = 0; c < boardLate[r].length; c++) {
            const cell = boardLate[r][c];
            for (let item of cell) {
                if (item.id === 'feed' || item.id === 'feed_late') hasSmallFeed = true;
                if (item.id === 'bossfeed' || item.id === 'bossfeed_late') hasBossFeed = true;
            }
        }
    }
    const shouldShow = hasSmallFeed || hasBossFeed;
    container.style.display = shouldShow ? 'block' : 'none';
    document.getElementById('lateSmallFeedInterval').style.display = hasSmallFeed ? 'flex' : 'none';
    document.getElementById('lateBossFeedInterval').style.display = hasBossFeed ? 'flex' : 'none';
    if (hasSmallFeed) {
        const sel = document.getElementById('lateSmallFeedIntervalSelect');
        if (sel) sel.value = currentValues['小关喂豆间隔']?.index ?? 0;
    }
    if (hasBossFeed) {
        const inp = document.getElementById('lateBossFeedIntervalInput');
        if (inp) inp.value = currentValues['fw_boss关喂豆间隔']?.data?.秒 || '4';
    }
}
// 随后在 renderAllBoards 和 Tab 切换中调用 updateLateFeedIntervalsVisibility()
说明：该区域只在后期 Tab 且棋盘上有小关喂豆或 Boss 喂豆操作时显示，控件修改会实时更新 currentValues 并刷新预览。

3. 前期自定义 Boss 喂豆间隔动态显示（棋盘下方）
问题：前期（自定义布阵）的 Boss 喂豆间隔（frame_wj_boss_custom_喂豆间隔）没有独立的显示位置，且与后期的间隔输入混在一起。

解决：在前期棋盘下方添加独立区域，显示条件为棋盘上有 Boss 喂豆操作（不依赖开关状态），只显示一个输入框（秒数）。

HTML 新增（位于 <div class="tab-content" id="tabEarly"> 内的 gridEarly 之后）：
html
<div id="earlyFeedIntervals" style="display:none; margin-top:10px; padding:8px; background:#f0fdf4; border-radius:6px; border:1px solid #bbf7d0;">
    <div style="font-size:13px; font-weight:600; margin-bottom:4px;">⏱️ 前期自定义Boss喂豆间隔设置</div>
    <div style="display:flex; flex-wrap:wrap; gap:12px;">
        <div id="earlyBossFeedInterval" style="display:flex; align-items:center; gap:6px;">
            <label style="font-size:12px;">Boss关喂豆间隔（秒）：</label>
            <input type="number" id="earlyBossFeedIntervalInput" style="width:60px; padding:3px 4px; border:1px solid #d0d7de; border-radius:4px; font-size:12px;" min="1" max="10">
        </div>
    </div>
</div>
JavaScript 新增（与后期逻辑类似，但独立处理）：
javascript
function initEarlyFeedIntervals() {
    const inp = document.getElementById('earlyBossFeedIntervalInput');
    inp.value = currentValues['frame_wj_boss_custom_喂豆间隔']?.data?.秒数 || '3';
    inp.addEventListener('input', function() {
        let val = this.value.trim();
        if (val === '') val = '3';
        if (!currentValues['frame_wj_boss_custom_喂豆间隔']) {
            currentValues['frame_wj_boss_custom_喂豆间隔'] = { data: {} };
        }
        currentValues['frame_wj_boss_custom_喂豆间隔'].data.秒数 = val;
        updatePreview();
    });
}

function updateEarlyFeedIntervalsVisibility() {
    const container = document.getElementById('earlyFeedIntervals');
    if (!container) return;
    const currentTab = document.querySelector('.tab.active')?.dataset.tab;
    if (currentTab !== 'early') {
        container.style.display = 'none';
        return;
    }
    let hasBossFeed = false;
    for (let r = 0; r < boardEarly.length; r++) {
        for (let c = 0; c < boardEarly[r].length; c++) {
            const cell = boardEarly[r][c];
            for (let item of cell) {
                if (item.id === 'bossfeed' || item.id === 'bossfeed_late') {
                    hasBossFeed = true;
                    break;
                }
            }
            if (hasBossFeed) break;
        }
        if (hasBossFeed) break;
    }
    container.style.display = hasBossFeed ? 'block' : 'none';
    if (hasBossFeed) {
        const inp = document.getElementById('earlyBossFeedIntervalInput');
        if (inp) inp.value = currentValues['frame_wj_boss_custom_喂豆间隔']?.data?.秒数 || '3';
    }
}
// 在 renderAllBoards 和 Tab 切换中同时调用 updateEarlyFeedIntervalsVisibility
说明：此区域与后期独立，修改 frame_wj_boss_custom_喂豆间隔，不影响后期的 fw_boss关喂豆间隔。

4. 美化编队切换控件（右侧）
问题：前期棋盘下的编队切换控件样式简陋，字体小，交互反馈不明显。

解决：重新设计 renderTeamSwitchControls 函数，采用卡片式布局，加入图标、颜色反馈、更大的复选框和输入框。

最终版本（紧凑美化版）代码如下（完整替换原函数）：

javascript
function renderTeamSwitchControls() {
    const container = document.getElementById('teamSwitchControls');
    if (!container) return;
    const currentTab = document.querySelector('.tab.active')?.dataset.tab;
    if (currentTab !== 'early') {
        container.innerHTML = '';
        return;
    }
    const switches = [
        { id: 'frame_wj_custom_前期使用编队', label: '前期切换编队', inputId: 'frame_wj_custom_前期切换到第几编队', icon: '🎯' },
        { id: 'frame_wj_custom_回到后期切换编队', label: '后期切换编队', inputId: 'frame_wj_custom_切换到第几编队', icon: '🔄' }
    ];
    let html = `
        <div style="background:#f8fafc; border-radius:8px; padding:6px 10px; margin-top:2px; border:1px solid #e2e8f0;">
            <div style="font-size:12px; font-weight:600; color:#1e293b; margin-bottom:4px; display:flex; align-items:center; gap:4px;">
                <span>📋</span> 编队切换
                <span style="font-size:10px; font-weight:400; color:#94a3b8;">（仅前期）</span>
            </div>
            <div style="display:flex; flex-direction:column; gap:3px;">
    `;
    switches.forEach(sw => {
        const isChecked = currentValues[sw.id]?.index === 1;
        const inputVal = currentValues[sw.inputId]?.data?.编队 || '';
        const borderColor = isChecked ? '#3b82f6' : '#e2e8f0';
        const bgColor = isChecked ? '#eff6ff' : '#ffffff';
        html += `
            <div style="display:flex; align-items:center; gap:4px; background:${bgColor}; border:1px solid ${borderColor}; border-radius:5px; padding:3px 6px; transition:all 0.2s ease;">
                <span style="font-size:12px; line-height:1;">${sw.icon}</span>
                <label style="font-size:12px; font-weight:500; color:#1e293b; display:flex; align-items:center; gap:4px; cursor:pointer; user-select:none; white-space:nowrap;">
                    <input type="checkbox" id="chk_${sw.id}" ${isChecked?'checked':''} style="width:14px; height:14px; cursor:pointer; accent-color:#2d7aff; flex-shrink:0; margin:0;">
                    ${sw.label}
                </label>
                <div style="display:${isChecked ? 'inline-flex' : 'none'}; align-items:center; gap:2px; margin-left:2px;">
                    <input type="text" id="inp_${sw.inputId}" placeholder="1-6" value="${inputVal}" style="width:36px; padding:1px 2px; border:1px solid #3b82f6; border-radius:3px; font-size:11px; font-weight:500; text-align:center; background:#fff; outline:none;" />
                    <span style="font-size:10px; color:#94a3b8;">号</span>
                </div>
                ${!isChecked ? `<span style="font-size:10px; color:#94a3b8; margin-left:4px;">未启用</span>` : ''}
            </div>
        `;
    });
    html += `</div></div>`;
    container.innerHTML = html;
    // 绑定事件（略，详见源码）
}
同时修改了 updateRightPanelSwitches 和 Tab 切换逻辑，确保编队控件在切换到前期时渲染，切换到后期时隐藏。

5. 修复前后期 Boss 喂豆开关独立性问题
问题：在 buildFullInstance 中，当检测到棋盘上有 Boss 喂豆操作时，fw_boss关是否需要喂豆 会被自动设置为 { index: 1 }，但用户可能并不想开启后期的这个开关，从而导致后期异常开启。

原因：fw_boss关是否需要喂豆 虽然在 userControlledKeys 中被备份，但未在 userOnlyKeys 中恢复，导致后续被棋盘驱动覆盖。

解决：在 buildFullInstance 的第 5 节（恢复用户控制的开关）中，将 fw_boss关是否需要喂豆 加入 userOnlyKeys 列表：

javascript
const userOnlyKeys = [
    '是否抛花', '是否魔甘收尾(牢玩家专属)', '使用三叶草', '魔甘高级选项',
    '小关是否加速', 'fw_boss关是否加速',
    'fw_boss_补给智能选取', 'fw_boss_是否开相机神器',
    '从boss关选卡界面开启', '无尽全自动循环', '刷掉僵尸',
    'fw_boss关是否需要喂豆'  // 新增
];
效果：现在 fw_boss关是否需要喂豆 的值将完全由用户在 UI 中的操作决定，不会被棋盘自动覆盖，前后期 Boss 喂豆开关互不干扰。

🧪 最终功能验证清单
功能点	预期行为	验证结果
选项树中显示编队切换及输入	在自定义布阵 → Yes 下出现 frame_wj_custom_前期使用编队 和 frame_wj_custom_回到后期切换编队，各自 Yes 分支出现编队编号输入	✅
选项树中显示自定义 Boss 喂豆间隔	在 frame_wj_custom_boss关是否喂豆 → Yes 下出现 frame_wj_boss_custom_喂豆间隔 输入	✅
后期喂豆间隔动态显示	切换到后期 Tab，棋盘上有小关喂豆或 Boss 喂豆操作时，棋盘下方显示相应间隔控件	✅
前期自定义 Boss 喂豆间隔动态显示	切换到前期 Tab，棋盘上有 Boss 喂豆操作时，棋盘下方显示间隔控件	✅
编队切换控件仅前期显示	前期 Tab 显示，后期 Tab 隐藏	✅
编队切换控件勾选后显示输入框，取消后隐藏并清空数据	交互正常，数据同步到 currentValues	✅
前后期 Boss 喂豆开关独立	开启前期的 frame_wj_custom_boss关是否喂豆 不会自动开启后期的 fw_boss关是否需要喂豆	✅
导出 JSON 包含完整 option 树	所有开启的开关及子项正确生成，未开启的开关不导出	✅
🚀 部署与使用
将最终的 index.html 放置在任意 Web 服务器或本地浏览器中打开。

使用界面：

左侧：卡槽备注、布阵操作（拖拽/点击）、快捷开关、无尽选项树（预览）。

右侧：前后期棋盘、棋盘控制、前期转后期输入、右侧快速开关、编队切换控件。

布阵操作：点击左侧操作项（如卡1、守卫菇等）选中，再点击棋盘格子放置；支持批量放置（勾选后连续放置）。

快捷开关：开启/关闭对应的总开关，自动联动依赖项。

导出配置：点击“导出 JSON”下载 .json 文件，或复制预览区 JSON。

重置：点击“默认选项”恢复所有配置到初始状态。

📝 开发者注意事项
OPTION_DEFS 是核心配置字典，新增或修改 Option 时需同步更新此处以及导出逻辑 buildFullInstance 中的相关填充/恢复代码。

棋盘操作（placeOp）会修改 currentValues，并触发 updatePreview() 刷新预览。

动态显示区域（喂豆间隔、编队控件）都依赖 currentTab 和棋盘数据，需要确保在 Tab 切换和棋盘变化时调用相应的更新函数。

导出过滤：buildCleanNode 会过滤掉 index: 0 的普通开关，但识别开关（能量花、魔甘、三叶草）有反转逻辑，注意此特性。

全局变量：boardEarly、boardLate、currentValues、OPTION_DEFS 等均在全局作用域，修改时注意一致性。

📚 参考文件
本次修改参考了以下 JSON 文件中的 Option 定义：

frame_custom.json – 自定义布阵相关（编队切换、Boss 喂豆间隔）

option.json – 通用开关及喂豆间隔定义

Flower.json、Magic_Gan.json、Patch_Plants.json – 其他功能模块

✅ 总结
本次开发圆满解决了所有已知问题，工具现在支持：

完整的前后期布阵操作

智能的开关联动与独立控制

显眼的喂豆间隔设置（前后期分别显示）

美观且易用的编队切换控件

准确无误的 JSON 导出

所有修改已集成到最终的 index.html 中，可直接投入使用。后续若需扩展新功能，可参照本文档的结构进行增量开发。