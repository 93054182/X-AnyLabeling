# SAM3 边界框/掩码过滤器设计

**日期**: 2026-04-09
**状态**: 设计中

## 概述

在 Remote-Server 的 SAM3 (Segment Anything 3) 模型下，添加一个输出过滤器 UI 组件，允许用户选择是否显示边界框（rectangle）和掩码（polygon）。

## 需求背景

SAM3 模型推理后可能同时返回边界框和掩码两种类型的 shape。用户希望能够独立控制每种类型的显示状态：
- 默认只显示掩码
- 可通过 checkbox 切换边界框和掩码的显示

## 功能需求

### UI 组件

在 auto_labeling 工具栏的 spacer（位于关闭按钮前）之前，添加：

1. **QGroupBox**: "输出过滤"
   - 默认隐藏
   - 仅在选择 SAM3 模型时显示

2. **QCheckBox**: "边界框"
   - 默认未选中
   - 控制 `shape_type == "rectangle"` 的 shape 是否显示

3. **QCheckBox**: "掩码"
   - 默认选中
   - 控制 `shape_type == "polygon"` 的 shape 是否显示

### 显示逻辑

- 当 `remote_server_select_combobox` 选择的 model_id 为 `"segment_anything_3"` 时，显示 QGroupBox
- 否则隐藏 QGroupBox

### 过滤逻辑

在 `RemoteServer.predict_shapes()` 返回结果前，根据 checkbox 状态过滤：

```python
if self.current_model_id == "segment_anything_3":
    filtered_shapes = []
    for shape in shapes:
        if shape.shape_type == "rectangle" and self.bbox_enabled:
            filtered_shapes.append(shape)
        elif shape.shape_type in ("polygon", "rotation") and self.mask_enabled:
            filtered_shapes.append(shape)
    shapes = filtered_shapes
```

## 技术设计

### 涉及文件

1. **anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.ui**
   - 添加 QGroupBox 和两个 QCheckBox

2. **anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.py**
   - 连接 checkbox 信号
   - 根据 model_id 控制 group 显示/隐藏
   - 新增方法获取 filter 状态

3. **anylabeling/services/auto_labeling/remote_server.py**
   - 新增 `set_sam3_filter_state(bbox_enabled, mask_enabled)` 方法
   - 新增成员变量 `self.bbox_enabled`, `self.mask_enabled`
   - 在 `predict_shapes()` 中应用过滤逻辑

### 实现步骤

#### Step 1: UI 修改

在 `auto_labeling.ui` 的 `horizontalSpacer` 之前添加组件。

#### Step 2: Python UI 逻辑

在 `auto_labeling.py` 中：

```python
# 初始化 checkbox
self.sam3_filter_group.hide()
self.sam3_bbox_checkbox.stateChanged.connect(self.on_sam3_filter_changed)
self.sam3_mask_checkbox.stateChanged.connect(self.on_sam3_filter_changed)

def on_sam3_filter_changed(self):
    # 通知 model 更新过滤状态
    if self.model_manager.loaded_model:
        model = self.model_manager.loaded_model
        if hasattr(model, 'set_sam3_filter_state'):
            model.set_sam3_filter_state(
                self.sam3_bbox_checkbox.isChecked(),
                self.sam3_mask_checkbox.isChecked()
            )

def update_remote_server_widgets(self, model_id):
    # 现有逻辑...
    # 新增：判断是否显示 SAM3 过滤器
    if model_id == "segment_anything_3":
        self.sam3_filter_group.show()
    else:
        self.sam3_filter_group.hide()
```

#### Step 3: Model 逻辑

在 `remote_server.py` 中：

```python
def __init__(self, model_config, on_message) -> None:
    # 现有代码...
    self.bbox_enabled = False
    self.mask_enabled = True

def set_sam3_filter_state(self, bbox_enabled, mask_enabled):
    self.bbox_enabled = bbox_enabled
    self.mask_enabled = mask_enabled

def predict_shapes(self, image, image_path=None, text_prompt=None, run_tracker=False):
    # 现有逻辑...
    # 返回前添加过滤
    if self.current_model_id == "segment_anything_3":
        shapes = self._filter_sam3_shapes(shapes)
    return AutoLabelingResult(shapes, replace=replace, description=description)

def _filter_sam3_shapes(self, shapes):
    filtered = []
    for shape in shapes:
        if shape.shape_type == "rectangle" and self.bbox_enabled:
            filtered.append(shape)
        elif shape.shape_type in ("polygon", "rotation") and self.mask_enabled:
            filtered.append(shape)
    return filtered
```

### 测试计划

1. **UI 显示测试**
   - 选择 Remote-Server → SAM3 模型，验证过滤器 group 显示
   - 切换到其他模型，验证过滤器 group 隐藏

2. **功能测试**
   - 默认状态：只显示掩码
   - 勾选边界框：同时显示边界框和掩码
   - 只勾选边界框：只显示边界框
   - 取消所有勾选：不显示任何 shape

## 风险与注意事项

1. **Video 模式兼容性**: SAM3 视频传播模式也需要应用相同的过滤逻辑
2. **UI 文本**: 固定使用中文，无需国际化
