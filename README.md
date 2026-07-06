# Camera Coordinate Framework

> 相机 App 自动化测试 · 坐标转化框架
> 基于布局树 dump + 项目 JSON 坐标库，给测试用例返回坐标，由用例自行 ADB 操作。

**只做一件事**：把"控件名"转成"项目特有的像素坐标"。不参与用例执行、不做 ADB、不做验证。

---

## 与 05_camera_aw 的关系

```
05_camera_aw/                  ← 已有的关键字驱动项目（uiautomator2 + 写死 locator）—— 不动
camera_coordinate_framework/   ← 本框架（dump → 查表 → 返回坐标）—— 新建
```

两套并存，测试用例可自由选择其一或混用。

---

## 5 分钟上手

### 1. 安装（占位）

```bash
pip install camera-coord-framework  # TODO: 发布到 PyPI
```

### 2. 准备项目 JSON

参考 `projects/example_project.json`，按你的设备型号和相机 App 标注一份。

### 3. 调用公共方法

```python
from camera_coord import Framework

fw = Framework(project="CameraA_Xiaomi13Pro")

# 拿到坐标，自己 ADB
coord = fw.switch_camera("前置")
adb.tap(*coord)

# 切模式
fw.switch_mode("夜景")
fw.switch_camera("后置")

# 设置项（自动判断 inline/menu）
fw.set_setting("闪光灯", "自动")
fw.set_setting("曝光补偿", "+1EV")  # menu 路径会自动展开

# 拍照
fw.take_photo()

# 滑动
start, end = fw.swipe("mode_strip_left", "mode_strip_right")
adb.swipe(*start, *end, duration_ms=300)
```

---

## 公共方法一览

| 方法 | 返回 | 说明 |
|------|------|------|
| `switch_camera(target)` | `Coordinate` | 切前后置 |
| `switch_mode(mode)` | `Coordinate` | 切模式 |
| `set_setting(name, value)` | `Coordinate` | 设置项选择 |
| `take_photo()` | `Coordinate` | 拍照按钮 |
| `open_menu()` | `Coordinate` | 打开菜单 |
| `swipe(from, to)` | `tuple[Coordinate, Coordinate]` | 滑动两端点 |
| `tap_widget(name)` | `Coordinate` | 通用 tap |

**重要约定**：框架只返回坐标，不执行任何 ADB 操作。

---

## 项目 JSON 结构（概要）

```json
{
  "project": "CameraA_Xiaomi13Pro",
  "device_model": "Xiaomi 13 Pro",
  "screen_size": [1440, 3200],
  "common_widgets": {
    "shutter":       [540, 2800],
    "switch_camera": [120, 200],
    "menu_button":   [1380, 200]
  },
  "modes": {
    "拍照": {
      "后置": {
        "闪光灯": {
          "type": "inline",
          "options": [
            { "value": "关闭", "coord": [200, 1500] },
            { "value": "自动", "coord": [400, 1500] },
            { "value": "开启", "coord": [600, 1500] }
          ]
        },
        "曝光补偿": {
          "type": "menu",
          "menu_path": ["menu_button", "摄影设置", "曝光补偿"],
          "options": [...]
        }
      }
    }
  }
}
```

完整 Schema 见 [docs/ARCHITECTURE.md §3](docs/ARCHITECTURE.md)。

---

## 离线标注流程

```bash
# 1. 把设备切到目标 (mode, facing, setting, value) 状态
# 2. dump 当前布局
adb shell uiautomator dump /sdcard/ui.xml
adb pull /sdcard/ui.xml

# 3. 用标注工具解析
camera-coord annotate parse ui.xml > projects/CameraA_Xiaomi13Pro.json

# 4. 在编辑器里补全归属
vim projects/CameraA_Xiaomi13Pro.json

# 5. 校验
camera-coord annotate validate projects/CameraA_Xiaomi13Pro.json
```

---

## 文档

- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — 完整架构设计（12 节）
- 项目根目录 README（本文件）— 快速入门

---

## 路线图

| 阶段 | 内容 | 状态 |
|------|------|------|
| 1 | types + project_loader + coord_resolver + framework + 示例 JSON | 🚧 进行中 |
| 2 | adb_helper + CLI + 离线标注工具 | ⏳ 待开始 |
| 3 | pytest 单元测试 + 集成测试 | ⏳ 待开始 |
| 4 | 高级归类器（启发式 + LLM） | ⏳ 未来 |

---

## 边界声明（再强调一次）

本框架**只做**：
- ✅ 读取（dump）Android 布局树
- ✅ 解析布局树为内部结构
- ✅ 加载项目 JSON 坐标库
- ✅ 业务级坐标查询（switch_camera 等）
- ✅ 输出坐标对象

本框架**不做**：
- ❌ ADB tap / swipe / keyevent
- ❌ 测试用例编写 / 执行
- ❌ 结果验证 / 断言
- ❌ 等待 / 同步（sleep 自己加）
- ❌ 日志 / 报告 / 排查
- ❌ 多设备调度

**调用方拿到 Coordinate 后，由调用方自行 adb.tap(x, y)。**

---

## License

TBD