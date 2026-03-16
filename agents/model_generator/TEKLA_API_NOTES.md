# Tekla Open API 学习笔记
> 来源：TSOpenAPIExamples (branch: 2024)
> 更新：2026-03-15

---

## 1. 基础连接

### 连接 Tekla 模型
```python
import clr
clr.AddReference("Tekla.Structures.Model")
clr.AddReference("Tekla.Structures.Geometry3d")

from Tekla.Structures.Model import Model

model = Model()
if not model.GetConnectionStatus():
    raise Exception("Tekla 未连接，请先打开 Tekla Structures")

# 提交所有修改（必须在所有 Insert/Modify 之后调用）
model.CommitChanges()
```

---

## 2. 创建梁 (Beam)

```python
from Tekla.Structures.Model import Beam
from Tekla.Structures.Geometry3d import Point

beam = Beam()
beam.StartPoint = Point(0, 0, 4500)       # 单位：毫米
beam.EndPoint = Point(6000, 0, 4500)
beam.Profile.ProfileString = "HEA400"     # 截面规格
beam.Material.MaterialString = "S235JR"   # 材料等级
beam.Insert()

model.CommitChanges()
```

**注意事项：**
- `StartPoint` / `EndPoint` 单位全部是 **毫米**
- `ProfileString` 必须与 Tekla 截面库中的名称完全一致（区分大小写）
- 梁也可以用来创建柱，只需让 StartPoint.Z < EndPoint.Z 且 X/Y 相同

---

## 3. 创建柱 (Column)

Tekla API 中柱和梁都是 `Beam` 类，区分方式是起止点方向：

```python
from Tekla.Structures.Model import Beam
from Tekla.Structures.Geometry3d import Point

column = Beam()
column.StartPoint = Point(0, 0, 0)         # 底部
column.EndPoint = Point(0, 0, 4500)        # 顶部（Z方向）
column.Profile.ProfileString = "HSS203X203X9.5"
column.Material.MaterialString = "A500"
column.Insert()

model.CommitChanges()
```

---

## 4. 创建直角网格 (Grid)

```python
from Tekla.Structures.Model import Grid

grid = Grid()
# 坐标格式：空格分隔的绝对坐标值（毫米）
# 也支持重复格式：如 "0 3*6000" 表示 0, 6000, 12000, 18000
grid.CoordinateX = "0 6000 12000 18000"   # X 轴轴线位置
grid.CoordinateY = "0 8000 16000"          # Y 轴轴线位置
grid.CoordinateZ = "0 4500 9000"           # 标高（楼层）

# 轴线标签（可选，数量需与坐标数量一致）
grid.LabelX = "A B C D"
grid.LabelY = "1 2 3"
grid.LabelZ = "L1 L2 L3"

grid.Insert()
model.CommitChanges()
```

**⚠️ 重要：坐标格式**
- 坐标是**绝对值**（毫米），空格分隔
- 支持重复语法：`"0 5*6000"` = `"0 6000 12000 18000 24000 30000"`
- `LabelX/Y/Z` 的数量必须比坐标数量多1（因为有 n+1 个区间标签）或完全匹配

---

## 5. 创建放射状网格 (RadialGrid)

```python
from Tekla.Structures.Model import RadialGrid
from Tekla.Structures.Geometry3d import Angle

# 角度单位设置
Angle.CurrentUnitType = Angle.UnitType.Degrees

radial_grid = RadialGrid()
radial_grid.IsMagnetic = False
radial_grid.RadialCoordinates = "0 5*5000"      # 半径方向：0, 5000, 10000...
radial_grid.AngularCoordinates = "0 6*15"        # 角度方向：0, 15, 30...度
radial_grid.CoordinateZ = "0 5000 10000"         # 标高
radial_grid.RadialLabels = "R1 R2 R3 R4 R5"
radial_grid.AngularLabels = "A1 A2 A3 A4 A5 A6 A7"
radial_grid.LabelZ = "Z1 Z2 Z3"

# 延伸长度（可选）
radial_grid.ArcStartExtension = 2000.0
radial_grid.ArcEndExtension = 2000.0
radial_grid.ExtensionBelowZ = 2000.0
radial_grid.ExtensionAboveZ = 2000.0

radial_grid.Insert()
model.CommitChanges()
```

**修改已有网格：**
```python
radial_grid.Modify()   # 修改后调用 Modify()，不是重新 Insert()
model.CommitChanges()
```

---

## 6. 截面规格 (CrossSection)

```python
from Tekla.Structures.Catalogs import CrossSection

section = CrossSection("W310X97")
if section.Select():
    outer_surface = section.OuterSurface
    for point in outer_surface:
        print(f"X={point.X}, Y={point.Y}")
        print(f"Chamfer type: {point.Chamfer.Type}")
```

---

## 7. 遍历模型对象

```python
from Tekla.Structures.Model import ModelObjectEnumerator
from Tekla.Structures.Model.UI import ModelObjectSelector

selector = ModelObjectSelector()
obj_enum = selector.GetSelectedObjects()  # 获取用户选中的对象

while obj_enum.MoveNext():
    part = obj_enum.Current  # 转换为具体类型
    if part:
        print(f"ID: {part.Identifier.ID}")
        print(f"Profile: {part.Profile.ProfileString}")
        print(f"Material: {part.Material.MaterialString}")
```

---

## 8. 导出 XML 关键要点

```python
# 获取构件属性
mark = ""
part.GetReportProperty("CAST_UNIT_POS", mark)  # 注意：Python 中需用 ref 方式

# 获取 Assembly
assembly = part.GetAssembly()
assembly.GetReportProperty("LENGTH", length)
assembly.GetReportProperty("WEIGHT", weight)

# 获取子构件
secondaries = assembly.GetSecondaries()
sub_assemblies = assembly.GetSubAssemblies()
```

---

## 9. 坐标系与变换平面

```python
from Tekla.Structures.Model import WorkPlaneHandler, TransformationPlane

plane_handler = model.GetWorkPlaneHandler()
current_plane = plane_handler.GetCurrentTransformationPlane()

# 切换到构件局部坐标系
bolt_coord_sys = bolt.GetCoordinateSystem()
bolt_plane = TransformationPlane(bolt_coord_sys)
plane_handler.SetCurrentTransformationPlane(bolt_plane)

# 操作完成后恢复原坐标系
plane_handler.SetCurrentTransformationPlane(current_plane)
```

---

## 10. Python 调用注意事项（⚠️ 重要）

### 必须使用 pythonnet (clr)
```bash
pip install pythonnet
```

### DLL 引用路径
Tekla 的 DLL 默认在安装目录，需要先添加路径：
```python
import sys
import clr

# Tekla 2025 默认安装路径
tekla_path = r"C:\Program Files\Tekla Structures\2025.0\bin"
sys.path.append(tekla_path)

clr.AddReference("Tekla.Structures.Model")
clr.AddReference("Tekla.Structures.Geometry3d")
clr.AddReference("Tekla.Structures.Datatype")
clr.AddReference("Tekla.Structures.Catalogs")
```

### Python vs C# 差异
| C# | Python |
|----|--------|
| `new Model()` | `Model()` |
| `new Point(x, y, z)` | `Point(x, y, z)` |
| `ref` 参数 | 直接赋值，部分需特殊处理 |
| `foreach (var x in list)` | `for x in list:` |
| `myEnum.MoveNext()` | 同样使用 `.MoveNext()` |

---

## 11. 常见坑

1. **未调用 CommitChanges()** — Insert() 后必须 CommitChanges()，否则模型不更新
2. **坐标单位** — 全部是毫米，不是米！
3. **ProfileString 拼写** — 必须与 Tekla 截面库完全一致，如 `W310X97` 不能写成 `W310x97`
4. **Modify() vs Insert()** — 新建用 Insert()，修改已有对象用 Modify()
5. **RadialGrid 的角度** — 坐标值是相邻差值，不是绝对值（如 `"0 6*15"` 不是 6 个 15 度，是从 0 开始每隔 15 度）
6. **Grid 标签数量** — LabelX 标签数量需与轴线数量对应
7. **Tekla 必须开着** — GetConnectionStatus() 返回 False 时必须停止，不能继续

---

## 12. 明天任务快速参考

从 XML 建轴线网格的核心流程：
```python
import xml.etree.ElementTree as ET

tree = ET.parse("extracted.xml")
root = tree.getroot()

# 解析轴线
x_coords = [gl.get("coordinate") for gl in root.findall(".//GridLine[@direction='X']")]
y_coords = [gl.get("coordinate") for gl in root.findall(".//GridLine[@direction='Y']")]
z_coords = [e.get("value") for e in root.findall(".//Elevation")]

# 创建网格
grid = Grid()
grid.CoordinateX = " ".join(x_coords)
grid.CoordinateY = " ".join(y_coords)
grid.CoordinateZ = " ".join(z_coords)
grid.Insert()
model.CommitChanges()
```
