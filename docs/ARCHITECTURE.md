# Camera Coordinate Framework — 架构设计文档

> 相机 App 自动化测试 · 坐标转化框架
> 与 `05_camera_aw/` 并存，独立运行。本框架**只做 dump → 坐标转化**，不参与用例执行。

---

## 1. 框架定位

### 1.1 一句话定义

**基于布局树 dump，把项目特定的控件坐标查出来交给测试用例，由用例自行 ADB 操作。**

### 1.2 边界声明（极其重要）

| 框架**做** | 框架**不做** |
|----------|------------|
| 读取（dump）Android 设备布局树 | ❌ ADB tap / swipe / keyevent |
| 解析布局树为内部结构 | ❌ 测试用例编写 / 执行 |
| 加载项目 JSON 坐标库 | ❌ 结果验证 / 断言 |
| 业务级坐标查询（switch_camera 等） | ❌ 等待 / 同步（sleep 自己加） |
| 输出坐标对象（Coordinate） | ❌ 日志 / 报告 / 排查 |
| | ❌ 多设备调度 |

**核心契约**：调用方拿到的是 `Coordinate` 对象（或元组），由调用方自行 `adb.tap(x, y)`。

### 1.3 与 05_camera_aw 的关系

```
05_camera_aw/              # 已有关键字驱动项目（uiautomator2 + 写死 locator）—— 不动
camera_coordinate_framework/   # 新建独立框架（dump → 查表 → 返回坐标）—— 本文档主题
```

两套并存，测试用例可以选择其一，或混用。

---

## 2. 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│  测试用例（用户自己写）                                              │
│                                                                  │
│    from camera_coord import Framework                            │
│    coord = fw.switch_camera("前置")                               │
│    adb.tap(*coord)                                               │
│                                                                  │
│    coord = fw.set_setting("闪光灯", "自动")                       │
│    adb.tap(*coord)                                               │
└──────────────────────────────────────────────────────────────────┘
                          ↓ 调用
┌──────────────────────────────────────────────────────────────────┐
│  Framework (Façade)                                              │
│    ├── switch_camera(target) → Coordinate                        │
│    ├── switch_mode(mode) → Coordinate                            │
│    ├── set_setting(name, value) → Coordinate                     │
│    ├── take_photo() → Coordinate                                 │
│    ├── open_menu() → Coordinate                                  │
│    └── swipe(from, to) → tuple[Coordinate, Coordinate]           │
└──────────────────────────────────────────────────────────────────┘
                          ↓ 内部调用
┌──────────────────────────────────────────────────────────────────┐
│  CoordResolver（核心查询引擎）                                      │
│    query(project, widget_name) → Coordinate                       │
│    query_setting(project, name, value) → Coordinate              │
│    query_menu_path(project, *path) → list[Coordinate]            │
└──────────────────────────────────────────────────────────────────┘
                          ↑ 加载
┌──────────────────────────────────────────────────────────────────┐
│  ProjectLoader                                                    │
│    load(path) → ProjectConfig（immutable）                        │
└──────────────────────────────────────────────────────────────────┘
                          ↑ 来自
┌──────────────────────────────────────────────────────────────────┐
│  projects/<project_name>.json（项目特有的坐标库）                  │
│    按 设备型号 + 项目 维度组织                                       │
└──────────────────────────────────────────────────────────────────┘
```

**离线标注工具**（tools/annotate_tool.py）独立运行，产出上述 JSON：

```
[人工操作] 切换 mode/cam/setting
       ↓
[工具] adb shell uiautomator dump → 解析 → 输出可编辑 JSON 骨架
       ↓
[人在编辑器中] 补全 (mode, cam, setting, value) 归属
       ↓
[工具] 校验 + 入库 projects/*.json
```

---

## 3. JSON 坐标库 Schema

设计决策：**嵌套树 (a) + 每模式独立 (iii)**——不跨模式共享菜单，宁冗余不耦合。

### 3.1 顶层结构

```json
{
  "project": "CameraA_Xiaomi13Pro",
  "device_model": "Xiaomi 13 Pro",
  "screen_size": [1440, 3200],
  "android_version": "14",
  "camera_app_version": "5.6.0",
  "annotation_date": "2026-07-06",
  "common_widgets": { ... },
  "modes": { ... }
}
```

### 3.2 common_widgets（跨模式通用控件）

```json
"common_widgets": {
  "shutter":         [540, 2800],   // 拍照快门
  "video_shutter":   [540, 2800],   // 录像快门（与拍照同位置，多数 App 共用）
  "switch_camera":   [120, 200],    // 前后置切换按钮
  "menu_button":     [1380, 200],   // 打开设置菜单
  "thumbnail":       [80, 2800],    // 缩略图（拍完返回的预览图）
  "back":            [50, 100],     // 返回按钮
  "gallery":         [80, 2800]     // 相册入口
}
```

### 3.3 modes（模式枚举）

每个模式独立维护一份设置项坐标表。

```json
"modes": {
  "拍照": {
    "前置": {
      "闪光灯":  { "type": "inline",  "options": [...] },
      "HDR":     { "type": "inline",  "options": [...] },
      "曝光补偿": { "type": "menu",    "menu_path": [...] }
    },
    "后置": { ... }
  },
  "夜景": {
    "后置": { ... }
  }
}
```

#### 3.3.1 设置项类型

**类型 A：inline（预览直控）**

```json
"闪光灯": {
  "type": "inline",
  "options": [
    { "value": "关闭", "coord": [200, 1500] },
    { "value": "自动", "coord": [400, 1500] },
    { "value": "开启", "coord": [600, 1500] }
  ]
}
```

**类型 B：menu（菜单深层）**

```json
"曝光补偿": {
  "type": "menu",
  "menu_path": ["menu_button", "摄影设置", "曝光补偿"],
  "options": [
    { "value": "-1EV", "coord": [200, 1800] },
    { "value": "0",    "coord": [400, 1800] },
    { "value": "+1EV", "coord": [600, 1800] }
  ]
}
```

`menu_path` 是从根菜单到目标设置项的**名称路径**，运行时按顺序点击。

### 3.4 滑动支持

滑动不存为独立设置项，而是公共方法的两个端点：

```json
"common_widgets": {
  "mode_strip_left":  [50, 2200],    // 模式条最左
  "mode_strip_right": [1390, 2200]   // 模式条最右
}
```

调用：
```python
start, end = fw.swipe("mode_strip_left", "mode_strip_right")
adb.swipe(*start, *end, duration_ms=300)
```

---

## 4. 公共方法 API（用户接口）

所有公共方法**返回坐标，不执行 ADB**。

### 4.1 方法清单

| 方法 | 返回 | 说明 |
|------|------|------|
| `switch_camera(target)` | `Coordinate` | 切前后置（`"前置"` / `"后置"`） |
| `switch_mode(mode)` | `Coordinate` | 切模式（按当前 facing 找入口坐标） |
| `set_setting(name, value)` | `Coordinate` | 设置项选择（自动判断 inline/menu） |
| `take_photo()` | `Coordinate` | 拍照按钮 |
| `open_menu()` | `Coordinate` | 打开菜单按钮 |
| `swipe(from, to)` | `tuple[Coordinate, Coordinate]` | 滑动两端点 |
| `tap_widget(name)` | `Coordinate` | 通用 tap（直接查 common_widgets） |

### 4.2 调用约定

```python
from camera_coord import Framework

fw = Framework(project="CameraA_Xiaomi13Pro")

# 单步操作
coord = fw.switch_camera("前置")
adb.tap(*coord)

# 复杂操作链
fw.switch_mode("夜景")
fw.switch_camera("后置")
fw.set_setting("闪光灯", "自动")
fw.set_setting("ISO", "800")
fw.take_photo()

# 滑动
start, end = fw.swipe("mode_strip_left", "mode_strip_right")
adb.swipe(*start, *end, duration_ms=300)
```

### 4.3 上下文追踪（轻量）

框架**不维护** `current_facing` / `current_mode` 状态——这是测试用例自己的责任。
框架方法调用时**需要显式传入** facing（如果业务需要）：

```python
# 显式指定 facing
fw.set_setting("闪光灯", "自动", facing="后置", mode="拍照")
```

或在 `Framework` 初始化时设定一次默认上下文：

```python
fw = Framework(
    project="CameraA_Xiaomi13Pro",
    default_facing="后置",
    default_mode="拍照",
)
```

---

## 5. 公共方法内部实现

```
framework.switch_camera(target)
    ↓
1. coord = project.common_widgets["switch_camera"]
2. return Coordinate(x, y)

framework.set_setting(name, value, facing, mode)
    ↓
1. setting = project.modes[mode][facing][name]
2. if setting.type == "inline":
       coord = setting.find_option(value).coord
       return coord
   elif setting.type == "menu":
       path_coords = [project.common_widgets[n] for n in setting.menu_path]
       option_coord = setting.find_option(value).coord
       return (path_coords, option_coord)  # 调用方按顺序点击

framework.take_photo()
    ↓
1. return project.common_widgets["shutter"]

framework.swipe(from, to)
    ↓
1. start = project.common_widgets[from]
2. end = project.common_widgets[to]
3. return (start, end)
```

---

## 6. 离线标注流程

### 6.1 现状：半自动 + 人工确认（B）

```
1. 用户在设备上手动切换到目标 (mode, facing, setting, value) 状态
2. 工具执行: adb shell uiautomator dump /sdcard/ui.xml
3. 工具解析 XML，提取所有节点的 bounds，输出 JSON 骨架
4. 用户在编辑器中补全归属（哪个节点对应哪个 (mode, facing, setting, value)）
5. 工具校验 JSON，写入 projects/<project>.json
```

### 6.2 理想：自动归类 + 人工确认（C，预留扩展点）

```
1. 启发式归类器（class + text 模式匹配）猜测归属
2. LLM 兜底（截图 + 节点列表 → 建议归属）
3. 人工在 JSON 里确认/改写
```

**架构预留**：标注工具拆成 **dump 解析器** + **归类器接口（Classifier）**。现状用 `ManualClassifier`（人工），未来加 `HeuristicClassifier`、`LLMClassifier`。

---

## 7. 设备适配

| 维度 | 策略 |
|------|------|
| **分辨率 / 屏幕尺寸** | 不同设备型号 → 不同的 JSON（`CameraA_Xiaomi13Pro.json` / `CameraA_Xiaomi14Pro.json`） |
| **型号判断** | 离线标注时人工判断要不要新建项目 JSON |
| **设备预检测** | keyevent 控制解锁 / 亮屏 / 切到目标 App（**框架不负责，由用户自己写**） |
| **多设备并行** | 同一份 JSON 复用（同型号多台设备），框架无状态支持 |

---

## 8. 部署形态

| 形态 | 描述 |
|------|------|
| **Python SDK** | `from camera_coord import Framework` |
| **CLI 工具** | `camera-coord query --project A --widget shutter` |
| **离线标注工具** | `camera-coord annotate dump/parse/validate` |

入口：`Framework(project=...)` 或 CLI 子命令。

---

## 9. 等待 / 同步 / 验证 —— 不在框架内

- 等待：`time.sleep(N)`，由用例自己加
- 同步：无（用例自己 sleep 或靠设备稳定性）
- 验证：`adb shell ls /sdcard/DCIM/` 或读 logcat，由用例自己写
- 日志：`logging` 由用例自己配

框架**保持极简**，只交付坐标。

---

## 10. 设计决策记录

| # | 决策 | 选择 | 理由 |
|---|------|------|------|
| 1 | 屏幕识别路线 | A：UI 层级 Dump | dump 仅用于离线标注辅助 |
| 2 | 控件匹配 | 离线人工标注 | 项目一直在变，自动匹配扛不住 |
| 3 | 坐标形式 | 固定像素坐标 | 用户拍板，简单 |
| 4 | 设置项范围 | C：预览 + 菜单 | 覆盖全部相机场景 |
| 5 | JSON 结构 | a 嵌套树 + iii 每模式独立 | 简洁无耦合 |
| 6 | 设备适配层 | 分辨率/型号判断 + keyevent 预检测 | 用户拍板 |
| 7 | 多设备策略 | iii 项目 JSON 复用 | 单项目内布局稳定 |
| 8 | 测试用例写法 | Python SDK + YAML 混合（用户视角） | 工程极简 |
| 9 | 离线工具 | CLI + 编辑器，B 现状 C 理想 | 用户拍板 |
| 10 | 框架部署 | CLI + SDK，Python | 用户拍板 |
| 11 | 结果验证 | B 控件存在性检查 | 用户拍板 |
| 12 | 等待同步 | A 固定 sleep | 用户拍板 |
| 13 | 失败排查 | B 日志 + dump 快照 | 用户拍板 |
| 14 | **核心边界** | **只做 dump → 坐标查询，不参与用例执行** | **用户最终拍板** |

---

## 11. 文件结构

```
camera_coordinate_framework/
├── camera_coord/                  # 主包
│   ├── __init__.py
│   ├── types.py                   # dataclass 数据类型
│   ├── exceptions.py              # 自定义异常
│   ├── adb_helper.py              # ADB dump 封装
│   ├── layout_parser.py           # 解析 dump XML
│   ├── project_loader.py          # 加载项目 JSON
│   ├── coord_resolver.py          # 核心查询引擎
│   ├── business_methods.py        # 公共方法实现
│   ├── framework.py               # Façade 入口
│   └── cli.py                     # CLI 入口
├── projects/                      # 项目坐标库
│   └── example_project.json       # 示例配置
├── tools/
│   └── annotate_tool.py           # 离线标注脚本
├── tests/                         # pytest 单元测试
│   ├── conftest.py
│   ├── test_coord_resolver.py
│   ├── test_project_loader.py
│   └── test_business_methods.py
├── docs/
│   └── ARCHITECTURE.md            # 本文档
├── README.md
├── pyproject.toml
└── .gitignore
```

---

## 12. 后续路线

1. **阶段 1**（最小可用）：types + project_loader + coord_resolver + framework + 示例 JSON
2. **阶段 2**：adb_helper + CLI + 离线标注工具（dump/parse/validate）
3. **阶段 3**：单元测试 + 集成测试
4. **阶段 4**：高级归类器（启发式 + LLM 预留接口）