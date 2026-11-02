# DSH 语音插件源码级对比：dsh-voice-mode vs dsh-talk

> 比对对象：
> - 主选候选 `qishuilalala/dsh-voice-mode`（下称 **VM**）
> - 备选候选 `PerryLink/dsh-talk`（下称 **TALK**）
>
> 方法：`git clone --depth 1` 到 `fishy-audit/repos/dsh-voice-mode`、`fishy-audit/repos/dsh-talk`，**逐个读源码**（不是只读 README）。
> 本地环境（只读，未改动）：DSH `0.2.1-rc.2`（`@deepseek-ai/dsh-desktop-runtime`）、Node `v24.19.0`（`desktop-runtime.json` 内声明 `nodeVersion: 24.21.0`）。
>
> 标注约定：
> - **〔已验证·源码〕** = 我读到了实现/类型/常量，并给出 文件:行号。
> - **〔已验证·命令〕** = git / npm registry / 文件系统实测。
> - **〔未验证·README〕** = 只有文档声称，源码未复核。
> - **〔NOT_TESTED〕** = 需要真机/真跑才能确认，本轮**没有**安装、没有启动插件、没有发任何消息。所有运行时行为一律属于此类，除非源码可直接判定。

---

## 0. 结论先行

| | 结论 |
| --- | --- |
| **主选** | **`dsh-voice-mode` 0.7.34**（本地流式 ASR + 服务端 VAD 打断 + 逐句流式 TTS + 字幕 + 完整会话稳定性设计） |
| **备选** | **`dsh-talk` 0.3.26**（架构干净、体积小、但**没有真正的语音打断**、没有字幕、没有逐句 TTS、自动朗读要靠模型主动调 `speak`） |
| 一句话理由 | VM 是"语音模式"产品；TALK 是"语音输入按钮 + 一个 speak 工具"。任务要求的 11 个维度里，VM 有 9 个维度有真实实现，TALK 只有 5 个且其中 2 个是脆的（Web Speech 依赖 + 打断不真停播）。 |
| Android 移植要点 | VM 的可移植性远高于 TALK：**它完全不碰 Web Speech API**，只依赖 `getUserMedia` + `AudioWorklet` + `POST` PCM 三个 WebView 里都有的东西；TALK 的默认 `stt.engine=auto` 在 Android WebView 上会**必然走进死路**（见 §8.2，源码可证）。 |

---

## 1. 版本 / 提交 / 兼容声明

| 项 | dsh-voice-mode | dsh-talk |
| --- | --- | --- |
| package.json `version` | **0.7.34** | **0.3.26** |
| npm latest（registry 实测〔已验证·命令〕） | 0.7.34，发布 `2026-10-28T09:05:56Z` | 0.3.26，发布 `2026-10-24T14:52:58Z` |
| 最新 commit（`git log -1`〔已验证·命令〕） | `3524db4`，**2026-10-31 03:33:42 +0800**，*fix(market): 根 manifest 补 exports 入口映射* | `6a5f768`，**2026-11-01 17:23:47 +0800**，*docs: add the legacy documentation baseline* |
| 仓库形态 | 根目录 = npm 包外壳；**真实插件在 `plugin/dsh-voice-mode/`**（根 `main` 指向 `./plugin/dsh-voice-mode/lib/index.js`） | 根目录即插件包（`src/` + `scripts/prepare.mjs` 构建出 `lib/`） |
| `engines.dsh` | **`>=0.1.1-rc.2 <0.3.0-0`** | **`>=0.2.1-0 <0.3.0`** |
| `engines.node` | `>=18.0.0` | `^22.19.0 \|\| >=24.0.0` |
| `dependencies` | `sherpa-onnx ^1.13.6`、`sherpa-onnx-node 1.13.6`、`@deepseek-ai/schemastery ^3.18.5` | **`zod 4.4.3`（仅此一个）** |
| `peerDependencies` | `@deepseek-ai/cordis ^4.0.1`、`react ^18.2.0` | `@deepseek-ai/cordis ^4.0.5` + 9 个 `@deepseek-ai/dsh-*`（全部 `optional: true`，见下） |
| 与本机 0.2.1-rc.2 是否在声明范围内 | ✅ 在范围内；且 `scripts/verify-dual.sh` 的已验证矩阵**显式包含 `0.2.1-rc.2`**（共 21 个版本） | ✅ 在范围内；但 `dshWorkshop.compatibility.dshVersions` 只列到 `0.2.1-alpha.1`，`0.2.1-rc.2` 属"范围内未声明实测" |
| 文件体积（`git` 工作树，不含 .git）〔已验证·命令〕 | 304 文件 / **13.95 MB**（含 12MB 截图与文档） | 88 文件 / **0.75 MB** |
| 发布包 unpacked（npm 实测〔已验证·命令〕） | **1,412,065 B ≈ 1.35 MB** | **1,236,668 B ≈ 1.18 MB** |

TALK 的 peer 声明（重点：**全 optional**，所以缺包不报错，只是功能降级）——`dsh-talk/package.json`：

```json
"peerDependenciesMeta": {
  "@deepseek-ai/dsh-session": { "optional": true },
  "@deepseek-ai/dsh-api-remotes": { "optional": true },
  "@deepseek-ai/dsh-client-connection": { "optional": true },
  "@deepseek-ai/dsh-client-locale": { "optional": true },
  "@deepseek-ai/dsh-client-ui-settings": { "optional": true },
  "@deepseek-ai/dsh-client-ui-slots": { "optional": true },
  "@deepseek-ai/dsh-client-ui-conversation": { "optional": true }
}
```

> ⚠️ 实测：本机 DSH 0.2.1-rc.2 的 `app.asar/dsh/node_modules/@deepseek-ai/` 里**没有 `dsh-client-ui-slots` 这个包**〔已验证·命令〕。TALK 的 `src/client/index.ts:67-70` 已就地声明 `interface SlotsFace` 兜住这一点（自述注释也承认"registry 的 owning package 跨 host line 不同"），所以不是 bug，但说明它是在多个宿主线之间猜着写的。

---

## 2. 中文语音识别（引擎 / 模型 / 是否要 API Key）

### 2.1 dsh-voice-mode：本地 sherpa-onnx 流式 zipformer2 + SenseVoice 定稿，**零 API Key**

**引擎：`sherpa-onnx`（Node 侧 WASM/原生 addon），模型随首次使用从 HuggingFace 懒下载。**

`plugin/dsh-voice-mode/src/asr-host.ts:52-72`（原样）：

```ts
/** 模型仓库与文件清单（SHA256 固定，供应链校验）。 */
export const MODEL_REPO = 'csukuangfj/sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30'
const MODEL_FILES: ModelFileSpec[] = [
  { file: 'encoder.int8.onnx', sha256: '5ac51e27981bb4dab01bb9be4958453ba50c3b61c063ddda0eab23fd3671aa4f' },
  { file: 'decoder.onnx', sha256: '06522ad63cec0fdf6809f4e1db9bb4f7d710c34582e3b35db62c60eccafac7e' },
  { file: 'joiner.int8.onnx', sha256: 'b34584dc6f561089e1d747fedebb3765f2caa72c927ef54d7ca55e5ae40a814b' },
  { file: 'tokens.txt', sha256: '6193c7ea1c96d0d9a1e9652789b40d13a8a913b434a5451e93158f5a09fd6652' },
]

/** P2-1 Silero VAD 模型：官方 sherpa 文档下载源（csukuangfj/vad，~2MB）。 */
export const VAD_REPO = 'csukuangfj/vad'
const VAD_FILES: ModelFileSpec[] = [
  { file: 'silero_vad.onnx', sha256: 'a35ebf52fd3ce5f1469b2a36158dba761bc47b973ea3382b3186ca15b1f5af28' },
]

/** P4-1 SenseVoice 定稿模型（int8 ~228MB，带标点 + ITN；模型总体积 160+228=388MB ≤500MB 约束）。 */
export const SENSE_REPO = 'csukuangfj/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17'
const SENSE_FILES: ModelFileSpec[] = [
  { file: 'model.int8.onnx', sha256: 'c71f0ce00bec95b07744e116345e33d8cbbe08cef896382cf907bf4b51a2cd51' },
  { file: 'tokens.txt', sha256: 'f449eb28dc567533d7fa59be34e2abca8784f771850c78a47fb731a31429a1dc' },
]
```

**模型大小（源码注释 + README 双证）**：
- zipformer2 流式：`encoder.int8.onnx ≈ 153.7 MB` + decoder + joiner + tokens ≈ **160 MB**（`asr-host.ts:2-11`；README §模型与缓存）
- SenseVoice 定稿重译：**≈ 228 MB**，默认 **开**（`VoiceSettingsValue.senseVoice` 默认 `true`，`src/index.ts:282`）
- Silero VAD：**≈ 2 MB**
- **合计 ≈ 390 MB** 首次下载；缓存目录：Windows `%LOCALAPPDATA%\dsh-voice-mode\models`，类 Unix `~/.cache/dsh-voice-mode/models`（`src/index.ts:177-180`，原样）：

```ts
const defaultModelCacheDir = (): string =>
  process.platform === 'win32'
    ? join(process.env.LOCALAPPDATA ?? join(homedir(), 'AppData', 'Local'), 'dsh-voice-mode', 'models')
    : join(homedir(), '.cache', 'dsh-voice-mode', 'models')
```

**API Key：不需要**。识别 100% 在本机 host 进程内跑 WASM/原生；音频不上传任何第三方（README §权限与数据流声明〔未验证·README〕，但源码里 ASR 路径确实没有任何 outbound ASR 请求，只有模型下载走 HF/hf-mirror）。

**下载是"懒 + 校验 + 断点续传 + 镜像回退"**，`src/models.ts:16-22`：

```ts
export const HOST_PRIMARY = 'https://huggingface.co'
export const HOST_FALLBACK = 'https://hf-mirror.com'
export const ALLOWED_MODEL_HOSTNAMES = ['huggingface.co', 'hf.co', 'hf-mirror.com'] as const
```

### 2.2 dsh-talk：三选一（Web Speech / FunASR HTTP / whisper.cpp），**本地路径要额外二进制**

引擎枚举，`src/wire.ts:28-32`（原样）：

```ts
/** The engines the settings panel may select for speech-to-text. */
export const STT_ENGINES = ['auto', 'web', 'funasr', 'whisper'] as const

/** The engines the settings panel may select for text-to-speech. */
export const TTS_ENGINES = ['auto', 'browser', 'edge-tts', 'piper'] as const
```

解析规则，`src/service.ts:153-160`（原样）：

```ts
export function resolveSttEngine(resolved: ResolvedConfig): 'web' | 'funasr' | 'whisper' {
  if (resolved.sttEngine === 'funasr') return 'funasr'
  if (resolved.sttEngine === 'whisper') return 'whisper'
  if (resolved.sttEngine === 'web') return 'web'
  if (resolved.funasrUrl !== null) return 'funasr'
  if (resolved.whisperModelPath !== null) return 'whisper'
  return 'web'
}
```

- **web**：浏览器 `SpeechRecognition`/`webkitSpeechRecognition`（Chrome/Edge 的云端识别）。**不是本地模型**，模型大小无从谈，识别在浏览器厂商的云上跑。**不需要你自己配 API Key**（用的是浏览器内置通道）。
- **funasr**：HTTP 打到你自己起的 FunASR 服务（OpenAI 兼容 `/v1/audio/transcriptions`）。**需要你另外部署一个 FunASR 服务**，模型大小取决于你部署什么。凭证取决于你的网关。
- **whisper**：`subprocess.spawn` 调 `whisper-cli`，**模型是你自己在 `stt.whisper.modelPath` 指的 ggml 文件**（仓库不带、也不下载）。

`src/engine.ts:103-115`（原样）：

```ts
export function whisperArgv(resolved: ResolvedConfig, wavPath: string, outBase: string, language: string): readonly string[] {
  const argv = [
    resolved.whisperCommand,
    '--model', resolved.whisperModelPath ?? '',
    '--file', wavPath,
    '--output-txt',
    '--output-file', outBase,
    '--no-prints',
  ]
  const tag = resolved.whisperLanguage ?? language
  if (tag !== 'auto' && tag !== '') argv.push('--language', tag)
  return argv
}
```

`src/engine.ts:248-274`（FunASR，原样，注意它是**整段一次性 POST**，没有流式）：

```ts
export async function transcribeFunasr(resolved: ResolvedConfig, wav: Uint8Array, signal: AbortSignal): Promise<string> {
  const base = resolved.funasrUrl ?? ''
  if (base === '') throw new EngineFailure('funasr: no endpoint configured', false)
  const url = base.endsWith('/v1/audio/transcriptions') ? base : `${base.replace(/\/+$/u, '')}/v1/audio/transcriptions`
  const form = new FormData()
  form.append('file', new Blob([wav as BlobPart], { type: 'audio/wav' }), 'speech.wav')
  if (resolved.funasrModel !== null) form.append('model', resolved.funasrModel)
  ...
  const text = (payload as Record<string, unknown>)['text']
  if (typeof text !== 'string') throw new EngineFailure('funasr response has no text field', false)
  return text.replace(/\s+/gu, ' ').trim()
}
```

> ⚠️ **源码级缺口**：`transcribe(wav)` 注释写"16 kHz mono WAV"，但客户端送上来的是 **MediaRecorder 的原始容器字节**（`audio/wav` / `webm;codecs=opus` / `audio/mp4`，见 `TalkMicButton.tsx:86`），**中间没有任何转码或重采样**（全仓 grep `resample|OfflineAudioContext|wav` 无转换代码，只有 engine 侧的路径拼接）。所以只要浏览器落到 webm/mp4，whisper-cli / FunASR 收到的是非 WAV 字节。**〔已验证·源码〕这是设计层面的缺陷，不是配置问题。**

### 2.3 小结（中文识别）

| | VM | TALK |
| --- | --- | --- |
| 中文引擎 | sherpa-onnx zipformer2-**zh**-int8（流式）+ SenseVoice-zh-en-ja-ko-yue（定稿） | Web Speech（浏览器云）/ 你自己的 FunASR / 你自己的 whisper.cpp |
| 模型体积 | ~160MB（+228MB 定稿 +2MB VAD），自动下载校验 | 仓库不带模型；whisper 要你自备 ggml（base ~142MB / small ~466MB 量级） |
| API Key | **不需要** | 不需要（web）/ 取决你的 FunASR 网关 / 不需要 |
| 中文准确度决定权 | 插件（模型固定、SHA 锁定） | 用户环境（换浏览器/换模型就变） |

---

## 3. 实时语音输入 / 流式识别

### 3.1 VM：真·流式（增量 PCM 上行 + 部分结果下行）

**上行协议**：客户端 AudioWorklet 每 **1024 样本 = 64ms** 出一帧 16kHz 单声道 f32，攒到 100ms 节拍 POST 一次增量（`asr.ts:184-189`: `PARTIAL_INTERVAL_MS = 100`、`PARTIAL_MIN_S = 0.4`、`PARTIAL_MAX_S = 30`、`BUFFER_SIZE = 1024`）。host 只吃"尚未喂过"的增量。

**host 接口签名**（`src/asr-host.ts:103-115`，原样）：

```ts
  feed(
    sessionId: string,
    samples: Float32Array,
    final: boolean,
    offset?: number,
    /** 客户端段身份（epoch = segmentEpoch 快照；旧世代请求被忽略/清理）。 */
    epoch?: number,
    /** 批 7N：manual 模式下用户是否按住 mic/Ctrl。auto 模式忽略。 */
    manualPressed?: boolean,
  ): Promise<{ text: string; loading?: boolean; endpoint?: boolean; isSpeech?: boolean }>
  /** 播放期打断检测通道（vadOnly）：AI 朗读中客户端常规 partial 断流，
   *  此方法只喂独立检测 VAD（不进 ASR 流、不碰端点 VAD），返回帧级 isSpeech。 */
  detect(sessionId: string, samples: Float32Array): Promise<{ isSpeech: boolean }>
```

**增量喂流 + 端点判定**（`asr-host.ts:550-613` 摘要，关键段落原样）：

```ts
    if (offset + samples.length > seg.fed) {
      const skip = Math.max(seg.fed - offset, 0)
      const inc = samples.subarray(skip)
      seg.stream.acceptWaveform(rec.config.featConfig.sampleRate, inc)
      seg.fed = offset + samples.length
      ...
      while (rec.isReady(seg.stream)) rec.decode(seg.stream)
      text = rec.getResult(seg.stream).text
      seg.lastText = text
      if (!final) {
        const vad = await ensureSessionVad(seg)
        if (vad) {
          ...
          vad.acceptWaveform(inc)
          isSpeech = vad.isDetected()
          if (!vad.isEmpty()) {
            let spokenMs = 0
            while (!vad.isEmpty()) {
              const sp = vad.front()
              spokenMs = (sp.samples.length / 16000) * 1000
              vad.pop()
            }
            const confirmMs = endpointConfirmMs(seg.lastText, spokenMs)
            ...
```

**HTTP 协议**（`asr-host.ts:984-1082`）：`POST /voice-mode/asr?sessionId=&final=&reset=&epoch=&offset=[&vadOnly=1][&manual=1]`，body 为裸 f32 LE PCM，上限 `MAX_ASR_BYTES = 4MB`（`asr-host.ts:165`）。返回 `{text, endpoint?, isSpeech?}` 或 `202 {loading:true}`（模型未就绪）。非活跃会话一律 403（`asr-host.ts:1030-1033`）。

**端点（断句）语义确认窗口**（`asr-host.ts:169-191`，这是个很细的设计，原样）：

```ts
const VAD_CONTINUE_RMS = 0.02
const CONFIRM_CONJUNCTION_MS = 800
const CONFIRM_LONG_SENTENCE_MS = 350
const CONFIRM_LONG_SENTENCE_S = 8
const CONFIRM_MIN_MS = 200
export const CONJUNCTION_TAIL = /(然后|还有|以及|并且|而且|此外|再说|接着|然后呢|比方说|比如说|比如|例如|等等|或者|或是|还有呢)$/

export function endpointConfirmMs(text: string, spokenMs: number): number {
  const tail = text.trimEnd()
  if (CONJUNCTION_TAIL.test(tail)) return CONFIRM_CONJUNCTION_MS
  if (spokenMs > CONFIRM_LONG_SENTENCE_S * 1000) return CONFIRM_LONG_SENTENCE_MS
  return CONFIRM_MIN_MS
}
```

即：话说完了还要再等 200–800ms 看用户是不是"然后……"接着讲。这是中文口语端点的实战细节，TALK 完全没有对应物。

**定稿**用 SenseVoice 整段重译（带标点+ITN），与 zipformer 流式定稿 race，超时 20s 降级（`asr-host.ts:626-663`），SenseVoice 解码跑在 `worker_threads`（`src/sense-worker.ts`，避免 228MB 同步 decode 卡死事件循环）。

### 3.2 TALK：**没有流式识别**（除浏览器 Web Speech 自带的 interim）

- Web Speech 路径设 `recognition.interimResults = true; recognition.continuous = true`（`TalkMicButton.tsx:256-257`），每次 `onresult` 把**整个 results 集合重建**后 `writeDraft`（`TalkMicButton.tsx:286-298`）：
  ```ts
      const buildTranscript = (event: { results: ArrayLike<ArrayLike<{ transcript: string }>> }): string => {
        const parts: string[] = []
        for (let index = 0; index < event.results.length; index += 1) {
          const text = event.results[index]?.[0]?.transcript ?? ''
          if (text.trim() !== '') parts.push(text.trim())
        }
        return parts.join(' ')
      }
      recognition.onresult = (event) => {
        lastFull = buildTranscript(event)
        armSilence()
        if (lastFull !== '') writeDraft(lastFull)
      }
  ```
  这是"浏览器给我的 interim"，不是插件自己的流式；且**只有 draft 更新，没有字幕/状态条**。
- FunASR / whisper 路径：`MediaRecorder` 整段录完 → `blob.arrayBuffer()` → base64 → `talk/transcribe` → **一次性**返回全文（`TalkMicButton.tsx:351-370`）。**零流式**。

---

## 4. 流式 TTS（是否按句朗读 / 用什么 TTS）

### 4.1 VM：**逐句流式**，三种引擎可热切

**分句器**：`src/segmenter.ts:52-67`（原样），中英日终止标点 + 软切点兜底：

```ts
export function splitSentences(chunk: string): { sentences: string[]; tail: string } {
  const sentences: string[] = []
  let start = 0
  // CJK 终止符 + ASCII 终止符；孤立的英文句点仅在空白/文末后算终止
  // （避免拆散 "3.14"、URL）。
  const re = /[。！？!?；;…\n]+|\.(?=\s|$)/g
  let m: RegExpExecArray | null = null
  let lastEnd = 0
  while ((m = re.exec(chunk)) !== null) {
    const end = m.index + m[0].length
    sentences.push(chunk.slice(start, end))
    start = end
    lastEnd = end
  }
  return { sentences, tail: chunk.slice(lastEnd) }
}
```

**调用链（谁调谁）**：

`ctx.on('llm/stream')` → `tapActiveStream()` → `SentenceSegmenter.feed(chunk.text)` → `queue.enqueue(sessionId, s)` → `TtsQueue.pump()` → `engine.synthesize(sentence)` → listener → `broadcast('audio', frame)` → SSE `/voice-mode/stream` → 客户端 `createAudioEngine().push(frame)` → Web Audio `decodeAudioData` 起播。

`src/index.ts:2142-2200`（原样，节选）：

```ts
async function* tapActiveStream(
  sessionId: string,
  inner: AsyncIterable<StreamChunk>,
  queue: TtsQueue,
  broadcast: (event: string, payload: unknown) => void,
  onTurn: (state: 'listening' | 'agent-speaking') => void,
  speak = true,
  muted: () => boolean = () => false,
): AsyncIterable<StreamChunk> {
  const segmenter = new SentenceSegmenter()
  ...
    for await (const chunk of inner) {
      // 只朗读最终答复的 text-delta（Q7）；reasoning/tool-call 不读。
      if (chunk.type === 'text-delta' && chunk.text) {
        ...
        for (const s of segmenter.feed(chunk.text)) {
          ...
          if (speak && !muted()) queue.enqueue(sessionId, s)
        }
      }
      ...
      yield chunk
    }
```

**SSE 帧结构**（`src/tts-queue.ts:73-87`，原样）：

```ts
export interface TtsChunkFrame {
  sessionId: string
  /** 句序号（句级不变；客户端按句拼帧）。 */
  sentenceId: number
  /** chunk 序号（句内递增；单 chunk 场景恒为 0）。 */
  chunkId: number
  /** true = 本句最后一个 chunk，此后客户端拼帧完成可起播。 */
  final: boolean
  /** 句子文本（剥离 markdown 后；仅 final 帧携带）。 */
  text?: string
  /** base64 音频分片。 */
  audio: string
  /** 音频 MIME（client 据此构造 Blob 类型：audio/wav 或 audio/mpeg）。 */
  mime?: string
}
```

**TTS 引擎三选一**（`VoiceSettingsValue.ttsEngine: 'edge' | 'vits' | 'kokoro'`，`src/index.ts:193`）：

| 引擎 | 实现 | 是否需要 key / 二进制 | 备注 |
| --- | --- | --- | --- |
| `edge`（**默认**） | 内联 `msedge-tts 2.1.1`（`lib/msedge-tts.cjs` 737KB）打微软 Edge「大声朗读」在线接口 | 不需 key；**需联网** | 微软随时可能改握手，仓库有每日探针 `.github/workflows/edge-tts-probe.yml`；CHANGELOG 0.7.31 记录 2026-10-26 就翻车过一次 |
| `vits` | sherpa-onnx VITS `csukuangfj/sherpa-onnx-vits-zh-ll`（纯中文 5 说话人，~130MB），跑在 `child_process.fork` 子进程 `lib/tts-vits-worker.cjs` | 本地，无 key | 全离线 |
| `kokoro` | `kokoro-int8-multi-lang-v1_1`（中英多语，int8 主模型 ~109MB），sherpa-onnx-node **原生 addon** | 本地，无 key | **有坑**：Electron 宿主下必须找到真 Node（见 §10） |

**为什么跑子进程**（`src/tts-local.ts:9-13`，原样）：

```
 * 合成跑在独立子进程（fork → lib/tts-vits-worker.cjs）——sherpa 的 generate()
 * 是同步 CPU 运算，主线程运行会阻塞事件循环、卡住文字流转发。
 * 主线程只做模型下载/校验与消息转发，文字照常流畅、声音跟着读。
```

### 4.2 TALK：**整段一次性合成，不分句**

`speak` 工具把**整条文本**（上限默认 20000 字符）交给引擎一次性合成：

`src/service.ts:213-248`（节选原样）：

```ts
  async speak(text: string, options: SpeakOptions): Promise<SpeakOutcome> {
    const spoken = sanitizeText(text, this.resolved.maxSpeakChars)
    if (spoken === '') throw new Error('dsh-talk: speak text must not be empty')

    if (options.interrupt !== false) this.interrupt()

    const engine = resolveTtsEngine(options.engine, this.resolved)
    const utteranceId = randomUUID()

    if (engine === 'browser') { ... }

    const localEngine = engine // edge-tts | piper
    const controller = new AbortController()
    this.activeSyntheses.set(utteranceId, controller)
    const signal = options.signal === undefined ? controller.signal : AbortSignal.any([controller.signal, options.signal])
    const startedAt = Date.now()
    try {
      const result = await synthesize(this.ctx.subprocess, this.resolved, localEngine, spoken, signal)
      ...
```

- 客户端把整段音频一次性 `new Audio(url)` 播（`TalkMicButton.tsx:453-462`），或 `speechSynthesis.speak(utterance)`（`TalkMicButton.tsx:441-447`）。**没有句子粒度、没有边合成边播**。
- 全仓 grep `caption|subtitle|字幕` → **0 命中**；`sentence|分句` 只有 `tool.ts:28` 一句英文注释——**没有分句器**〔已验证·源码〕。
- TTS 引擎：`browser`（speechSynthesis）/ `edge-tts`（**外部 Python CLI**，`--voice --rate --text --write-media out.mp3`）/ `piper`（**外部二进制** + `.onnx` 语音）。三个本地引擎全部依赖**用户自己装的东西**（`src/engine.ts:61-92`）。

**关键差异（对产品影响最大的一条）**：VM 自动朗读 AI 回复（`llm/stream` tap），TALK **不会自动朗读**——只有两种情况会响：① 模型主动调用 `speak` 工具；② 事件播报（回合结束/待审批/出错，固定话术 "Turn complete." 等）。所以 TALK 的"回复会被朗读"其实依赖模型自觉调工具〔已验证·源码：全仓无 `llm/stream` 或 `system-prompt` 钩子〕。

---

## 5. 本地语音引擎

| | VM | TALK |
| --- | --- | --- |
| ASR 本地 | ✅ sherpa-onnx（WASM + 可选原生 addon），模型自动下载校验 | ⚠️ 只能说"可以本地"：`whisper-cli` 二进制 + ggml 模型**要自己装自己放**；FunASR 要自己起服务 |
| VAD 本地 | ✅ Silero VAD（sherpa-onnx `createVad`，2MB，host 侧） | ❌ 只有 RMS 能量阈值（`src/client/vad.ts`，见 §6.2） |
| TTS 本地 | ✅ VITS（WASM 子进程）/ Kokoro（原生 addon 子进程） | ⚠️ piper 二进制 + onnx 语音，要自己装 |
| 模型分发 | 插件负责（懒下载 / SHA256 / 断点续传 / 镜像回退） | **不负责**（文档让你自己准备） |
| 落地程度 | 开箱即用（除首次 390MB 下载） | 开箱只有"浏览器云"这一条能跑 |

VM 的 VAD 工厂（`asr-host.ts:334-350`，原样）：

```ts
  /** Silero VAD 实例工厂（端点 VAD 与检测 VAD 共用，threshold 可调）。 */
  const newVad = (vadPath: string, threshold = 0.5, minSilenceDuration = 0.5): SherpaVad =>
    createVad({
      sileroVad: {
        model: vadPath,
        threshold,
        minSilenceDuration,
        minSpeechDuration: 0.25,
        maxSpeechDuration: 20,
        windowSize: 512,
      },
      sampleRate: 16000,
      numThreads: 1,
      provider: 'cpu',
      debug: 0,
      bufferSizeInSeconds: 30,
    })
```

---

## 6. 语音打断（用户开口 → 自动停止播报）

### 6.1 VM：**真·服务端 Silero VAD 帧级检测 + 三级确认 + 回声门控 + 完全静音收尾**

**完整判定链**（`docs/adr/0003-server-side-vad.md:17-25`，文档写的链路，与源码一致〔已验证·源码〕）：

```
采集帧（浏览器） → HTTP POST /asr?vadOnly=1（100ms 节拍）
  → host Silero VAD.isDetected()（threshold 0.35）
  → JSON 下行 isSpeech
  → 客户端连续 confirmFrames 次为真 → 回声门控 → hardBreak
```

**检测通道是独立 VAD 实例**（不与端点 VAD 共享状态），`asr-host.ts:361-376`（原样）：

```ts
  /** 播放期检测通道 VAD 池：按会话独立实例（跨段纪元存活）。只喂 vadOnly 音频，
   *  不与端点 VAD 共享状态——朗读期回声/人声若混入端点 VAD 会污染断句判定。 */
  const detectVads = new Map<string, SherpaVad>()
  /** 检测 VAD 最近使用时刻（90s 清扫依据；reset/dispose 之外的生命周期兜底）。 */
  const detectVadLastUse = new Map<string, number>()
  const ensureDetectVad = async (sessionId: string): Promise<SherpaVad | null> => {
    const existing = detectVads.get(sessionId)
    if (existing) return existing
    const vadPath = await ensureVadModel()
    if (!vadPath) return null
    // 检测 VAD 更灵敏（0.35 vs 端点 0.5）：打断要抓轻声/正常音量的人声前沿，
    // 0.5 会让 VAD 闪烁（真→假→真），confirmMs 飙到 1~2.7s。端点 VAD 仍 0.5（断句保守）。
    const vad = newVad(vadPath, 0.35)
    detectVads.set(sessionId, vad)
    return vad
  }
```

**detect 实现（fail-closed）**，`asr-host.ts:771-783`（原样）：

```ts
    detect: async (sessionId, samples) => {
      // 打断根治：检测通道 fail-closed——VAD 模型缺失返回 false（不打断，
      // 客户端计数随之清零，防残留累计误触发）。
      const vad = await ensureDetectVad(sessionId)
      if (!vad) return { isSpeech: false }
      detectVadLastUse.set(sessionId, Date.now())
      if (samples.length > 0) vad.acceptWaveform(samples)
      const speech = vad.isDetected()
      // 检测 VAD 只消费帧级状态：排空已完成段队列（sherpa 内部按段 malloc，
      // 不排空会随打断次数累积泄漏——对抗审查 Important#3）。
      while (!vad.isEmpty()) vad.pop()
      return { isSpeech: speech }
    },
```

**三档灵敏度 = 连续确认帧数**，`src/client.tsx:59-61`（原样）：

```ts
 * 打断灵敏度三档 → isSpeech 连续确认帧数（墙钟节拍 100ms/拍 + 上行往返 → 确认阶段约 0.3/0.2/0.1s；语义对齐旧能量持续时长档位）。 */
const INT_CONFIRM_FRAMES: Record<0 | 1 | 2, number> = { 0: 3, 1: 2, 2: 1 }
```

**客户端状态机 + 迟滞 + 回声门控 + hardBreak**，`src/client.tsx:2691-2758`（原样，这是本报告里最值得给 Android 端抄的一段）：

```ts
          onIsSpeech: (speech) => {
            fixtureRecorder.noteIsSpeech(speech)
            if (bargeInMode === 'manual') return
            // 打断检测只在 AI 朗读中有意义：非播放期清零计数。
            if (!bus.ui.playing) {
              isSpeechTrueCount = 0
              interruptFirstAt = 0
              echoRejectStreak = 0
              bus.setUi({ isSpeech: speech, echoDelayMs: bus.echoDelayMs(), echoLevels: engineRef.current?.echoLevels() })
              return
            }
            if (speech === true) {
              isSpeechFalseRun = 0
              isSpeechTrueCount++
              if (isSpeechTrueCount === 1) interruptFirstAt = Date.now()
              if (isSpeechTrueCount >= confirmFrames) {
                // A2.5 回声门控：残差未明显高于回声地板 → 判回声，不打断（防外放回声自打断）。
                if (engineRef.current && !engineRef.current.aboveEchoFloor(cfg.echoGateDb ?? 6)) {
                  ...
                  isSpeechTrueCount = 0
                  interruptFirstAt = 0
                  if (++echoRejectStreak === 3) bus.setNotice(tk('echoRejectHint'))
                  return
                }
                const confirmMs = interruptFirstAt > 0 ? Date.now() - interruptFirstAt : 0
                ...
                interruptFirstAt = 0
                isSpeechTrueCount = 0
                echoRejectStreak = 0
                fixtureRecorder.mark('interrupt', `confirmMs=${confirmMs}`)
                resetIdle()
                bus.resetTelemetry()
                bus.setUi({ interruptConfirmMs: confirmMs })
                void hardBreak()
              }
            } else {
              // 迟滞衰减而非清零：单拍「假」不衰减，连续 2 拍假才减 1。
              isSpeechFalseRun++
              if (isSpeechFalseRun >= 2) {
                isSpeechFalseRun = 0
                isSpeechTrueCount = decayConfirmCount(isSpeechTrueCount, () =>
                  engineRef.current?.invalidateSalvageSeed(),
                )
                if (isSpeechTrueCount === 0) interruptFirstAt = 0
              }
            }
            ...
```

**回声门控（信号级判据）**，`src/asr.ts:1477-1485`（原样）：

```ts
    aboveEchoFloor(marginDb = 6) {
      // 无地板（刚开播）→ 保守拒绝打断：放行会让外放回声在首播秒内误触发自打断
      // （实测外放 auto 根因之一）；代价是首播前 ~2s 无法自动打断（真机可接受）。
      // 用峰值（跨音节间隙）而非瞬时值：瞬时值在音节停顿处掉回回声水平被误拒。
      const pass = echoFloorRms > 0 && echoPeak > echoFloorRms * Math.pow(10, marginDb / 20)
      if (!pass && gateQueryScope) preRollSeedWall = null
      return pass
    },
```

**hardBreak 做了什么**（`client.tsx:2623-2669`，5 件事，全部"立刻做、不等慢操作"）：
1. `bus.skipAudio()` —— 本地播放队列清空 + 停 `AudioBufferSourceNode`，立刻静音
2. `bus.unduckAudio()` —— 恢复增益
3. `cancelAutoSend()` / `submitInFlightRef=false` / `correctCycleRef++` —— 取消在途自动发送，防"打断后旧文本被发出"（幽灵提交）
4. `bus.cancelTurn(sid)` + `POST /voice-mode/cancel {keepAsr, reason:'interrupt'}` → host `queue.cancel()`（`tts-queue.ts:555-564`，`q.epoch++` 让积压/在途全部作废）+ `readMuted.set()` 让本回合余句不再入队
5. `engineRef.discardSegment()`（非 hold 时）—— 丢弃被打断的半截 ASR 段，防残缺文本被 autosend

**打断方式三档**（`bargeInMode: 'auto' | 'manual' | 'detect'`，默认 `detect`，`src/index.ts:277`）：
- `detect`（默认）：进模式时读 `track.getSettings().echoCancellation`，false → 自动降级 `manual`（`asr.ts:1311-1319`，原样）：

```ts
    const aecOn = stream.getAudioTracks()[0]?.getSettings().echoCancellation === true
    if (!aecOn) {
      console.warn('[dsh-voice-mode] native browser echoCancellation is not in effect (speakers may self-interrupt); use headphones or manual barge-in')
    }
    if (config.bargeInMode === 'detect') {
      runtimeBargeInMode = aecOn ? 'auto' : 'manual'
    }
```
- `manual`：只认按住 mic / Ctrl；host 侧还有一道 `if (bargeInMode() === 'manual' && !manualPressed) return { text: '' }` 的 fail-closed 守卫（`asr-host.ts:517-519`）。
- 外放强回声环境下这是"物理边界"的正解（ADR-0001 / ADR-0006 有完整推演）。

**打断吃句的抢救**（ADR-0012）：播放期一直维护一个 2s 的 AEC 后预录环（`asr.ts:296-304`，`PRE_ROLL_MS = 2000`），真打断时把"语音起始-250ms"起的音频并进新识别段。

### 6.2 TALK：**没有语音打断**（只有"按下麦克风按钮 = 请求停播"，且停不干净）

源码事实〔已验证·源码〕：

1. 唯一的打断触发点是**按下录音按钮**，`TalkMicButton.tsx:238-242`（原样）：

```ts
  const start = async (): Promise<void> => {
    const snapshot = settingsRef.current
    if (snapshot === undefined || snapshot.record.enabled !== true) return
    // Speak-to-interrupt: starting to talk stops host playback.
    if (snapshot.interrupt) void interrupt()
```

2. host `interrupt()` **只 abort 在途合成**，不碰已发给浏览器的音频，`service.ts:518-527`（原样）：

```ts
  /** Stop every in-flight local synthesis (the user started talking). */
  interrupt(): TalkInterruptResult {
    let stopped = false
    for (const controller of this.activeSyntheses.values()) {
      stopped = true
      controller.abort()
    }
    this.activeSyntheses.clear()
    return { stopped }
  }
```

3. 客户端**也没有**停播代码。全仓 grep `speechSynthesis.cancel|speechSynthesis.pause|\.pause()` → **0 命中**〔已验证·源码〕。播放对象是局部变量、无句柄：
   - `TalkMicButton.tsx:459`：`const player = new Audio(url)`（函数内局部，播完只 `URL.revokeObjectURL`，从不 pause）
   - `TalkMicButton.tsx:442-447`：`const utterance = new SpeechSynthesisUtterance(...)` + `speechSynthesis.speak(utterance)`

> **结论**：TALK 的"说话即打断"在源码上是**"按按钮 → 请 host 别再合成下一句"**。已经在播的那一段会一直播完。README 的"说话即打断"〔未验证·README〕与实现不符。

4. TALK 有 VAD，但**只用于"静音自动结束录音"**，与打断无关，且是**纯 RMS 能量阈值**不是模型，`src/client/vad.ts:12-42`（原样）：

```ts
export function rmsOf(samples: Float32Array): number {
  if (samples.length === 0) return 0
  let sum = 0
  for (let index = 0; index < samples.length; index += 1) {
    const sample = samples[index]!
    sum += sample * sample
  }
  return Math.sqrt(sum / samples.length)
}

export function isSpeechFrame(rms: number, threshold: number): boolean {
  return rms >= threshold
}

export function advanceSilence(previousSilenceMs: number, frameMs: number, speech: boolean): number {
  return speech ? 0 : previousSilenceMs + frameMs
}

export function silenceTrips(silenceMs: number, thresholdMs: number): boolean {
  return silenceMs >= thresholdMs
}
```

使用点 `TalkMicButton.tsx:385-397`（原样，注意 `advanceSilence(..., 500, ...)` 的 500 是**写死的 500ms 假帧长**，而实际 tick 是 `setInterval(..., 500)`，靠巧合对上了）：

```ts
      const analyser = analyserRef.current
      const vad = settingsRef.current?.record.vad
      if (analyser !== null && vad?.enabled === true) {
        const buffer = new Float32Array(analyser.fftSize)
        analyser.getFloatTimeDomainData(buffer)
        silenceMsRef.current = advanceSilence(silenceMsRef.current, 500, isSpeechFrame(rmsOf(buffer), vad.energyThreshold))
        if (silenceTrips(silenceMsRef.current, vad.silenceMs)) {
          stop()
          return
        }
      }
```

---

## 7. 语音字幕

### 7.1 VM：有字幕，落在 `shell.overlay` 槽位（固定定位浮层）

**槽位注册**，`src/client.tsx:519-529`（原样）：

```ts
  ctx.slots.inject('shell.overlay', () =>
    ctx.slots.register(
      {
        name: 'shell.overlay',
        id: 'voice-mode-overlay',
        order: 100,
        inject: (): VoiceSlotActions => ({ bus }),
      },
      VoiceOverlay,
    ),
  )
```

**组件**，`src/client.tsx:4307-4386`（原样，节选）：

```tsx
export function VoiceOverlay({ bus }: OverlayProps): React.ReactElement {
  useLang() // 语言切换时重渲染
  const [b, setB] = useState(() => ({ active: bus.activeSessionId, ui: bus.ui }))

  useEffect(() => {
    return bus.subscribe(setB)
  }, [bus])

  // 播放引擎只推当前语音会话的帧（模式隔离）；有朗读才显示浮层。
  if (!b.ui.playing || b.ui.boot?.showCaptions === false) return <></>

  return (
    <div
      role="status"
      aria-live="polite"
      style={{
        position: 'fixed',
        right: 16,
        bottom: 96, // 上移，避免盖住底部输入框/麦克风按钮
        zIndex: 9999,
        ...
      }}
    >
      ...
      <span
        key={b.ui.playingCaption ?? 'idle'}
        className="dshvm-caption"
        style={{ whiteSpace: 'normal', overflowWrap: 'anywhere', wordBreak: 'break-word', flex: 1, minWidth: 0 }}
      >
        {b.ui.playingCaption ?? t('reading')}
      </span>
      <button onClick={() => bus.skipAudio()} aria-label={t('skipReading')} ...>
        {t('skipReading')}
      </button>
    </div>
  )
}
```

- **DOM 节点**：`ctx.slots.inject('shell.overlay', …)` 渲染出的 `<div role="status" aria-live="polite" style="position:fixed;right:16px;bottom:96px;z-index:9999">`，字幕文本是其中 `<span class="dshvm-caption">`。
- **字幕随"真实起播"推进，不领先音频**：`captionQueue`（`client.tsx:632`）+ 只在 `setUi({playing:true, playingCaption: frame.text})` 时推进（`client.tsx:703`、`762`）。
- 可配：`showCaptions`、`captionFontSize` 0-3（12/14/18/24px）、`captionMaxWidth` 0-2（50/70/90vw）×（`client.tsx:4333`、`4350`）。
- **a11y**：`role="status"` + `aria-live="polite"`；有专门测试 `test/caption-a11y.test.mjs`、`test/statusbar-aria.test.mjs`。
- 另有输入框上方状态条：`conversation.input.dock` → `VoiceStatusBar`（`client.tsx:507-517`）。

### 7.2 TALK：**完全没有字幕/字幕浮层**

全仓 grep `caption|subtitle|字幕` → **0 命中**；连 `sentence|分句` 也只有 `src/tool.ts:28` 一个英文注释里的 "sentence"（讲的是渲染句式，不是分句器）〔已验证·源码〕。它只在 `conversation.input.left` 挂一个 `<button>`（`TalkMicButton.tsx:470-484`）：

```tsx
  return (
    <button
      type="button"
      data-dsh-talk-mic
      data-recording={recording ? 'true' : 'false'}
      data-disabled={disabled ? 'true' : 'false'}
      title={mic.phase === 'error' ? mic.error : undefined}
      disabled={disabled}
      onClick={() => { if (recording) stop(); else start() }}
    >
      {label}
    </button>
  )
```

无转录预览、无"正在朗读哪一句"、无 aria-live。

---

## 8. 手机端麦克风权限 / WebView 的坑

### 8.1 VM 的 getUserMedia（约束很讲究）

`src/asr.ts:1288-1299`（原样）：

```ts
    const myStream = await navigator.mediaDevices.getUserMedia({
      audio: {
        channelCount: 1,
        echoCancellation: true,
        // L0（研究核实）：AGC/NS 是时变非线性，放在 AEC 前破坏线性回声路径假设——
        // WebRTC GainController2 与 Speex 均把增益放 AEC 之后。禁用后原生 AEC 与
        // 自研 AEC 的输入才线性，回声消得净；AGC 放大近端语音由「残差地板门控」
        // 的归一化语义替代，不需要采集端增益。
        noiseSuppression: false,
        autoGainControl: false,
      },
    })
```

错误分类（用户看得懂的话术），`src/client.tsx:3032-3050`（原样）：

```ts
    } catch (e) {
      if (enterEpochRef.current !== myEnter) return
      // getUserMedia 被拒等：留在 off，红点提示（Q16）
      engineRef.current = null
      ...
      setLocalMode('off')
      const msg =
        e instanceof DOMException
          ? e.name === 'NotAllowedError'
            ? t('micDenied')
            : // UX：NotReadableError = 麦克风被占用（其他程序/标签页）——与「不可用」
              // 混同时用户无从下手，分原因给可行动文案。
              e.name === 'NotReadableError'
              ? t('micBusy')
              : t('micUnavailable')
          : t('startFail').replace('{err}', String(e instanceof Error ? e.message : e))
```

移动端专门的处理：
- **AudioWorklet → ScriptProcessor 回退**（`asr.ts:1343-1394`）：`audioWorklet.addModule` 失败自动降级，避免"进了语音模式但静默无反应"。
- **采样率协商**：`new AC({ sampleRate: SAMPLE_RATE })` 后读回 `ctxRate = audioCtx.sampleRate`（Safari/Android 会忽略选项），交给 `resampleLinear` 重采样到 16k（`src/resample.ts`）。
- **手势栈内 resume AudioContext**（`asr.ts:1326-1332`）：iOS Safari / 部分 WebView 非手势栈创建的 AudioContext 是 suspended。
- **触屏长按菜单压制**：`client.tsx:3549-3564`，非被动 `touchstart` + `preventDefault`（React 的 onTouchStart 是 passive，无效）。
- **没有 `isSecureContext` 显式检查**（全仓 grep 无命中）——靠 host 默认只绑回环 + README 说明。**这是它比 TALK 弱的一点**。
- README「已知限制」里 iOS 章节（〔未验证·README〕）：需 HTTPS/localhost；被拒后到 设置→Safari→麦克风 开启；后台/锁屏暂停、回前台恢复（可能丢句）。

### 8.2 TALK 的 getUserMedia（**在 Android WebView 上会撞死路**）

安全上下文检查，`TalkMicButton.tsx:325-334`（原样）：

```ts
    if (typeof window !== 'undefined' && window.isSecureContext === false) {
      setMic(foldMic(micRef.current, { kind: 'failed', message: 'microphone requires a secure origin (https or localhost)' }))
      return
    }
    const mime = supportedMime()
    if (mime === undefined || navigator.mediaDevices === undefined) {
      setMic(foldMic(micRef.current, { kind: 'failed', message: 'MediaRecorder is unavailable in this browser' }))
      return
    }
```

**但错误处理把原因糊掉了**，`TalkMicButton.tsx:334-340`（原样）：

```ts
    let stream: MediaStream
    try {
      stream = await navigator.mediaDevices.getUserMedia({ audio: true })
    } catch {
      setMic(foldMic(micRef.current, { kind: 'failed', message: 'microphone permission denied' }))
      return
    }
```

（对比 VM 分 NotAllowedError / NotReadableError / 其它三档；TALK 全部报"permission denied"。）

**⚠️ 最致命的 WebView 死路（源码可证）**：

1. `start()` 只有在 `snapshot.stt.resolved === 'web'` 时用 Web Speech（`TalkMicButton.tsx:247`）。
2. Android WebView **不实现 `webkitSpeechRecognition`**（Chromium 的 SpeechRecognition 不在 WebView 暴露）→ `webRecognition()` 返回 `undefined`（`TalkMicButton.tsx:70-82`）。
3. 于是代码落到 `MediaRecorder` 分支，录完调 `transcribe(base64)` → `talk/transcribe`。
4. 但 host 端会直接抛错，`src/service.ts:499-503`（原样）：

```ts
  async transcribe(audioData: string): Promise<TalkTranscript> {
    const engine = resolveSttEngine(this.resolved)
    if (engine === 'web') {
      throw new Error('dsh-talk: the web STT engine transcribes in the browser; configure funasr or whisper for host transcription')
    }
```

**净效果**：默认配置（`stt.engine: auto`，未配 funasr/whisper）在 Android WebView 上 = 按麦克风 → 录一段 → 转写永远失败。必须显式配置 `funasr`（要有服务）或 `whisper`（要有二进制+模型）才能用。**这不是我猜的，是两条源码路径合起来的必然结果**〔已验证·源码〕。这直接决定了 §10 的移植结论。

> 关于 WebView 权限本身（App 侧）：Android WebView 里 `getUserMedia` 要求宿主 App 实现 `WebChromeClient.onPermissionRequest()` 并对 `RESOURCE_AUDIO_CAPTURE` 授权，且页面必须在安全上下文（`https://` 或 `http://127.0.0.1` 这类 potentially-trustworthy origin）。**这两个插件都不管这件事**（它们看不见 App 代码）。因为本机 DSH Web GUI 是 `http://127.0.0.1:57009`，**若 WebView 里加载的是 127.0.0.1 则满足安全上下文**；若走局域网 IP（例如 `http://192.168.x.x:57009`）则不满足，麦克风直接不可用。**〔平台知识，NOT_TESTED〕**

---

## 9. 前台/后台切换、页面隐藏时是否停止

### 9.1 VM：隐藏时**停麦克风**（隐私/battery），可见时**自动恢复**；不主动停 TTS

`src/client.tsx:3360-3381`（原样）：

```ts
      if (document.hidden) {
        if (localRef.current === 'on' && engineRef.current && !onceRef.current) {
          // M2：隐藏 tab 暂停收音（隐私），可见时恢复。toggle 与 hold 都关麦——hold 之前只
          // endHeld 不 stop，麦克风后台常开（隐私缺口）。
          if (bootNow().mode === 'hold') {
            engineRef.current?.endHeld(true)
            holdCtrlRef.current = false
            setHolding(false)
          }
          pausedForHiddenRef.current = true
          void engineRef.current.stop()
        }
      } else if (pausedForHiddenRef.current && localRef.current === 'on') {
        pausedForHiddenRef.current = false
        void engineRef.current?.start().catch(() => {
          // 无手势恢复失败（Safari/iOS）→ 完整退出语音模式（停 ASR + 让出 host）。
          // 仅 setLocalMode('off') 不够：bus/host 侧仍活跃 → 「按钮 off + 状态条在 +
          // AI 还在读」的分裂态（local 与 bus.active 漂移）。
          void exitModeRef.current('manual')
        })
      }
```

另有：
- `visibilitychange` 回可见时重报 focused session（`client.tsx:3165-3177`）。
- `bus.warmAudio()` 挂在一次性 `pointerdown`/`keydown`（`client.tsx:432-436`），处理"浏览器挂起 AudioContext 后回来没声"。
- 空闲自动退出：`idleTimeoutMinutes` 默认 5（`index.ts:267`）。

### 9.2 TALK：**完全没有任何页面生命周期处理**

全仓 grep `visibilitychange|pagehide|document.hidden|beforeunload|wakeLock` → **0 命中**〔已验证·源码〕。手机切后台/锁屏时 MediaRecorder、Web Speech、`Audio` 播放都保持原状（由系统节流决定何时掐死）。**这是长时间会话 + 移动端上明确的能力缺口。**

---

## 10. 插件与宿主（DSH host）的接口面

### 10.1 VM 注册了什么

`plugin/dsh-voice-mode/src/index.ts:171`（原样）：

```ts
export const inject = ['webServer', 'settings', 'sessions']
```

**host 半边（无 tool！）**〔已验证·源码：全仓 grep `ctx.tools|defineTool` → 0 命中〕：

| 事件 / 服务 | 位置 | 说明 |
| --- | --- | --- |
| `ctx.webServer.register({kind:'prefix'|'exact', path, handler})` | `index.ts:1224…`（17 处） | 自己造 HTTP 面 |
| `ctx.on('llm/stream', (options, next)) → AsyncIterable<StreamChunk>` | `index.ts:1102` | 无损 tap，只读 `text-delta`，其余 `yield` 透传 |
| `ctx.on('system-prompt/assemble', (assembly, ctx, next))` | `index.ts:1092` | 只给活跃语音会话注入"口语化"提示词 |
| `ctx.on('agent/turn-stopping', ({agent}))` | `index.ts:1195` | 回合真结束 → TTS 队列 `noteTurnEnd` |
| `ctx.on('agent/error', ({agent}))` | `index.ts:1207` | 同上（error 收场也会残留 openTurns） |
| `ctx.on('agent/pre-step', ({agent, turn, step}, next))` | `index.ts:1215` | 维护 turn/step，用于"停止朗读"的回合边界判据 |
| `ctx.settings.register(ns, schema, {base})` 或 `ctx.settingsScope` | `index.ts:864-951` | 双路径兼容（0.1.7+ 官方移除 register，改走插件自己的 `/voice-mode/settings`） |
| `ctx.get('sessions').get(id)` | `index.ts:739` | 会话存在性校验 |

**HTTP 面（17 个路由，全部自带回环+同源校验）**：
`GET /voice-mode`（健康检查）、`/config`、`POST /toggle`、`POST /asr`、`POST /cancel`、`GET /stream`（SSE）、`/preview`、`/correct`、`/models/status|retry|clean|download`、`/voices`、`/settings`、`/mode`、`/focus`。

**SSE 事件词汇表**（`index.ts` 各 `broadcast()` 调用点）：
`audio`（`{sessionId, sentenceId, chunkId, final, text?, audio(base64), mime?}`）、`mode`（`{active, ownerTabId}`）、`focus`（`{sessionId, tabId}`）、`display`（`{readAloud, showCaptions, showStatusBar}`）、`turn`（`{sessionId, state, turn}`）、`latency`（`{sessionId, stage:'first-llm-token'|'first-sentence-text'}`）、`tool`（`{sessionId, name}`）、`tts-idle`、`tts-error`、`tts-skip`（`{sessionId, text, streak}`）、`asr-progress`（`{file, percent}`）、`asr-ready`、`asr-error`（`{file, reason?}`）、`replaced`。

**client 半边槽位**（`dsh.client.inject` 9 个锚点 + 4 类槽）：`conversation.input.activity`（single！）/ `conversation.input.right` / `conversation.input.dock` / `shell.overlay` / `settings.plugin.item` / `settings.section` / `plugins.bundle.config`。

> ⚠️ **与你本机环境的直接冲突**：`conversation.input.activity` 在 0.2.1-rc.2 里是 `kind: "single"`（`dsh-client-ui-conversation/lib/client.js`，实测），而**已装的 `@deepseek-ai/dsh-experimental-voice-input-bundle` 也注册在这个槽**（`dsh-experimental-client-ui-voice-input/lib/client.js`：`ctx.slots.inject("conversation.input.activity", …)`）。VM 作者已经想到这点并做了兜底：`client.tsx:440-476` 用 `activity.mounted` 计数 + 1.5s 超时判定"activity 槽被别的注册者遮住"→ 自动在 `conversation.input.right` 补一个按钮，且保证同页只出现一个麦克风。**TALK 用 `conversation.input.left`（kind: list），没有这个冲突。**

### 10.2 TALK 注册了什么

**host 半边有 1 个 tool + 1 个 service + 1 个 projection unit + 3 个事件**：

| 接口 | 位置 | 签名 |
| --- | --- | --- |
| `ctx.tools.register(speakTool(service, maxSpeakChars))` | `index.ts:101` | tool `name: 'speak'`（`tool.ts:82`） |
| `new TalkService(ctx, config, resolved, 5)` → `ctx.get('talk')` | `index.ts:88-95` / `service.ts:169` | `class TalkService extends TypertRemoteService`，命名空间 `'talk'` |
| `ctx.sessionProjections.register({key:'talk:speech', …})` | `index.ts:105-118` | 会话投影单元，客户端 `useProjection('talk:speech')` 读 |
| `ctx.on('agent/status', ({agent, status}))` | `index.ts:122` | running→idle 播 "Turn complete." |
| `ctx.on('agent/error', ({agent}))` | `index.ts:133` | 播 "Something went wrong." |
| `ctx.on('approval/request', (request, next))` | `index.ts:141` | waterfall，先 `next()` 再播，**绝不阻塞审批门** |

`inject`：`export const inject = ['tools', 'subprocess']`（`index.ts:40`）。

**`speak` 工具的真实签名**（`src/tool.ts:80-136`，原样节选）：

```ts
export function speakTool(service: TalkService, maxChars: number) {
  return defineTool({
    name: 'speak',
    description: 'Speak text aloud to the user through the configured text-to-speech engine (browser voice, edge-tts, or piper). ...',
    parameters: {
      text: { type: 'string', required: true as const, description: 'The exact text to speak aloud.' },
      engine: { type: 'string', enum: ['auto', 'browser', 'edge-tts', 'piper'], description: 'TTS engine override; default is the configured engine.' },
      voice: { type: 'string', description: 'Voice name override for the resolved engine.' },
      interrupt: { type: 'boolean', description: 'Interrupt any speech currently playing before starting (default true).' },
    },
    output: {
      schema: {
        type: 'object',
        properties: {
          spoken: { type: 'boolean' }, engine: { type: 'string' }, audioBytes: { type: 'integer' },
          durationSec: { type: 'number' }, error: { type: 'string' }, utteranceId: { type: 'string' },
          audioPath: { type: 'string' }, audioExport: { type: 'string', enum: ['file','unsupported','disabled','failed'] },
          exportNote: { type: 'string' },
        },
        additionalProperties: false,
      },
      render: renderSpeak,
    },
    async execute(args, exec) { ... }
  })
}
```

**Remote（Typert）6 个 invocation**（`src/wire.ts:385-392` 的 `TALK_INVOCATIONS`）：

| id | 参数 | 返回 |
| --- | --- | --- |
| `talk/status` | — | `TalkStatus` |
| `talk/audio` | `utteranceId: string` | `TalkAudio \| null` |
| `talk/transcribe` | `audioData: string`（base64） | `TalkTranscript` |
| `talk/applySettings` | `settings: TalkSettingsInput` | `TalkSettingsResult` |
| `talk/interrupt` | — | `TalkInterruptResult` |
| `talk/latest` | `sessionId: string` | `TalkLatest \| null` |

**会话日志事件**：`'dsh-talk/speech'`（`src/vocabulary.ts:21-44`），载荷类型：

```ts
export interface DshTalkSpeechEvent {
  kind: 'tts'
  utteranceId: string
  engine: SpeechTtsEngine          // 'browser' | 'edge-tts' | 'piper'
  text: string
  audioBytes: number
  reason: SpeechReason             // 'speak-tool' | 'turn-end' | 'approval' | 'error'
  error?: string | undefined
  interrupted?: boolean | undefined
  voice?: string | undefined
  rate?: number | undefined
  pitch?: number | undefined
}
```

写入是**带门控的**（`src/speech.ts:53-57`，原样）：

```ts
export function appendSpeechEvent(session: Session, data: DshTalkSpeechEvent): boolean {
  if (!KNOWN_SESSION_EVENT_TYPES.has(SPEECH_EVENT)) return false
  session.append(SPEECH_EVENT, data)
  return true
}
```

**client 槽位**：`conversation.input.left`（id `talk-mic`, order 20）+ `settings.plugins.tab`（id `talk`, order 50）（`src/client/index.ts:126-140`）。

> 架构对比一句话：**TALK 走"DSH 一等公民"接口（tool + Typert Remote + session projection + 事件），VM 走"自家 HTTP 面 + slot"**。前者更符合宿主规范、天然多客户端；后者不污染模型上下文、能力更强但所有跨进程通信都是自造的（17 条路由 + SSE），可移植性取决于宿主是否愿意转发 HTTP。

---

## 11. 依赖体积 / 是否需要额外二进制或 Python

| | VM | TALK |
| --- | --- | --- |
| npm 运行时依赖 | `sherpa-onnx`（unpacked **15.1 MB** 的 WASM/js）+ `sherpa-onnx-node`（59 KB，含**平台原生包**：win-x64 21.9MB / linux-x64 31.2MB / linux-arm64、darwin-arm64 31.8MB…）+ schemastery | **`zod` 一个** |
| 插件自身产物 | `lib/client.js` 295KB + `lib/index.js` 169KB + `lib/msedge-tts.cjs` **737KB**（内联微软 TTS 客户端）+ 2 个 worker | 仓库无 `lib`（`prepare` 构建）；npm 包 1.18MB |
| 额外二进制 | ❌ 不需要（全在进程内 WASM/addon + 子进程复用 `process.execPath`） | ✅ **需要**：`edge-tts`（Python CLI）、`piper`（二进制+onnx）、`whisper-cli`（二进制+ggml）、FunASR HTTP 服务 |
| 需要 Python | ❌ | ✅ 若要 `edge-tts`（它是 pip 包） |
| 运行时模型下载 | ~390MB（ASR 160 + SenseVoice 228 + VAD 2），SHA256 锁定，可断点续传/换镜像 | 0（也不帮你下） |
| Electron 特例 | ⚠️ Kokoro 用 `sherpa-onnx-node` 原生 addon，Electron（V8 内存笼）拒绝 external buffer → 必须找一个**真 Node ≥18**（`src/tts-runtime.ts:40-51`，错误码 `kokoro_needs_node`） | 无（纯 JS + 子进程） |

VM 的 Electron 判定（`src/tts-runtime.ts:65-80`，原样节选）：

```ts
  const electron = opts.electron ?? !!process.versions.electron
  let result: NativeRuntime | null
  if (!electron) {
    result = { env }
  } else {
    const childEnv: NodeJS.ProcessEnv = { ...env }
    delete childEnv.ELECTRON_RUN_AS_NODE
    const probe = opts.probe ?? probeNode
    const candidates = [env.DSHVM_NODE, 'node'].filter((c): c is string => typeof c === 'string' && c.length > 0)
    ...
    result = execPath ? { execPath, childEnv } : null
  }
```

> 对本机（Electron 桌面宿主）的含义：VM 的 **edge 与 vits 引擎开箱可用**，**kokoro 需要系统里另装真 Node**（或设 `DSHVM_NODE`）。这一点对 Android（Node 在 dsh-Go 引擎里跑）会变成"能不能加载 arm64 原生 addon"的问题，见 §12。

---

## 12. 长时间语音会话的稳定性设计

### 12.1 VM：这是它最强的一块，逐条给证据

| 机制 | 位置 | 解决的真实现象 |
| --- | --- | --- |
| `segmentEpoch` / `detectGeneration` / `resetGen` / `turnGen` / `q.epoch` **5 套代际计数器** | `asr.ts:261`、`asr.ts:292`、`asr-host.ts:238`、`index.ts:696`、`tts-queue.ts:250` | 弃段/定稿后迟到的 partial、reset 后旧 vadOnly 响应毒化水位、旧回合状态覆盖新回合、打断后孤儿泵继续广播 |
| host 段 90s 空闲回收（30s 一扫） | `asr-host.ts:166-167`、`asr-host.ts:700-766` | 页面崩溃/断电后 stream/VAD 悬挂、`finalized` 缓存随会话只增不消 |
| `finalized` 定稿缓存 32 条 LRU + `finalizing` 并发定稿复用 | `asr-host.ts:614-697` | final 重试丢句、重复定稿开销 |
| 模型下载失败 60s 退避 | `asr-host.ts:254-256`、`312-313`、`381-382` | 源不可达时每拍 partial 都触发下载，把增量识别拖慢数百 ms |
| body 空闲看门狗 120s + 仅 headers 阶段超时 60s | `models.ts:123-153` | 上游"接受连接后永不吐头"→ Promise 永不 settle → 后续每次 ASR 永久等待（真机"进语音模式后麦克风无响应、控制台无错"根因） |
| SHA256 校验结果缓存（size+ino+**ctime**） | `models.ts:97-98` | 239MB 全量哈希 0.6-1.6s 卡主线程（每次打断杀子进程后都重走） |
| `dispose()` 后禁止懒重建（三处守卫） | `asr-host.ts:251`、`410-420` | 热重载每代泄漏一块 150MB WASM / 一个常驻 worker 线程 |
| SSE 背压 | `index.ts:2015-2019`（`SSE_MAX_BUFFERED = 4MB`） | 慢消费者把全部音频帧驻留内存 |
| SSE 连接上限 8 + 同 tabId 顶替 + 25s 心跳 | `index.ts:1989-2047` | 页面重载后旧 EventSource 没关 → "能说话但回复不朗读" |
| TTS 队列上限 500 句 + 单句 3 次重试+退避 | `tts-queue.ts:543-546`、`597-615` | 长回复"跳句"、Edge 偶发 ETIMEDOUT 静默丢句 |
| 空音频整句丢弃（不占句号、不计失败） | `tts-queue.ts:627-631` | 纯情绪标签句合成出 44 字节空 WAV |
| 打断限流 2 次/秒 | `index.ts:1796-1799` | 抖动触发打断风暴 |
| 双 tab 抢占 + 让出语义（`mode`/`focus` 广播 + 2s 重入冷却） | `client.tsx:2783-2790` | 两个标签页同时进语音模式互踢振荡 |
| PCM 体量上限 4MB / JSON 体 16KB | `asr-host.ts:165`、`index.ts:2087` | 无界内存 |
| 测试规模 | 78 个 test 文件 / **237 个 test() 用例**〔已验证·命令〕 | 每个 bug 都配了回归测试（`barge-in-*`、`echo-tail`、`sse-backpressure`、`model-download-idle`、`dispose` 相关…） |

### 12.2 TALK：有界但不深

| 机制 | 位置 |
| --- | --- |
| 音频内存缓存 `MAX_UTTERANCE_COUNT = 64` + 字节上限（默认 8MiB，上限 64MiB），LRU 驱逐 | `service.ts:42`、`414-430` |
| 每会话语音记录表上限 32 会话 | `service.ts:45`、`370-374` |
| 导出音频目录**双维度**有界（`maxFiles` 20 / `maxAgeMinutes` 1440）+ 只删自己命名的文件 | `audio-export.ts:1-19`、`76` |
| 单次合成 16MiB 上限；录音 1..600s（默认 60） | `engine.ts:31`、`config.ts:20-23` |
| VAD 静音自动结束（RMS 0.01 / 1500ms） | `config.ts:56-62` |
| 设置写回带时间戳备份 + 合并后先校验再写（拒绝写入"加载不起来的配置"） | `service.ts:542-558` |
| 测试规模 | 18 个 spec / **108 个 `it()`**〔已验证·命令〕 |

**缺口**：无页面生命周期处理（§9.2）、无重试/退避（合成失败就跳过该句）、无 SSE/长连接（每次 Remote 调用一次性返回整段 base64）、无并发/竞态代际设计（因为是无状态请求-响应，暂时不需要，但也意味着没法做流式）。

---

## 13. 逐维打分总表

| 维度 | VM | TALK | 胜 |
| --- | --- | --- | --- |
| 中文识别引擎 | sherpa zipformer2-zh + SenseVoice（本地，SHA 锁定，零 key） | Web Speech / 自备 FunASR / 自备 whisper.cpp | **VM** |
| 实时流式识别 | ✅ 64ms 帧 / 100ms 增量 POST / partial + 语义端点确认 | ❌（只有浏览器 interim；本地引擎整段批处理） | **VM** |
| 流式 TTS 按句朗读 | ✅ 分句器 + 逐句入队 + SSE 逐句播 | ❌ 整段一次性合成 | **VM** |
| 本地语音引擎 | ✅ ASR/VAD/TTS 全本地可离线 | ⚠️ 依赖自装二进制 | **VM** |
| 语音打断（VAD 检测） | ✅ Silero VAD 帧级 + 3 档确认 + 回声门控 + hardBreak 全静音 | ❌ 按按钮才"请求"停，且**不会真的停播** | **VM**（压倒性） |
| 语音字幕 | ✅ `shell.overlay` 浮层 + a11y | ❌ 无 | **VM** |
| 长会话稳定性 | ✅ 5 套代际 + 回收 + 退避 + 背压 + 237 用例 | ⚠️ 有界缓存，无生命周期 | **VM** |
| 手机 WebView 麦克风 | ✅ 错误分类 + worklet 回退 + 采样率自适应 + iOS 手势栈 resume（缺 isSecureContext 检查） | ⚠️ 有 isSecureContext 检查但错误糊成一类；**默认引擎在 WebView 上死路** | **VM** |
| 前后台切换 | ✅ hidden 停麦 + 可见恢复 + 无手势失败即退模式 | ❌ 完全没有 | **VM** |
| 宿主接口规范性 | ⚠️ 自造 17 条 HTTP 路由 + SSE，只注册事件不注册 tool | ✅ tool + Typert Remote + session projection + 事件，一等公民 | **TALK** |
| 依赖/体积 | ⚠️ 15MB WASM + 平台原生包 + 390MB 模型 | ✅ 只有 zod；但功能要靠外部二进制 | **TALK**（体积）/ 平（可运维） |
| 与已装 `voice-input-bundle` 冲突 | ⚠️ 抢 `conversation.input.activity`（single），已有兜底 | ✅ 用 `conversation.input.left`（list） | **TALK** |
| 与 DSH 0.2.1-rc.2 兼容 | ✅ 显式在 21 版已验证矩阵内 | ✅ 声明范围内 | 平 |

**9 : 4**，且 TALK 赢的 4 项里，只有"宿主接口规范性"是长期架构优势，其余是体积/冲突这类可工程规避的问题。

---

## 14. 最终结论

### 14.1 选型

- **主选：`dsh-voice-mode` v0.7.34** —— 唯一一个把这 11 个维度当产品做完了的实现。尤其**语音打断**这一项，TALK 在源码层面根本不具备（§6.2），这不是调参能补的，是要重写播放层的工程量。
- **备选：`dsh-talk` v0.3.26** —— 保留作为"**架构参考**"和"**你要接官方 speech-to-text 服务时的样板**"：它的 Typert Remote 声明、`talk:speech` 会话投影、`appendSpeechEvent` 的 known-type 门控、`speak` 工具的 output schema/render 分离，都是 DSH 一等公民写法的正面教材，值得 VM 借。

### 14.2 理由（按重要性）

1. **打断是语音模式的灵魂，只有 VM 有**。VM 有完整的"检测→确认→回声门控→硬停播→撤会话→弃半句→抢救半句"闭环（§6.1）；TALK 只有"按按钮请 host 别合成下一句"，且在播的那一段停不掉（§6.2 三步源码证据）。
2. **VM 不依赖 Web Speech**。这是 Android 移植的生死线（下一节）。
3. **VM 有字幕、有逐句 TTS、有中文端点语义、有本地全离线路径**——四个维度 TALK 全 0。
4. **VM 的稳定性是"打出来的"**：5 套代际计数器、90s 段回收、SSE 背压、模型下载看门狗，每一条都挂着具体真机故障的现象描述；237 个用例。这是长时间语音会话的必需成本，TALK 侧等于从零做。
5. TALK 的优势（体积、宿主规范性、无槽位冲突）都是**可工程规避**的；VM 的优势（打断、字幕、流式）是**要从零写**的。

### 14.3 风险与建议（给主会话）

- VM 的 `conversation.input.activity` 是 single 槽，会和已装的 `@deepseek-ai/dsh-experimental-voice-input-bundle` 抢位。VM 自带兜底（1.5s 未挂上就退到 `conversation.input.right`），但**建议装 VM 时先禁用 `dsh-experimental-voice-input-bundle`**（两者都做"语音输入"，语义重叠；VM 的 ASR 明显更强）。
- VM 默认 TTS 是 **Edge 云端**（微软"大声朗读"非官方接口），2026-10-15 / 2026-10-26 各翻车一次。若要稳定，建议切 `vits`（纯中文、全离线、130MB）。
- VM 首次使用要下 **~390MB** 模型。建议在装机引导里显式告知 + 提供 `modelHost=hf-mirror.com`。

---

## 15. 移植到 Android 原生（dsh-Go）需要重写哪部分

> 前提事实（来自同目录 `31-dsh-go-local-source-authority.md` 对 `D:\code\dshgo` 的只读核对，此处**引用**，未重复验证）：dsh-Go 里插件**不能调用任何 Android API**；原生视图只能从白名单组件里挑（v1 16 种 / v2 42 种）；`overlay` 能力"画在 App 自己界面上，**不覆盖其他 App**"，且需要点一下才起；WebView 里跑的是同一套 DSH Web GUI。
>
> 本轮**没有**在任何 Android 设备/模拟器上跑过这两个插件。以下所有"Android 侧会怎样"均为**平台知识 + 源码推导**，标注为〔NOT_TESTED〕的部分请以真机为准。

### 15.1 关键结论：VM 的"浏览器侧"能在 WebView 活着，但"host 侧 ASR"必须换实现

把 VM 的三层拆开看：

| # | 层 | 具体实现（源码位置） | Android 上能不能原样用 | 结论 |
| --- | --- | --- | --- | --- |
| A | **浏览器采集** | `getUserMedia` + `AudioWorklet`（`asr.ts:1288`、`1343-1367`）+ 16k 重采样（`resample.ts`） | ✅ Android System WebView（Chromium）支持 `getUserMedia`、`MediaStreamAudioSourceNode`、`AudioWorklet`；**但要求宿主 App 实现 `onPermissionRequest` 授 `RESOURCE_AUDIO_CAPTURE`，且页面在安全上下文**〔NOT_TESTED〕 | **基本可复用**（需要 App 侧补权限桥） |
| B | **HTTP/SSE 传输** | 17 条 `/voice-mode/*` 路由 + `/stream` SSE | WebView 页面本来就访问 `127.0.0.1:57009`，同源即可 | **可复用**（前提：dsh-Go 的 webServer 与桌面同一套） |
| C | **host 侧 ASR/TTS 引擎** | `sherpa-onnx`（WASM）+ `sherpa-onnx-node`（**原生 addon**，平台包只有 win-x64/win-ia32/linux-x64/linux-arm64/darwin-x64/darwin-arm64，**没有 android-arm64**）+ `fork` 子进程 | ❌ 原生 addon 在 Android 上**没有对应预编译包**；`fork(process.execPath)` 在 Android 上能否起 Node 子进程也存疑 | **必须重写**（见 15.3） |
| D | **dmux：断句/端点/打断判定** | Silero VAD（sherpa-onnx）+ `endpointConfirmMs` + 3 档确认帧 + 回声门控 | 算法本身可移植；但它在 C 层，位置会跟着 C 走 | **重写（跟着 C 走）** |
| E | **字幕渲染** | `shell.overlay` 槽位 + `position:fixed` 的 React 浮层 | 若仍跑 WebView 内的 Web GUI：**原样可用**。若要变成"原生悬浮字幕"：dsh-Go 的 `overlay` 能力**只能画在 App 界面内、不能覆盖其他 App**，且由 App 绘制（`MediaOverlays.kt`） | **两选一**：留在 WebView 里（零成本）或接 `overlays` 能力（需重画） |
| F | **播放（TTS 起播/打断静音）** | `AudioContext.decodeAudioData` + `AudioBufferSourceNode` + `skipAudio()` | WebView 可用（自动播放策略需一次用户手势，VM 已有 `warmAudio()` 挂在 pointerdown） | **可复用** |

### 15.2 必须先说清楚的三件事（否则 Android 端一定踩）

1. **Web Speech API 在 Android WebView 里不可用。**
   `webkitSpeechRecognition` / `SpeechRecognition` 在 Android WebView 中**不暴露**（Chromium 的语音识别依赖 Google 语音服务与浏览器授权流程，WebView 没有这条通道）。所以：
   - TALK 的 `web` STT 引擎在 WebView 上 → `webRecognition()` 恒 `undefined`（`TalkMicButton.tsx:70-82`）→ 落到 MediaRecorder → `talk/transcribe` → host 抛 "the web STT engine transcribes in the browser"（`service.ts:499-503`）。**默认配置下麦克风功能整体不可用**〔已验证·源码推导〕。
   - VM **完全不使用 Web Speech**（全仓 `SpeechRecognition|speechSynthesis` 在 `src/` 下 **0 命中**〔已验证·源码〕），所以这一刀砍不到它。**这是选 VM 的第一位理由。**
2. **`speechSynthesis`（浏览器 TTS）在 WebView 里不可靠。** 它依赖系统 TTS 引擎 + WebView 是否暴露；`getVoices()` 常返回空数组，`onend` 有时不触发。VM 只在 Kokoro/VITS/Edge 三条路径里跑，**不用 `speechSynthesis`**；TALK 的 `browser` TTS 引擎会直接撞这个坑〔NOT_TESTED，但这是 Android WebView 的已知行为〕。
3. **dsh-Go 的原生能力不覆盖"任意 UI"**：`overlay` 只给一行字幕且不越 App 边界；`composer.strip` 只有一格、≤24 字符。所以"把 VM 的麦克风按钮 + 状态条 + 字幕浮层原样搬成原生组件"做不到——要么继续用 WebView 里的 Web GUI（推荐），要么大幅降级交互（只剩一个 24 字符状态条 + 一行字幕）。

### 15.3 重写清单（按"必须做"排序）

#### ① 必须重写：host 侧 ASR/TTS 引擎（把 sherpa-onnx Node addon 换成 Android 原生）

- **现状**：ASR 走 `sherpa-onnx-node` 原生 addon（无 android 平台包）+ `sherpa-onnx` WASM；TTS VITS/Kokoro 走 `fork` 子进程里的原生 addon（`tts-local.ts`、`tts-runtime.ts`）。
- **Android 方案（三选一，建议 a）**：
  - **a) 原生 sherpa-onnx Android AAR**（`org.k2fsa.sherpa.onnx`，k2-fsa 官方有 Android 预编译）：`OnlineRecognizer`（zipformer2 流式）、`Vad`（Silero）、`OfflineRecognizer`（SenseVoice）、`OfflineTts`（VITS/Kokoro）全部有 Java/Kotlin API。**这是最贴的替换**：模型文件同名同格式（`.onnx` + `tokens.txt`）可直接复用 VM 的 SHA256 清单。
  - b) 继续用 WASM：`sherpa-onnx` 的 wasm 版在 Android WebView 里跑（JS/WASM 都能加载），但手机上 160MB encoder + 228MB SenseVoice 的 WASM 推理会非常慢且吃内存。**不推荐**。
  - c) 把 ASR 留在桌面/服务器，手机只为客户端（`/voice-mode/asr` 是 HTTP，天然可远程）。**如果 dsh-Go 接的是远端引擎，这条零成本**。
- **需要新建的桥**：Kotlin 侧暴露 `feed(sessionId, pcmF32, final, offset, epoch, manualPressed)` / `detect(sessionId, pcmF32)` / `reset(sessionId)` / `modelStatus()`，**签名照着 `asr-host.ts:103-136` 抄**，返回结构 `{text, loading?, endpoint?, isSpeech?}` 保持不变——这样 VM 的浏览器侧代码一行都不用改。

#### ② 必须重写：麦克风采集（如果不用 WebView 的 getUserMedia）

两种路线：
- **路线 A（推荐，最省）**：保留 WebView 里的 `asr.ts` 采集逻辑，只补宿主 App 三件事：
  1. `WebChromeClient.onPermissionRequest()` 授 `PermissionRequest.RESOURCE_AUDIO_CAPTURE`；
  2. `WebSettings` 允许 `setMediaPlaybackRequiresUserGesture(false)`（否则 TTS 起播被拦）；
  3. **保证页面 origin 是安全上下文**（`http://127.0.0.1:53000` 这类可以；局域网 IP **不行**）。
  同时补 VM 缺的那道检查：`if (!window.isSecureContext) → 明确提示`（VM 现在没有 `isSecureContext` 守卫，TALK 有）。
- **路线 B（要原生采集体验时）**：Kotlin `AudioRecord`（16kHz mono PCM16）+ 自带 AEC（`AcousticEchoCanceler`）/ NS（`NoiseSuppressor`）/ AGC 开关，用 `AudioTrack` 播放 TTS，通过 `ctx.dshGo` 的 `micro` 通道把 PCM 送进插件。
  **注意**：VM 的 AEC 设计是"**信任浏览器原生 AEC，自研 NLMS 只兜底**"（ADR-0001），并用 `track.getSettings().echoCancellation` 判断降级到 manual 打断（`asr.ts:1311-1319`）。换成 `AudioRecord` 后**没有这个信号**，必须用 `AcousticEchoCanceler.isAvailable()` 重写这段判定，并对"外放自打断"重新做降级策略（否则一定会出现"AI 说两个字就被自己打断"）。这是路线 B 最大的一块看不见的工作量。
- **路线 B 还必须重写**：`AudioWorklet` 的 64ms 分帧 + 16k 重采样（`audio-worklet.ts:26-31`：`CHUNK=1024`、`RATIO=sampleRate/16000`）→ 换成 `AudioRecord` 的 `read(shortArray, 0, 320)`（20ms）或 1024 帧。

#### ③ 必须重写：页面生命周期（Android 的"后台"和浏览器不同）

- VM 靠 `document.visibilitychange` 停麦/恢复（`client.tsx:3360-3381`）。Android 上 `visibilitychange` 只能表达"App 进后台"；**App 真被杀时不会触发任何 JS**。
- 需要 Android 侧补：
  - `onPause`/`onStop` → 主动暂停采集 + 释放 `AudioRecord`；
  - 长时间语音会话需要 **前台服务 + 常驻通知**（否则 Doze/省电策略会掐掉录音）；
  - 回前台恢复采集时**必须重新走一次用户手势**（`AudioRecord` 不需要手势，但 WebView 的 AudioContext 需要；VM 已在 pointerdown/keydown 上 `warmAudio()`，WebView 场景够用）〔NOT_TESTED〕。

#### ④ 需要适配：字幕

- 若继续跑 WebView：`shell.overlay` 的 `VoiceOverlay`（`client.tsx:4307`）原样工作，**零改动**。
- 若要原生字幕：接到 dsh-Go 的 `overlays` 能力，但**只能一行、≤240 字符、画在 App 界面内**。VM 现在的字幕带 4 档字号/3 档宽度/跳过按钮，原生版要砍到"一行文本 + 可点跳过（`composer.strip` 一格）"。
- 若要做**跨 App 悬浮字幕**：**dsh-Go 插件层做不到**（需 `SYSTEM_ALERT_WINDOW`，只给 App 自己的桌宠用）。要么改 App 自己（非插件），要么放弃。

#### ⑤ 需要适配：打断的音频通路

- VM 的"立刻静音"是 `bus.skipAudio()`（清队列 + `AudioBufferSourceNode.stop()`，`client.tsx:629` 的 `activeSrcs`）。WebView 场景可复用。
- `AudioRecord` 场景要换成"清 `AudioTrack` 缓冲 + 停止播放线程"，并且**回声消除的参考信号**（VM 的 `onPlaybackRef` 喂 AEC 参考池，`client.tsx:688-701`）要改成从 `AudioTrack` 取播报样本。这块是原生实现里最容易漏的（漏了就会复述回环："AI 听到自己"）。

#### ⑥ 不必重写（可直接沿用）

- `/voice-mode/*` 的全部 HTTP 路由与 SSE 事件词汇（若 webServer 在 dsh-Go 相同）。
- `segmenter.ts`（分句 + markdown 剥离，纯函数）。
- `endpointConfirmMs` / `CONJUNCTION_TAIL`（中文端点语义，纯函数）。
- `mic-gesture.ts`（长按/右键/触屏手势，纯逻辑）、`wakeword.ts`（唤醒词匹配，纯函数）。
- `emotion.ts`、`llm-correct.ts`、`settings-store.ts`、`security.ts`（`isLoopbackRequest`/`sameOriginRequest`/`RateLimiter`）。
- `src/models.ts` 的下载/校验/续传/镜像回退（纯 Node `fetch`+`fs`，Android 上的 Node 引擎照样能跑——用于下载模型到 App 私有目录）。
- 全部 78 个测试文件（约 237 用例）作为回归基线。

### 15.4 一句话给 Android 端

> **照抄 VM 的浏览器侧（采集/分帧/打断状态机/字幕/播放）+ 照抄 VM 的算法纯函数（分句/端点/唤醒/手势）+ 把 `asr-host.ts` 的三个引擎换成 sherpa-onnx 的 Android AAR（保持 `feed/detect/reset` 签名不变）+ 自己补 WebView 麦克风权限桥与前台服务。TALK 只在"要不要接官方 `speech-to-text` 服务接口"这一件事上值得参考，别照抄它的播放/打断层。**

---

## 附录 A 命令与证据留痕

```text
# 克隆（需走本机代理 127.0.0.1:7897；直连 GitHub 443 超时）
git -c http.proxy=http://127.0.0.1:7897 clone --depth 1 https://github.com/qishuilalala/dsh-voice-mode.git
git -c http.proxy=http://127.0.0.1:7897 clone --depth 1 https://github.com/PerryLink/dsh-talk.git

# HEAD
dsh-voice-mode  3524db40938f16e7c8f0726e30647afe5bfbef25  2026-10-31 03:33:42 +0800
dsh-talk        6a5f768b2607ec1204fc256ede492c2abd39a3e1  2026-11-01 17:23:47 +0800

# npm
dsh-voice-mode latest=0.7.34  time=2026-10-28T09:05:56.209Z  unpacked=1412065
dsh-talk       latest=0.3.26  time=2026-10-24T14:52:58.297Z  unpacked=1236668

# 体量
dsh-voice-mode/plugin/dsh-voice-mode/lib/: client.js 295.1KB, index.js 169.4KB,
       msedge-tts.cjs 737.2KB, sense-worker.mjs 3.4KB, tts-vits-worker.cjs 5.0KB
dsh-voice-mode 工作树 304 文件 / 13.95MB；dsh-talk 88 文件 / 0.75MB

# 依赖 unpacked（npm registry）
sherpa-onnx 1.13.6       15,101,190 B (~14.4MB)
sherpa-onnx-node 1.13.6      59,654 B  + 平台包 win-x64 21.9MB / linux-x64 31.2MB / darwin-arm64 31.8MB
                          （无 android 平台包）

# 用例
dsh-voice-mode: 78 test 文件 / 237 个 test()/it()
dsh-talk:       18 spec 文件 / 108 个 it()

# 本机 DSH 内置语音包（用于冲突判断）
app.asar/dsh/node_modules/@deepseek-ai/
  dsh-experimental-voice-input-bundle        0.2.1-rc.2   → cordis.patch.yml 插 4 行
  dsh-experimental-client-ui-voice-input     0.2.1-rc.2   → ctx.slots.inject("conversation.input.activity", …)
  dsh-experimental-speech-to-text-sensevoice 0.2.1-rc.2   → 依赖 sherpa-onnx-node 1.13.8
  dsh-experimental-speech-to-text            0.2.1-rc.2   → Service "speechToText" 注册表
app.asar/dsh/node_modules/@deepseek-ai/dsh-client-ui-slots  → 不存在（TALK 的 optional peer）
```

## 附录 B 关键文件地图

**dsh-voice-mode**（根 `plugin/dsh-voice-mode/`）
```
src/index.ts        2200 行  host：设置/HTTP 17 路由/SSE/llm 流 tap/回合状态
src/client.tsx      4409 行  client：麦克风按钮/状态条/字幕浮层/播放引擎/打断状态机
src/asr.ts          1744 行  client：AudioWorklet 采集/partial 上行/vadOnly 检测/AEC/预录环
src/asr-host.ts     1082 行  host：zipformer2 流式/Silero VAD/SenseVoice/段管理
src/models.ts        494 行  模型下载+SHA256+续传+镜像回退
src/tts-queue.ts     761 行  逐会话 TTS 队列/epoch 打断/重试/SSE 帧
src/tts-local.ts     619 行  VITS/Kokoro 子进程引擎
src/tts-runtime.ts    84 行  Electron 下为 Kokoro 找真 Node
src/segmenter.ts     194 行  分句 + markdown 剥离
src/audio-worklet.ts  74 行  64ms 分帧 + 16k 重采样
src/wakeword.ts      117 行  唤醒词容错匹配
src/mic-gesture.ts   573 行  长按/右键/触屏手势
src/security.ts       80 行  回环/同源/限流
test/                78 文件 / 237 用例
docs/adr/            12 个 ADR（AEC/VAD/传输/打断降级/情绪标签/让位语义…）
```

**dsh-talk**（根）
```
src/index.ts            143 行  host 插件体（service/tool/projection/3 事件）
src/service.ts          581 行  talk 服务（speak 管线/音频缓存/设置写回）
src/engine.ts           275 行  edge-tts/piper/whisper-cli/FunASR 命令行与 HTTP
src/client/TalkMicButton.tsx 486 行  麦克风按钮（Web Speech/MediaRecorder/播放）
src/client/vad.ts        43 行  RMS 静音检测
src/wire.ts             393 行  Typert 线协议 + 6 个 invocation 描述符
src/vocabulary.ts        86 行  语音事件载荷 + 投影 schema
src/speech.ts            58 行  dsh-talk/speech 会话事件（known-type 门控）
src/audio-export.ts     248 行  有界音频导出目录
tests/                   18 spec / 108 用例
```
