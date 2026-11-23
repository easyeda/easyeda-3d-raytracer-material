# 光追材质资源包 · 开发指南

> [English](DEVELOPMENT.md) | 简体中文
>
> 文件名：`DEVELOPMENT.zh-CN.md`（随资源包分发，与 `README.md` 同级；英文版为 `DEVELOPMENT.md`）
>
> 面向两类读者：
> **A. 产品/设计** —— 了解资源包是什么、怎么让它生效；
> **B. 制作材质包的人 / AI** —— 按本文规范即可独立产出效果逼真的材质包。
>
> 本文只讲**资源包里放什么、字段怎么填、取值怎么定、怎么验证**，不涉及代码开发。
>
> 文中数值来自 2026-11 实测：客户端 **4.1.109**、`smt-gl-engine 4.1.44`。
> 标注「实测」的结论为本地验证结果，接口若在后续版本变更，以本机客户端的 `lceda-pro --help` / `lceda-pro doc api` 输出为准。
> 注意区分发行版：中国大陆版可执行文件为 `lceda-pro`（本文命令均以它为例）；国际版 EasyEDA Pro 的可执行文件为 `easyeda-pro`，命令形式相同，把 `lceda-pro` 原样替换为 `easyeda-pro` 即可（如 `easyeda-pro --help`、`easyeda-pro doc api`）。请以本机实际安装的版本为准，不要混用两个发行版。

---

## 目录

1. 资源包是什么
2. 目录结构与文件职责
3. `material_table.json` —— 颜色→PBR 调色板（最容易做错的一环）
4. `inner_material_table.json` —— PCB 内置材质覆盖
5. `param.json` —— 渲染参数
6. 贴图规范（通道、颜色空间、尺寸、命名）
7. HDR 环境光规范
8. 颜色空间：一切都按线性浮点写
9. `scene_model/`：是否添加、如何开发；`debug/` 为什么不会有
10. 加载与生效机制（生产链路 / 热加载）
11. 资源包在运行时的行为（不需要写代码）
12. 用 AI 生成材质包的完整流程
13. 验证与调参方法
14. 常见坑（汇总）
15. 发版检查清单

---

## 一、资源包是什么

光追渲染器启动时，会从扩展存储里读取一个 **zip 资源包**，其中可以包含：

- 材质表（每个面/器件的外观参数）
- 内置材质覆盖（PCB 基板/阻焊/丝印/镀层）
- PBR 贴图（基础色/法线/粗糙度/金属度）
- HDR 环境贴图
- 渲染参数（渲染器 / 背景 / 灯光）
- 可选场景模型（OBJ+MTL，如桌面摆件）

zip 的**根目录直接放这些文件**，没有外层文件夹。

### 体积参考（实测）

一套 154.7 MB 的 PNG 素材，可以做到：

| 档位 | 总大小 | 单张 | 说明 |
|---|---:|---:|---|
| 原始 | 154.7 MB | 最大 25.4 MB | 大量 RGBA（alpha 全 255）、未优化、含 3 组重复文件 |
| 严格无损 | **56.5 MB** | 最大 8.6 MB | 去 alpha、去重、PNG level 9，像素逐位一致 |
| 近无损 1024 | 12.2 MB | ≤1.9 MB | **不可用于本引擎**（见下） |

> ⚠️ **贴图尺寸不能降低。** 引擎纹理图集单元 `UNIT_SIZE = 2048`，贴图必须是 2048×2048，否则直接抛错：
> `Texture size mismatch: 1024×1024, expected 2048×2048`。
> 因此体积优化只能做**编码层压缩**（去 alpha、去重、PNG level 9），发版用 `lossless/` 档。

---

## 二、目录结构与文件职责

```
raytracer-assets.zip
├─ material_table.json          # 必需：颜色→PBR 调色板（数组）
├─ inner_material_table.json    # 可选：覆盖 PCB 内置材质（对象）
├─ param.json                   # 可选：背景 / 光照 / 渲染参数
├─ *.png / *.hdr                # 贴图（被材质表按文件名引用）
└─ scene_model/                 # 可选：场景道具（OBJ+MTL）
    ├─ manifest.json
    ├─ <name>.obj
    └─ <name>.mtl
```

> 本包不包含 `debug/`（引擎的调试输出），详见第九章。

| 文件 | 必需 | 作用 |
|---|---|---|
| `material_table.json` | ✅ 是 | 颜色→PBR 调色板；缺省时器件只能得到哑光灰 |
| `inner_material_table.json` | 否 | 覆盖焊盘/基板/阻焊/丝印等命名材质 |
| `param.json` | 否 | 渲染器、背景、灯光 |
| `*.hdr` | 视情况 | 被 `param.json` 的 `background.texture` 引用 |
| `*.png` | 视情况 | 被材质条目的 `*Texture.name` 引用，尺寸 2048×2048 |
| `scene_model/` | 否 | 场景道具；是否添加见第九章 |

> `debug/` 不随资源包分发，也不需要制作，详见第九章。

---

## 三、`material_table.json` —— 颜色→PBR 调色板

### 3.1 它到底是什么（最容易误解）

它不是「本板每个面的材质清单」，而是引擎在 **构建几何时** 使用的**颜色→材质查找表**：

```
MaterialComponent.getIndex(phong):
  1) phong 命中内置材质（H6 字典）        → 用 inner_material_table 的 PBR
  2) 否则存在 material_table（hasMaterialTable）
        → 在表里按 baseColor 距离找最近条目，用该条目的 PBR
  3) 否则                                  → pbr = fromPhong(diffuse)
                                             即 metallic 0 / roughness 1（哑光灰）
```

EDA 器件的 3D 模型不带 PBR，只有一个基础色（如 `0.208` 灰 = 器件本体、`0.592,0.463,0` = 镀层、`0.855` 灰 = 引脚）。调色板就是把这些编码色映射到真实材质。

**关键技巧**：条目的 `phong.diffuse` 只用于**匹配**，`pbr.color` 决定**实际渲染颜色**。两者可以不同——例如把 `0.208` 灰映射成近黑环氧 `[0.078,0.063,0.063]`。

### 3.2 结构

数组，每个元素是「Phong（光栅化）+ PBR（光追）」双版本：

```json
[
  {
    "phong": {
      "ambient":  [1, 1, 1],
      "diffuse":  [0.2078, 0.2078, 0.2078],
      "specular": [0.1255, 0.1255, 0.1255],
      "shininess": 5,
      "alpha": 1
    },
    "pbr": {
      "color": [0.0032, 0.0032, 0.0034],
      "metallic": 0,
      "roughness": 0.45,
      "ior": 1.5,
      "iorLevel": 0.5,
      "clearcoatTintColor": [1, 1, 1],
      "clearcoatIntensity": 0.15,
      "clearcoatRoughness": 0.12,
      "clearcoatIor": 1.5,
      "normalIntensity": 0.6,
      "normalTexture": null,
      "metallicTexture": null,
      "colorTexture": null,
      "roughnessTexture": null
    }
  }
]
```

**`phong` 字段**（仅用于 2D 光栅化预览，光追不读取，但 `diffuse` 例外）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `ambient` | `[r,g,b]` | 光栅化环境色，光追忽略 |
| `diffuse` | `[r,g,b]` | **颜色匹配键**。引擎用它查找调色板条目；必须与器件/模型的编码色一致，不能改 |
| `specular` | `[r,g,b]` | 光栅化高光色，光追忽略 |
| `shininess` | number | 光栅化高光锐度，光追忽略 |
| `alpha` | 0~1 | 透明度；光追按不透明处理，保持 1 |

**`pbr` 字段**（光追实际使用的参数，全部为线性空间）：

| 字段 | 类型 / 范围 | 说明 | 常用取值 |
|---|---|---|---|
| `color` | `[r,g,b]` 0~1 | 基础色（albedo）；金属写反射率而非“涂装色” | 见第八章 |
| `metallic` | 0~1 | 金属度；0 = 介质，1 = 纯金属 | 金属 1.0，塑料/环氧/阻焊 0 |
| `roughness` | 0~1 | 粗糙度，越小越镜面 | 金属 0.2~0.35；环氧 0.45~0.55；阻焊 0.35~0.45 |
| `ior` | ≥1 | 折射率，决定介质表面的基础反射率 | 金属 1.5；阻焊 1.42~1.45；哑光件 1.0 |
| `iorLevel` | 0~1 | 菲涅尔强度，与 `ior` 配合控制高光强弱；过高会在亮环境下整片泛白 | 金属 0.5；阻焊 0.15~0.25；哑光 0 |
| `clearcoatTintColor` | `[r,g,b]` | 清漆层的着色（通常白色） | `[1,1,1]` |
| `clearcoatIntensity` | 0~1 | 清漆强度；0 = 关闭 | 环氧 0.10~0.15；阻焊 0.06~0.18 |
| `clearcoatRoughness` | 0~1 | 清漆粗糙度，越小漆面越亮 | 0.10~0.30 |
| `clearcoatIor` | ≥1 | 清漆折射率 | 1.5 |
| `normalIntensity` | 0~2 | 法线强度；无法线贴图时影响内置微观法线 | 金属 0.3~0.4；塑料 0.6；无 1.0 |
| `colorTexture` | `{name, scale}` / `null` | 基础色贴图（sRGB） | 场景道具使用 |
| `normalTexture` | `{name, scale}` / `null` | 法线贴图（线性，+Y 向上） | 同上 |
| `roughnessTexture` | `{name, scale}` / `null` | 粗糙度贴图（灰度） | 同上 |
| `metallicTexture` | `{name, scale}` / `null` | 金属度贴图（灰度） | 同上 |

> 贴图对象的两个子字段：`name` 为 zip 内文件名（必须逐字符一致），`scale` 为 UV 平铺倍数（1 = 不重复）。

### 3.3 本包调色板的构成（1384 条，1280 个唯一颜色）

实测统计：

- 绝大多数是 `metallic 0 / roughness 0.25` 的彩色介质；
- 170 条 `metallic 1 / roughness 0.25` 的金属；
- 灰阶覆盖 0.02~1.0 共 50+ 档，用于命中器件编码色。

### 3.4 推荐生成规则

按 `phong.diffuse` 分类，写入独立设计的 `pbr`：

| 类别 | 判定条件 | metallic | roughness | clearcoat | pbr.color（线性） |
|---|---|---:|---:|---:|---|
| 黑色环氧本体 | 中性灰 luminance < 0.16 | 0 | 0.45 | 0.15 | sRGB 0.045 → 线性 0.0032 |
| 深灰本体 | 中性 0.16–0.32 | 0 | 0.50 | 0.10 | sRGB 0.12 → 线性 0.0137 |
| 引脚/焊盘/端子 | 中性 ≥ 0.32 | 1.0 | 0.33 | 0 | 0.66–0.76 |
| 亮金属 | 中性 ≥ 0.88 | 1.0 | 0.28 | 0 | 0.74（纯白除外） |
| 金色镀层 | b < 0.10 且 g/r ∈ [0.60, 0.92] | 1.0 | 0.26 | 0 | `[0.749,0.604,0.224]` |
| 铜色 | b < 0.10 且 g/r ∈ [0.30, 0.60] | 1.0 | 0.30 | 0 | `[0.620,0.300,0.140]` |
| 彩色介质 | 其余 | 0 | 0.48 | 0.10 | 原色 sRGB→线性 |
| 白色塑件/丝印 | 纯白 | 0 | 0.55 | 0.05 | `[0.90,0.895,0.88]` 线性 |

**去重**：同一 `phong.diffuse` 只保留一条（匹配取最近，重复无意义，只会增大体积）。

### 3.5 硬性要求

| 要求 | 说明 |
|---|---|
| 数组不能为空 | 空数组时 `hasMaterialTable = false`，等同没有调色板，器件全部退回哑光灰 |
| `phong.diffuse` 保留原始编码 | 它是匹配键；改成设计色会导致匹配不上，颜色错乱 |
| 同一 `diffuse` 只保留一条 | 匹配取最近条目，重复条目没有意义，只会增大体积 |
| 条目数量与顺序不限 | 匹配靠颜色、不靠索引，与具体板子无关 |

---

## 四、`inner_material_table.json` —— 内置材质覆盖

### 4.1 规则

对象，key 必须是引擎内置材质名；未知 key 会被跳过并告警。

字段与 §3.2 的 `pbr` **完全一致**（没有 `phong`，因为内置材质本身就带 Phong 定义）：

```json
{
  "gold": {
    "color": [0.749, 0.604, 0.224],
    "metallic": 0.85, "roughness": 0.30,
    "ior": 1.5, "iorLevel": 0.5,
    "clearcoatTintColor": [1,1,1],
    "clearcoatIntensity": 0, "clearcoatRoughness": 0, "clearcoatIor": 1.5,
    "normalIntensity": 0.25,
    "normalTexture": null, "metallicTexture": null,
    "colorTexture": null, "roughnessTexture": null
  }
}
```

### 4.2 常用内置材质名

**PCB 会真正用到的（建议全部覆盖）**

| 分组 | 名称 |
|---|---|
| 表面处理 | `gold` `silver` `aluminum` `osp` |
| 丝印 | `silk_white` `silk_black` |
| 基板 | `substrate_yellow`(FR4) `substrate_white`(陶瓷) `substrate_grey` `substrate_coppery` `substrate_fpc` |
| 阻焊 | `solder_mask_<颜色>_with_copper` / `_without_copper`，颜色：`green red blue yellow black purple white coppery soft_yellow` |

**其余有效但非 PCB 表面的名称**（器件或摆件模型可能引用，按需覆盖）

| 类别 | 名称 | 说明 |
|---|---|---|
| 金属 | `brass` `bronze` `chrome` `coppery` | 黄铜 / 青铜 / 铬 / 紫铜，用于器件金属件与摆件 |
| 宝石与矿物 | `obsidian` `pearl` `jade` `emerale` `ruby` `turquoise` | 黑曜石 / 珍珠 / 玉 / 祖母绿 / 红宝石 / 绿松石，装饰类模型 |
| 塑料 | `plastic_black` `plastic_white` `plastic_red` `plastic_green` `plastic_yellow` `plastic_cyan` | 常见塑封外壳色，器件本体可能引用 |
| 橡胶 | `rubber_black` `rubber_white` `rubber_red` `rubber_green` `rubber_yellow` `rubber_cyan` | 线缆外套、脚垫等柔性材质 |
| FPC 补强 | `stiffen_layer_black_film` `stiffen_layer_steel` `stiffen_layer_transparent_orange` `stiffen_layer_transparent_white` `stiffen_layer_transparent_yellow` | 柔性板补强层，仅 FPC 模型引用 |
| 透明基板 | `substrate_transparent_acrylic` `substrate_fpc_acrylic` `substrate_glossy_transparent_pet` `substrate_frosted_transparent_pet` `substrate_glossy_transparent_pvc` `substrate_frosted_transparent_pvc` `substrate_black_transparent_acrylic` | 亚克力 / PET / PVC 的透明与磨砂面板 |
| 阻焊变体 | `solder_mask_acrylic` `solder_mask_acrylic_with_copper` `solder_mask_acrylic_without_copper` | 亚克力质感的阻焊，仅特殊板型使用 |
| 焊球 | `raytracer_solder_bump` | BGA / CSP 焊球 |
| 遮蔽 | `absolute_black` | 纯黑遮蔽材质 |
| 占位 | `zero` | 空材质占位，一般不需要覆盖 |

> 引擎内置字典共 81 个名字；写错名字不会报错，只会被跳过，对应材质维持默认外观。

### 4.3 `with_copper` 与 `without_copper` 的区别

引擎按「阻焊下方是否有铜」选择两个变体：

| 变体 | 物理含义 | 建议参数 |
|---|---|---|
| `_with_copper` | 阻焊压在线路上，涂层更致密 | roughness 0.36–0.40，clearcoat 0.12–0.18，颜色可略亮 ×1.05 |
| `_without_copper` | 阻焊压在基板上，偏哑光 | roughness 0.42–0.46，clearcoat 0.06–0.10 |

两者差值不要太大，否则同块板上铜区/非铜区会出现明显色带。

### 4.4 推荐参数（实测观感最好）

| 材质 | color（线性） | metallic | roughness | ior | iorLevel | clearcoat |
|---|---|---:|---:|---:|---:|---:|
| gold | `0.749,0.604,0.224` | 0.85 | 0.30 | 1.5 | 0.5 | 0 |
| silver | `0.820,0.820,0.824` | 0.85 | 0.24 | 1.5 | 0.5 | 0 |
| aluminum | `0.520,0.520,0.540` | 1.0 | 0.32 | 1.5 | 0.5 | 0 |
| silk_white | sRGB 0.90 → 线性 0.787 | 0 | 0.50 | 1.5 | 0.5 | 0 |
| substrate_yellow | sRGB 0.72,0.64,0.42 → 线性 | 0 | 0.78 | 1.5 | 0.5 | 0 |
| 阻焊（带铜） | sRGB 目标色 → 线性 ×1.05 | 0 | 0.38 | 1.45 | **0.20** | 0.14 |
| 阻焊（不带铜） | 同上 | 0 | 0.44 | 1.42 | **0.20** | 0.08 |

> `iorLevel` 是阻焊最容易翻车的地方：常见配置为 0.5，但在「高亮柔光箱 HDR」下会整片泛白。实测 0.15–0.25 能保留漆面菲涅尔又不过曝。

---

## 五、`param.json` —— 渲染参数

```json
{
  "renderer": {
    "rendererMaxSample": 4096,
    "rendererEnableDenoise": true,
    "displayDevice": "sRGB",
    "viewTransform": "ACES",
    "lutName": "", "lutIntensity": 1,
    "maxRayBounces": 6,
    "hideRenderPlane": false,
    "hideRenderTriangle": false,
    "hideUnselected": false,
    "selectMaterialId": 0,
    "useBlueNoise": true
  },
  "background": {
    "useColor": false,
    "texture": "studio.hdr",
    "intensity": 0.35,
    "rotation": 0,
    "hide": true,
    "hideColor": "#000000"
  },
  "light": {
    "a0": { "color": "#ffffff", "power": 0.8, "normalize": false,
            "size": {"width": 0.011, "height": 0.011},
            "position": [-0.06, 0.23, 508.9], "rotate": [-45, 0, 25],
            "hideRender": false, "cameraVisible": true, "castShadow": true,
            "rayVisibilityDiffuse": true, "rayVisibilityGlossy": true,
            "spread": 80, "colorMapTileId": -1 },
    "a1": { "...": "默认值即可" },
    "a2": { "...": "默认值即可" }
  }
}
```

### 5.1 渲染器

| 字段 | 建议值 | 作用 | 说明 |
|---|---|---|---|
| `rendererMaxSample` | 1024 ~ 65536 | 采样上限 | 渐进累积，设大不会变慢，只决定“最多干净到哪” |
| `rendererEnableDenoise` | true | 降噪 | 开启后低采样下也能出图，建议常开 |
| `displayDevice` | `sRGB` | 显示色彩空间 | 另有 `Display p3` |
| `viewTransform` | `ACES` | 色调映射 | `Standard` 更平；ACES 高光过渡更接近摄影 |
| `lutName` | `""` | LUT 文件 | 留空表示不用；使用的文件需打进包 |
| `lutIntensity` | 1 | LUT 强度 | 仅在 `lutName` 非空时有效 |
| `maxRayBounces` | 6 | 最大反弹次数 | 8 以上收益递减，仅增加耗时 |
| `hideRenderPlane` | false | 隐藏平面 | 调试用，正常出图保持 false |
| `hideRenderTriangle` | false | 隐藏三角面 | 调试用 |
| `hideUnselected` | false | 只渲染选中对象 | 调试用 |
| `selectMaterialId` | 0 | 选中材质高亮 | 调试用，出图保持 0 |
| `useBlueNoise` | true | 蓝噪声采样 | 减少低频噪点，建议开启 |

### 5.2 背景

| 字段 | 说明 |
|---|---|
| `useColor` | true = 纯色背景（用 `color`）；false = 用 HDR 环境 |
| `color` | `useColor:true` 时的背景色（线性）；与 HDR 互斥 |
| `texture` | HDR 文件名，**必须与 zip 内文件完全一致** |
| `intensity` | HDR 能量倍率。实测 0.3–0.5 最稳；>0.8 容易让阻焊整片泛白 |
| `rotation` | 水平旋转（弧度），用于调整主光方位 |
| `hide` | true = **只照明不显示**（产品图常用黑底）；false = 背景可见 |
| `hideColor` | `hide:true` 时的背景色，**按线性解释**：`#0B0C0F` 会显示成约 `#3A3A3A`，要纯黑请写 `#000000` |

### 5.3 灯光（重要：版本差异）

资源包可配置三盏区域灯 `a0` / `a1` / `a2`：

| 字段 | 类型 | 说明 |
|---|---|---|
| `color` | 十六进制字符串 | 灯光颜色（线性解释） |
| `power` | number | 功率；引擎内部按 `power × 场景尺度²` 换算，不要按物理瓦数理解 |
| `normalize` | boolean | 是否归一化；建议 false |
| `size` | `{width, height}` | 发光面尺寸；与 `position` 同一尺度 |
| `position` | `[x, y, z]` | 相对场景中心的位置 |
| `rotate` | `[x, y, z]` | 发光面朝向（角度） |
| `hideRender` | boolean | 渲染时隐藏发光面；true 时只发光不可见 |
| `cameraVisible` | boolean | 相机是否能看到发光面 |
| `castShadow` | boolean | 是否投射阴影 |
| `rayVisibilityDiffuse` | boolean | 是否参与漫反射照明 |
| `rayVisibilityGlossy` | boolean | 是否参与镜面/光泽照明 |
| `spread` | number（度） | 发光角；180 = 半球，数值越小越聚光 |
| `colorMapTileId` | number | 灯光贴图索引；无贴图时为 -1 |

版本行为：

- 资源包在 `param.json` 中配置的 `a0` 主光，**初始化时**会被应用；
- **实测 4.1.109**：运行期 `setRenderConfigurations({ light })` 回读成功，但导出画面逐像素不变；热加载 `importPackage` 时 `light` 同样不生效；
- 因此**照明应以 HDR 为主**，灯只作为可选补光，并务必在目标版本上验证。

灯参数单位提示：引擎内部会按场景尺度换算（`power * scale²`、`position` 归一化），推荐组合为 `power 0.8 / size 0.011 / position z=508`，**不要按物理直觉随意放大**。

---

## 六、贴图规范

| 用途 | 格式 | 颜色空间 | 备注 |
|---|---|---|---|
| 基础色 `colorTexture` | PNG | **sRGB** | 文件名建议带 `basecolor` / `Color` |
| 法线 `normalTexture` | PNG | **线性** | 切线空间，**+Y 朝上**（OpenGL 约定） |
| 粗糙度 `roughnessTexture` | PNG | **线性** | 灰度，单通道即可 |
| 金属度 `metallicTexture` | PNG | **线性** | 灰度，单通道即可 |
| 环境光 `background.texture` | HDR / RGBE | 线性 | 等距柱状投影，**不进入图集**（1024×512 / 2048×1024 均可） |

> 前四类 PBR 贴图会进入引擎纹理图集，**必须 2048×2048**；HDR 走独立采样，不受该限制。

### 6.1 尺寸与体积

| 要求 | 说明 |
|---|---|
| 尺寸严格 2048×2048 | 引擎图集 `UNIT_SIZE = 2048`，其他尺寸会抛 `Texture size mismatch` |
| 压缩只在编码层做 | RGB→L 灰度、去 alpha、PNG level 9；**不能降分辨率** |
| 不要输出 RGBA | alpha 通常全 255，纯冗余；8-bit 足够，16-bit 会让体积翻约 4 倍 |

### 6.2 命名与引用

```json
"colorTexture": { "name": "my_mask_basecolor.png", "scale": 1 }
```

| 子字段 | 类型 | 说明 |
|---|---|---|
| `name` | string | zip 内文件名，必须逐字符一致 |
| `scale` | number | UV 平铺倍数；1 = 原样，2 = 平铺 2×2 |

| 规则 | 说明 |
|---|---|
| `name` 逐字符一致 | 必须与 zip 内文件名完全相同，含中文、大小写、空格 |
| 文件缺失不报错 | 贴图不显示，表现为材质全黑或只剩纯色 |
| 同一文件可复用 | 多条材质可引用同一文件，不必重复打包（重复只会让 zip 变大） |

### 6.3 UV 与平铺

| 规则 | 说明 |
|---|---|
| `scale` 是平铺倍数 | >1 时贴图必须**无缝可平铺**，否则会看到接缝 |
| 不支持 alpha 镂空 | 光追按不透明处理，镂空需要几何实现 |
| 依赖 UV | PCB 本体 UV 由引擎生成，给阻焊加法线贴图可能拉伸，需先验证 |

---

## 七、HDR 环境光规范

### 7.1 规格

| 项目 | 规格 |
|---|---|
| 投影 | 等距柱状（equirectangular）：`az = u·360° − 180°`，`el = (0.5 − v)·180°`（v=0 为天顶） |
| 格式 | Radiance `.hdr`（`#?RADIANCE` / `FORMAT=32-bit_rle_rgbe`），逐扫描线 RLE |
| 尺寸 | 建议 1024×512 或 2048×1024，长宽比 2:1 |
| 方向 | 保持标准 `-Y +X`，当前版本不支持其他 orientation 变体 |

### 7.2 一个可直接复用的「工作室」环境

四柔光箱，实测观感自然、不易过曝：

| 用途 | az | el | 角宽 aw×ah | 强度 I | 软边 |
|---|---|---:|---:|---:|---:|
| 主光 key | −40° | 40° | 26°×17° | 22 | 11° |
| 顶光 top | −8° | 76° | 64°×14° | 4.5 | 12° |
| 补光 fill | 62° | 10° | 34°×20° | 7 | 11° |
| 轮廓光 rim | 165° | 30° | 38°×12° | 13 | 11° |

基础环境（避免纯黑导致死阴影）：

```
base = 0.004 + 0.007·(el>0 ? 1 : 0.5) + 0.006·exp(-|el|/18)
R = base, G = base·0.985, B = base·1.015
```

柔光箱叠加：

```
daz = |wrap180(az − az0)|
del = |el − el0|
fx  = max(0, 1 − max(0, daz − aw/2)/soft)
fy  = max(0, 1 − max(0, del − ah/2)/soft)
f   = fx·fy
core= max(0, 1 − max(daz/(aw/2), del/(ah/2))²)
I_eff = I · (0.65·f + 0.35·core·f)
RGB += I_eff · color
```

### 7.3 能量与观感

- HDR 是**唯一可靠的照明来源**（见 5.3）；
- 亮度基准：`background.intensity` 0.30–0.50 对应「板面亮度中位数 p50 ≈ 16–25/255」的产品图观感；
- 若发现阻焊整片发白：先降 `intensity`，再降阻焊 `iorLevel`，最后才考虑改色；
- 想看环境倒影又不想要黑底：`hide:false`，但柔光箱会成为可见亮斑——产品图一般 `hide:true` + 后期合成。

---

## 八、颜色空间：一切都按线性浮点写

**这是上一版最大的翻车点。** 引擎把 `pbr.color`、`hideColor` 当作**线性**值直接参与光照，不做 sRGB 解码。

| 写法 | 结果 |
|---|---|
| 直接抄设计稿 sRGB 值（如 `#12305E` → `0.070,0.188,0.369`） | 渲染明显偏亮、发灰/发白 |
| 先转线性（`#12305E` → `0.006,0.0296,0.1119`） | 与设计稿一致 |

转换规则（sRGB 分量 u，取值 0~1）：

```
u ≤ 0.04045      →  u / 12.92
u > 0.04045      →  ((u + 0.055) / 1.055) ^ 2.4
```

十六进制色先按 `分量/255` 归一化再套用上式。常用对照：

| 设计色 (sRGB) | 线性值 |
|---|---|
| `#12305E` | `0.006, 0.0296, 0.1119` |
| `#0A1B38` | `0.0030, 0.0116, 0.0429` |
| `#C9A24A`（金） | `0.5721, 0.3744, 0.0742` |
| `#909090` | `0.2789` |

要点：

| 要点 | 说明 |
|---|---|
| 金属的 `pbr.color` 是反射率 | 物理正确值，不是“涂装色”；金色写 `[0.749,0.604,0.224]` 而不是亮黄 |
| `phong.diffuse` 保留原始编码 | 它是匹配键，不是显示色 |
| `hideColor` / `light.color` | 同样按线性解释；纯黑用 `#000000` |

---

## 九、`scene_model/`：是否添加与如何开发

### 9.1 要不要加

| 需求 | 是否添加 |
|---|---|
| 需要桌面摆件（电烙铁、焊锡丝、镊子、桌面垫等场景图） | 添加 |
| 只出板子的产品图，只要干净背景 | 不添加（可省约 27 MB 体积） |

添加后，道具会按世界坐标与 PCB 一起出现在场景中；不添加时场景里只有板子本身。

### 9.2 目录与清单

```
scene_model/
├─ manifest.json
├─ <name>.obj
└─ <name>.mtl
```

`manifest.json` 列出全部道具：

```json
{
  "version": "1.0.0",
  "elements": [
    { "name": "Soldering Iron",
      "obj": "scene_model/scene_soldering_iron.obj",
      "mtl": "scene_model/scene_soldering_iron.mtl" }
  ]
}
```

| 字段 | 说明 |
|---|---|
| `name` | 显示名称，可自定义 |
| `obj` / `mtl` | 相对 zip 根的路径，必须与实际文件逐字符一致 |
| 顺序 | 不影响渲染 |

### 9.3 OBJ 要求

| 项 | 要求 |
|---|---|
| 坐标 | **世界坐标**，零变换装载；位置完全由顶点决定 |
| 单位 | 与 PCB 一致（mil），否则会过大或过小 |
| `mtllib` | 必须指向同目录的 MTL，文件名与 manifest 一致 |
| `usemtl` | 引用的材质名必须在 MTL 中有对应 `newmtl` |
| 规模 | 建议不超过 5 万顶点/件，过大会显著拖慢加载 |

### 9.4 MTL 要求

| 项 | 要求 |
|---|---|
| 材质名 | 与 OBJ 的 `usemtl` 保持一致，**不能改名** |
| `Kd` | 决定该部件的颜色编码，光追通过它匹配 `material_table.json` |
| 贴图 | MTL 里不要写 `map_*`；道具贴图通过材质表引用（见 9.5） |

### 9.5 道具贴图如何生效

1. 在 MTL 中设置 `Kd`（例如 `0.800 0.800 0.800`）；
2. 在 `material_table.json` 中增加一条 `phong.diffuse = [0.8, 0.8, 0.8]` 的条目；
3. 在该条目的 `pbr` 中填写 `colorTexture` / `normalTexture` / `roughnessTexture`，`name` 与包内文件一致；
4. 引擎构建道具时按颜色匹配到该条目，贴图随之生效。

> 贴图必须 2048×2048；同一套贴图可被多个部件引用。

### 9.6 `debug/` 为什么不会出现在包里

`debug/`（`camera.json`、`used_material.json`、`plane.json` 等）是引擎在**导出场景时**生成的调试数据，只对导出它的那一块板有效（材质槽位、相机、板层平面都绑定具体几何）。

本包不包含 `debug/`，制作时也不需要生成它；资源包只负责外观，不携带相机与板层数据。

---

## 十、加载与生效机制

```
eext-common 菜单「开启光追渲染」
  → 未安装则从 modules.lceda.cn 下载 zip（流式，带进度）
  → 存入扩展存储键 'raytracer resource pack'
  → eda.pcb_RayTracerEngine.init(uuid)
  → pro-api → pro-ui → pro-pcb → smt-gl-engine
  → 通过 uuid 反查 IndexedDB 取 zip
  → 解包，构造 getAsset(path) 闭包
  → MaterialComponent.decode()
  → 渲染
```

解码顺序：

1. `inner_material_table.json` → 覆盖内置材质（**先**执行）
2. `material_table.json` → 清空并重建调色板
3. `debug/used_material.json` → 恢复材质槽位（若包内存在；本包不含）

### 10.1 为什么「初始化时加载」很重要

`getIndex` 的颜色匹配发生在**构建几何时**。若包在几何构建后才加载：

- 调色板不会重新匹配（器件继续用旧的 / `fromPhong` 的材质）；
- `usedMaterials` 的槽位索引仍指向旧表，新表长度不同就会**张冠李戴**。

**实测**：热加载一个 5 条新表后，把 0–4 号材质设为纯红，整块板（基板、丝印、焊盘、阻焊）全部变红——证明索引直接落到槽位 0–4。

### 10.2 热加载（`importPackage`）的正确用法

- 适合：条目**数量与顺序都不变**的包（例如本板导出→改参数→回灌）；
- 不适合：换板复用；
- 需要热验证资源包时，正确顺序是：**先停止光追 → 等待 2 秒 → 重新开启（重建几何）→ 再导入资源包**；
- 最稳妥的方式：把包写入扩展存储后重启光追，让它在初始化时加载（生产路径）。

---

## 十一、资源包在运行时的行为（不需要写代码）

| 行为 | 说明 | 建议 |
|---|---|---|
| 初始化时加载 | 调色板匹配发生在构建几何时，之后换包不会重新匹配，材质槽位还可能错位 | 资源包随初始化一起加载 |
| 热加载有前提 | 只适合条目数量与顺序都不变的包 | 换板复用时走初始化加载 |
| 重复初始化会报错 | 引擎初始化不是幂等的，渲染中再次初始化会抛 `this.currentTask !== null` | 先停止光追、等待，再重新开启；或关闭 3D 页重开 |
| 相机不属于资源包能力 | 4.1.109 中运行期修改相机不生效 | 本包不包含 `debug/`，无需处理相机 |
| 导出分辨率 = 画布尺寸 | 资源包没有分辨率字段 | 需要固定尺寸时调整窗口/画布 |
| 灯的生效时机 | `param.json` 的灯在初始化时应用；运行期改灯不生效 | 照明以 HDR 为主 |
| 贴图尺寸 | 图集内的 PBR 贴图必须 2048×2048；HDR 不进入图集 | HDR 可用 1024×512 |

---

## 十二、用 AI 生成材质包的完整流程

### 12.1 能做什么、不能做什么

| 可生成 | 说明 |
|---|---|
| `inner_material_table.json` | ✅ 最推荐：给 PCB 换肤（镀层、基板、阻焊） |
| `material_table.json` | ✅ 由「颜色→PBR」规则批量生成调色板 |
| PBR 贴图集 | ✅ AI 出图可行，注意通道与颜色空间 |
| `param.json` | ✅ 背景/渲染器参数 |
| HDR 环境 | ✅ 程序化生成最可靠（见第七章） |
| `scene_model/` | ⚠️ 可生成简单摆件，几何精度不保证 |
| 替代 PCB 几何 | ❌ 几何由 EDA 数据生成 |
| 决定材质槽位索引 | ❌ 由引擎按颜色匹配烘焙 |

### 12.2 直接给 AI 的提示词模板

```
请生成一个嘉立创EDA光追通用材质包，输出 zip（根目录直接放文件），包含：
material_table.json   —— 1384 条左右的“颜色→PBR”调色板
inner_material_table.json —— 27 条内置材质覆盖
param.json            —— 工作室渲染参数
studio.hdr            —— 1024x512 工作室环境光

硬约束：
1. 所有颜色用 0~1 线性浮点数组，禁止十六进制字符串；设计色请先做 sRGB→线性转换。
2. material_table 每条：phong.diffuse 保留给定编码色（匹配键），pbr.color 写设计色。
3. 金属(焊盘/引脚) metallic=1，roughness 0.25~0.35；黑色环氧 metallic=0，roughness 0.45~0.55。
4. 阻焊：roughness 0.36~0.46，ior 1.42~1.45，iorLevel 0.15~0.25，clearcoat 0.06~0.18。
5. HDR 采用等距柱状 + RGBE；四柔光箱，主光强度 20~25，环境基底 0.004~0.008。
6. 场景道具按第九章可选；不包含 debug/、相机与板子几何。
7. JSON 为 UTF-8，字段严格按给定 schema，多余字段可忽略。
```

### 12.3 生成后的自检

```text
□ 结构：根目录只有 material_table.json / inner_material_table.json / param.json / *.hdr / *.png
□ 调色板：数组非空；每条 phong.diffuse 保留原始编码
□ 内置材质：key 全部在引擎字典内
□ 数值：所有 color 分量 0~1；metallic / roughness 0~1
□ 金属条目 metallic = 1；阻焊 iorLevel ≤ 0.3
□ 引用：每个 *Texture.name 与 background.texture 都能在包内找到同名文件
□ 贴图：进入图集的贴图均为 2048×2048
□ 观感：初始化加载后检查——阻焊深色有漆面、焊盘有金属反射、器件本体近黑、阴影不死黑
```

---

## 十三、验证与调参方法

### 13.1 验证流程

| 步骤 | 操作 | 说明 |
|---|---|---|
| 1 | 打开 PCB 的 3D 预览页 | 光追依赖该页面，未打开时无法初始化 |
| 2 | 菜单「开启光追渲染」 | 确认资源包已加载 |
| 3 | 观察画面 | 画面逐步变干净说明采样在累积 |
| 4 | 等待 2～5 分钟后另存 PNG | 检查材质层次与噪点 |
| 5 | 换板重复 1～4 | 确认没有色带、错位或多余道具 |

### 13.2 调参顺序（按影响从大到小）

| 顺序 | 调整项 | 影响 |
|---|---|---|
| 1 | 背景 `intensity` | 整体曝光，最先调 |
| 2 | HDR 柔光箱强度 / 软边 | 明暗结构与阴影柔和度 |
| 3 | 阻焊 `iorLevel` + `clearcoat` | 漆面层次，是否整片泛白 |
| 4 | 调色板分类阈值 | 本体是否够黑、引脚是否有金属感 |
| 5 | `viewTransform` | Standard 偏平，ACES 更接近摄影 |
| 6 | `maxRayBounces` / 采样 | 只影响画面干净程度 |

### 13.3 常见现象的定位

| 现象 | 优先检查 |
|---|---|
| 整板发白/发灰 | 颜色是否忘了 sRGB→线性；`intensity` 是否过高 |
| 阻焊整片高光 | `iorLevel` 是否 ≥0.5；HDR 主光是否过亮 |
| 焊盘像黄色塑料 | 调色板缺金属条目，或缺 `material_table.json` |
| 器件本体是浅灰 | 同上，落到了 `fromPhong` 的 metallic 0 / roughness 1 |
| 整图纯黑 | HDR 文件名不匹配 / HDR 未打进包 / `hide:true` 且无其他光源 |
| 导入后整板变红/错乱 | 热加载索引错位，改用 dispose→init 后导入 |
| 导入后多出元件/摆件 | 包里带了 `scene_model/` |

---

## 十四、常见坑（汇总）

| 现象 | 原因 | 解决 |
|---|---|---|
| 材质全黑 | 贴图缺失或 name 写错；`material_table` 为空 | 校验文件名；确保数组非空 |
| 贴图发灰/过曝 | 基础色未按 sRGB 处理（或反之） | 基础色 sRGB，法线/粗糙/金属度线性 |
| 凹凸方向反了 | 法线贴图 Y 轴方向不对 | 使用 +Y up（OpenGL） |
| 内置材质没生效 | key 拼写与引擎字典不一致 | 见 4.2 名单 |
| 包很大 | 贴图 8K / RGBA / 未优化 | 保持 2048 + 灰度 + 去 alpha + PNG level 9 |
| 报 `Texture size mismatch` | 贴图不是 2048×2048 | 恢复为 2048（图集单元固定 2048） |
| 上传后重启丢失 | 未走生产链路 | 资源包需经 `exportPackage` 导出并替换 URL 常量 |
| 颜色整体偏亮 | 颜色按 sRGB 写 | 转线性（第八章） |
| 热加载后材质错位 | 索引已固化 | 初始化时加载 / dispose→init |
| 导入别的板出现重复几何 | 包内含 `scene_model/` | 不需要道具时移除该目录与 manifest 条目 |
| `init()` 一直不返回 | 3D 预览页未打开 | 先打开 3D 预览 |
| 导出图尺寸不对 | 分辨率 = 画布尺寸 | 固定窗口大小 |
| 灯光调了没反应 | 4.1.109 运行期 `light` 不生效 | 以 HDR 照明为主 |
| `Error: this.currentTask !== null` | 重复调用 `init()`：`startRender()` 检测到渲染循环已在运行（引擎无幂等保护） | 先 `dispose()` 并等待，再 `init()`；或关闭/重开 3D 页；避免并发调用 |

---

## 十五、发版检查清单

| 步骤 | 操作 | 说明 |
|---|---|---|
| 1 | 更新 `raytracer-assets/assets` | 放入新的材质表与贴图 |
| 2 | 执行 `npm run build:prod` | 产出 zip |
| 3 | 上传到 `modules.lceda.cn/raytracer-assets/<version>.<hash>/` | 版本目录与 hash 需对应 |
| 4 | 修改 `eext-common/packages/raytracer/index.ts` | `RESOURCE_PACK_URL`、`RESOURCE_PACK_VERSION`、`RESOURCE_PACK_SIZE`（进度百分比依赖，必须同步） |
| 5 | 重新构建扩展 | 生效新地址 |
| 6 | 两块板各验证一次 | 阻焊/基板/丝印/焊盘/本体正常，且无多余几何 |

---

## 附：本文档对应的实测版本

| 组件 | 版本 |
|---|---|
| 嘉立创EDA 专业版 | 4.1.109 |
| smt-gl-engine | 4.1.44 |
| pro-api | 0.3.84 |

接口行为（尤其是 `light`、`camera`、`importPackage` 的索引语义）随版本可能变化，升级后请按第十三章复验。
