# 干净环境验证清单

在本机之外的全新 Windows 上验证 `video-en2zh-dub`。本机环境已被 `SSLKEYLOGFILE` 等历史问题污染，
不具代表性，发布前必须过这一关。

---

## 一、虚拟机配置（VirtualBox）

| 项目 | 建议值 | 说明 |
| --- | --- | --- |
| 类型 / 版本 | Microsoft Windows / Windows 10 64-bit | 见下方「Win10 还是 Win11」 |
| 内存 | 8192 MB | 宿主机 32GB，分 8GB 不影响日常使用 |
| CPU | 4 核 | **别只给 2 核**，Whisper 是 CPU 多线程，核心数直接决定转写耗时 |
| 硬盘 | VDI 动态分配 60GB | 实际占用约 25–30GB，动态分配不一次吃满 |
| 网络 | NAT | 默认即可，edge-tts 与模型下载需要外网 |
| 显存 | 128MB | 不影响流水线，纯 CPU 干活 |

**Win10 还是 Win11**：Win10 已于 2025-10-14 停止支持，但作为一次性测试环境完全够用、也最省事。
若想长期保留这台 VM，建议 Win11——但 VirtualBox 装 Win11 需自行启用虚拟 TPM 与安全启动，折腾程度高出一截。

**性能提示**：若发现 VM 异常卡顿，检查宿主机是否开启了 Hyper-V 或 WSL2（会用掉虚拟化层，
导致 VirtualBox 走兼容后端而掉速）。当前宿主机未强制开启，一般没问题。

---

## 二、第一层：脚本层验证（不需要 WorkBuddy）

目标：确认三个脚本在全新环境能装、能跑、产物正确。

### 1. 装基础环境

- [ ] Python 3.12（官方安装包，**务必勾选 "Add Python to PATH"**）
- [ ] ffmpeg：装好后 `ffmpeg -version` 应能执行
- [ ] 重启一次终端，确保 PATH 生效

### 2. 装依赖（最慢的一步）

```bash
python -m pip install torch --index-url https://download.pytorch.org/whl/cpu
python -m pip install -r requirements.txt
```

**先装 CPU 版 torch**：直接装 whisper 时可能拉到数 GB 的 CUDA 包，而本项目纯 CPU 就够。
即便如此仍要 5–15 分钟，属正常现象，不要中断。

记录实际耗时：______ 分钟

### 3. 跑自检

```bash
python scripts\check_env.py
```

**预期：四项依赖 + audioop + ffmpeg + ffprobe 全 OK，SSLKEYLOGFILE 未设置，退出码 0。**

这一步是重点：干净机器上不应出现 `BROKEN` 或 `MISS`。若出现，说明发布版仍有环境假设没去干净，必须修。

**已知的高频缺失项**（上一轮干净环境实测踩到，现已写进 requirements，但值得确认）：

- [ ] Python 3.13+ 需 `audioop-lts` —— 标准库已移除 audioop，pydub 依赖它
- [ ] Windows 缺 VC++ 运行库 → torch/whisper 导入崩溃 → `pip install msvc-runtime`
- [ ] `ffprobe.exe` 是否随 ffmpeg 一起装了 —— 有些包只带 ffmpeg.exe，pydub 解码 mp3 会失败
- [ ] torch 是否 CPU 版（自检会显示 `+cpu` 或提示含 CUDA）
- [ ] **是否出现「ffmpeg 仅在当前目录」的 WARN** —— 这种状态当下能用但换目录即失效，应加入 PATH
- [ ] Windows 无管理员权限时，`msvc-runtime`（pip 包）是否成功替代 vc_redist 安装程序

### 4. 跑转写

```bash
python scripts\transcribe.py -i input.mp4 -o input.srt
```

- [ ] 首次运行会下载 `base` 模型（约 140MB），能正常完成
- [ ] 输出 `完成：写出 N 条字幕`
- [ ] `input.srt` 条数与本机实测（166 条）接近，末条结束时间约 9:29

记录耗时：______ 秒（本机 6 核为 57 秒，VM 4 核预计 2–4 分钟）

### 5. 准备中文字幕

第一层不装 WorkBuddy，翻译环节二选一：

- 从宿主机复制一份已翻译好的 `output.srt` 进来
- 或手工写一个 5–10 条的小 `output.srt` 用于跑通链路

### 6. 跑配音

```bash
python scripts\synthesize.py -i output.srt -o output.mp3 --report
```

- [ ] 逐条合成，片段数应等于字幕条数
- [ ] `--report` 能输出每条的「语音时长 / 字幕槽」占比
- [ ] 时长 ≈ 末条字幕结束 + 0.5 秒

小样本（10 条以内）测试时可用 `--limit 10` 先跑通，再跑全量。

记录：全量 166 条耗时 ______ 分钟（本机改 sleep 1.5 后预计 6–8 分钟，VM 略慢）

### 7. 跑合成

```bash
ffmpeg -y -i input.mp4 -i output.mp3 -c:v copy -c:a aac -map 0:v:0 -map 1:a:0 output.mp4
```

- [ ] 不卡住（验证 `-y` 的有效性）
- [ ] 产物时长与原视频接近，差值应在 1 秒内

顺手验证双轨（这条在发布版里只在本机测过）：

```bash
ffmpeg -y -i input.mp4 -i output.mp3 -map 0:v:0 -map 0:a:0 -map 1:a:0 -c:v copy -c:a aac -b:a 128k ^
  -metadata:s:a:0 language=eng -metadata:s:a:0 title="English" ^
  -metadata:s:a:1 language=zho -metadata:s:a:1 title="中文配音" ^
  -disposition:a:0 0 -disposition:a:1 default output_dual.mp4
ffprobe -v error -show_entries stream=index,codec_type:stream_tags=language -of csv=p=0 output_dual.mp4
```

预期三条流，语言标记 eng / zho。

再确认默认轨已交给中文（audio 两行应分别为 `default=0` 与 `default=1`）：

```bash
ffprobe -v error -show_entries stream=index,codec_type:stream_disposition=default -of csv=p=0 output_dual.mp4
```

---

## 三、第二层：agent 层验证（需要 WorkBuddy）

第一层全绿后再做。目标：确认 SKILL.md 的指令能被 agent 正确执行。

1. 在 VM 中安装并登录 WorkBuddy
2. 把 `video-en2zh-dub/` 放到 skill 目录，工作目录放 `input.mp4`
3. **只给一句需求**，例如「把这个视频翻译成中文配音」，全程不干预
4. 观察重点：
   - [ ] agent 是否主动执行了第 0 步环境自检
   - [ ] 能否自己找到正确的 Python 解释器（不靠人提示）
   - [ ] 翻译环节是否遵守了「4.5 字/秒控字数」的约束
   - [ ] 是否执行了三项字幕自检
   - [ ] 遇到 `--report` 的溢出提示时，是否会先看**中位占比**再决定对策
         —— 中位数接近 1.0 时盲目全局加速是帮倒忙，应识别为短碎片问题
   - [ ] 是否把步骤 1、3 放到了后台执行
   - [ ] 产物是否用 present_files 交付

**任何一项需要人工纠正才做对，都说明 SKILL.md 写得不够明确，要回头改文档而不是怪 agent。**

---

## 四、传文件的注意事项

- 用 VirtualBox 共享文件夹传 `input.mp4` 与 skill 目录
- **不要把工作目录设在共享文件夹里**：Windows 下它表现为 `\\vboxsvr\...` 的 UNC 路径，
  Python 与 ffmpeg 处理 UNC 路径时有各种意外。**先复制到 VM 本地磁盘再跑**。
- 建议工作目录用 `C:\work` 这类**无空格**路径。本机工作目录带空格是个历史巧合，
  趁这次也验证一下无空格路径下同样正常。

---

## 五、验证完要回填的数据

跑完之后，把这几项回填进发布文档，它们是最有说服力的材料：

| 项目 | 值 |
| --- | --- |
| `pip install` 耗时 | ______ |
| 转写耗时（9.5 分钟视频） | ______ |
| 配音耗时（166 条） | ______ |
| 合成耗时 | ______ |
| 溢出中位占比 | ______ |
| 成片音视频时长差 | ______ |
| 干净环境 check_env 结果 | 全 OK / 有 BROKEN |
