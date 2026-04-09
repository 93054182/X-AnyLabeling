# SAM3 边界框/掩码过滤器实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标:** 在 Remote-Server 的 SAM3 模型下添加边界框/掩码过滤器 UI 组件，允许用户独立控制两种类型的 shape 显示

**架构:**
1. UI 层：在 auto_labeling.ui 中添加 QGroupBox 和两个 QCheckBox
2. 逻辑层：在 auto_labeling.py 中处理 checkbox 信号和显示逻辑
3. 模型层：在 remote_server.py 中添加过滤状态存储和结果过滤逻辑

**技术栈:** PyQt6, Python 3.8+

---

## Task 1: 修改 UI 文件添加过滤器组件

**文件:**
- Modify: `anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.ui`

在 `horizontalSpacer`（第 472-484 行）之前添加 SAM3 过滤器组件。

- [ ] **Step 1: 在 auto_labeling.ui 中添加 SAM3 过滤器组件**

在第 471 行 `</item>` 之后，第 472 行 `<item>` 之前插入以下内容：

```xml
     <item>
      <widget class="QGroupBox" name="sam3_filter_group">
       <property name="title">
        <string>输出过滤</string>
       </property>
       <property name="visible">
        <bool>false</bool>
       </property>
       <layout class="QVBoxLayout" name="sam3_filter_layout">
        <property name="spacing">
         <number>2</number>
        </property>
        <property name="leftMargin">
         <number>6</number>
        </property>
        <property name="topMargin">
         <number>6</number>
        </property>
        <property name="rightMargin">
         <number>6</number>
        </property>
        <property name="bottomMargin">
         <number>6</number>
        </property>
        <item>
         <widget class="QCheckBox" name="sam3_bbox_checkbox">
          <property name="text">
           <string>边界框</string>
          </property>
          <property name="checked">
           <bool>false</bool>
          </property>
         </widget>
        </item>
        <item>
         <widget class="QCheckBox" name="sam3_mask_checkbox">
          <property name="text">
           <string>掩码</string>
          </property>
          <property name="checked">
           <bool>true</bool>
          </property>
         </widget>
        </item>
       </layout>
      </widget>
     </item>
```

- [ ] **Step 2: 运行应用验证 UI 加载**

```bash
cd C:\Users\22207\Desktop\X-AnyLabeling
xanylabeling
```

预期：应用正常启动，组件加载成功（但默认隐藏）

- [ ] **Step 3: 提交**

```bash
git add anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.ui
git commit -m "feat(sam3): 添加边界框/掩码过滤器 UI 组件"
```

---

## Task 2: 在 auto_labeling.py 中添加 checkbox 初始化和信号连接

**文件:**
- Modify: `anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.py`

- [ ] **Step 1: 在 __init__ 方法中添加 checkbox 初始化**

找到 `# --- Configuration for: button_reset_tracker ---` 部分（约第 216 行），在其后添加：

```python
# --- Configuration for: SAM3 filter checkboxes ---
self.sam3_filter_group.hide()
self.sam3_bbox_checkbox.stateChanged.connect(self.on_sam3_filter_changed)
self.sam3_mask_checkbox.stateChanged.connect(self.on_sam3_filter_changed)
```

- [ ] **Step 2: 添加 checkbox 状态变化处理方法**

在文件末尾添加新方法（在其他 on_* 方法附近）：

```python
def on_sam3_filter_changed(self):
    """Handle SAM3 filter checkbox state changes"""
    if self.model_manager.loaded_model:
        model = self.model_manager.loaded_model
        if hasattr(model, "set_sam3_filter_state"):
            model.set_sam3_filter_state(
                self.sam3_bbox_checkbox.isChecked(),
                self.sam3_mask_checkbox.isChecked(),
            )
```

- [ ] **Step 3: 提交**

```bash
git add anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.py
git commit -m "feat(sam3): 添加过滤器 checkbox 初始化和信号连接"
```

---

## Task 3: 在 auto_labeling.py 中添加 SAM3 模型检测逻辑

**文件:**
- Modify: `anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.py`

修改 `update_remote_server_widgets` 方法，在选择 SAM3 模型时显示过滤器。

- [ ] **Step 1: 修改 update_remote_server_widgets 方法**

找到 `def update_remote_server_widgets(self, model_id):` 方法（约第 1496 行），在方法末尾（在 `for widget_item in widgets_config:` 循环之后）添加：

```python
# Show SAM3 filter only for segment_anything_3 model
if model_id == "segment_anything_3":
    self.sam3_filter_group.show()
else:
    self.sam3_filter_group.hide()
```

确保这段代码在 `for widget_item in widgets_config:` 循环之后，且在方法返回之前。

- [ ] **Step 2: 提交**

```bash
git add anylabeling/views/labeling/widgets/auto_labeling/auto_labeling.py
git commit -m "feat(sam3): 添加模型检测逻辑控制过滤器显示"
```

---

## Task 4: 在 remote_server.py 中添加过滤器状态存储

**文件:**
- Modify: `anylabeling/services/auto_labeling/remote_server.py`

- [ ] **Step 1: 在 __init__ 方法中添加过滤器状态变量**

找到 `# Segment Anything 3` 部分（约第 62 行），在现有 SAM3 变量之后添加：

```python
# Segment Anything 3 filter state
self.bbox_enabled = False
self.mask_enabled = True
```

- [ ] **Step 2: 添加 set_sam3_filter_state 方法**

在 `set_mask_fineness` 方法（约第 136 行）之后添加新方法：

```python
def set_sam3_filter_state(self, bbox_enabled, mask_enabled):
    """Set SAM3 bbox and mask filter state.

    Args:
        bbox_enabled: Whether to show bounding boxes (rectangle shapes)
        mask_enabled: Whether to show masks (polygon shapes)
    """
    self.bbox_enabled = bbox_enabled
    self.mask_enabled = mask_enabled
```

- [ ] **Step 3: 提交**

```bash
git add anylabeling/services/auto_labeling/remote_server.py
git commit -m "feat(sam3): 添加过滤器状态存储和设置方法"
```

---

## Task 5: 在 remote_server.py 中添加结果过滤逻辑

**文件:**
- Modify: `anylabeling/services/auto_labeling/remote_server.py`

- [ ] **Step 1: 添加 _filter_sam3_shapes 辅助方法**

在 `_reset_video_session` 方法之前（约第 824 行）添加新方法：

```python
def _filter_sam3_shapes(self, shapes):
    """Filter shapes based on SAM3 bbox/mask checkbox state.

    Args:
        shapes: List of Shape objects to filter

    Returns:
        Filtered list of Shape objects
    """
    if not self.current_model_id:
        return shapes

    # Only apply filter for SAM3 model
    if self.current_model_id != "segment_anything_3":
        return shapes

    filtered = []
    for shape in shapes:
        if shape.shape_type == "rectangle" and self.bbox_enabled:
            filtered.append(shape)
        elif shape.shape_type in ("polygon", "rotation") and self.mask_enabled:
            filtered.append(shape)
    return filtered
```

- [ ] **Step 2: 在 predict_shapes 中应用过滤**

找到 `predict_shapes` 方法中构建 `AutoLabelingResult` 的部分（约第 256 行）：

```python
return AutoLabelingResult(
    shapes, replace=replace, description=description
)
```

将其修改为：

```python
# Apply SAM3 filter if applicable
filtered_shapes = self._filter_sam3_shapes(shapes)
return AutoLabelingResult(
    filtered_shapes, replace=replace, description=description
)
```

- [ ] **Step 3: 在 _handle_video_prompt 中应用过滤**

找到 `_handle_video_prompt` 方法中构建 `AutoLabelingResult` 的部分（约第 418 行）：

```python
return AutoLabelingResult(
    shapes, replace=self.replace, description=""
)
```

将其修改为：

```python
# Apply SAM3 filter if applicable
filtered_shapes = self._filter_sam3_shapes(shapes)
return AutoLabelingResult(
    filtered_shapes, replace=self.replace, description=""
)
```

- [ ] **Step 4: 在 _handle_video_point_prompt 中应用过滤**

找到 `_handle_video_point_prompt` 方法中构建 `AutoLabelingResult` 的部分（约第 612 行）：

```python
return AutoLabelingResult(shapes, replace=False, description="")
```

将其修改为：

```python
# Apply SAM3 filter if applicable
filtered_shapes = self._filter_sam3_shapes(shapes)
return AutoLabelingResult(filtered_shapes, replace=False, description="")
```

- [ ] **Step 5: 提交**

```bash
git add anylabeling/services/auto_labeling/remote_server.py
git commit -m "feat(sam3): 添加结果过滤逻辑"
```

---

## Task 6: 测试和验证

- [ ] **Step 1: 启动应用并加载 Remote-Server 模型**

```bash
cd C:\Users\22207\Desktop\X-AnyLabeling
xanylabeling
```

操作：
1. 点击 AI 按钮打开自动标注面板
2. 选择 Remote-Server 模型
3. 选择 SAM3 (Segment Anything 3) 模型

预期：显示"输出过滤"groupbox，包含"边界框"（未勾选）和"掩码"（已勾选）

- [ ] **Step 2: 测试默认过滤状态**

使用 SAM3 进行推理，预期结果：只显示掩码（polygon），不显示边界框（rectangle）

- [ ] **Step 3: 测试边界框勾选**

勾选"边界框"checkbox，进行推理。预期：同时显示边界框和掩码

- [ ] **Step 4: 测试只显示边界框**

取消"掩码"勾选，只保留"边界框"勾选。预期：只显示边界框

- [ ] **Step 5: 测试取消全部勾选**

取消两个 checkbox 的勾选。预期：不显示任何 shape

- [ ] **Step 6: 测试模型切换验证隐藏逻辑**

切换到其他 Remote-Server 模型（非 SAM3）。预期："输出过滤"groupbox 隐藏

- [ ] **Step 7: 最终提交**

如果测试通过：

```bash
git commit --allow-empty -m "test(sam3): 验证边界框/掩码过滤器功能"
```

---

## 验证检查清单

完成所有任务后，验证以下功能点：

- [ ] UI 组件正确添加且默认隐藏
- [ ] 选择 SAM3 模型时显示过滤器
- [ ] 选择其他模型时隐藏过滤器
- [ ] 默认状态：掩码选中，边界框未选中
- [ ] 只显示掩码：正确过滤
- [ ] 只显示边界框：正确过滤
- [ ] 同时显示两者：正确过滤
- [ ] 都不选中：不显示任何 shape
- [ ] checkbox 状态变化时正确通知 model
