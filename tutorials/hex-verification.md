# anomalyco/hex 验证报告（候选 8 号）

- 验证日期：2026-11-10
- 仓库：https://github.com/anomalyco/hex
- 快照数据：6,046 Star / 370 Fork / 193 个 open issues（验证当日用已连接 GitHub 账号只读核实）；语言 Rust；MIT 协议
- 验证方式：官方 README/docs/源码/Issues 核查 + 第三方 Linux 实测交叉验证 + 本沙箱（Ubuntu 24.04 x86_64、AVX2、glibc 2.39、无显示器）真实下载安装并转写中文语音

## 结论：暂缓 ⏸️

不是放弃：最关键的未知数「中文听写」实测过关且明显好于预期。
暂缓的理由是另外两个硬伤（见下「为什么暂缓」）：付费主力平台没法完成真机实测、Windows/Linux 仍是 beta 且 Windows 安装包未签名。等再启动条件满足后再做，届时中文引擎部分无需重验。

---

## 一、中文听写验证：过关 ✅（且推翻了初筛假设）

### 1. 初筛假设是错的：中文不靠 Parakeet
- Parakeet 系列只是英文/欧洲语言模型：Parakeet Unified English = 英文专用，Parakeet v3 = 25 种欧洲语言，均不含中文。
- Hex 的模型目录（约 14 个模型）是**按语言推荐模型**的，中文有专门模型：
  - **Qwen3-ASR 0.6B —— 中文官方推荐**（覆盖 30 种语言，中文为其主力）
  - **SenseVoice Small —— 中文备选，标注「最快」**（支持中文、粤语、英、日、韩）
  - Whisper large-v3-turbo —— 中文第三选择，优势是支持 Vocabulary 词表提示
- 源码内嵌的官方基准（`src/transcription/model_benchmarks.rs`）：Qwen3-ASR 在 **FLEURS 中文测试集 CER 7.57%**（字错误率约 7.6%，属可用且不错的水平）。
- 界面本地化：简体中文界面自 app-v2.3.0 起就有；v3.3.0 新增繁体中文界面（见官方 Releases）。

### 2. 本沙箱实测（真实跑通，非文档转述）
环境：官方 Linux .deb（27.7 MB，v3.3.0）解包运行；模型用官方命令 `hex model install qwen3_asr06_b --language zh` 从 Hugging Face 下载 **Qwen3-ASR 0.6B，850 MB，校验通过**，约 5 分钟下完。

转写结果（`hex model transcribe --language zh`，CPU 推理）：

| 样本 | 时长 | 输出 | 判定 |
|---|---|---|---|
| FunASR 官方中文示例（真人录音） | 5.5s | 「欢迎大家来体验达摩院推出的语音识别模型。」 | ✅ 与官方标准文本逐字一致，零错误，带标点 |
| 雷军演讲录音片段（真人自然口语，截取 60–80s 段） | 20s | 「还是想办一个比较大的聚会，然后呢，让我们的新朋友、老朋友一起聚一聚。今天的话呢，我们就在北京的国家会议中心呢，举办了这么一个活动。现场呢，来了很多人，大概有三千五百人。」 | ✅ 流畅准确：语气词、数字「三千五百人」、专名「国家会议中心」全对 |
| 台湾普通话公告样本 | 5.6s | 「開放時間：早上九點至下午五點。」 | ✅ 内容准确，但输出为**繁体字**（样本本身为台湾腔调；简体用户需注意字形输出可能随口音/样本漂移到繁体） |

速度参考：本沙箱为共享弱 CPU（4 核），5.5s 音频含模型加载共约 10s、20s 音频约 27s；第三方在 8 核 Xeon 纯 CPU 上实测 9s 中文片段 3.0–4.1s（Qwen3-ASR）。普通用户电脑只会更快。

### 3. 第三方实测交叉验证
AI SI Daily 于 2026-11-01 在 Debian 13（纯 CPU、无 GUI）实测了 Linux 引擎：中文提示词用 Qwen3-ASR 转写 3.0–4.1s 完成，快于 Whisper 的 11.4–13.7s 且错误更少。其结论与本报告一致：「Does it support Chinese? Yes.」

### 4. 中文的已知短板（教程将来必须正面处理，不能回避）
- **中英混读的代码词是弱项**：第三方实测中，中文提示里夹的英文标识符会被拆/错，如 useEffect 被拆成「use effect」、UserList 变「user list」、Vitest 三个模型都没对。Hex 的解法是 Vocabulary 词表（Whisper/Qwen 可读词表作提示），但效果未实测。
- **长音频/音乐开头可能漏识别**：雷军样本前 25s（活动开场段）转写输出为空；中段纯人声则完全正常。听写场景是短句按住说，影響有限，但不能拿它当长录音转写工具卖。
- 微软云 TTS 在本沙箱网络下连不上，未能自制「中英混读自定义文本」做第四组样本；混读结论来自第三方实测 + 官方词表功能存在性，未亲自复测。

---

## 二、平台验证：Mac 唯一稳定，Windows/Linux 仍是 beta ⚠️

| 平台 | 状态（官方文档原文口径） | 硬性要求 |
|---|---|---|
| macOS | 主力/稳定 | **仅 Apple 芯片 + macOS 15 或更新**；Intel Mac 不支持 |
| Windows | **Beta**（docs/windows.md 标题即「Windows Beta」） | x64/ARM64；**安装包未代码签名**，SmartScreen 会警告；快捷键 Ctrl+Space |
| Linux | **Beta**（docs/linux.md 标题即「Install The Linux Beta」） | x86_64/ARM64；桌面只瞄准 i3/X11 与 wlroots 系 Wayland（Hyprland/Sway 等），GNOME/KDE 走 portal 兜底；**必须有 Vulkan 驱动（纯 CPU 推理也要）**；CPU 须 AVX2；.deb/.rpm 须 glibc ≥ 2.39 |

本沙箱实测到的边界：
- ✅ .deb 在 Ubuntu 24.04 上解包即跑，二进制、CLI、模型下载、CLI 转写全部可用 —— **识别引擎层验证完成**。
- ❌ **桌面端到端流程无法验证**：沙箱无显示器、无 Vulkan 驱动、无 ALSA 音频、无 systemd 用户会话，`hex start` 后台服务始终不应答，`hex model list` 都要求主程序在运行。即：快捷键录音、自动粘贴、设置向导这些小白真正会卡住的环节，我们和第三方测评者一样**都没真机跑过**（第三方同样只测了引擎 CLI、未测 GUI）。

---

## 三、为什么暂缓（而不是现在做）

1. **卖点与本生意的核心承诺冲突**：这门生意卖的是「我实测过一遍」。但付费主力人群（Mac/Windows 小白）的桌面流程，我们现在无法在任何一台真机上实测；拿 beta + 未签名的 Windows 版去卖小白教程，售后和退款风险高，也违背实测承诺。
2. **安装本身太简单，撑不起客单价**：全平台都是一条命令/一个安装包。能卖的只剩「中文模型选型 + 词表调教 + 850 MB 模型在国内网络的下载避坑（Hugging Face 直连慢/不稳）」的实战包，而这部分恰恰需要真机长测才能写扎实。
3. **中文空白已在收窄**：已有较完整的中文介绍（含模型表和中文实测，AI SI Daily 中文版）及多篇中文收录/解析；空白只剩「中文听写专项实战」一个角度，窗口不像 ha-mcp 那样干净。
4. **保鲜期短**：迭代极快（10 月 31 日 v3.0.0 重写发布，到 11 月 10 日已 v3.3.0；193 个 open issues），教程大概率月级过期，维护成本计入后不如先做排名更靠前的选题。

## 四、再启动条件（满足任一即升级为「做」）

- 能拿到一台 Apple 芯片 Mac 或 Windows 真机，把桌面端到端流程（安装向导 → 模型下载 → 快捷键听写 → 粘贴 → 词表）完整实测一遍；
- 或 Windows 版脱离 beta / 安装包完成代码签名；
- 升级后角度固定为「中文听写实战包」：Qwen3-ASR / SenseVoice / Whisper 三模型中文与中英混读对比实测、Vocabulary 词表调教、模型下载镜像避坑，定价统一 9.9 元。

## 五、证据来源
- 仓库与 README/文档（当前 dev 分支）：https://github.com/anomalyco/hex ；Linux 文档 https://github.com/anomalyco/hex/blob/3a7a712315d83efc2aec8d8c7570cd53a3a86c28/docs/linux.md ；Windows 文档 https://github.com/anomalyco/hex/blob/dev/docs/windows.md ；Releases https://github.com/anomalyco/hex/releases/
- 模型目录与中文基准：仓库源码 `src/transcription/model_benchmarks.rs`（FLEURS zh CER 7.57）、`src/transcription/artifacts.rs`（qwen3_asr 模型条目）；第三方模型表整理 https://flaviocopes.com/hex/
- 第三方 Linux 引擎实测（含中文样本）: https://www.aisidaily.com/posts/hex-voice-dictation/ ；产品概览 https://www.alterhunt.com/tools/hex-dictation
- 本沙箱实测样本来源：FunASR 官方中文示例与 sherpa-onnx SenseVoice 中文测试音频（https://k2-fsa.github.io/sherpa/onnx/sense-voice/python-api.html ）
- 相关 Issues 旁证：#944 繁体中文界面（已完成）、#129 Qwen3-ASR 识别提示、#177 CJK 文本替换的词边界 bug（已关闭）——说明中文用户群真实存在且官方在持续修 CJK 问题

*验证全程只读：未做任何 GitHub 写操作，未发布、未上架。实测产生的临时文件与模型已清理。*
