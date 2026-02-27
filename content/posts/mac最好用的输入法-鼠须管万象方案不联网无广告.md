---
title: "Mac 最好用的输入法（鼠须管 + 万象输入法方案 完美）：不联网，无广告"
date: 2026-02-26
lastmod: 2026-02-27
description: "在 Mac 上用鼠须管（Squirrel）搭配万象拼音，实现本地、不联网、无广告的高质量中文输入，含数字键选词、中英翻译、以词定字与迁移指南。"
categories: ["效率与工具"]
tags: ["Mac", "输入法", "鼠须管", "RIME", "万象拼音", "环境搭建"]
keywords: ["Mac 输入法推荐", "鼠须管 万象拼音", "RIME 配置教程", "不联网输入法", "无广告输入法"]
ShowToc: true
faq:
  - q: "鼠须管 + 万象拼音真的不联网吗？"
    a: "常规输入与词库调用可在本地完成，不依赖云账号和广告服务，配置文件也可自行审计。"
  - q: "数字键不能选词一般是什么原因？"
    a: "多由方案键位冲突或自定义配置覆盖引起，检查 default.custom.yaml 与 wanxiang.custom.yaml 的按键配置即可。"
draft: false
---

## 前言

在 Mac 上想找一款**不联网、无广告、可高度自定义**的中文输入法，很多人会选 **RIME** 系。  
其中 **鼠须管（Squirrel）** 是 macOS 上的 RIME 前端，搭配 **万象拼音** 方案，可以做到：

- **完全本地**：词库与计算都在本机，不依赖云、不联网
- **无广告、无隐私上传**：数据不出本机
- **可定制**：皮肤、快捷键、方案切换、翻译模式等均可自己调

本文基于实际使用与配置整理：从安装到「鼠须管 + 万象」的推荐配置，再到常用功能与多机迁移，方便你一步到位用上这套方案。

---

## 一、为什么选「鼠须管 + 万象拼音」？

- **鼠须管（Squirrel）**：RIME 官方推荐的 macOS 前端，稳定、占用小、支持自定义外观与按键。
- **万象拼音（rime_wanxiang）**：基于 RIME 的拼音方案，词库全、支持**中英翻译**（输入中文看英文、输入英文看释义）、**以词定字**、**数字键选词**（已修复与声调的冲突）、**动态日期/时间/农历/计算器**（Lua 脚本）等，且开源可自维护。

组合起来：**不联网、无广告、无账号、数据全在本地**，适合在意隐私与稳定性的用户。

---

## 二、安装与初次配置

### 2.1 安装鼠须管

1. 打开 [RIME 官网下载页](https://rime.im/download/)，选择 **鼠须管 (Squirrel)** 下载并安装。
2. 安装后在 **系统设置 → 键盘 → 输入法** 中添加「鼠须管」，并勾选「使用大写键切换 ABC」等按需设置。
3. 点击菜单栏输入法图标 → **「用户设定...」**，会打开配置目录：`~/Library/Rime`。

### 2.2 安装万象拼音方案

**方式 A：用 Plum（东风破）一键安装（推荐）**

在终端执行（需已安装 Git）：

```bash
cd ~
git clone --depth=1 https://github.com/rime/plum.git
cd plum
bash rime-install amzxyz/rime_wanxiang
```

国内网络若较慢，可配置 Git 代理后再执行；安装完成后，Plum 会把方案与词库写入 `~/Library/Rime`。

**方式 B：手动下载**

从 [amzxyz/rime_wanxiang](https://github.com/amzxyz/rime_wanxiang) 下载仓库，将 `*.schema.yaml`、`*.dict.yaml`、`lua/`、`opencc/`、`dicts/` 等按目录结构复制到 `~/Library/Rime`。

### 2.3 配置「只用万象」、避免无候选框

若之前装过多种方案（如 rime_ice、luna_pinyin 等），又未全部保留，容易在 `schema_list` 里留下**不存在的方案**，导致 RIME 报错「missing input schema」或**没有候选框**。

建议在 `default.custom.yaml` 里**只保留已安装的方案**，例如仅万象：

```yaml
patch:
  "menu/page_size": 9
  schema_list:
    - schema: wanxiang
    - schema: wanxiang_english
```

保存后，菜单栏点击 **「重新部署」**，再按 `Control + ` `（反引号/Tab 上方）` 切换到「万象拼音」。

### 2.4 关闭「按应用强制英文」（避免误以为不能打中文）

若在 `squirrel.custom.yaml` 里对大量 App 配置了 `app_options` 且 `ascii_mode: true`，会导致在这些应用里**默认英文、看不到中文候选**。  
若不需按应用锁定英文，可清空或删掉该段：

```yaml
patch:
  app_options: {}
```

---

## 三、万象拼音常用功能（不联网、本地完成）

以下功能均在本地生效，无需联网。

### 3.1 数字键选词

- 已通过方案配置**释放被声调占用的数字键**，直接按 **1～9** 即可选择对应候选词。

### 3.2 中英翻译（输入时看释义）

- **快捷键**：`Control + E`（开启/关闭）
- **效果**：输入拼音时，候选词旁显示英文释义；输入英文时显示中文释义。  
  例：输入 `huanying` → 候选「欢迎 (Welcome)」。

### 3.3 以词定字

- 输入完整拼音后：
  - **`[`**（左中括号）→ 上屏该词的**首字**
  - **`]`**（右中括号）→ 上屏该词的**末字**  
  例：输入 `shuxiguan`，按 `[` 上屏「鼠」，按 `]` 上屏「管」。

### 3.4 动态内容（Lua）

- `orq` 或 `/rq`：当前日期
- `osj` 或 `/sj`：当前时间
- `onl` 或 `/nl`：农历
- `ov` + 算式：计算器，如 `v1+2*3` → 7

### 3.5 符号与短语

- 输入 **`;`**（分号）作引导符，再按约定字母可出常用符号或短语（依方案配置）。

---

## 四、推荐外观与快捷键（可选）

在 `squirrel.custom.yaml` 中可设置皮肤与候选栏样式，例如：

```yaml
patch:
  style:
    font_face: "PingFang SC"
    font_point: 16
    candidate_list_layout: linear
    inline_preedit: true
  preset_color_schemes:
    mac_light:
      name: "简约白"
      back_color: 0xffffff
      hilited_back_color: 0xd05b21
      # ... 其他颜色
```

在 `default.custom.yaml` 中可配置：

- 逗号/句号翻页
- `Control+Shift+3` 中英标点
- `Control+Shift+4` 简繁切换
- 小键盘 0～9 选词

按需调整后**重新部署**即可生效。

---

## 五、保持更新与日常维护

- **万象拼音**：定期查看 [amzxyz/rime_wanxiang](https://github.com/amzxyz/rime_wanxiang) 的更新，将最新 `wanxiang.*`、`dicts/`、`lua/` 等覆盖到 `~/Library/Rime`，再点击「重新部署」。
- **多机同步**：在 `installation.yaml` 中配置 `sync_dir: "/你的云盘路径/RimeSync"`，通过菜单栏「同步」在多台 Mac 间合并用户词库（仍为本地 + 自建同步，不经过第三方服务器）。
- **卡顿/异常**：可删除 `~/Library/Rime/build` 目录后重新部署；若出现无候选框，优先检查 `schema_list` 是否包含未安装的方案，以及 `app_options` 是否误开全局英文。

---

## 六、迁移到新 Mac 或 Windows

- **备份**：打包 `~/Library/Rime` 下所有 `*.custom.yaml`、`*.schema.yaml`、`*.dict.yaml`、`lua/`、`opencc/`、`custom_phrase.txt` 等。
- **新 Mac**：安装鼠须管后，将备份覆盖到 `~/Library/Rime`，再「重新部署」并切换方案为万象拼音。
- **Windows（小狼毫）**：安装 [小狼毫](https://rime.im/download/) 后，将除 `squirrel.custom.yaml` 外的配置与词库复制到 `%AppData%\Rime`；外观需在 `weasel.custom.yaml` 中单独配置（如 `horizontal: true` 等），详见各平台文档。

---

## 小结

- **鼠须管 + 万象拼音** 在 Mac 上可实现**不联网、无广告、全本地**的高质量中文输入。
- 安装后建议**只保留万象**在 `schema_list` 中，并关闭按应用强制英文，可避免「无候选框」等问题。
- 数字键选词、中英翻译、以词定字、日期时间农历与计算器等，均在本地完成，无需上传数据。
- 通过 `sync_dir` 可自建多端词库同步；迁移到新机时备份 `~/Library/Rime` 并重新部署即可。

如果你愿意进一步固化配置，可以把 `default.custom.yaml`、`squirrel.custom.yaml`、`wanxiang.custom.yaml` 用 Git 或网盘保存一份，换机或重装时直接覆盖，即可快速恢复「鼠须管 + 万象」的完美体验。
