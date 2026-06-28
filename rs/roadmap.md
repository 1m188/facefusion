# FaceFusion Rust 移植 — 工作路线图

---

## 一、总体目标

### 1.1 项目定位

将 Python 版 [FaceFusion](https://github.com/facefusion/facefusion)（v3.6.1）完全移植到 Rust，最终实现一个**零配置、跨平台、单入口**的人脸操作工具。用户下载 zip 解压即用，无需安装 Python / Conda / pip / CUDA Toolkit / FFmpeg，所有依赖预打包、对用户透明。

### 1.2 核心交付物

| 产物 | 说明 |
|------|------|
| `facefusion` / `facefusion.exe` | 单可执行文件，CLI 入口，静态链接除 ONNX Runtime 外的一切 |
| `facefusion-offline-{platform}.zip` | 全量离线包：二进制 + 全部模型 + ONNX Runtime + FFmpeg，无网络可用 |
| `facefusion-online-{platform}.zip` | 轻量包：二进制 + ONNX Runtime + FFmpeg（~50MB），首次运行自动下载模型 |
| `facefusion-ui` | Web UI 服务（第二阶段交付），Rust 原生前端 + axum 后端 |

### 1.3 完整架构（全功能视图）

```
facefusion-rs/                             # Cargo workspace
├── Cargo.toml
│
├── crates/
│   ├── facefusion-core/                   # 核心库（平台无关）
│   │   ├── src/
│   │   │   ├── lib.rs                     # 公共导出
│   │   │   ├── state.rs                   # 全局状态管理
│   │   │   ├── config.rs                  # TOML 配置加载
│   │   │   ├── types.rs                   # 类型定义
│   │   │   ├── context.rs                 # AppContext（CLI / UI）
│   │   │   ├── process_state.rs           # 状态机
│   │   │   ├── memory.rs                  # 系统内存限制
│   │   │   ├── sanitizer.rs               # 输入清理/范围裁剪
│   │   │   ├── normalizer.rs              # 数据标准化
│   │   │   ├── filesystem.rs              # 文件操作/格式判断
│   │   │   ├── common.rs                  # 通用工具（OS检测/类型转换）
│   │   │   ├── time_helper.rs             # 时间工具（日期/耗时/格式化）
│   │   │   ├── temp_helper.rs             # 临时目录/文件管理
│   │   │   │
│   │   │   ├── face/                      # 人脸分析子系统
│   │   │   │   ├── detector.rs            # 4模型检测 + NMS
│   │   │   │   ├── landmarker.rs          # 68点关键点
│   │   │   │   ├── recognizer.rs          # ArcFace 嵌入
│   │   │   │   ├── classifier.rs          # 性别/年龄/种族
│   │   │   │   ├── analyser.rs            # Face 对象创建
│   │   │   │   ├── masker.rs              # 4种蒙版
│   │   │   │   ├── selector.rs            # 排序 + 过滤
│   │   │   │   ├── geometry.rs            # 仿射变换、人脸对齐
│   │   │   │   └── cache.rs               # 帧哈希缓存
│   │   │   │
│   │   │   ├── vision.rs                  # 图像 I/O、缩放、混合、色彩匹配
│   │   │   ├── ffmpeg.rs                  # FFmpeg 操作（提取/合并/音频）
│   │   │   ├── audio.rs                   # Mel 频谱生成
│   │   │   │
│   │   │   ├── inference/                 # ONNX 推理子系统
│   │   │   │   ├── pool.rs                # Session 池管理
│   │   │   │   ├── session.rs             # 单 Session 封装
│   │   │   │   ├── provider.rs            # 执行提供者
│   │   │   │   └── model_helper.rs        # 模型元数据提取（分辨率/步长/锚点）
│   │   │   │
│   │   │   ├── download.rs                # 模型下载 + 校验
│   │   │   ├── hash.rs                    # CRC32 哈希
│   │   │   ├── voice_extractor.rs         # 声音分离
│   │   │   │
│   │   │   ├── processors/                # 可插拔处理器系统
│   │   │   │   ├── mod.rs                 # Processor trait + 模块加载器
│   │   │   │   ├── pixel_boost.rs         # 高分辨分块（公共工具）
│   │   │   │   ├── live_portrait.rs       # LivePortrait 公共工具
│   │   │   │   ├── swapper.rs             # 换脸
│   │   │   │   ├── enhancer.rs            # 人脸增强
│   │   │   │   ├── editor.rs              # 人脸编辑
│   │   │   │   ├── expression_restorer.rs # 表情还原
│   │   │   │   ├── age_modifier.rs        # 年龄修改
│   │   │   │   ├── background_remover.rs  # 背景移除
│   │   │   │   ├── frame_enhancer.rs      # 全帧超分
│   │   │   │   ├── frame_colorizer.rs     # 黑白上色
│   │   │   │   ├── lip_syncer.rs          # 唇形同步
│   │   │   │   ├── deep_swapper.rs        # 预训练艺人换脸
│   │   │   │   └── debugger.rs            # 调试可视化
│   │   │   │
│   │   │   ├── workflow/                  # 处理工作流
│   │   │   │   ├── mod.rs
│   │   │   │   ├── image_to_image.rs      # 单图
│   │   │   │   └── image_to_video.rs      # 视频
│   │   │   │
│   │   │   ├── jobs/                      # 作业系统
│   │   │   │   ├── manager.rs             # CRUD + 状态目录
│   │   │   │   ├── runner.rs              # 执行引擎 + 重试
│   │   │   │   ├── store.rs               # 参数注册
│   │   │   │   └── helper.rs              # 工具函数
│   │   │   │
│   │   │   ├── benchmark.rs               # 性能基准测试
│   │   │   │
│   │   │   └── uis/                       # Web UI 子系统
│   │   │       ├── mod.rs
│   │   │       ├── server.rs              # HTTP 服务启动
│   │   │       ├── layout/                # 页面布局
│   │   │       │   ├── default.rs
│   │   │       │   ├── benchmark.rs
│   │   │       │   ├── jobs.rs
│   │   │       │   └── webcam.rs
│   │   │       └── component/             # 可复用 UI 组件
│   │   │           ├── source.rs
│   │   │           ├── target.rs
│   │   │           ├── output.rs
│   │   │           ├── preview.rs
│   │   │           ├── about.rs
│   │   │           ├── face_detector.rs
│   │   │           ├── face_landmarker.rs
│   │   │           ├── face_masker.rs
│   │   │           ├── face_selector.rs
│   │   │           ├── processors.rs
│   │   │           ├── download.rs
│   │   │           ├── webcam.rs
│   │   │           └── ...
│   │   └── Cargo.toml
│   │
│   ├── facefusion-models/                 # 模型清单 + 自动下载
│   │   ├── src/lib.rs
│   │   └── models.toml
│   │
│   └── facefusion-bundle/                 # 离线包构建工具
│       ├── src/main.rs
│       └── Cargo.toml
│
├── src/
│   ├── main.rs                            # CLI 入口（clap derive）
│   └── cli/
│       ├── mod.rs
│       ├── args.rs                        # 参数定义
│       ├── router.rs                      # 命令路由
│       ├── table.rs                       # 表格渲染
│       └── locale.rs                      # i18n
│
└── assets/
    └── facefusion.toml                    # 默认配置
```

### 1.4 完整层级关系

```
┌──────────────────────────────────────────────────────┐
│  main.rs (clap derive, CLI entry)                   │  ← 二进制入口层
├──────────────────────────────────────────────────────┤
│  cli/ (args, routing, table, locale)                │
├──────────────────────────────────────────────────────┤
│  facefusion-core/                                    │  ← 核心库层
│                                                      │
│  ┌─ workflow/         工作流编排                      │
│  ├─ processors/       11个处理器（可插拔）            │
│  ├─ face/             人脸分析管线（检测→识别→分类）  │
│  ├─ inference/        ONNX 推理池                    │
│  ├─ jobs/             作业管理（草稿→队列→执行→重试） │
│  ├─ uis/              Web UI 子系统                  │
│  ├─ benchmark.rs      性能基准                       │
│  ├─ download.rs       模型管理                       │
│  ├─ vision.rs         视觉处理基础                    │
│  ├─ ffmpeg.rs         FFmpeg 桥接                    │
│  ├─ audio.rs          音频处理                       │
│  └─ voice_extractor   声音分离                       │
├──────────────────────────────────────────────────────┤
│  facefusion-models/    模型清单 + 自下载             │  ← 模型清单层
└──────────────────────────────────────────────────────┘
```

### 1.5 核心接口设计

#### Processor Trait（所有处理器的统一契约）

```rust
#[async_trait]
pub trait Processor: Send + Sync {
    /// 处理器标识名，如 "face_swapper"
    fn name(&self) -> &'static str;

    /// 注册 CLI 参数到 clap App
    fn register_args(&self, cmd: Command) -> Command;

    /// 将解析后的 CLI 参数应用到处理器状态
    fn apply_args(&mut self, args: &ArgMatches);

    /// 预检查：模型是否下载、hash 是否匹配、环境是否就绪
    fn pre_check(&self) -> Result<()>;

    /// 预处理：线程池外的初始化（创建 ONNX Session、分配缓冲区）
    async fn pre_process(&mut self) -> Result<()>;

    /// 核心方法：处理单帧 FrameDataBag
    async fn process_frame(&self, receive: FrameDataBag) -> Result<FrameDataBag>;

    /// 后处理：管线结束后释放资源
    async fn post_process(&mut self) -> Result<()>;

    /// 返回推理会话池，供统一的池管理
    fn inference_pool(&self) -> &InferencePool;
}
```

#### FrameDataBag（管线流转数据结构）

```rust
pub struct FrameDataBag {
    pub target_frame:    Option<VisionFrame>,  // 目标帧 (H, W, C), u8
    pub temp_frame:      Option<VisionFrame>,  // 当前处理中间帧
    pub source_frame:    Option<VisionFrame>,  // 源帧（参考人脸）
    pub reference_frame: Option<VisionFrame>,  // 参考帧（人脸选择匹配用）
    pub audio_frame:     Option<AudioFrame>,   // mel 频谱 (mel_bins, time_steps)
    pub voice_frame:     Option<AudioFrame>,   // 提取后的人声音频频谱
    pub source_faces:    Vec<Face>,            // 源帧检测到的人脸
    pub target_faces:    Vec<Face>,            // 目标帧检测到的人脸
    pub debug_frame:     Option<VisionFrame>,  // 调试叠加层
    pub masks:           HashMap<MaskType, Mask>, // 生成的蒙版集合
}
```

#### Face（人脸数据对象）

```rust
pub struct Face {
    pub bounding_box:  BoundingBox,        // [x1, y1, x2, y2]
    pub score_set:     FaceScoreSet,       // {detector, landmarker} 置信度
    pub landmark_set:  FaceLandmarkSet,    // {5点, 68点}
    pub angle:         f32,                // 人脸朝向 0/90/180/270
    pub embedding:     Embedding,          // 512维 ArcFace 原始嵌入
    pub embedding_norm: Embedding,          // 512维 L2 归一化嵌入
    pub gender:        u8,                 // 0=male, 1=female
    pub age:           u8,                 // 0..8 年龄段
    pub race:          u8,                 // 0..6 种族
}
```

#### VisionFrame / AudioFrame / Embedding

```rust
pub type VisionFrame = Array3<u8>;           // ndarray (H, W, C)
pub type AudioFrame  = Array2<f32>;          // ndarray (mel_bins, time_steps)
pub type Embedding    = Array1<f32>;         // ndarray 512
pub type Mask         = Array2<f32>;         // ndarray (H, W)，值 [0.0, 1.0]
pub type BoundingBox  = [f32; 4];
```

#### InferencePool（ONNX 会话池）

```rust
pub struct InferencePool {
    sessions:   HashMap<String, Arc<Mutex<Session>>>,
    provider:   ExecutionProvider,
    device_id:  usize,
}

pub struct Session {
    inner:          ort::Session,
    input_buffer:   Array4<f32>,
    output_buffer:  Array4<f32>,
}
```

#### Workflow Trait（工作流契约）

```rust
#[async_trait]
pub trait Workflow {
    /// 初始化（创建临时目录、复制源文件等）
    async fn setup(&self, state: &State) -> Result<()>;

    /// 运行处理管线
    async fn run(&self, processors: &[Box<dyn Processor>]) -> Result<()>;

    /// 收尾（合并视频、恢复音频、清理临时文件）
    async fn finalize(&self, state: &State) -> Result<()>;
}

// 两种工作流
pub struct ImageToImage;
pub struct ImageToVideo;
```

### 1.6 完整数据流（多处理器链式调用）

以同时启用 `face_swapper` + `face_enhancer` 处理一段视频为例：

```
视频输入
  │
  ▼
FFmpeg::extract_frames()   → 帧序列 (frame_0001.png ... frame_0240.png)
  │
  ▼
┌─ for each frame ─────────────────────────────────────────────┐
│                                                              │
│  FrameDataBag { target_frame, temp: None, source_faces: [], │
│                 target_faces: [], masks: {} }                │
│    │                                                         │
│    │  (一次性预处理：face_analyser 批量检测源帧人脸)           │
│    │  face_analyser(source_frame)                            │
│    │    ├─ detector::detect()        → bbox + score          │
│    │    ├─ landmarker::detect()      → 68                    │
│    │    ├─ recognizer::calculate()   → embedding             │
│    │    └─ classifier::classify()    → gender/age/race       │
│    │  → bag.source_faces                                        │
│    │                                                         │
│    ├─ 1. face_analyser(target_frame)                         │
│    │    └─ (同上) → bag.target_faces                         │
│    │                                                         │
│    ├─ 2. face_selector::select()                             │
│    │    ├─ sort_and_filter(target_faces)                     │
│    │    └─ find_match(reference, target) → 匹配对            │
│    │                                                         │
│    ├─ 3. face_masker::create()                               │
│    │    ├─ create_box_mask()      → bag.masks["box"]         │
│    │    └─ create_area_mask()     → bag.masks["area"]        │
│    │                                                         │
│    ├─ 4. face_swapper::process_frame(bag)                    │
│    │    ├─ warp_face_by_landmark_5()  → 对齐人脸              │
│    │    ├─ pixel_boost tile()         → 分块（可选）         │
│    │    ├─ onnx_session.run()         → 推理换脸             │
│    │    ├─ paste_back()               → 贴回目标帧            │
│    │    └─ bag.temp_frame ← 更新                            │
│    │                                                         │
│    ├─ 5. face_enhancer::process_frame(bag)                   │
│    │    ├─ warp_face_by_bounding_box() → 截取人脸区域         │
│    │    ├─ onnx_session.run()          → 超分/去噪           │
│    │    ├─ blend(temp, enhanced, mask) → 混合回 temp_frame   │
│    │    └─ bag.temp_frame ← 更新                            │
│    │                                                         │
│    ├─ 6. (处理器链结束)                                        │
│    │    conditional_match_frame_color(source, temp)          │
│    │    → write temp_frame to disk                           │
│    │                                                         │
│    └─ advance to next frame                                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
  │
  ▼
FFmpeg::merge_video()     → temp_*.png → output.mp4
FFmpeg::restore_audio()   → output.mp4 ← original audio
  │
  ▼
清理临时文件 → 输出完成
```

### 1.7 错误处理

全局统一错误类型，`thiserror` 派生，禁止 `panic!` / `unwrap()`，所有错误通过 `?` 逐层传播至 `main.rs` 统一 `eprintln` 输出。

```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    // ── 推理 ──
    #[error("ONNX session error: {0}")]
    Ort(#[from] ort::Error),
    #[error("Model not downloaded: {name}")]
    ModelNotFound { name: String },
    #[error("Inference failed for {model}: {reason}")]
    InferenceFailed { model: String, reason: String },

    // ── 文件 & 下载 ──
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
    #[error("Download failed for {url}: HTTP {status}")]
    DownloadFailed { url: String, status: u16 },
    #[error("Hash mismatch: {path} (expected {expected}, got {actual})")]
    HashMismatch { path: String, expected: String, actual: String },

    // ── FFmpeg ──
    #[error("FFmpeg not found in bundle or system PATH")]
    FfmpegNotFound,
    #[error("FFmpeg exited code {code}: {stderr}")]
    FfmpegFailed { code: i32, stderr: String },

    // ── 图像 ──
    #[error("Image decode failed: {0}")]
    ImageDecode(#[from] image::ImageError),
    #[error("No face detected in {side} image")]
    NoFaceDetected { side: &'static str },
    #[error("Frame dimension mismatch: got {0}x{1}x{2}, expected {3}x{4}x{5}")]
    DimensionMismatch(usize, usize, usize, usize, usize, usize),

    // ── 配置 ──
    #[error("Config parse error: {0}")]
    Config(#[from] toml::de::Error),
    #[error("Invalid arg '{arg}': {value} not in [{min}, {max}]")]
    ArgOutOfRange { arg: String, value: String, min: f64, max: f64 },

    // ── 通用 ──
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

### 1.8 日志

`tracing` + `tracing-subscriber` 替代 Python `logging`。格式保持 `[模块] 消息`：

```
[face_swapper] Loading inswapper_128.onnx
[workflow] Rendering frame 42 / 240
[download] inswapper_128.onnx: 100% ██████████ 534MB/534MB
```

### 1.9 文件路径 & 可移植性

**默认行为（便携）**：所有数据存放在可执行文件所在文件夹下，解压即用，搬走即清。不与系统目录交互。

| 路径 | 默认（便携）解析 |
|------|-----------------|
| 应用根目录 | `std::env::current_exe().parent().unwrap()` |
| 运行时依赖 (onnxruntime, ffmpeg) | `{app_dir}/runtime/` |
| 模型缓存 | `{app_dir}/models/` |
| 配置文件 | `{app_dir}/facefusion.toml` |
| 临时目录 | `{app_dir}/temp/` |
| 作业目录 | `{app_dir}/jobs/` |

**可选行为（系统目录）**：通过配置文件 `[paths] system_dirs = true` 或环境变量 `FACEFUSION_SYSTEM_DIRS=1` 切换，数据存放于系统标准目录，供多版本、多用户共享模型缓存。

| 路径 | 系统目录解析 |
|------|-------------|
| 模型缓存 | `{dirs::cache_dir()}/facefusion/models/` |
| 配置文件 | `{dirs::config_dir()}/facefusion/facefusion.toml` |
| 临时目录 | `{dirs::cache_dir()}/facefusion/temp/` |
| 作业目录 | `{dirs::state_dir()}/facefusion/jobs/` |

### 1.10 关键依赖

| 用途 | Rust Crate | 原 Python 对应 |
|------|-----------|---------------|
| ONNX 推理 | `ort` | `onnxruntime` |
| 图像处理 | `image` + `imageproc` + 部分手写 | `opencv-python` |
| 科学计算 | `ndarray` + `ndarray-stats` | `numpy` + `scipy` |
| CLI | `clap` (derive) | `argparse` |
| 配置 | `serde` + `toml` | `configparser` |
| 日志 | `tracing` + `tracing-subscriber` | `logging` |
| HTTP 下载 | `reqwest` + `indicatif` | `curl` + `tqdm` |
| 哈希 | `crc32fast` | `zlib.crc32` |
| JSON | `serde_json` | `json` |
| 音频 I/O | `hound` | `soundfile` |
| 错误 | `thiserror` + `anyhow` | Exception |
| 异步 | `tokio` | asyncio |
| 并行 | `rayon` | `concurrent.futures` |
| 路径 | `dirs` | `appdirs` |
| FFmpeg | `std::process::Command` | `subprocess` |
| Web 服务 | `axum` + `tower-http` | Gradio |
| 前端框架 | `dioxus` 或 `leptos` | — |
| 摄像头 | `nokhwa` | `cv2.VideoCapture` |
| UUID | `uuid` | `uuid` |

### 1.11 与 Python 版的差异总览

| 维度 | Python（当前） | Rust（目标） |
|------|---------------|-------------|
| 运行环境 | Python 3.10+ / Conda | 无需 |
| 包管理 | pip install N个包 | Cargo 编译时解决 |
| CUDA | 用户配 CUDA Toolkit | CPU 模式无需；GPU 需 CUDA 系统库 |
| FFmpeg | 用户手动安装 | 捆绑分发 |
| 模型 | 用户手动下载 | 首次运行自动 / 离线包预装 |
| ONNX Runtime | pip 安装 | 捆绑 .dll/.so/.dylib |
| 跨平台 | 源码级（每台配环境） | 发行级（CI 多平台构建） |
| 二进制 | N/A | ~30MB / ~2GB（离线全量包） |

---

## 二、第一阶段：核心功能 CLI

### 2.1 目标

产出可用的 CLI 工具，支持 `face_swapper` + `face_enhancer` 两个处理器，全功能命令行参数，单图/视频处理，解压即用。

本阶段的**交付物**也是后续所有阶段的**基础骨架**——Cargo workspace 结构、核心 trait、推理基础设施、视觉/FFmpeg 基础模块一经建立，后续全部处理器和工作流均在此骨架之上添加。

### 2.2 第一阶段架构（子集视图，相对总体架构的覆盖范围）

```
facefusion-rs/
├── crates/
│   ├── facefusion-core/
│   │   └── src/
│   │       ├── config.rs              ✓
│   │       ├── state.rs               ✓
│   │       ├── types.rs               ✓
│   │       ├── context.rs             ✓
│   │       ├── process_state.rs       ✓
│   │       ├── memory.rs              ✓
│   │       ├── sanitizer.rs           ✓
│   │       ├── normalizer.rs          ✓
│   │       ├── filesystem.rs          ✓
│   │       ├── common.rs              ✓
│   │       ├── time_helper.rs         ✓
│   │       ├── temp_helper.rs         ✓
│   │       ├── face/
│   │       │   ├── detector.rs        ✓
│   │       │   ├── landmarker.rs      ✓
│   │       │   ├── recognizer.rs      ✓
│   │       │   ├── classifier.rs      ✓
│   │       │   ├── analyser.rs        ✓
│   │       │   ├── masker.rs          ✓  (area mask)
│   │       │   ├── selector.rs        ✓  (many + left-right + filter)
│   │       │   ├── geometry.rs        ✓
│   │       │   └── cache.rs           ✓
│   │       ├── vision.rs              ✓
│   │       ├── ffmpeg.rs              ✓
│   │       ├── audio.rs               （桩）
│   │       ├── inference/             ✓
│   │       │   ├── pool.rs            ✓
│   │       │   ├── session.rs         ✓
│   │       │   ├── provider.rs        ✓
│   │       │   └── model_helper.rs    ✓
│   │       ├── download.rs            ✓
│   │       ├── hash.rs                ✓
│   │       ├── voice_extractor.rs     （桩：无操作）
│   │       ├── processors/
│   │       │   ├── mod.rs             ✓  (Processor trait)
│   │       │   ├── swapper.rs         ✓
│   │       │   ├── enhancer.rs        ✓
│   │       │   └── pixel_boost.rs     ✓
│   │       ├── workflow/
│   │       │   ├── mod.rs             ✓  (Workflow trait)
│   │       │   ├── image_to_image.rs  ✓
│   │       │   └── image_to_video.rs  ✓
│   │       ├── jobs/                  未实现（第四阶段）
│   │       ├── uis/                   未实现（第二阶段）
│   │       └── benchmark.rs           未实现（第四阶段）
│   ├── facefusion-models/             ✓
│   └── facefusion-bundle/             ✓
├── src/
│   ├── main.rs                        ✓
│   └── cli/
│       ├── args.rs                    ✓
│       ├── router.rs                  ✓
│       ├── table.rs                   ✓
│       └── locale.rs                  ✓  (仅 en)
└── assets/
    └── facefusion.toml                ✓
```

### 2.3 分步实施方案

#### 1.0 — 项目骨架
- [ ] `Cargo.toml` workspace 初始化，`facefusion-core` / `facefusion-models` / `facefusion-bundle` 三个 crate
- [ ] `facefusion-core/src/lib.rs`：空模块树，所有 trait 定义占位
- [ ] `facefusion-models/src/lib.rs` + `models.toml`：模型 URL / hash / 尺寸 清单
- [ ] `src/main.rs`：clap App，仅打印版本号 `--version`
- [ ] CI：GitHub Actions 跨平台编译矩阵 `windows-latest / macos-latest / ubuntu-latest`
- [ ] CI 中加入 `cargo clippy -- -D warnings` + `cargo fmt --check`

#### 1.1 — 配置 & 类型 & 基础工具
- [ ] `config.rs`：TOML 加载/保存，`State` 结构体（serde）
- [ ] `types.rs`：`Face` / `BoundingBox` / `FaceScoreSet` / `FaceLandmarkSet` / `Embedding` / `VisionFrame` / `AudioFrame` / `FrameDataBag` / `Mask`
- [ ] `context.rs`：`AppContext` 枚举（CLI / UI）
- [ ] `process_state.rs`：状态机（`Checking → Processing → Stopping → Pending`）
- [ ] `state.rs`：`StateManager` — 全局状态读写，CLI/UI 双上下文隔离
- [ ] `memory.rs`：系统内存限制（Windows SetProcessWorkingSetSize / Unix RLIMIT_DATA）
- [ ] `sanitizer.rs`：输入清理、整数/浮点范围裁剪
- [ ] `normalizer.rs`：数据标准化（颜色通道、FPS、分辨率约束）
- [ ] `filesystem.rs`：文件操作封装（格式判断 `is_audio/is_image/is_video` 等）
- [ ] `common.rs`：通用工具（平台检测 `is_linux/is_macos/is_windows`、类型转换）
- [ ] `time_helper.rs`：时间工具（当前日期时间、耗时计算、时间差拆分、"X 前"人性化描述）
- [ ] `temp_helper.rs`：临时目录创建/清理、临时帧路径模式解析

#### 1.2 — 推理基础设施
- [ ] `inference/provider.rs`：ONNX Runtime 初始化，provider 选择（CPU 默认，GPU 通过 `--execution-provider cuda` 启用）
- [ ] `inference/session.rs`：单个 `Session` 创建，预分配 `input_buffer` / `output_buffer`
- [ ] `inference/pool.rs`：`InferencePool` — `HashMap` 池，按需创建 + 复用
- [ ] `inference/model_helper.rs`：ONNX 模型元数据提取（输入分辨率/步长/锚点），供 detector 使用
- [ ] `download.rs`：`reqwest` 下载器，支持断点续传、进度条（`indicatif`）、多源 fallback（GitHub → HuggingFace）
- [ ] `hash.rs`：`CRC32` 校验
- [ ] 集成测试：下载 `inswapper_128.onnx`，创建 Session，跑一次前向

#### 1.3 — 视觉处理基础
- [ ] `vision.rs`：
  - 图像 I/O：`image` crate 读写 PNG / JPEG / WebP
  - resize、normalize、HWC ↔ CHW 转换、u8 ↔ f32 转换
  - warp_affine、blend（alpha 混合）
  - 色彩匹配（直方图均衡化 + 颜色迁移）
  - 高斯模糊（用于边缘羽化）
- [ ] 单元测试：已知输入验证输出像素值

#### 1.4 — FFmpeg 集成
- [ ] `ffmpeg.rs`：
  - `extract_frames(src, resolution, fps, trim)` → 帧序列 PNG 到临时目录
  - `merge_video(frames_dir, fps, resolution, output)` → 输出视频
  - `restore_audio(src, output)` → 从源视频恢复音频
  - `replace_audio(src, audio_path, output)` → 替换音轨
  - `detect_fps()` / `detect_resolution()` / `count_frames()`
- [ ] 便携 FFmpeg 查找：优先 `{app_dir}/runtime/ffmpeg` → 系统 PATH
- [ ] 集成测试：解压一段视频再合并，逐帧像素对比

#### 1.5 — 人脸分析管线
- [ ] `face/detector.rs`：
  - `detect_faces(frame)` → `Vec<(Bbox, Score)>`
  - 模型：`retinaface_10g` / `scrfd_2.5g`（第一阶段）；`yolo_face` / `yunet`（第三阶段补充）
  - 旋转感知：`detect_faces_by_angle(frame, 0/90/180/270)`
  - `apply_nms(bboxes, scores, iou_thresh)` — NMS 后处理
  - `ModelInitializer` 提取 ONNX 模型元数据（分辨率/步长/锚点）
- [ ] `face/landmarker.rs`：
  - `detect_landmarks(frame, bbox)` → 68 点
  - 模型：`2dfan4`
- [ ] `face/recognizer.rs`：
  - `calculate_embedding(frame, landmarks_5)` → `(Embedding, EmbeddingNorm)`
  - 模型：`arcface_w600k_r50`
- [ ] `face/classifier.rs`：
  - `classify(frame, landmarks_5)` → `(Gender, AgeGroup, Race)`
  - 模型：`fairface`
- [ ] `face/analyser.rs`：
  - `create_faces(frame, bboxes, scores, landmarks_5)` → `Vec<Face>`
  - 整合检测→关键点→嵌入→分类全流程
  - `get_one_face()` / `get_average_face()` / `get_many_faces()`
- [ ] `face/geometry.rs`：
  - `estimate_matrix_by_landmark_5(landmarks, template, size)` → 仿射矩阵
  - `warp_face_by_landmark_5(frame, landmarks, template, size)` → 对齐人脸
  - `paste_back(frame, crop, mask, matrix)` → 贴回
  - `convert_5_to_68()` / `convert_68_to_5()` — 关键点格式转换
  - `estimate_face_angle(landmarks_68)` → 角度
  - 预定义 `WARP_TEMPLATE_SET`（7 种模板）
- [ ] `face/cache.rs`：按帧字节哈希缓存 `Vec<Face>`
- [ ] 集成测试：用已知图片验证人脸检测/嵌入/关键点/分类结果

#### 1.6 — 处理器：face_swapper
- [ ] `processors/mod.rs`：`Processor` trait 定义 + `load_processor(name)` 动态加载
- [ ] `processors/swapper.rs`：`FaceSwapper` 实现 `Processor` trait
  - 模型：`inswapper_128` / `blends swap_256`
  - `weight` 参数控制混合比例
  - pixel_boost 高分辨分块支持（`processors/pixel_boost.rs`）
- [ ] `face/masker.rs`：`area_mask`（基于 landmark 的凸包蒙版）
- [ ] `face/selector.rs`：`many` 模式 + `left-right` 排序 + 性别/年龄/种族过滤
- [ ] 集成测试：输入一对图片，验证换脸输出

#### 1.7 — 处理器：face_enhancer
- [ ] `processors/enhancer.rs`：`FaceEnhancer` 实现 `Processor` trait
  - 模型：`codeformer` / `gfpgan_1.2` / `gfpgan_1.3` / `gfpgan_1.4`
  - `blend` 参数控制修复强度
- [ ] 集成测试：输入低质量人脸图片，验证增强输出

#### 1.8 — 工作流
- [ ] `workflow/mod.rs`：`Workflow` trait + `conditional_process(state, processors)` 工厂函数
- [ ] `workflow/image_to_image.rs`：单图处理
  - `setup()` → 复制目标图片到临时目录
  - `run()` → 逐处理器调用 `process_frame()`
  - `finalize()` → 导出输出图片，清理临时文件
- [ ] `workflow/image_to_video.rs`：视频逐帧处理
  - `setup()` → 创建临时目录
  - `run()` → FFmpeg 提取帧 + rayon 多线程处理 + FFmpeg 合并
  - `finalize()` → 恢复音频 + 清理临时文件
  - 进度回调 + 处理状态机（支持 `Ctrl+C` 中断）
- [ ] 集成测试：跑通图片→图片和视频→视频的完整管线

#### 1.9 — CLI
- [ ] `cli/args.rs`：clap derive 构建完整命令行：
  - 子命令：`run` / `headless-run` / `download` / `version`
  - `--source` / `--target` / `--output` 路径
  - `--processors` 选择（face_swapper / face_enhancer）
  - `--face-detector-model` / `--face-detector-size` / `--face-detector-score`
  - `--face-landmarker-model` / `--face-landmarker-score`
  - `--face-selector-mode` / `--face-selector-order` / `--face-selector-gender` / `--face-selector-age-range`
  - `--face-mask-types` / `--face-mask-blur` / `--face-mask-padding`（仅 "area" 有效）
  - `--face-swapper-model` / `--face-enhancer-model`
  - `--output-image-quality` / `--output-video-quality` / `--output-video-encoder` / `--output-video-fps`
  - `--execution-provider` / `--execution-device-id` / `--execution-thread-count`
  - `--trim-frame-start` / `--trim-frame-end` / `--temp-frame-format`
  - `--log-level` / `--config-path` / `--temp-path`
- [ ] `cli/router.rs`：命令分发
- [ ] `cli/table.rs`：表格渲染（设备列表、模型列表等）
- [ ] `cli/locale.rs`：i18n 字符串（初始仅 en）
- [ ] 端到端测试：CLI 完整命令换脸 + 增强

#### 1.10 — 打包 & 分发包构建
- [ ] `facefusion-bundle/src/main.rs`：
  - 参数：`--platform` / `--mode online|offline`
  - 下载 ONNX Runtime 对应平台的动态库
  - 下载 FFmpeg 对应平台二进制
  - 离线模式：下载全部模型文件 → `hash` 校验
  - 在线模式：仅打包 `models.toml`（运行时按需下载）
  - 产出 zip：`facefusion-{mode}-{platform}.zip`
- [ ] CI 产物自动上传到 GitHub Releases
- [ ] 三平台产物验证：解压后执行 `facefusion --version` + 简单换脸测试

### 2.4 第一阶段功能清单

| 功能 | 说明 |
|------|------|
| 换脸 (face_swapper) | ins_swapper_128 / blendswap_256，含 pixel_boost 高分分块 |
| 人脸增强 (face_enhancer) | codeformer / gfpgan_1.2~1.4 |
| 图片输入/输出 | PNG / JPEG / WebP |
| 视频输入/输出 | 含原始音频保留 |
| 人脸检测 | retinaface / scrfd，含旋转感知 |
| 68点关键点 | 2dfan4 |
| 人脸嵌入 | arcface 512维 |
| 人脸分类 | 性别 / 年龄 / 种族 |
| 人脸选择 | many 模式 + left-right 排序 + 性别/年龄/种族过滤 |
| 人脸蒙版 | area mask（landmark 凸包） |
| CLI | 全参数 run / headless-run / download |
| 模型下载 | 首次运行自动下载，进度条 |
| 离线包 | facefusion-bundle 构建全量 zip |
| 跨平台 | Win / macOS / Linux，GitHub Actions CI 构建 |
| 便携默认 | 默认所有数据在 exe 目录下，可选切换系统目录 |

---

## 三、第二阶段：Web UI

### 3.1 目标

在第一阶段 CLI 基础上增加完整的 Web 界面，用户通过浏览器完成全部操作，体验对齐 Python 版 Gradio UI。功能范围仍为第一阶段的 `face_swapper` + `face_enhancer` 两个处理器。

### 3.2 Web UI 设计

#### 技术选型

- **前端框架**：Dioxus（全栈 Rust，SSR + 客户端 Hydration）或 Leptos（细粒度响应式），无需 JS 前端分离
- **通信方式**：HTTP Server-Sent Events（SSE）实现处理进度实时推送；WebSocket 用于 Webcam 实时帧流
- **静态资源**：CSS / 前端 WASM 通过 `include_bytes!` 编译进二进制

#### UI 布局

4 个页面布局。JOBS 和 BENCHMARK 在第四阶段实现前隐藏，Phase 2 仅展示 DEFAULT 和 WEBCAM 两个 tab。

```
┌──────────────────────────────────────────────────────┐
│  FaceFusion  [DEFAULT]  [WEBCAM]                    │
├──────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌─────────────────────┐ │
│  │ SOURCE   │  │ TARGET   │  │ PREVIEW / OUTPUT    │ │
│  │ 上传/拖放 │  │ 上传/拖放 │  │ 实时预览 + 下载     │ │
│  │          │  │          │  │                     │ │
│  └──────────┘  └──────────┘  └─────────────────────┘ │
│                                                      │
│  ┌─ FACE DETECTOR ─────────────────────────────────┐ │
│  │ Model: [retinaface v] Size: [640 v] Score: [0.5] │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌─ FACE SELECTOR ─────────────────────────────────┐ │
│  │ Mode: [many v] Order: [left-right v]            │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌─ FACE SWAPPER ──────────────────────────────────┐ │
│  │ Model: [inswapper_128 v] Weight: [100]           │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌─ FACE ENHANCER ─────────────────────────────────┐ │
│  │ Model: [codeformer v] Blend: [80]                │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌─ OUTPUT ────────────────────────────────────────┐ │
│  │ Encoder: [h264 v] Quality: [80] FPS: [30]        │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  [START] [STOP] [CLEAR]                             │
│  ████████████░░░░░░░░  58%  ETA 01:23               │
└──────────────────────────────────────────────────────┘
```

#### Webcam 布局

Phase 2 采用浏览器端方案：调用 `getUserMedia()` 获取摄像头流，帧通过 WebSocket 发送到后端处理，结果实时回传渲染到 `<canvas>`。

> **与第四阶段的关系**：Phase 2 的浏览器端 Webcam 用于 Web UI 中实时预览，是用户通过浏览器操作的场景。第四阶段新增后端直采（`nokhwa` crate）用于 CLI 场景（`facefusion webcam` 命令无浏览器运行）。两者是**场景互补**，不是替代关系。

### 3.3 `uis/` 模块结构

```
facefusion-core/src/uis/
├── mod.rs                  # UIServer: axum 启动, SSE/WS 路由注册
├── server.rs               # HTTP 服务: listen + graceful shutdown
├── layout/
│   ├── default.rs          # 主处理布局
│   ├── webcam.rs           # 实时摄像头布局
│   ├── jobs.rs             # 作业列表布局（功能桩）
│   └── benchmark.rs        # 基准测试布局（功能桩）
└── component/
    ├── source.rs           # 源文件上传组件
    ├── target.rs           # 目标文件上传组件
    ├── output.rs           # 输出设置组件
    ├── preview.rs          # 实时预览（SSE 图片流）
    ├── about.rs            # 关于页（版本/许可/作者）
    ├── face_detector.rs    # 检测器参数组件
    ├── face_landmarker.rs  # 关键点模型选项组件
    ├── face_selector.rs    # 人脸选择参数组件
    ├── face_masker.rs      # 蒙版参数组件
    ├── face_swapper.rs     # 换脸参数组件
    ├── face_enhancer.rs    # 增强参数组件
    ├── processors.rs       # 处理器选择
    ├── download.rs         # 模型下载面板
    ├── webcam.rs           # Webcam 控制 + Canvas
    └── webcam_options.rs   # Webcam 参数
```

### 3.4 后端 API 设计

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/process` | POST | 接收源/目标文件 + 参数 JSON，启动处理，返回 task_id |
| `/api/progress/{task_id}` | GET (SSE) | 实时推送处理进度 `{frame, total, eta}` |
| `/api/preview/{task_id}` | GET (SSE) | 处理中逐帧预览图片 |
| `/api/result/{task_id}` | GET | 完成后下载输出文件 |
| `/api/stop/{task_id}` | POST | 中断正在执行的处理 |
| `/api/status` | GET | 获取可用设备、已下载模型等运行时信息 |
| `/ws/webcam` | WebSocket | 摄像头帧收发（双向 binary） |
| `/` | GET | 主页面（Dioxus/Leptos SSR 渲染） |

### 3.5 第二阶段分步方案

#### 2.0 — HTTP 服务骨架
- [ ] `axum` 集成：HTTP Server 启动、静态文件服务、Graceful Shutdown
- [ ] SSE 基础设施：`tokio::sync::broadcast` 推送通道
- [ ] WebSocket 基础设施：`axum::extract::ws`

#### 2.1 — 核心 API
- [ ] `POST /api/process`：接收 multipart 上传 + JSON 参数，转为 `State`，调用 `workflow::run()`
- [ ] `GET /api/progress/{id}`：SSE 推送帧进度（处理线程通过 channel 上报）
- [ ] `GET /api/preview/{id}`：SSE 逐帧推送处理后的临时图片
- [ ] `GET /api/result/{id}`：返回最终输出文件
- [ ] `POST /api/stop/{id}`：设置 `ProcessState::Stopping`

#### 2.2 — 前端渲染
- [ ] Dioxus/Leptos 项目初始化，组件树搭建
- [ ] `default.rs` 布局完整实现（源/目标/输出/参数面板）
- [ ] 文件拖放上传
- [ ] 参数面板：face_detector / face_selector / face_masker / face_swapper / face_enhancer / output 全部组件
- [ ] 进度条 + ETA 显示（接收 SSE 更新）
- [ ] 处理结果下载按钮

#### 2.3 — 实时预览 & Webcam
- [ ] 处理中逐帧预览（`<img>` 标签 + SSE blob URL）
- [ ] Webcam 页面：`getUserMedia()` → WebSocket 发送 → 后端处理 → WebSocket 返回 → `<canvas>` 渲染
- [ ] Webcam 参数面板（分辨率、帧率、镜像）

#### 2.4 — 主题 & 样式
- [ ] 暗色/亮色主题切换
- [ ] 响应式布局（桌面端为主，移动端基础可用）

#### 2.5 — 集成 & 打包
- [ ] `facefusion ui` 子命令启动 Web Server
- [ ] 首次启动自动打开浏览器 `--open-browser`
- [ ] 前端 WASM + CSS 编译进二进制（`include_bytes!`）
- [ ] 产物更新：在线包 / 离线包均包含 Web UI

### 3.6 第二阶段功能清单

| 功能 | 说明 |
|------|------|
| Web UI — 主处理页 | 源/目标上传、全部参数面板、进度条、结果下载 |
| Web UI — 实时预览 | 处理中逐帧预览 |
| Web UI — Webcam | 浏览器摄像头 + 实时处理预览 |
| API | RESTful + SSE + WebSocket |
| 主题 | 暗色/亮色切换 + 响应式布局 |
| CLI 扩展 | `facefusion ui` 启动 Web Server |

---

## 四、第三阶段：剩余处理器

### 4.1 目标

移植剩余 9 个处理器，完善 Voice Extractor、音频处理、蒙版和选择器。Web UI 自动展示新增处理器选项。

### 4.2 第三阶段功能清单

- [ ] **剩余处理器移植**：
  - `face_editor`（LivePortrait 人脸编辑：眉毛方向 / 眼球注视 / 眼睛开合 / 嘴型 / 头部姿态）
  - `expression_restorer`（表情还原：微笑 / 张嘴 / 惊讶 / 生气）
  - `age_modifier`（年龄修改：FRAn，0←年轻 → 100 年老）
  - `background_remover`（背景移除：BEN，自定义填充色 / 去溢色）
  - `frame_enhancer`（全帧超分：clear_reality_x4 / real_esrgan_x4~x8 / ultra_sharp_x4）
  - `frame_colorizer`（黑白上色：DDColor）
  - `lip_syncer`（唇形同步：EDTalk + voice_extractor 预处理）
  - `face_debugger`（调试叠加：边界框 / 关键点 / 置信度 / 性别 / 年龄 / 种族）
  - `deep_swapper`（100+ 预训练艺人换脸：Druuzil 系列，morph 混合参数）
- [ ] `live_portrait.rs` 完善：LivePortrait 公用工具（表情限制、姿态限制、欧拉角旋转矩阵）
- [ ] **audio.rs**：Mel 频谱生成（STFT + mel 滤波器组 + 帧对齐）
- [ ] **voice_extractor.rs**：kim_vocal_1/2 + uvr_mdxnet 声音分离
- [ ] **face/detector.rs** 补充：yolo_face / yunet 模型支持
- [ ] **face_masker.rs** 完善：box / occlusion (XSeg) / area（全部区域）/ region (BiSeNet) 四种蒙版
- [ ] **face_selector.rs** 完善：reference 模式（嵌入距离匹配）、全部 8 种排序模式、全部过滤条件

---

## 五、第四阶段：周边系统

### 5.1 目标

补充 Webcam 实时流、作业管理、性能基准测试和多语言支持。

### 5.2 第四阶段功能清单

- [ ] **Webcam 实时流**：后端直接调用摄像头（`nokhwa` crate），不走浏览器 `getUserMedia()`
- [ ] **作业管理系统**（`jobs/`）：
  - `manager.rs`：draft → queue → submit → run → completed / failed
  - `runner.rs`：多步骤顺序执行、输出拼接、失败重试
  - `store.rs` / `helper.rs`：参数注册 + 工具函数
  - CLI：`job-list` / `job-create` / `job-submit` / `job-run` / `job-retry` / `job-delete`
  - Web UI：作业列表页完整可用
- [ ] **benchmark.rs**：多分辨率（240p~2160p）性能基准测试，warm/cold 模式，相对 FPS 报告
  - CLI：`benchmark` 子命令
  - Web UI：基准测试页完整可用
- [ ] **多语言 i18n**：`cli/locale.rs` + UI 端国际化，初始支持 en + zh_CN

---

## 六、技术决策记录

| # | 决策 | 理由 |
|---|------|------|
| 1 | 语言：Rust | ONNX Runtime 绑定最成熟（`ort` crate）、图像生态最丰富、天然适合单文件分发 |
| 2 | Go 被排除 | `onnxruntime_go` 社区维护，cgo 破坏静态链接和交叉编译，数值计算生态极弱 |
| 3 | C++ 被排除 | 分发打包困难，包管理碎片化，语言优势不足以抵消构建工程复杂度 |
| 4 | 推理引擎：ONNX Runtime | 模型格式不变，`ort` crate 封装 C API |
| 5 | 图像处理：`image` + `imageproc` + 手写核心算法 | `opencv-rust` 绑定的编译成本太高（需 CMake + OpenCV 源码），复杂变换（warp_affine、blend、NMS）手写即可，代码量有限 |
| 6 | ONNX 动态链接 | 捆绑 .dll/.so/.dylib 而非静态链接，避免 GPL/ONNX Runtime 许可证复杂度和链接问题 |
| 7 | FFmpeg 外部进程 | 与 Python 版一致，`std::process::Command` 调用，不引入 FFmpeg C API 的编译依赖 |
| 8 | Web 框架：Dioxus 或 Leptos | 全栈 Rust，SSR + 客户端 Hydration，无需 JS 前端分离；第二阶段确定最终选型 |
| 9 | 便携默认 | 所有数据默认存 exe 目录，不依赖系统路径；通过配置可选切换标准系统目录供高级用户多版本共享 |
| 10 | 异步运行时：tokio | Rust 生态标准，ONNX 推理在 `spawn_blocking` 中执行，FFmpeg 子进程用 `tokio::process` |
| 11 | 多线程视频处理：rayon | 每帧人脸分析/处理器由 rayon 线程池并行执行，`InferencePool` 中 `Arc<Mutex<Session>>` 保证 Session 线程安全复用 |
