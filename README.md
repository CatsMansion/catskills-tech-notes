# CatsMansion 新技术总结 / CatsMansion New-Tech Notes

> 本仓库汇总 CatsMansion 组织下自研 skill 背后的技术方法。每项技术在对应 skill 仓库有完整可运行代码；这里只讲「为什么这么做、怎么做、踩过什么坑」。
>
> English summary at the bottom.

## 目录 / Contents
1. [ai-watermark-margin — AI 水印留白裁剪](#1-ai-watermark-margin)
2. [consistent-character-mv-anchor — MV 角色锚点一致性](#2-consistent-character-mv-anchor)

---

## 1. ai-watermark-margin

**一句话**：让 AI 修图工具的水印落在「留白」里，修完再把留白裁掉，水印消失、画面无损。

### 问题
AI 图像编辑 / 生成通道（Miora 等）会在输出图上盖一个生成器水印。如果水印压在内容区，直接裁会伤画面；不裁又没法用。

### 方法
1. **编辑前**：用 `add_margin.py` 给画布加一圈纯色留白（默认 ~10%，可只加某一边，颜色白 / 黑 / 任意 `#rrggbb`）。
2. **送 AI 编辑**：生成器水印会盖在「内容 + 留白」上；留白足够大时水印落在留白里。
3. **编辑后**：用 `crop_margin.py` 按「已知留白颜色」把留白裁掉，恢复原始构图。水印随留白一起消失。

**关键修正**：`crop_margin.py` 的 `detect()` 必须**显式传入留白颜色**（不是从角点猜）。单边留白、角点是内容时，角点猜色会误判、裁不干净还留水印。自测发现并修掉了这个 bug。

### 自测（正 / 负对照）
- 阳性对照：水印在留白内 → 裁掉，尺寸恢复到 600×400 ✅
- 阴性对照：水印压在内容上、无留白 → 不裁，水印保留 ✅

两者都过，证明方法只去掉「留白里的水印」、不会误伤内容。

### 边界 / 伦理
- 只去掉**生成器自己的**印章；用户仍需声明 AI 使用。
- 不用于去除第三方版权水印。

→ 完整代码：`CatsMansion/ai-watermark-margin`

---

## 2. consistent-character-mv-anchor

**一句话**：用「锁定锚图 + 图像到图像」保证 MV 多帧角色身份一致，并用双判据抠图 + 最大连通域清理干掉背景水印。

### 问题
- 多帧 / 视频里同一角色容易「长歪」（identity drift）。
- 抠图（绿幕 / 纯色背景）会残留背景水印或杂点。
- 构图基线（留白 / 占比 / 中心）帧间不一致会跳。

### 方法
1. **身份锁**：固定一张锚图，所有帧走 image-to-image + 锚图参考，锁住脸 / 发型 / 配色。
2. **双判据抠图**：`key_green.py` 先用「边界洪水填充 + 颜色距离」找背景色区域，再**只保留最大前景连通域**——背景水印、孤立杂点（不属于主体连通域）被一并丢掉。
3. **特征程序化锁**：对关键特征（挑染位置、耳饰左右）做程序化约束，不让模型自由发挥。
4. **静态区锁**：帧间一致性靠锁定静态区域。
5. **构图基线测量**：`measure_bbox.py` 量 L/R/T/B 留白、占比 %、底边留白、中心，新帧逐项对齐。

### 自测
合成 512 帧：绿背景残留 0 像素、浅色水印残留 0 像素；`measure_bbox` 正确排除水印。✅

### 边界 / 伦理
- 只用**自有角色**；客户 / 第三方资产绝不外发。
- 对外分享前必须脱敏（本仓库方法已剔除所有客户数据与内部设定）。

→ 完整代码：`CatsMansion/consistent-character-mv-anchor`

---

## 发布状态 / Publish status
- 两项技术均已作为独立 skill 发布到 GitHub（CatsMansion org）。
- agentskill.sh 首次导入仍受平台侧 GitHub 429 限流影响，需在浏览器手动 Connect GitHub + import（webhook 已挂两个 skill 仓库，import 后自动同步后续 push）。

## License
MIT

---

## English summary

**ai-watermark-margin** — Drop AI generator watermarks losslessly. Pad a blank margin (~10%) around the canvas *before* sending the image to an AI editor, so the generator's stamp lands in the margin; crop the margin back off afterward. Watermark gone, content intact. Key fix: `crop_margin.py` must take the *known* margin color explicitly (corner-guessing fails on one-sided margins). Positive/negative controls both pass.

**consistent-character-mv-anchor** — Keep a character identity-consistent across MV frames/video. Lock a reference anchor image + image-to-image for identity; dual-criteria chroma key (border flood-fill + color distance) keeps only the largest connected foreground component to drop background watermarks/specks; programmatic feature locking + static-region lock + composition-baseline measurement (`measure_bbox.py`) for frame-to-frame stability. Self-tested on 512 synthetic frames: 0 leftover background/watermark pixels.
