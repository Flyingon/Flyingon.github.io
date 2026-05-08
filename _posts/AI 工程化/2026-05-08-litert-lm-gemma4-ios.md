---
layout: post
title: LiteRT-LM 在 iOS 上跑 Gemma 4：一次从 GPU 到回滚的工程复盘
category: AI 工程化
tags: [LiteRT-LM, Gemma 4, iOS, GPU, 本地大模型]
keywords: litert-lm, gemma 4, ios gpu, metal, stablehlo, local llm
---

这篇记录一次很具体的工程排查：在 iOS App 里集成 LiteRT-LM，跑 `Gemma-4-E2B-it.litertlm` 做本地 Vision LLM。目标不是证明某条路线一定正确，而是把当天踩到的坑、判断依据和最后保留下来的实现写清楚。

结论先放前面：

- 当前公开 LiteRT-LM iOS 路线下，**Gemma 4 可以作为本地模型继续保留**。
- 稳定配置是 **Main executor 走 GPU，Vision executor 走 CPU**。
- 不要把 Vision executor 强行切到 GPU；当前会在 `STABLEHLO_COMPOSITE` prepare、Metal texture 分配或内存压力处失败。
- 不切 Gemma-3n。Gemma-3n 的 Vision 子图要求 GPU，但公开 runtime 路径下 `vision=gpu` 创建 engine 失败，没有可用 CPU fallback。
- 最终代码回到单模型 Gemma 4 路线，并保留下载、多源 fallback、运行时动态加载、耗时日志和降内存参数。

---

## 一、背景：为什么选 LiteRT-LM 和 Gemma 4

AppScanFlow 原本已经有一条稳定的识别链路：

```text
Vision OCR
  -> 规则分类
  -> 字段提取
  -> 摘要
  -> 保存结果
```

这条链路的优点是稳定、轻、快；缺点也明显：遇到非标准票据、跨语言字段、版式复杂图片时，需要大量规则兜底。

所以本地 LLM 的目标不是替代所有逻辑，而是增加两种可选能力：

| 模式 | 作用 |
| --- | --- |
| OCR + Text LLM | 先 OCR，再让本地 LLM 做分类、字段复核、摘要 |
| Vision LLM | 直接把图片交给本地多模态模型，返回结构化 JSON |

模型侧选择 `Gemma-4-E2B-it.litertlm` 的原因很现实：

- `.litertlm` 模型约 2.59GB，移动端还能接受。
- GGUF 视觉路线通常还需要主模型 + `mmproj`，体积更大，当前不适合作为默认 App 内模型。
- LiteRT-LM 是 Google 官方边缘端 LLM runtime，理论上和 Gemma 4 edge variants 配套。
- Google AI Edge Gallery 宣传里已经把 Gemma 4 E2B/E4B 放到了移动端场景里。

但是“Gallery 能跑”不等于“第三方 iOS App 用公开 dylib/C API 路线也能同样跑”。这正是这次排查的核心。

---

## 二、最初现象：GPU 不像想象中那么简单

真机环境：

```text
Device: iPhone14,3
iOS: 26.3.1
Xcode: 26.4
Model: gemma-4-E2B-it.litertlm
Runtime: LiteRT-LM v0.11.x C API bridge
```

一开始最自然的想法是：既然要提速，那就把 GPU 开起来。

LiteRT-LM 的 engine settings 里有两个 backend 概念：

```text
main backend
vision backend
```

也就是说，Gemma 4 多模态模型不是一个整体 backend 开关，而是至少拆成两段：

- Main executor：文本主模型、prefill/decode。
- Vision executor：图像 encoder / adapter / patch 处理。

试下来以后，真正稳定的是：

```text
main=gpu
vision=cpu
```

而不是：

```text
main=gpu
vision=gpu
```

Vision GPU 路线的典型失败日志是：

```text
Node number 905 (STABLEHLO_COMPOSITE) failed to prepare.
ERROR: Node number 905 (STABLEHLO_COMPOSITE) failed to prepare.
RunPrefillAsync status: INTERNAL:
  runtime/executor/vision_litert_compiled_model_executor.cc
  external/litert/litert/cc/litert_compiled_model.cc
```

后来在更高 token、patch 或 speculative decoding 配置下，还遇到 iOS 直接杀进程：

```text
The process has been terminated by the operating system because it is using too much memory.
Domain: IDEDebugSessionErrorDomain
Code: 11
Failure Reason: Debug session ended with code 9: Terminated due to memory issue.
```

这个时候就能把问题分成两类：

| 问题 | 本质 |
| --- | --- |
| `STABLEHLO_COMPOSITE failed to prepare` | 图编译 / delegate prepare 阶段失败，还没真正开始识别 |
| iOS memory issue / jetsam | 模型、vision patch、KV cache、Metal resource 累积超过系统可承受范围 |

---

## 三、STABLEHLO_COMPOSITE prepare 失败到底是什么

`STABLEHLO_COMPOSITE` 不是普通业务报错，也不是 JSON 解析问题。

它大致发生在这条链路：

```text
.litertlm 模型
  -> LiteRT-LM 拆出 vision encoder / adapter / main decoder
  -> LiteRT 创建 vision compiled model
  -> GPU delegate 尝试 prepare 某个 StableHLO composite 节点
  -> Metal backend 无法完成 lowering / kernel binding / tensor allocation
  -> engine 创建或 image prefill 失败
```

所以它的本质是：**模型里的 Vision 子图和当前 iOS LiteRT GPU delegate 能力不匹配**。

从 App 层能调的参数其实很有限：

- 降低输入图像尺寸。
- 降低 vision patch 数。
- 降低输出 token 数。
- 关闭 speculative decoding / MTP。
- 避免先尝试失败的 GPU Vision 路线，防止失败后内存资源更糟。

但如果某个 composite op 没有被当前 Metal delegate 支持，或者官方 Gallery 用了我们公开包里没有的 kernel / accelerator / 静态链接路径，那 App 层没有办法真正修复。

底层要修，大概只有三条路：

1. **补 LiteRT Metal backend**
   - dump 出失败节点对应的 composite op。
   - 在 LiteRT GPU delegate 里补 lowering 或 kernel。
   - 重新构建 iOS runtime。

2. **换成和 Gallery 一致的 runtime artifact**
   - Gallery 可能静态链接了 Metal accelerator。
   - 也可能用了不同版本的 runtime、模型包、allowlist 或编译缓存。

3. **重新转换模型**
   - 让 Vision 子图避开当前 public runtime 不支持的 composite。
   - 这条最重，而且需要官方转换链路足够透明。

---

## 四、为什么试过 Gemma-3n，最后又放弃

中间尝试过一个方案 B：保留 Gemma 4 的现状，同时新增 Gemma-3n，看看它是不是更接近 iOS Gallery 的公开 allowlist。

这个方向很快也被证伪了。

Gemma-3n 的 Vision 子图明确要求 GPU：

```text
INVALID_ARGUMENT: Vision backend constraint mismatch.
Model requires one of [gpu] but Vision backend is CPU
```

也就是说它不像 Gemma 4 那样可以走：

```text
main=gpu
vision=cpu
```

对 Gemma-3n 来说，`vision=cpu` 不是慢一点，而是模型 metadata 直接拒绝。

可是切到：

```text
main=gpu
vision=gpu
```

又会失败：

```text
Node number 32 (STABLEHLO_COMPOSITE) failed to prepare.
Failed to create engine:
  runtime/executor/vision_litert_compiled_model_executor.cc
  external/litert/litert/cc/litert_compiled_model.h
```

这说明 Gemma-3n 在当前公开 iOS runtime 路径下没有有效 fallback：Vision CPU 不允许，Vision GPU 又 prepare 失败。

所以最后回滚这个分支是正确选择。工程上最怕“为了验证官方说法，把主线拖成一团泥”。Gemma-3n 这条路可以记为证据，但不应该进入产品实现。

---

## 五、最终保留的实现

最后保留下来的实现很克制：**只支持 Gemma-4-E2B-it，且只让 Main executor 使用 GPU**。

### 5.1 模型管理

当前模型目录只保留一个模型：

```swift
static let gemma4E2B = LocalAIModel(
    id: "gemma-4-E2B-it",
    displayName: "Gemma-4-E2B-it",
    fileName: "gemma-4-E2B-it.litertlm",
    downloadURL: URL(string: "https://huggingface.co/litert-community/gemma-4-E2B-it-litert-lm/resolve/main/gemma-4-E2B-it.litertlm")!,
    mirrorDownloadURLs: [
        URL(string: "https://hf-mirror.com/litert-community/gemma-4-E2B-it-litert-lm/resolve/main/gemma-4-E2B-it.litertlm")!
    ],
    expectedSizeBytes: 2_588_147_712
)
```

下载保留多源 fallback：

```text
Hugging Face official repo
  -> hf-mirror
```

并保留基本完整性校验，避免下载到半截文件后直接进入运行时。

### 5.2 Runtime bridge

Swift 侧不依赖尚未稳定的 Swift SDK，而是走 C API bridge：

```text
Swift
  -> LiteRTGemmaRuntime
  -> LiteRTLMRuntimeBridge
  -> dlopen / dlsym
  -> libLiteRtLm.dylib
  -> libLiteRt.dylib / Metal accelerator / TopK sampler
```

动态加载时会先加载 Gemma constraint provider：

```swift
libGemmaModelConstraintProvider.dylib
libLiteRtLm.dylib
```

其他 accelerator / sampler dylib 交给 LiteRT-LM 自己按名称加载，不在 Swift 侧手动乱 preload，避免 Objective-C duplicate class warning 和插件注册顺序问题。

### 5.3 Backend 策略

最终策略非常明确：

```swift
private func selectedBackendPlan() -> BackendPlan {
    let key = "settings.models.mainGPUAcceleration"
    let isMainGPUEnabled = UserDefaults.standard.object(forKey: key) as? Bool ?? true
    return BackendPlan(main: isMainGPUEnabled ? "gpu" : "cpu", vision: "cpu")
}
```

也就是：

| 设置 | main | vision |
| --- | --- | --- |
| Main GPU 开 | GPU | CPU |
| Main GPU 关 | CPU | CPU |

不再尝试 Vision GPU。

### 5.4 内存控制

Vision LLM 入口先压图：

```swift
private static let maxVisionInputDimension: CGFloat = 640
private static let maxVisionPatches = 768
private static let maxVisionOutputTokens = 1536
```

生成时写临时 JPEG，quality 为 `0.88`。这样能控制 vision patch 数和输入 tensor 压力。

Engine settings 里也做了限制：

```swift
api.settingsSetMaxTokens(settings, isVision ? 3072 : 4096)
api.settingsSetMaxImages?(settings, isVision ? 1 : 0)
api.settingsSetSpeculativeDecoding?(settings, false)
```

`speculative decoding` / MTP 理论上能提速，但当天真机验证发现它会扩大内存压力。当前先关掉，等上游 runtime 更稳定后再单独测。

### 5.5 耗时日志

为了判断 GPU 到底有没有收益，最终实现里加了关键阶段耗时：

```text
[LiteRT-LM][time] engine settings=...
[LiteRT-LM][time] engine create=...
[LiteRT-LM][time] text generateContent elapsed=...
[LiteRT-LM][time] vision temp image write elapsed=...
[LiteRT-LM][time] vision sendMessage elapsed=...
[LiteRT-LM][time] vision total elapsed=...
```

这个比“感觉快不快”可靠很多。尤其在移动端，首轮 engine 创建、模型 mmap、Metal shader 初始化、prefill、decode 是完全不同的性能阶段，混在一起看很容易误判。

---

## 六、返回 JSON 不完整的问题

有一次 CPU 路径返回了半截 JSON：

```json
{
  "documentType": "invoice",
  "detectedCategory": "financial_transaction",
  "categoryConfidence": 0.95,
  "title": "Transaction Record",
  "language": "ko",
  "fields": [
    {
      "name": "Merchant",
      "value": "신용카드",
      "confidence": 0.90
    },
    {
      "name": "Buyer",
      "va
```

这不是“模型完全识别错了”，更像是输出被截断。常见原因有：

- `maxOutputTokens` 太小。
- prompt 要求 JSON 太长。
- 模型在低速 CPU 路径上被提前取消。
- session / conversation API 返回内容没有完整拼接。

因此错误信息里保留 raw response prefix 是必要的。它能区分：

| 现象 | 判断 |
| --- | --- |
| 完全空 | runtime / conversation 失败 |
| 半截 JSON | token 限制、截断或取消 |
| 非 JSON 文本 | prompt / sampling / 模型服从性问题 |
| 字段错但 JSON 完整 | 识别质量问题 |

---

## 七、这次搜索到值得关注的官方 issue

后续如果继续跟 LiteRT-LM iOS / Gemma 4 GPU 路线，最值得关注这几个：

### 7.1 Gallery #692

[google-ai-edge/gallery #692](https://github.com/google-ai-edge/gallery/issues/692)

这个 issue 和当前问题最贴近：Gemma 4 LiteRT-LM 在 iOS 第三方 App 里初始化失败，而且 public iOS allowlist 没列 Gemma 4。

它问的其实就是我们最关心的问题：

> Gemma 4 `.litertlm` 在 iOS 第三方 App 里到底是不是官方支持？如果支持，公开 runtime/API 路径是什么？

### 7.2 LiteRT #6745

[google-ai-edge/LiteRT #6745](https://github.com/google-ai-edge/LiteRT/issues/6745)

这个 issue 提到 iOS arm64 `libLiteRtMetalAccelerator.dylib` 在 prebuilt 包里的问题，还指出 Gallery iOS 可能把 Metal accelerator 静态链接进了 binary。

这和我们的判断高度一致：**Gallery 能跑，不代表公开 dylib 路线完整等价**。

### 7.3 LiteRT-LM #2151

[google-ai-edge/LiteRT-LM #2151](https://github.com/google-ai-edge/LiteRT-LM/issues/2151)

这个关注 iOS Mach-O bundling、companion dylibs、`dlopen` by basename 和 `.framework` 打包问题。只要官方后续修 public iOS dylib 路线，这类 issue 很可能会出现相关信号。

### 7.4 LiteRT-LM #2154

[google-ai-edge/LiteRT-LM #2154](https://github.com/google-ai-edge/LiteRT-LM/issues/2154)

这个要求官方提供 public shared-library build target。当前 C API + dylib wrapper 路线，本质上也卡在这里。

### 7.5 LiteRT-LM #2187

[google-ai-edge/LiteRT-LM #2187](https://github.com/google-ai-edge/LiteRT-LM/issues/2187)

这是 Gemma-4-E2B vision 部分性能慢的问题，不是 iOS，但可以观察官方怎么解释 vision pipeline 性能、backend 支持和调优方向。

---

## 八、工程上的判断

这一天最大的收获不是“GPU 能不能开”，而是几个边界判断。

### 8.1 不要把模型支持理解成 App 集成支持

官方说 Gemma 4 支持移动端，可能指的是：

- Gallery App 支持。
- Android Kotlin 路线支持。
- 特定 prebuilt / static linked runtime 支持。
- 特定设备和系统版本支持。

但我们真正需要的是：

```text
第三方 iOS App
  + public LiteRT-LM C API
  + public iOS arm64 dylibs
  + App Store 可打包方式
  + Gemma-4-E2B-it.litertlm
  + Vision LLM
```

这是一条更窄、更具体的路线。只要其中一个组件不完整，就会出现“官方 demo 很快，我们这里很慢或跑不起来”的落差。

### 8.2 GPU 不一定整体更快

移动端 LLM 不是简单的“开 GPU = 快”。

GPU 可能加速 decode，但也可能增加：

- engine 初始化成本
- shader 编译成本
- tensor upload 成本
- Metal resource 压力
- Vision encoder prepare 风险
- 内存峰值

所以必须分阶段打点：

```text
image prepare
engine create
prefill
first token
decode
total
```

只看总耗时，很难判断到底哪一段在拖后腿。

### 8.3 fallback 要尊重模型 metadata

Gemma 4 可以 `vision=cpu`，Gemma-3n 不行。后者 metadata 明确要求 Vision backend 是 GPU。

因此 fallback 不是 App 想怎么 fallback 就怎么 fallback，而要看模型包里的 backend constraint。

### 8.4 先保主线，再保探索

Gemma-3n 双模型方案作为探索是有价值的，但不应该为了探索把主线复杂化。

最终回到：

```text
Gemma 4 单模型
main GPU 可开关
vision 固定 CPU
Vision LLM 限制输入和输出
保留耗时日志
保留 OCR 默认兜底
```

这是当前最稳的工程点。

---

## 九、后续策略

短期不再继续硬怼 `vision=gpu`。继续做三件事：

1. **保留 Gemma 4 当前实现**
   - 单模型。
   - Main GPU 可开关。
   - Vision CPU。
   - 降低 patch / token / speculative decoding 内存风险。

2. **关注官方 issue**
   - 特别是 Gallery #692 和 LiteRT #6745。
   - 等官方明确 Gemma 4 iOS public runtime 支持路径。

3. **如果要继续底层排查，先准备 upstream issue**
   - 设备型号、iOS、Xcode、LiteRT-LM tag。
   - 模型 URL 和 size。
   - backend 配置。
   - 完整 `STABLEHLO_COMPOSITE failed to prepare` 日志。
   - 说明 Gallery 能跑但 public dylib route 不等价。

在官方给出更清晰的 iOS runtime 之前，当前实现已经足够务实：能用、能测、能回退，也不会把 App 绑死在一个不稳定的 GPU Vision 假设上。

