# SAM3 Remote Server 添加 BBox/Mask 过滤控件 - 设计文档

**日期:** 2026-04-09
**状态:** 已批准

## 需求概述

在 SAM3 Remote Server 自动标注模式下，添加两个复选框控件用于过滤服务器返回的形状类型：

- **"边界框"** - 控制是否显示矩形 (rectangle) 类型的形状
- **"掩码"** - 控制是否显示多边形 (polygon) 类型的形状

### 核心特性

| 特性 | 说明 |
|------|------|
| 过滤方式 | 本地过滤，不发送到服务器 |
| 处理对象 | 仅处理服务器返回的新数据，不影响已有标注 |
| 默认状态 | 仅 "掩码" 选中 |
| 标签文本 | 中文："边界框" 和 "掩码" |
| 布局 | 水平排列 |

---

## 设计方案

### 方案选择：方案 A - 固定控件 + 动态显示

在 `auto_labeling.ui` 中添加固定控件，通过 `update_visible_widgets` 机制控制显示。

**选择理由：**
1. 实现简单，与现有控件（如 `toggle_preserve_existing_annotations`）一致
2. 易于维护和调试
3. 不需要修改服务器端代码

---

## 实现细节

### 1. UI 布局 (auto_labeling.ui)

在 `toggle_preserve_existing_annotations` 和 `button_classes_filter` 之间添加：

```xml
<item>
  <widget class="QCheckBox" name="checkbox_bbox">
    <property name="text">
      <string>边界框</string>
    </property>
  </widget>
</item>
<item>
  <widget class="QCheckBox" name="checkbox_mask">
    <property name="text">
      <string>掩码</string>
    </property>
  </widget>
</item>
```

### 2. 代码修改 (auto_labeling.py)

#### 2.1 添加到 `supported_remote_widgets` 列表

```python
supported_remote_widgets = [
    # ... 现有控件 ...
    "checkbox_bbox",
    "checkbox_mask",
]
```

#### 2.2 初始化默认状态（`__init__` 方法）

```python
# 默认：仅 mask 选中
self.checkbox_mask.setChecked(True)
self.checkbox_bbox.setChecked(False)
```

#### 2.3 添加过滤逻辑

```python
def filter_shapes_by_bbox_mask_preference(self, shapes):
    """根据 bbox/checkbox 选择过滤形状

    Args:
        shapes: 从服务器返回的形状列表

    Returns:
        过滤后的形状列表
    """
    show_bbox = self.checkbox_bbox.isChecked()
    show_mask = self.checkbox_mask.isChecked()

    filtered = []
    for shape in shapes:
        if shape.shape_type == "rectangle" and show_bbox:
            filtered.append(shape)
        elif shape.shape_type in ("polygon", "point") and show_mask:
            filtered.append(shape)
        else:
            # 其他类型默认保留
            filtered.append(shape)
    return filtered
```

#### 2.4 修改结果处理流程

在处理服务器返回的 `AutoLabelingResult` 时，先过滤再发送信号。

---

## 数据流

```
┌─────────────────────────────────────────────────────────────┐
│                     服务器返回数据                            │
│  shapes: [rectangle, polygon, polygon, rectangle, ...]      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              filter_shapes_by_bbox_mask_preference()          │
│  ┌─────────────┬─────────────┬─────────────┐               │
│  │ ☑ 边界框    │ ☑ 掩码      │ 结果        │               │
│  ├─────────────┼─────────────┼─────────────┤               │
│  │ ✓          │ ✓          │ 全部保留    │               │
│  │ ✓          │ ✗          │ 仅 rectangle │               │
│  │ ✗          │ ✓          │ 仅 polygon   │               │  ← 默认
│  │ ✗          │ ✗          │ 空          │               │
│  └─────────────┴─────────────┴─────────────┘               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   添加到画布显示                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 边界情况处理

| 情况 | 处理方式 |
|------|----------|
| 两个都选中 | 显示所有类型 |
| 两个都不选 | 显示空（不报错） |
| 服务器返回其他 shape_type | 默认保留（不过滤） |
| 切换 checkbox | 仅影响新收到的数据 |
| 非远程服务器模型 | 控件隐藏 |

---

## 修改文件清单

| 文件 | 修改内容 |
|------|----------|
| `anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.ui` | 添加两个 QCheckBox 控件 |
| `anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.py` | 添加过滤逻辑、初始化、绑定信号 |

---

## 验证步骤

### 功能测试
1. 选择 Remote Server 模型 → 确认两个 checkbox 显示
2. 选择其他模型 → 确认两个 checkbox 隐藏
3. 默认状态：仅 "掩码" 选中

### 过滤测试
1. 仅选 "边界框" → 运行推理 → 只显示 rectangle
2. 仅选 "掩码" → 运行推理 → 只显示 polygon
3. 两者都选 → 运行推理 → 显示全部

### 兼容性测试
1. 确认不影响已有标注
2. 确认不影响其他模型的推理结果
