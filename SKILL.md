---
name: 开发Operit侧边栏UI工作台
---

# 开发 Operit 侧边栏 UI 工作台

本技能用于开发“能在 Operit 侧边栏看到、带图形化界面（Compose DSL）的 UI 工作台”，
例如清理/监控/调试类的看板或控制台。技能界面里能看到本技能的开/关开关，表示它已被
正确放进 `/sdcard/Download/Operit/skills/` 并被识别。

## 一、先分清：普通 JS 包还是 ToolPkg

默认不要一上来就做 ToolPkg。按需选择：

- 只要“新增/修改工具函数、参数、返回结构、读普通资源” → 写普通 JS 包脚本（METADATA + 工具函数）。
- 需要“配置界面 / 工具箱页面 / 侧边栏 UI 工作台 / 注册宿主入口 / hook / prompt hook” → 才用 ToolPkg。
- 只是少量配置项，优先用参数或 env，不要为加配置而做 UI。
- 想让侧边栏“插件区”出现入口，必须走 ToolPkg：普通 JS 包没有注册侧边栏入口的能力。

纯 JS 包照常只提供工具调用，不能注册 `main_sidebar_plugins`。

## 二、ToolPkg 工程骨架

开发目录固定：`/sdcard/Download/Operit/dev_package/{toolpkg_id}/`

```text
dev_package/com.operit.xxx/
├── manifest.json              # toolpkg_id / main / display_name / subpackages
├── tsconfig.json              # 参考 ../types，typeRoots 指向 ../types
├── src/
│   ├── main.ts                # ToolPkg 主上下文：注册 UI 路由 + 侧边栏/工具箱入口
│   ├── packages/<id>.ts       # AI 直调工具（METADATA 注释块保留）
│   ├── shared/                # 纯逻辑（扫描/查询等），UI 与工具共用
│   └── ui/<name>/index.ui.ts  # Compose DSL 看板（UI 工作台）
└── dist/                      # tsc 编译产物
```

常用关键点：
- 普通工具仍要带 `/* METADATA {…} */` 注释块，这样被当作外部包时也能按“工具包”识别。
- UI 模块在 `.ui.ts` 里 `export default function Screen(ctx)`，用 `ctx.UI`、`ctx.useState`、`ctx.useRef`。
- `main.ts` 用全局 `ToolPkg.registerUiRoute` 与 `ToolPkg.registerNavigationEntry` 注册。
- 单一源码工程用 `import`/`export`；不要 `require`。

## 三、main.ts：让侧边栏出现入口

关键：`registerNavigationEntry` 的 `surface` 用 `"main_sidebar_plugins"`（主侧边栏插件区），
再在工具箱加一个入口便于打开；UI 运行时用 `"compose_dsl"`。

```ts
const ROUTE = "toolpkg:com.operit.xxx:ui:workbench";
function registerToolPkg() {
  ToolPkg.registerUiRoute({
    id: "workbench",
    route: ROUTE,
    runtime: "compose_dsl",
    screen: Screen,
    params: {},
    keepAlive: true,
    title: { zh: "XX 工作台", en: "XX Workbench" }
  });
  ToolPkg.registerNavigationEntry({
    id: "xxx_sidebar",
    route: ROUTE,
    surface: "main_sidebar_plugins",   // 主侧边栏插件区可见
    title: { zh: "XX 工作台", en: "XX Workbench" }
  });
  ToolPkg.registerNavigationEntry({ id: "xxx_toolbox", route: ROUTE, surface: "toolbox", title: { zh: "XX", en: "XX" } });
}
exports.registerToolPkg = registerToolPkg;
```

安装并启用后，入口在“主侧边栏插件区”出现；若没立即出现，重载/重启一次 Operit 让
ToolPkg 主上下文重新执行 `registerToolPkg`。

## 四、Compose DSL：UI 工作台与“开关”

UI 文件是 `src/ui/<name>/index.ui.ts`，`export default function Screen(ctx)`。
宿主每次渲染会重建 Screen，普通 setTimeout 会捕获“首次渲染的陈旧闭包”，所以把最新值
写进 `ctx.useRef` 的“控制舱” `ctl`，定时回调一律从 `ctl.current` 读。

迷你例子（含 UI.Switch 开关）：

```ts
export default function Screen(ctx: any): any {
  const UI = ctx.UI;
  const s = (v: any): string => String(v == null ? "" : v);
  const [autoOn, setAutoOn] = ctx.useState("wb_auto", false);   // 开关状态（持久 key）
  const [text, setText] = ctx.useState("wb_text", "");
  const [logs, setLogs] = ctx.useState("wb_logs", [] as string[]);
  const ctl: any = ctx.useRef("wb_ctl", { autoOn: false, logs: [] });
  ctl.current.autoOn = autoOn;                                   // 写回最新值
  function toggle() { const nx = !autoOn; setAutoOn(nx); ctl.current.autoOn = nx; }
  const Row = (k: any[]) => UI.Row({ spacing: 8, fillMaxWidth: true, verticalAlignment: "center" }, k);
  const Card = (k: any[]) => UI.Card({ padding: { horizontal: 12, vertical: 12 } },
    [UI.Column({ spacing: 8, fillMaxWidth: true }, k)]);
  return UI.LazyColumn({ spacing: 10, padding: { horizontal: 10, vertical: 8 }, fillMaxWidth: true },[
    Card([
      UI.Text({ text: "UI 工作台（带开关）", style: "titleMedium" }),
      Row([
        UI.Text({ text: "开关：" + (autoOn ? "开" : "关") }),
        UI.Switch({ checked: autoOn, onCheckedChange: toggle })  // ← 技能/看板里要的开关
      ]),
      UI.TextField({ value: text, onValueChange: setText, singleLine: true, fillMaxWidth: true }),
      UI.Button({ text: "记录", onClick: () => { const nx = [...ctl.current.logs, text]; ctl.current.logs = nx; setLogs(nx); } })
    ]),
    Card(logs.length ? logs.map((l, i) => UI.Text({ text: (i + 1) + ". " + l, style: "bodySmall" }))
        : [UI.Text({ text: "暂无日志", style: "bodySmall" })])
  ]);
}
```

常用控件：`UI.Text / UI.Button / UI.TextField / UI.Switch / UI.Row / UI.Column / UI.Card /
UI.LazyColumn / UI.IconButton / UI.Surface / UI.CircularProgressIndicator`。
控件文字/颜色用 props（`text/style/color/maxLines/enabled/onClick/checked/onCheckedChange/weight`），
间距 `spacing`、`padding`、`horizontalArrangement`/`verticalAlignment`。

状态要点：
- `ctx.useState(key, default)` 会按 key 持久，用它存 dir/开关/日志等。
- 每次渲染把最新值写回 `ctl.current`，定时器/回调从 `ctl.current` 读，避免陈旧闭包。
- 关掉的开关不要残留旧 timer：`toggle` 里先 clearInterval 再决定是否新建。

## 五、构建、打包、安装、让入口出现

1. 编译：项目内 `rm -rf dist && tsc -p tsconfig.json`（0 报错）。
2. 打包：把 `manifest.json`（zip 根）+ `dist/**` 打进 `dev_package/<id>.toolpkg`。
3. 安装：`operit_editor:debug_install_toolpkg`，`source_path` 指到 `.toolpkg`，
   `enable_after_install=true`、`activate_after_install=true`。会拷贝到
   `/sdcard/Android/data/com.ai.assistance.operit/files/packages/<id>.toolpkg` 并刷新。
4. 若外部目录还留着同名的普通 `.js` 包，先删除旧文件避免工具冲突。
5. 若侧边栏还没出现入口：重载/重启 Operit 让 ToolPkg 主上下文重新执行 `registerToolPkg`。

## 六、易踩坑（经验沉淀）

- UI 层读内部 datastore/需鉴权的东西会被宿主拒（toolCall requires an active execution Call）：
  把它迁到“包工具”的工具层，UI 再经桥接调；读取结果可能是被屏蔽的特殊对象，
  先 JSON 往返还原成普通对象再判断字段。
- 不要在 UI 里直接调 datastore 读取类 API，改走工具层 active-call。
- `ctx.useState` 的类型参数可省：`ctx.useState("k", [] as T[])`，避免对 `any` 宿主报 TS2347。
- `UI.Text` 若想限制行数，辅助函数要多收一个 maxLines 可选参数，别在固定 3 参的封装里硬塞第 4 参。
- 大文件别用 create_file 一次写超大内容（易被截断作废）：用终端 `cat >`/`cat >>` 分段写。
- 定时自动刷新默认关，别在 onLoad 里对超大目录自动全量扫描，会卡。
- Android 沙箱的 Files.list("/") 看不到真实根；要做整盘，目标要落在
  `/storage/emulated/0`（内部存储 0/）这类可达目录，别拿系统根当扫描对象。
- 清理/删除类工具务必带“保护名单 + 预览(干跑)+确认”，整盘遍历更要跳过受限应用目录。

## 七、交付检查单

- [ ] manifest 与版本正确；`main` 指向 `dist/main.js`
- [ ] tsc 0 报错；`.toolpkg` 里 manifest 在 zip 根、dist 在其中
- [ ] ToolPkg 已安装且 enabled；外部目录无同 ID 旧 `.js`
- [ ] 侧边栏“插件区”能看到入口，点开能看到带开关的 UI
- [ ] 工具有 METADATA、UI 与工具都能独立工作；回归测试过
