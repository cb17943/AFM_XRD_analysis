# AFM 表面形貌分析（网页版）

在浏览器里直接处理 **Asylum Research MFP-3D / Cypher** 导出的 `.ibw` 数据：解析、去倾斜、算粗糙度、分水岭分割晶粒、统计颗粒，输出 2D／3D 图、汇总表和 Word 报告。

**全部计算在本机浏览器完成，文件不上传到任何服务器。** 没有后端、没有数据库、没有依赖。

在线体验：把本仓库开启 GitHub Pages 后，直接打开生成的网址即可（见下方部署）。

---

## 功能

| 模块 | 说明 |
|---|---|
| 解析 | 识别 ARDF v2 容器：384 B 头 + N 通道 × W × H float32 + 参数文本 + 尾部索引；取出 Height 通道与 ScanSize / ImagingMode / ScanRate / InvOLS 等参数 |
| 去面 | 一阶最小二乘平面拟合后相减（只去样品倾斜，不做逐行滤波） |
| 粗糙度 | Ra / Rq / Rmax / Rz（五点法）/ Ssk / Sku |
| 晶粒 | 高斯平滑 σ=1 px → 高度局部极大作种子 → 优先队列分水岭生长；剔除边缘截断与面积 < 12 px 的碎片，输出等效圆直径分布 |
| 颗粒 | 15 px 尺度高斯高通 → 8σ 阈值（下限 3 nm）→ 连通域，输出数量、面密度、等效直径、高度、覆盖率 |
| 自动分类 | 连续晶粒织构 / 平坦基底 + 孤立颗粒，两类走不同统计路径 |
| 出图 | Gwyddion 色标 + 色标条（标 nm）+ 1 µm scale bar；2D 高度图、3D 形貌、晶粒分割、总览、Ra/Rq 对比 |
| 导出 | PNG 图、CSV 表、Word 报告（.docx）、全部打包 ZIP |

## 使用

1. 用 Chrome / Edge / Safari 打开 `index.html`（双击即可，无需服务器）。
2. 把 `.ibw` 文件或整个数据文件夹拖进页面，或点「选择文件 / 选择文件夹」。
3. 等待处理完成，页面会给出汇总表、总览图和逐样品图集。
4. 点「打包 ZIP」一次性拿到图片 + CSV + Word。

选择文件夹时会自动包含子文件夹，不需要额外设置。

## 部署到 GitHub Pages

```bash
git init
git add index.html README.md
git commit -m "AFM web analyzer"
git branch -M main
git remote add origin https://github.com/<你的账号>/<仓库名>.git
git push -u origin main
```

然后在仓库页面 **Settings → Pages → Build and deployment**：

- Source 选 `Deploy from a branch`
- Branch 选 `main`，目录选 `/ (root)`
- 保存，等 1–2 分钟后访问 `https://<你的账号>.github.io/<仓库名>/`

> 只需要 `index.html` 一个文件就能运行。`src/` 与 `vendor/` 是源码与库，`build.py` 用来把它们重新内联成 `index.html`：
> ```bash
> python build.py
> ```

## 目录结构

```
afm-web/
├── index.html          ← 成品，单文件（含全部 JS/CSS/依赖，约 900 KB）
├── build.py            ← 把 src + vendor 内联成 index.html
├── src/
│   ├── core.js         ← .ibw 解析 + 数值分析（去面/粗糙度/分水岭/颗粒）
│   ├── draw.js         ← Canvas 绘图（2D/3D/晶粒/总览/统计图）
│   ├── export.js       ← CSV / Word(docx.js) / ZIP(jszip)
│   ├── ui.js           ← 交互流程
│   ├── style.css
│   └── index.template.html
└── vendor/             ← jszip.min.js、docx.umd.js（已内联进 index.html）
```

## 计算口径

- **去面**：只做一阶平面拟合，Ra 中包含真实长波起伏。若需要逐行/中值滤波后的数值，请自行在桌面软件处理。
- **晶粒**：等效圆直径 `2√(A/π)`；分水岭方向为"从晶粒顶部向谷底生长"，与 skimage 的 `watershed(-z, markers)` 一致；所有像素都有归属（不保留边界线）。
- **表面类型判据**：`Ra/Rmax < 0.06`，或（颗粒覆盖率 > 0.2% 且颗粒均高 > 3×Ra）→ 判为「平坦基底 + 孤立颗粒」。这是经验阈值，建议结合原图复核。
- **与桌面 Python 版差异**：同一份数据，Ra/Rq/Rmax/Rz 完全一致；晶粒数与平均粒径的偏差通常在 1% 以内（分水岭实现的像素级差异）。

## 已验证

用 13 个 MFP-3D 实测文件（5×5 µm、512×512、AC Mode）比对，与 Python 版结果：

| 样品 | Ra (nm) | 晶粒数 | 平均粒径 (nm) |
|---|---|---|---|
| F_1110A02_C | 6.16 / 6.16 | 1553 / 1556 | 132.3 / 132.3 |
| CVD80nm | 5.82 / 5.82 | — | — |
| F_1110A15_C | 0.60 / 0.60（颗粒 13 / 13） | — | — |

（左为网页版，右为 Python 版）

## 限制

- 只支持 ARDF v2 结构的 `.ibw`（MFP-3D / Cypher 的常见导出）。若你的文件是标准 Igor Binary Wave 或 `.spm`，需要另做解析器。
- 一次建议不超过 ~40 个文件；13 个 512×512 文件约 10–20 秒（取决于机器）。
- Word 中的图片为 PNG 内嵌，单个报告约 2–3 MB（每样品 3 图）。

## 许可

内部工具，随用随改。
