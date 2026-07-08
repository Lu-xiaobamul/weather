第一部分：HTML 框架（骨架与语义）
HTML 结构采用了自上而下、左（列表）右（详情）的经典后台/工具型布局。它不仅是内容的容器，更是 JavaScript 操作的目标地图。

区域	标签/ID	作用与交互意图
头部 <header>	#city-input, #add-btn, #location-btn	输入交互区。搜索框支持文本输入，添加按钮触发城市查询，定位按钮调用浏览器 Geolocation API。flex-col md:flex-row 确保在手机上垂直排列，在电脑上水平排列。
错误反馈	#error-msg	瞬时状态反馈。用于显示“城市不存在”、“网络错误”等提示，高度固定为 h-5，避免提示出现时页面发生跳动。
左侧列 (城市列表)	#city-list, #city-count	数据导航。动态渲染城市卡片，点击卡片切换选中状态，右上角“×”删除城市。overflow-y-auto 配合隐藏滚动条，使长列表滚动时不破坏毛玻璃美感。
右侧列 (详情)	#weather-detail, #empty-state	核心展示区。#weather-detail 默认隐藏 (hidden)，只有当选中城市并获取数据后才显示。#empty-state 在列表为空时显示占位提示。
详情子模块	#detail-temp, #detail-icon, #forecast-list	分别负责当前温度、天气图标（SVG容器）和未来5天预报的网格填充。
第二部分：CSS 样式与交互效果（视觉层）
代码采用 Tailwind CSS（功能性类） + 自定义 CSS（精细动效） 的组合策略。

1. 自定义 CSS 详解
.fade-in (入场动画)
css
@keyframes fadeIn { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: translateY(0); } }
交互效果：当天气详情切换或首次加载时，内容会从下方轻微上浮（8px）并淡入。
作用：提供流畅的视觉过渡，避免数据刷新时生硬地“闪现”，提升浏览舒适度。
.glass (毛玻璃特效)

css
background: rgba(255, 255, 255, 0.2);
backdrop-filter: blur(14px);
border: 1px solid rgba(255, 255, 255, 0.25);
交互效果：背景半透明，透过面板能看到底层的背景渐变色，且内容边缘有柔光边框。
作用：营造现代感、高级感的 UI 质感（即 Glassmorphism），让天气卡片看起来像是悬浮在背景之上的磨砂玻璃。
.scrollbar-hide (优雅滚动)

css
.scrollbar-hide::-webkit-scrollbar { display: none; }
.scrollbar-hide { -ms-overflow-style: none; scrollbar-width: none; }
交互效果：完全隐藏滚动条，但保留滚轮滚动功能。
作用：在深色毛玻璃背景下，原生滚动条会显得突兀，隐藏后界面更干净整洁。

2. 动态背景交互（由 JavaScript 驱动）
交互逻辑：<body> 的默认类是 bg-gradient-to-br from-blue-500 to-indigo-700。
动态变化：当 JavaScript 获取到天气数据后，会根据 weatherMap 中映射的 bg 字段（如 from-slate-800 to-gray-900 代表暴雨）直接修改 <body> 的 className。
效果：晴天是蓝色渐变，雨天是深灰/蓝黑渐变，雪天是冷白/灰蓝渐变。视觉上，整个页面的氛围与当前天气实时同步。

3. Tailwind 工具类交互细节
transition-colors duration-700（在 body 上）：背景颜色切换时有 700ms 的缓动过渡，使天气切换时背景渐变平滑改变。
hover:bg-white/20 与 hover:bg-red-500：城市卡片悬停时背景变亮，删除按钮悬停时变红，提供明确的可点击反馈。
focus:outline-none focus:ring-4 focus:ring-blue-300：输入框聚焦时出现蓝色外发光，提示用户当前输入焦点位置。

第三部分：JavaScript 交互逻辑（行为层）
JS 将页面拆分为 数据状态（Model）、UI 渲染（View） 和 用户操作（Controller）。

1. 数据状态管理（持久化）
cities 数组：存储所有城市对象 { id, name, lat, lon }。
localStorage 机制：loadCities() 和 saveCities() 负责读写本地缓存。
交互效果：用户关闭浏览器再打开，添加的城市和选中的城市依然存在，实现“无后端数据持久化”。

2. 核心交互函数链
用户操作	触发函数	内部逻辑与交互反馈
点击“添加” / 按回车	addCity()	1. 调用地理编码 API 将中文名转经纬度。
2. 检查重复，若存在则直接选中该城市。
3. 成功则 push 进数组，调用 renderCityList() 刷新左侧列表，并自动调用 selectCity() 拉取天气。
点击城市卡片	selectCity(id)	1. 更新 selectedCityId 并保存。
2. 调用 renderCityList()（高亮当前卡片）。
3. 请求 Open-Meteo 天气 API。
4. 拿到数据后调用 renderDetail()，隐藏空状态，显示详情面板。
点击“×”删除	deleteCity(e, id)	1. 阻止事件冒泡（e.stopPropagation()），防止触发卡片选中。
2. 过滤数组删除目标。
3. 如果删除的是当前选中的，自动选中列表第一个或显示空状态。
点击定位📍	navigator.geolocation	1. 获取经纬度。
2. 反向地理编码解析城市名。
3. 存在则选中，不存在则自动添加（省去用户手动输入）。
3. 视图渲染细节
renderDetail() 的复杂交互：
动态图标注入：使用 innerHTML 将预定义的 icons 对象（含 SVG 路径）注入到 #detail-icon 中。这比图片加载更快，且无跨域问题。
5日预报循环：for (let i = 1; i <= 5; i++) 从 API 返回的 daily 数据中取第 1 到第 5 项（第 0 项是今天），生成 5 张迷你卡片。
背景同步：document.body.className = ... 实现了“天气变，背景变”的沉浸式交互。
4. 错误处理与防御性交互
showError(msg)：将错误文本写入 #error-msg，并在 3 秒后自动清空。这是一种非阻塞式提示，不会像 alert 那样打断用户操作。
API 请求包裹 try-catch：当网络异常或查询不到城市时，捕获异常并展示友好的中文提示，防止页面白屏或崩溃。

