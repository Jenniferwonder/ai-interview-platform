# 11 · 实时语音链路（WebSocket + ASR/LLM/TTS）

> 对应源码：`modules/voiceinterview/`、DashScope SDK 集成、WebSocket Handler  
> 结束后的评估复用 [10 统一评估引擎](10-evaluation-engine.md)。

## 架构概览

```
浏览器 ──WebSocket──→ 服务端
  │                      ├── ASR（千问3 语音识别）
  │                      ├── VAD（服务端断句）
  │                      ├── LLM（流式生成回答）
  │                      └── TTS（千问3 语音合成）
  │                           ↓
  └────────── 音频流 ←───────┘
```

## 核心环节

### 1. WebSocket 全双工通信

- 为什么不用 HTTP：音频是持续流，HTTP 半双工 + 握手开销大
- `VoiceInterviewWebSocketHandler` 管理连接生命周期
- 心跳保活 + 超时自动暂停

### 2. ASR 语音识别（千问3）

- DashScope SDK 集成
- 实时流式识别，中间结果 → 字幕展示
- 最终结果 → 触发 LLM 回答

### 3. VAD 服务端断句

- 静音检测：`APP_VOICE_ASR_SILENCE_MS`（默认 1000ms）
- 句间合并：`APP_VOICE_USER_UTTERANCE_DEBOUNCE_MS`（默认 1600ms）
- 自动断句 + 手动提交双模式

### 4. LLM 流式生成 + 并发 TTS

```
LLM token 流 → 句子边界检测 → 拆为句子
                                ↓
                   每句独立调用 TTS（并发）
                                ↓
                   生成完毕 → 立即播放
```

- "边生成边合成边播放"：不等 LLM 全部生成完
- 句子级并发 TTS：前一句还在合成，后一句已开始
- 首包延迟 200ms

### 5. 回声防护

- AI 语音输出期间暂停收音
- 手动提交模式可选，避免误录

### 6. 开场白 TTS 启动预热

`VoiceInterviewWebSocketHandler` 在 `@PostConstruct` 中调用 `warmupOpeningAudioCache()`：读取 `voice-interview-opening.yml` 里的开场白模板，对每条调用 DashScope TTS，写入内存缓存 `openingAudioCache`，目标是用户进房时首句秒出。

**代价**：每次启动真实打云端 API，开发频繁重启会耗额度；日志中会出现一长串 `[TTS] Synthesis completed successfully`。

**开关**（默认关闭，适合本地开发）：

| 配置 | 环境变量 | 默认 |
|------|----------|------|
| `app.voice-interview.warmup-opening-audio-enabled` | `APP_VOICE_INTERVIEW_WARMUP_OPENING_AUDIO_ENABLED` | `false` |

- `false`：启动跳过预热，日志打印 `Opening audio cache warmup disabled`；首句走按需合成并仍可写入缓存。
- `true`：生产可开，降低首句语音延迟。

相关类：`VoiceInterviewProperties.warmupOpeningAudioEnabled`、`VoiceInterviewWebSocketHandler.warmupOpeningAudioCache`。

## 已知局限与改进方向

| 局限 | 改进方向 |
|------|----------|
| 端到端延迟偏高 | WebRTC 替代服务端音频中转 |
| 无耳机时回声泄漏 | 客户端 VAD 降噪 |
| TTS 音色单一 | 多音色支持 |
| 弱网音频断续 | 自适应码率 |

## 关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 通信协议 | WebSocket | 全双工、低延迟、浏览器原生支持 |
| 语音模型 | 千问3（ASR/TTS/LLM） | 统一 API Key，简化配置 |
| TTS 策略 | 句子级并发 | 减少首字延迟 |
| VAD 位置 | 服务端 | 客户端算力不确定 |
| 与文字面试的关系 | 共用评估引擎 | 评估结果可对比 |

## 核心要点

- **WebSocket vs WebRTC**：WebSocket 适合信令和通用数据传输，WebRTC 适合音视频直连（P2P 更低延迟），实际可按场景组合使用
- **为什么当前是级联而非端到端**：ASR → LLM → TTS 三级级联，每级可独立替换升级；端到端语音模型是演进方向但尚未成熟
- **VAD 为什么在服务端**：不依赖客户端浏览器兼容性和算力，行为可控、策略可统一调整
- **回声防护**：AI 输出期间暂停收音是最简单有效的策略，无需复杂的回声消除算法
