# GeoLabel Studio 遥感与应急标注工作台

本地部署的 FastAPI + React/Vite 可编辑原型。上传原图与元数据 → 转成统一模型 → 在原图上编辑 → 查看首次导入/当前标注或像素 Diff → 保存版本 → 导出含原图、索引 Mask、矢量与文本的 ZIP。前端使用 SVG 实现可交互画布，避免 CSS 缩放后坐标漂移；最终 Mask 由后端统一栅格化。

当前已部署 L20 的 8B / 32B Qwen 与 GPU SAM3，可在界面选择模型大小，默认 8B。见 [L20 运行说明](deploy/L20.md)。新增长按框选顶点、成组拖动、局部补点闭合与独立双语 VQA 窗口，见 [操作说明](docs/VQA-AND-GEOMETRY.md)。

支持明亮云白、护眼青绿、暖纸米色、深海夜色四套主题，右上角切换并自动记忆。模型已改为生成时按需启动、空闲 10 分钟自动释放显存。见 [主题与按需运行](docs/THEMES-AND-RUNTIME.md)。

支持独立的矩形马赛克：多框、块大小、撤销重做与原名打码导出，AI 辅助输入也使用打码图。它不创建语义标签或 Mask 对象。见 [马赛克说明](docs/MOSAIC.md)。SAM 候选可直接「采用并编辑顶点」，已选节点不会被重叠区域遮挡。

左侧图片卡片提供删除按钮，可移至回收站、撤销删除及恢复已保存标注。见 [图片管理](docs/IMAGE-LIBRARY.md)。

新增：Qwen3/Qwen3-VL QA 候选、SAM3 文本/框选分割候选审核、指针位置滚轮缩放、区域聚焦、原文件名导出。模型服务配置与本地 SAM3 启动方法见 [services/README.md](services/README.md)。

## 快速运行

建议 Python 3.11–3.13、Node.js 22.12+。在项目目录执行：

```powershell
./start.ps1 -Install -Demo
```

打开 http://127.0.0.1:8000 ，API 文档位于 http://127.0.0.1:8000/docs 。以后直接 `./start.ps1` 即可。`-Demo` 每次创建一条**合成示意数据**，不是实际遥感观测。Windows 需允许本地 PowerShell 脚本运行。

也可以分开运行（macOS/Linux 同理，将虚拟环境命令换为对应路径）：

```powershell
python -m venv .venv
./.venv/Scripts/python -m pip install -r backend/requirements.txt
./.venv/Scripts/python scripts/export_schema.py
./.venv/Scripts/python scripts/demo.py --seed
./.venv/Scripts/python -m uvicorn backend.app:app --host 127.0.0.1 --port 8000
# 另一个终端
cd frontend
npm install
npm run dev
```

开发前端位于 http://127.0.0.1:5173 ，通过 Vite 代理访问 API。生产式本地部署先 `npm run build`，再重启后端，后端会提供 `frontend/dist`。手机需要同一局域网下运行 `uvicorn backend.app:app --host 0.0.0.0 --port 8000` 并打开电脑局域网 IP；此原型没有登录鉴权，适用于可信本地网络。

## 三项交付物

- `backend/app.py`：文件接收、批量导入、读取、修改、历史版本、预览、Diff、ZIP 导出。
- `frontend/src/main.jsx`：上传表单、原图叠加、顶点编辑、画笔、对比、类别与文本管理。
- `backend/models.py` / `schema/sample.schema.json`：唯一权威数据模型与自动生成的 JSON Schema。`GET /api/schema` 同时提供 Schema。

## 数据集适配的实际边界

这些项目不是同一种文件格式。标签预设、文件格式适配、原生训练器适配分别处理，不能以名称相同宣称“完美兼容”。

| 项目 | 本原型支持 | 还需要的原生适配 |
|---|---|---|
| RescueNet | 官方语义 ID 0..10、单通道/调色板索引 PNG；精确转多边形后编辑；导出原始 ID Mask | 原始目录可用 manifest 批量组织；没有假造“官方 JSON” |
| DisasterM3 | 灾害等级示例预设、xBD 像素 WKT、统一多边形/文本、通用 VLM conversations 导出 | 具体发布版的多时相路径、任务字段、训练脚本须用样例进一步对齐；示例标签不是官方全任务标签表 |
| DORA | 保留原始 JSON 属性、统一区域/指令/回答编辑 | 完整任务 manifest、SAR/DEM/矢量工具轨迹执行和多时相配准尚未实现 |
| SegEarth-R1/R2 | 背景/目标预设、索引 Mask、文本指令/回答、多个区域 | EarthReason / LaSeRS 原生打包字段需额外 release adapter |
| PixDLM / DRSeg | 背景/目标预设、Mask 与推理文本编辑 | DRSeg 发布版原生字段需样例对齐 |
| 自采手机 / UAV / 遥感 | JPG、PNG、WebP、BMP、TIFF/GeoTIFF、JP2，支持从零绘制与自建类别 | HEIC、RAW、视频、点云、ENVI/HDF/NetCDF、多文件传感器产品暂未支持 |

官方来源：[DisasterM3](https://github.com/Junjue-Wang/DisasterM3)、[DORA](https://github.com/Junjue-Wang/DORA)、[RescueNet](https://github.com/BinaLab/RescueNet-A-High-Resolution-Post-Disaster-UAV-Dataset-for-Semantic-Segmentation)、[SegEarth-R1](https://github.com/earth-insights/SegEarth-R1)、[SegEarth-R2](https://github.com/earth-insights/SegEarth-R2)、[PixDLM](https://github.com/XIEFOX/PixDLM)。实际数据的使用许可遵循各来源。

### 现有格式适配器

1. **统一 JSON**：上传完整 Sample 或单独 Annotation。完整 Sample 的图像尺寸以实际文件为准；旧 ID/版本不覆盖本地管理信息。
2. **索引 Mask**：必须与原图同尺寸、单通道或 P 模式；类别 0 为背景，1..254 为前景。RGB 彩色 Mask 与 255 ignore 暂不自动推断。Mask 不重采样，`rasterio.features.shapes` 保留孔洞和离散分量。高碎片度 Mask 最多 30,000 个区域。
3. **LabelMe**：只接收 polygon，标签名需与 categories 匹配。
4. **COCO**：按上传图像的唯一 `file_name` 匹配，支持 polygon segmentation，保留原 category_id 和 annotation id。RLE 明确拒绝，前景 ID 0 或超出 254 需预先重映射。
5. **xBD**：读取 `features.xy[*].wkt`，将 no/minor/major/destroyed 映射至 1/2/3/4；un-classified 和灾前未给定等级不会猜测。
6. **GeoJSON**：只接受 Polygon/MultiPolygon。顶层 `coordinate_space: "pixel-edge"` 表示像素坐标；否则必须显式给出与影像一致的 `crs_name`，例如 EPSG:32650，通过逆仿射变换转换。不会默认为经纬度，也不会隐式重投影。

非统一 JSON 可在顶层声明 `categories: [{id,name,color}]`。原始元数据保留在 `source.metadata` 及 Annotation properties；区域属性保留在 `regions[].properties`。前端显示顶点数组并提供拖拽修改，复杂任意属性可通过 API 修改。多模态或灾前/灾后可暂时作为独立样本导入，在 properties 中保存共同 pair_id；尚未提供联动配准查看器。

## 交互流程

1. 导入原图，选择标签预设；已有标注可一同上传 JSON 或索引 Mask。空白图像只上传图片。
2. 选择右侧类别，点“多边形”，逐个落点，Enter/完成闭合；Esc 取消。
3. 选择区域，拖拽顶点。双击边线插入顶点；点击顶点后可删除，每个环至少三个点。“重新绘制”保留区域 ID、类别及属性，替换区域几何。
4. 画笔按原图像素直径涂抹；擦除写入背景。平移、缩放对编辑/对比分屏使用同一视窗。
5. 对比基线是**首次导入**；分屏上均有原图叠加；Diff 绿色表示新增、红色表示删除、黄色表示前景类别变化。
6. 保存生成新版本，基线保持不变。恢复历史只加载草稿，随后保存为新版本。撤销/重做最多 50 步。
7. 导出会先保存当前修改；保存校验失败则保留前端草稿，停止导出。自相交多边形/错误孔洞在保存时拒绝。切换/关闭页面提示未保存修改；没有浏览器崩溃恢复草稿功能。

## 坐标与一致性约定

- 原点左上角，x 向右，y 向下；原图像素边缘坐标 `[0,width] × [0,height]`，不是归一化坐标。多边形数组不重复末尾首点。
- 每个 region 的 `rings[0]` 是外环，其余为孔洞。类别必须显式存在；类别 ID 唯一，背景 ID 0 必须存在。
- SVG 通过 `getScreenCTM().inverse()` 计算鼠标/触摸对应的原图坐标，缩放和平移不改动实际顶点。
- Mask 是派生物：先依序栅格化所有区域，后按笔画顺序绘制，后写覆盖前写。橡皮擦写 0。笔画始终在多边形上层；新多边形不会盖住已有笔画，需撤销/恢复处理。矢量区域与笔画同时保留，可重复导出。
- 后端使用像素中心栅格化；索引 Mask → 多边形 → Mask 可逐像素回环，包含孔洞。前端显示后端生成的 Mask 叠加；拖拽/画笔的即时轮廓随后会被后端精确预览替换。
- TIFF/JP2 使用前 1–3 波段、每波段 2–98% 拉伸，仅供显示；单波段复制为灰度。原始文件及其 CRS/仿射信息完整保留。当前不含 SAR 标定、可选波段组合及配准。
- 普通图片不应用 EXIF 自动旋转，确保已有标注仍对应编码像素。界面提供同方向 PNG 预览；导出的原图保持原样。
- 原型限制单图 3200 万像素、单文件 128MB；需要大幅面 COG 金字塔和切片编辑时，应增设 tile/window API，将局部坐标变换回全图。不要将整个正射镶嵌图读入浏览器。

## 统一结构示意

```json
{
  "schema_version": "1.0",
  "id": "managed-id",
  "profile": "custom",
  "revision": 0,
  "coordinate_space": "pixel-edge",
  "image": {"filename":"uav.jpg","width":4000,"height":3000,"crs":null,"transform":null},
  "annotation": {
    "categories": [
      {"id":0,"name":"Background","color":"#182534"},
      {"id":1,"name":"Road blocked","color":"#e45b70"}
    ],
    "regions": [{"id":"r1","category_id":1,"rings":[[[100,100],[450,100],[450,300],[100,300]]],"properties":{"damage":"major"}}],
    "strokes": [],
    "image_labels": ["flood"],
    "instruction": "标出阻塞道路。",
    "response": "图像左上区域的道路有阻塞。",
    "properties": {"pair_id":"event001"}
  },
  "source": {"dataset":"local"}
}
```

Pydantic 在 JSON Schema 结构校验之外还执行：几何有效性、孔洞有效性、类别引用唯一性、坐标有限性及原图边界检查。

## 批量导入包

ZIP 根目录必须有 `manifest.json`。不递归猜测目录，不解压到任意用户路径；拒绝路径穿越、重复条目和展开大小超过 512MB 的压缩包。每包最多 500 个样本，任一样本失败则回滚本包。

```json
{"samples":[
  {"image":"images/001.jpg","mask":"masks/001.png","format":"mask","profile":"rescuenet"},
  {"image":"images/002.tif","metadata":"labels/002.json","format":"unified","profile":"segearth-r1"}
]}
```

也可以为每条记录直接放 `annotation` 对象并选择 `format:"unified"`。`examples/demo.zip` 由 `python scripts/demo.py` 生成，可直接从界面导入。

## 导出

ZIP 包包含：`images/` 原始图像、`previews/` 显示 PNG、`masks/` 单通道 uint8 PNG、`annotations/` 当前 Sample、`original_annotations/` 初始基线、`history/` 版本快照、`train.json` 训练索引、`manifest.json` 可重新导入清单、`schema.json`。

- **unified**：权威编辑数据和通用索引。
- **rescuenet**：强制校验 RescueNet 0..10 官方类别及名称；Mask 作为分割监督，JSON 作为本系统索引。
- **disasterm3-style**：通用 human/gpt conversations + image/mask 引用，保留 instruction/response；不是已认证的 DisasterM3 全任务原生格式，不能保证直接交给某个训练脚本无需映射。

导出包重新导入会创建新的本地 ID 和初始基线，**不会自动重建旧版本链**；原历史文件仍随导出包保存。

## API

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/api/profiles` / `/api/schema` | 标签预设 / JSON Schema |
| POST | `/api/samples` | multipart image、profile、format、metadata?、mask? |
| POST | `/api/import-zip` | multipart archive |
| GET | `/api/samples` / `/api/samples/{id}` | 列表 / 完整样本 |
| GET | `/api/samples/{id}/image` / `original` | 预览图 / 初始 Sample |
| PUT | `/api/samples/{id}` | `{expected_revision, annotation}`；冲突 409，校验失败 422 |
| POST | `/api/samples/{id}/preview?mode=overlay\|mask\|diff` | 请求体 Annotation，返回 PNG |
| GET | `/api/samples/{id}/revisions` / `/revisions/{revision}` | 历史版本 |
| GET | `/api/export?ids=id1,id2&target=unified` | 导出指定或全部样本 |

SQLite 使用事务与版本号防止覆盖并发修改；图像及显示预览在 `data/{uuid}/` 中。环境变量 `STUDIO_DATA` 可指定数据根目录。写入完成后才显示保存成功。数据库和 data 目录应一起备份。

## 验证与后续扩展

```powershell
./.venv/Scripts/python -m pytest backend/test_system.py -q
cd frontend
npm run build
```

测试覆盖 Mask 孔洞逐像素回环、版本冲突、不可变基线、Diff 颜色、保存/导出/再导入、无效几何、越界、画笔擦除、未知类别、ZIP 穿越和失败回滚、GeoTIFF 世界坐标转换。

本次实际验证结果见 `docs/VALIDATION.md`，界面截图见 `docs/workspace.png`。源码交付包已包含构建后的前端；若只运行打包版本，可创建虚拟环境、安装后端 requirements 后直接启动 uvicorn。依赖复现可使用 `pip install -r backend/requirements.txt -c backend/constraints-tested.txt`；前端提供 pnpm 锁文件，亦可使用 npm 安装。

生产扩展建议：COG/切片与多波段选择 → 通过原生样例确认各 release adapter → 灾前/灾后及 SAR 联动配准 → 登录/项目权限/对象存储/任务队列 → SAM 辅助分割与人工复核。GeoPandas 更适合后续批量矢量层重投影；当前单图像像素编辑核心使用 Shapely + Rasterio，避免把像素坐标误当地理 CRS。


## 区域修正与自动草稿

新增局部重画边界、区域增补/擦除、去边就近重连、同类区域合并与贯穿切割；所有操作先预览再应用，支持撤销。右侧质检面板可定位几何和 VQA 完整性问题。浏览器自动备份未保存草稿，刷新恢复，遇到旧版本冲突需选择处理。操作与限制见 [GEOMETRY-AND-DRAFTS.md](docs/GEOMETRY-AND-DRAFTS.md)。

VQA 中文 Question / Answer / think 编辑后默认自动翻译对应英文，可关闭或手动重试。待同步字段不能审核进入训练数据；手工英文编辑和更新的中文不会被旧翻译响应覆盖。详见 [VQA-AND-GEOMETRY.md](docs/VQA-AND-GEOMETRY.md)。
