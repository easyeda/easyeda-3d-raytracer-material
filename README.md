# Raytraced Material Resource Pack

**English** | [简体中文](#简体中文)

A ray-tracing material resource pack for the EasyEDA Pro 3D preview.

| Item | Value |
|---|---|
| Textures | 17, 2048×2048 |
| Materials | 1,384-entry color→PBR palette + 27 PCB built-in materials |
| Scene props | Soldering iron, solder wire, tweezers, desk mat |
| EDA support | EasyEDA Pro 4.1.109+ |

What it looks like:

- **PCB**: dark solder mask with visible copper layers; ENIG pads; matte FR4 edge; white silkscreen
- **Components**: near-black epoxy bodies, metallic pins and terminations
- **Scene**: soldering iron, solder wire and tweezers on a desk mat, with base color / normal / roughness maps
- **Lighting**: neutral studio environment, ACES tone mapping

---

## 1. Installation

The pack is distributed as repository files, so build the zip yourself before installing:

1. Download this repository (click **Code → Download ZIP**, or clone it);
2. Repack the files yourself: select the files at the **repository root** — `material_table.json`, `inner_material_table.json`, `param.json`, all `*.png` / `*.hdr` files, and the `scene_model/` folder — then create a zip with them directly at the archive root. Do not wrap them in an extra folder: when you unpack the archive, `material_table.json` must sit at the top level;
3. Open any PCB project;
4. Enter **3D Preview**;
5. Open the **ray tracer** and click **Enable ray tracing**;
6. On first launch a dialog offers **Download** or **Install** materials;
7. Choose **Install**, select the zip you just created, and wait for loading to finish.

> GitHub's **Download ZIP** wraps everything in a `<repo-name>/` folder, which the engine cannot read. Re-zip the contents as described in step 2.
> `README.md`, `DEVELOPMENT*.md` and `LICENSE` are ignored by the renderer and may be included or omitted; do not include the `.git/` folder.
> If choosing **Download** in the dialog shows `Failed to load resource: 418 (Invalid XPrivet Token)`, that is an authentication error when downloading from the official server and is unrelated to this pack. Use **Install** and pick your own zip instead.

---

## 2. FAQ

| Symptom | Fix |
|---|---|
| Entire image black | Make sure the 3D preview is open and ray tracing initialized successfully |
| `Texture size mismatch … expected 2048×2048` | Textures were downscaled while repacking; keep every texture at 2048×2048 |
| Pack seems not to load; materials stay default | The zip likely has an outer folder; repack so `material_table.json` sits at the archive root |
| Too bright / washed out | Lower `background.intensity` in `param.json` (0.25–0.45 recommended) |
| Props such as the soldering iron are missing | Props are placed at fixed world coordinates; rotate or zoom out |
| Slow loading | The pack includes 4 scene props; the first load takes a few seconds — do not import repeatedly |
| `Error: this.currentTask !== null` | Ray tracing was initialized twice: close and reopen the 3D preview, or wait for the previous initialization to finish |

---

## 3. Developers

The pack's file structure, field meanings, value conventions, texture and HDR requirements, scene-model authoring, verification and tuning are all documented in **[DEVELOPMENT.md](DEVELOPMENT.md)** (English) and **[DEVELOPMENT.zh-CN.md](DEVELOPMENT.zh-CN.md)** (中文).

Read the guide before creating or modifying a material pack, and follow its conventions.

---

## 4. Version

| Item | Value |
|---|---|
| Version | 1.0.0 |
| Updated | 2026-11-21 |
| Asset source | Scene models and textures come from the official resource pack (textures only losslessly re-encoded) |

## 5. License

1. Electric soldering iron, tweezers, and soldering tools are modified based on open-source models: "Old Soldering Workstation" (https://skfb.ly/ovRCs) by Wikiti is licensed under Creative Commons Attribution (http://creativecommons.org/licenses/by/4.0/).

2. This project under Apache License

---
---

# 简体中文

[English](#raytraced-material-resource-pack) | **简体中文**

用于嘉立创EDA 专业版 3D 预览的光线追踪材质资源包。

| 项目 | 值 |
|---|---|
| 贴图 | 17 张，2048×2048 |
| 材质 | 1384 条颜色→PBR 调色板 + 27 条 PCB 内置材质 |
| 场景道具 | 电烙铁、焊锡丝、镊子、桌面垫 |
| 适用EDA | 嘉立创EDA 专业版 4.1.109+ |

效果概览：

- **PCB 本体**：深色阻焊，可见铜箔层次；沉金焊盘；哑光 FR4 断面；白色丝印
- **器件**：近黑环氧本体、金属引脚与端电极
- **场景**：桌面垫上摆放电烙铁、焊锡丝、镊子，带基础色 / 法线 / 粗糙度贴图
- **照明**：中性工作室环境光，ACES 色调映射

---

## 一、安装

本包以仓库文件形式分发，安装前需要自己打包成 zip：

1. 下载本仓库（点击 **Code → Download ZIP**，或 clone）；
2. 自行打包：选中**仓库根目录**下的文件 —— `material_table.json`、`inner_material_table.json`、`param.json`、全部 `*.png` / `*.hdr` 贴图，以及 `scene_model/` 文件夹 —— 压缩成 zip，所有文件直接位于压缩包根目录。不要套一层外层文件夹：解压后 `material_table.json` 必须就在最顶层；
3. 打开任意 PCB 工程；
4. 进入 **3D 预览**；
5. 打开 **光追渲染器**，点击 **开启光追渲染**；
6. 第一次打开会弹窗提示「下载」或「安装」材质；
7. 点击 **安装**，选择刚打包好的 zip，等待加载完成即可。

> GitHub 的 **Download ZIP** 会把内容套在 `<仓库名>/` 文件夹里，引擎无法读取；请按第 2 步重新打包。
> `README.md`、`DEVELOPMENT*.md`、`LICENSE` 不参与渲染，放不放均可；不要打包 `.git/` 目录。
> 若在弹窗中选择「下载」后出现 `Failed to load resource: 418 (Invalid XPrivet Token)`，这是从官方服务器下载时的鉴权错误，与本包无关；请改用「安装」按钮选择自己打包的 zip。

---

## 二、常见问题

| 现象 | 处理 |
|---|---|
| 整张图纯黑 | 确认 3D 预览已打开，且光追已成功初始化 |
| 报错 `Texture size mismatch … expected 2048×2048` | 打包时贴图被缩小过；所有贴图必须保持 2048×2048 |
| 资源包似乎没生效；材质仍是默认外观 | zip 里多半套了外层文件夹；请重新打包，确保 `material_table.json` 位于压缩包根目录 |
| 画面太亮 / 发白 | 下调 `param.json` 中 `background.intensity`（建议 0.25–0.45） |
| 看不到电烙铁等道具 | 道具按世界坐标固定摆放，旋转或缩小视角查看 |
| 加载较慢 | 包含 4 个场景道具，首次加载需数秒；不要连续多次导入 |
| 报错 `Error: this.currentTask !== null` | 光追被重复初始化：先关闭 3D 预览页再重新打开，或等待上一次初始化结束后再操作 |

---

## 三、开发者

本包的文件结构、各字段含义、取值规范、贴图与 HDR 要求、场景模型写法、验证与调参方法，全部写在 **[DEVELOPMENT.md](DEVELOPMENT.md)**（英文）与 **[DEVELOPMENT.zh-CN.md](DEVELOPMENT.zh-CN.md)**（中文）。

制作或修改材质包前请先阅读该文档，按其中的规范修改本包即可。

---

## 四、版本

| 项 | 值 |
|---|---|
| 版本 | 1.0.0 |
| 更新日期 | 2026-11-21 |
| 素材来源 | 场景模型与贴图来自官方资源包（贴图仅做无损重编码） |


## 五、许可

1. 电烙铁、镊子、焊锡工具基于开源模型修改："Old Soldering Workstation" (https://skfb.ly/ovRCs) by Wikiti is licensed under Creative Commons Attribution (http://creativecommons.org/licenses/by/4.0/).

2. 本项目基于 Apache License 开源。
