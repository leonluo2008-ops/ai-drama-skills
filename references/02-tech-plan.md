# AI 漫剧/短剧制作工具 — 自研技术方案

> 版本: v1.0 | 日期: 2026-04-07 | 状态: 初稿

---

## 目录

1. [产品定位与差异化](#1-产品定位与差异化)
2. [系统架构设计](#2-系统架构设计)
3. [Agent 详细设计](#3-agent-详细设计)
4. [技术选型](#4-技术选型)
5. [关键技术难点与解决方案](#5-关键技术难点与解决方案)
6. [MVP 开发路线图](#6-mvp-开发路线图)
7. [成本模型](#7-成本模型)
8. [风险评估](#8-风险评估)

---

## 1. 产品定位与差异化

### 1.1 MVP 目标用户与场景

**核心用户画像（按优先级排列）：**

| 优先级 | 用户类型 | 典型场景 | 痛点 |
|--------|----------|----------|------|
| P0 | 短视频创作者/KOL | 批量生产漫剧内容投放抖音/快手/YouTube Shorts | 人工制作成本高，产出慢 |
| P1 | 小型 MCN / 内容工作室 | 日产 3-5 集漫剧，建立内容矩阵 | 缺乏美术团队，依赖外包 |
| P2 | 个人创作者/爱好者 | 讲述原创故事，制作动画短片 | 不会画画不会剪辑 |
| P3 | 品牌营销团队 | 制作品牌故事动画、产品宣传片 | 传统动画制作周期长 |

**MVP 聚焦场景：** 漫剧（漫画风格动画短剧），单集 1-5 分钟，竖屏（9:16）为主，适配短视频平台。

**不做（MVP 阶段）：**
- 长视频（>10 分钟）
- 真人风格视频
- 实时交互/游戏
- 复杂 3D 动画

### 1.2 与 OiiOii 的差异化方向

OiiOii 定位为通用 AI 动画平台，我们的差异化：

| 维度 | OiiOii | 我们的策略 |
|------|--------|-----------|
| **定位** | 通用动画制作（149 种风格） | **垂直聚焦漫剧/短剧**，深度优化 |
| **交互** | 纯 Web 对话式 | **对话 + 可视化时间线编辑器**双模式 |
| **风格一致性** | 依赖模型能力 | **角色 LoRA/参考图锚定 + 风格前缀注入** |
| **音效** | Suno（纯配乐） | **Suno + MiniMax TTS 双引擎**，支持角色配音 |
| **输出** | 单视频文件 | **一键适配多平台**（竖屏/横屏/方屏 + 字幕） |
| **成本** | 订阅制（较贵） | **按量计费 + 开源自部署选项** |
| **中文** | 英文为主 | **原生中文支持**，中文提示词工程 |
| **扩展性** | 封闭平台 | **开放 Agent 插件机制**，可接入更多模型 |

**核心护城河：** 中文漫剧场景的深度优化（角色一致性、节奏感、配音对齐），这是 OiiOii 作为英文通用平台难以做到的。

### 1.3 商业模式

**阶段 1（验证期）：**
- 按量计费（API 成本 + 加价）
- 单集 5 分钟漫剧：售价 ¥30-50，成本 ¥10-15
- 目标：验证需求，积累用户

**阶段 2（增长期）：**
- 订阅制：¥99/月（基础版 10 集），¥299/月（专业版 50 集）
- 按量付费：超出部分 ¥5/集
- 团队版：¥999/月（5 人协作，100 集）

**阶段 3（平台期）：**
- 模板市场（分镜模板、角色模板）
- 创作者社区（分享作品、模板）
- API 开放（让第三方集成）

**收入预估（月）：**
| 阶段 | 用户数 | ARPU | 月收入 |
|------|--------|------|--------|
| 验证期 | 50 | ¥200 | ¥10,000 |
| 增长期 | 500 | ¥150 | ¥75,000 |
| 平台期 | 5000 | ¥100 | ¥500,000 |

---

## 2. 系统架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        客户端层                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Web UI      │  │  CLI 工具    │  │  API (第三方集成) │   │
│  │  React       │  │  Python      │  │  RESTful         │   │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘   │
└─────────┼─────────────────┼───────────────────┼─────────────┘
          │                 │                   │
          ▼                 ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  认证鉴权 │ 速率限制 │ 路由 │ WebSocket (实时进度)    │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                    核心服务层                                 │
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│  │  项目管理服务   │  │  Agent 编排引擎 │  │  媒体服务    │  │
│  │  Projects      │  │  Orchestrator  │  │  Media       │  │
│  └────────────────┘  └───────┬────────┘  └──────────────┘  │
│                             │                                │
│  ┌────────────────┐  ┌──────▼─────────┐  ┌──────────────┐  │
│  │  用户服务       │  │  任务调度服务   │  │  模板服务    │  │
│  │  Users         │  │  Scheduler     │  │  Templates   │  │
│  └────────────────┘  └────────────────┘  └──────────────┘  │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                    Agent 层 (7 个 Agent)                      │
│                                                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │ 艺术总监 │→│  编剧   │→│ 角色    │→│ 场景    │          │
│  │ Director│ │ Writer  │ │ Char    │ │ Scene   │          │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │
│                                                    │        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────▼─┐       │
│  │ 后期合成 │←│ 音效总监│←│ 分镜师  │ │           │       │
│  │ Composer│ │ Audio   │ │ Storybd │ │           │       │
│  └─────────┘ └─────────┘ └─────────┘ └───────────┘       │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                    外部 API 适配层                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │ Veo 3.1  │ │ 即梦 CLI  │ │ MiniMax  │ │ Suno API    │  │
│  │ (聚鑫)   │ │ (Dreamina)│ │ TTS      │ │ (配乐)      │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│  │ 图床     │ │ LLM      │ │ ffmpeg   │                   │
│  │ (本机)   │ │ (GLM-5)  │ │ (视频处理)│                   │
│  └──────────┘ └──────────┘ └──────────┘                   │
└─────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                    基础设施层                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │PostgreSQL│ │ Redis    │ │ MinIO    │ │ Cloudflare   │  │
│  │ (数据库) │ │ (缓存/队列)│ │ (对象存储)│ │ Tunnel      │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心模块划分

| 模块 | 职责 | 技术要点 |
|------|------|----------|
| **Project Service** | 项目 CRUD、版本管理、协作权限 | PostgreSQL + 乐观锁 |
| **Agent Orchestrator** | Agent 流水线编排、上下文传递、人工审批卡点 | 状态机 + 事件驱动 |
| **Task Scheduler** | 异步任务调度、重试、优先级队列 | Celery + Redis |
| **Media Service** | 文件上传/下载、格式转换、元数据管理 | MinIO + ffmpeg |
| **User Service** | 注册登录、API Key 管理、配额管理 | JWT + PostgreSQL |
| **Template Service** | 风格模板、角色模板、分镜模板 | PostgreSQL JSONB |

### 2.3 Agent 编排引擎设计

#### 流水线模型

采用 **DAG（有向无环图）** 模型，默认为串行链，支持并行分支：

```
Director → Writer → [Character, Scene] → Storyboard → Audio → Composer
                          ↕ (并行)
```

**引擎核心接口：**

```python
class AgentOrchestrator:
    def create_pipeline(self, project_id: str, config: PipelineConfig) -> Pipeline
    
    def run_step(self, pipeline: Pipeline, step: Step) -> StepResult
    
    def pause_for_approval(self, pipeline: Pipeline, step: Step, 
                           preview_data: dict) -> Approval
    
    def resume(self, pipeline: Pipeline, approval: Approval) -> Pipeline
    
    def retry_step(self, pipeline: Pipeline, step: Step) -> StepResult
    
    def get_status(self, pipeline: Pipeline) -> PipelineStatus
```

#### 上下文传递机制

每个 Agent 的输出是一个 **结构化文档**（JSON Schema），存入项目的 `context` 表，后续 Agent 按需读取：

```python
@dataclass
class AgentContext:
    project_id: str
    step_name: str          # "director", "writer", "character", ...
    output_schema: dict     # 该 Agent 输出的 JSON Schema
    output_data: dict       # 实际输出数据
    artifacts: list[str]    # 产出的文件路径列表（图片、音频等）
    llm_messages: list      # 完整的 LLM 对话历史（用于多轮修改）
    created_at: datetime
    version: int            # 支持多轮修改的版本号
```

**上下文传递策略：**
1. **全量传递**：后续 Agent 可以读取所有前序 Agent 的输出
2. **摘要传递**：对于长上下文（如完整剧本），提取结构化摘要传给后续 Agent
3. **引用传递**：图片/视频等大文件只传路径，不传内容
4. **提示词注入**：将前序输出格式化为 LLM prompt 的一部分

#### 人工审批卡点

在以下环节设置**强制暂停**，等待用户确认后继续：

| 卡点位置 | 审批内容 | 默认行为 |
|----------|----------|----------|
| Writer 输出后 | 剧本内容 | 必须确认 |
| Character 输出后 | 角色设计图 | 必须确认 |
| Scene 输出后 | 场景设计图 | 必须确认 |
| Storyboard 每个镜头 | 分镜草图/视频预览 | 可配置（默认确认） |

### 2.4 状态管理方案

#### Pipeline 状态机

```
CREATED → RUNNING → PAUSED(APPROVAL) → RUNNING → ... → COMPLETED
                  → FAILED → RETRYING → RUNNING → ...
                  → CANCELLED
```

每个 Step 的状态：

```
PENDING → RUNNING → COMPLETED
                → FAILED → RETRYING → ...
                → AWAITING_APPROVAL → APPROVED → RUNNING → ...
                                   → REJECTED → RUNNING → ...
```

#### 数据库模型（核心表）

```sql
-- 项目表
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    title VARCHAR(200) NOT NULL,
    status VARCHAR(20) DEFAULT 'created',  -- created/running/completed/failed
    config JSONB NOT NULL DEFAULT '{}',     -- 项目配置（比例、时长、风格等）
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Pipeline 步骤
CREATE TABLE pipeline_steps (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id),
    step_order INT NOT NULL,
    agent_name VARCHAR(50) NOT NULL,        -- director/writer/character/...
    status VARCHAR(20) DEFAULT 'pending',
    input_data JSONB,
    output_data JSONB,
    llm_messages JSONB,                     -- LLM 对话历史
    artifacts JSONB,                        -- 产出文件列表
    version INT DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

-- 人工审批记录
CREATE TABLE approvals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_id UUID NOT NULL REFERENCES pipeline_steps(id),
    decision VARCHAR(10) NOT NULL,          -- approved/rejected
    feedback TEXT,                          -- 用户反馈（用于修改时传给 Agent）
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 3. Agent 详细设计

### 3.1 艺术总监 (Director Agent)

**职责：** 根据用户输入（故事梗概/关键词），确定影片整体创作方向。

**输入 Schema：**
```json
{
  "user_input": "一个关于勇敢小猫拯救森林的童话故事",
  "preferences": {
    "duration": "3-5分钟",
    "aspect_ratio": "9:16",
    "language": "中文",
    "mood": "温馨治愈",
    "target_audience": "3-8岁儿童"
  },
  "optional_references": ["https://img.aistar.work/ref1.jpg"]
}
```

**输出 Schema：**
```json
{
  "creative_direction": {
    "title": "《小橘的森林冒险》",
    "genre": "童话/冒险",
    "mood_board": {
      "color_palette": ["暖橙", "森林绿", "天空蓝"],
      "visual_style": "水彩绘本风",
      "emotion_arc": "平静→紧张→温馨→喜悦"
    },
    "style_reference": "吉卜力工作室风格，柔和色调，手绘质感",
    "pacing": "中速节奏，关键场景有情绪停留"
  },
  "production_specs": {
    "duration_seconds": 240,
    "aspect_ratio": "9:16",
    "scene_count_estimate": 8,
    "fps": 24
  }
}
```

**Prompt 策略：**
- 系统提示词包含：风格库（50+ 中文描述）、目标平台规范、节奏指导原则
- 输出格式强制 JSON，使用 function calling / structured output
- 多轮修改：保留对话历史，用户可说"风格再活泼一点"

**人工确认：** 不需要（快速启动，后续步骤可纠正）

---

### 3.2 编剧 (Writer Agent)

**职责：** 根据创作方向撰写完整动画剧本，输出结构化分场剧本。

**输入 Schema：**
```json
{
  "creative_direction": { "...来自 Director Agent 的输出" },
  "characters": [],                          // 可选，已知的角色
  "revision_notes": "第二幕节奏太慢，加快节奏"  // 多轮修改时
}
```

**输出 Schema：**
```json
{
  "title": "《小橘的森林冒险》",
  "characters": [
    {
      "name": "小橘",
      "role": "主角",
      "description": "一只勇敢的橘色小猫，体型娇小但充满勇气",
      "personality": "善良、勇敢、好奇",
      "visual_traits": "橘色毛发、绿色大眼睛、尾巴末端有一撮白毛"
    }
  ],
  "scenes": [
    {
      "scene_number": 1,
      "location": "森林边缘的小木屋",
      "time_of_day": "清晨",
      "mood": "宁静温馨",
      "description": "小橘在木屋前晒太阳，突然听到森林深处传来求救声",
      "characters_present": ["小橘"],
      "dialogue": [
        {
          "character": "小橘",
          "line": "那是谁在呼救？我必须去看看！",
          "emotion": "警觉而坚定"
        }
      ],
      "camera_direction": "远景→中景→特写小橘的表情",
      "duration_seconds": 25,
      "transition": "淡入"
    }
  ],
  "narration": [
    { "scene": 1, "text": "在森林边缘，住着一只叫小橘的猫咪..." }
  ]
}
```

**Prompt 策略：**
- 系统提示词包含：剧本结构规范、分场格式、对白风格指导
- 强制输出结构化 JSON，每个场景必须包含 duration_seconds
- 场景数根据总时长自动计算（建议每场景 15-45 秒）
- 旁白和对话分离，便于后续 TTS 处理

**人工确认：** ✅ **必须确认**（核心创作环节）

---

### 3.3 角色设计师 (Character Designer Agent)

**职责：** 根据剧本中的角色描述，设计所有角色的视觉形象。

**输入 Schema：**
```json
{
  "creative_direction": { "...风格参考..." },
  "characters": [ "...来自 Writer Agent 的角色列表..." ],
  "revision_notes": "小橘的眼睛再大一点"
}
```

**输出 Schema：**
```json
{
  "characters": [
    {
      "name": "小橘",
      "reference_images": [
        "https://img.aistar.work/char_xiaoju_front.png",
        "https://img.aistar.work/char_xiaoju_side.png"
      ],
      "prompt_template": "一只勇敢的橘色小猫，{pose}，{expression}，吉卜力风格，水彩绘本风，柔和色调",
      "style_prefix": "Studio Ghibli style, watercolor children's book illustration, warm color palette",
      "negative_prompt": "3D, realistic, photorealistic, dark, horror"
    }
  ]
}
```

**实现逻辑：**
1. LLM 根据角色描述 + 风格参考，生成精细的图片提示词
2. 调用即梦 CLI `text2image` 生成角色参考图
3. 可选：如果用户上传了参考图，使用 `image2image` 融合风格
4. 上传至图床，存储公网 URL（供 Veo 图生视频使用）

**Prompt 工程（图片生成）：**
```
风格前缀 + 角色核心特征 + 姿态 + 表情 + 场景描述 + 质量词
```
示例：
```
"Studio Ghibli style watercolor illustration, a brave small orange tabby cat 
with big green eyes and a white-tipped tail, sitting on a mossy log, curious expression, 
soft warm lighting, children's book art, hand-drawn texture, no text"
```

**人工确认：** ✅ **必须确认**（角色形象是后续所有画面的基础）

---

### 3.4 场景设计师 (Scene Designer Agent)

**职责：** 为剧本中每个场景设计背景画面，继承角色画风。

**输入 Schema：**
```json
{
  "creative_direction": { "...风格参考..." },
  "style_prefix": "Studio Ghibli style, watercolor...",
  "scenes": [ "...来自 Writer Agent 的场景列表..." ],
  "character_reference": "https://img.aistar.work/char_xiaoju_front.png",
  "revision_notes": "森林场景太暗，提亮"
}
```

**输出 Schema：**
```json
{
  "scenes": [
    {
      "scene_number": 1,
      "background_image": "https://img.aistar.work/scene_01_forest_edge.png",
      "prompt_template": "森林边缘的小木屋，清晨阳光，{weather}，{time_of_day}，{mood}",
      "mood_keywords": ["宁静", "温暖", "晨光"]
    }
  ]
}
```

**实现逻辑：**
1. LLM 为每个场景生成背景描述提示词（复用风格前缀）
2. 调用即梦 CLI 批量生成场景背景图
3. 确保与角色参考图的风格一致（使用相同的 style_prefix + negative_prompt）

**风格一致性关键：** 所有场景和角色使用相同的 `style_prefix` 和 `negative_prompt`，仅在主体描述部分变化。

**人工确认：** ✅ **必须确认**（场景质量直接影响最终效果）

---

### 3.5 分镜师 (Storyboard Agent)

**职责：** 将剧本转化为具体的视频镜头序列，生成每个镜头的视频片段。

**输入 Schema：**
```json
{
  "creative_direction": { "...节奏指导..." },
  "script": { "...完整剧本..." },
  "characters": { "...角色参考图..." },
  "scenes": { "...场景背景图..." },
  "mode": "fast"  // "fast"（参考图直出）| "precise"（宫格图确认）
}
```

**输出 Schema：**
```json
{
  "shots": [
    {
      "shot_number": 1,
      "scene_number": 1,
      "duration_seconds": 4,
      "description": "小橘在木屋前晒太阳的远景",
      "camera_movement": "缓慢推进",
      "character": "小橘",
      "character_pose": "慵懒地趴着",
      "character_expression": "满足地眯眼",
      "reference_image": "https://img.aistar.work/char_xiaoju.png",
      "scene_background": "https://img.aistar.work/scene_01.png",
      "composition_guide": "角色居中偏左，背景占60%，前景有花朵虚化",
      "veo_prompt": "A small orange tabby cat lounging peacefully on the porch of a cozy wooden cabin, 
                    forest edge background, morning sunlight filtering through trees, 
                    slow zoom in, Ghibli animation style, watercolor texture",
      "veo_negative_prompt": "blurry, low quality, distorted",
      "video_url": "https://img.aistar.work/video_shot_001.mp4",
      "audio": {
        "narration": "在森林边缘，住着一只叫小橘的猫咪",
        "narration_audio_url": "https://img.aistar.work/narration_001.mp3",
        "sound_effects": ["微风", "鸟鸣"]
      }
    }
  ],
  "total_duration_seconds": 240
}
```

**实现逻辑（两种模式）：**

**Fast 模式（参考图直出）：**
1. LLM 为每个镜头生成 Veo 提示词
2. 先用即梦生成镜头关键帧（角色 + 场景合成图）
3. 上传关键帧到图床
4. 调用聚鑫 Veo 3.1 `image2video` 批量生成视频

**Precise 模式（宫格图确认）：**
1. LLM 生成每个镜头的关键帧描述
2. 用即梦生成关键帧图片
3. 组合成宫格图供用户确认
4. 用户确认后才调用 Veo 生成视频

**视频生成策略：**
- 每个镜头默认 4-8 秒（Veo 3.1 支持 8 秒）
- 景别变化：远景(6-8s) → 中景(4-6s) → 特写(3-4s)
- 镜头运动：推进、拉远、平移、固定（避免复杂运镜）

**人工确认：**
- Fast 模式：不强制（但可在最终合成前整体预览）
- Precise 模式：每个镜头确认后生成

---

### 3.6 音效总监 (Audio Agent)

**职责：** 生成配乐、配音、音效，与视频时间线对齐。

**输入 Schema：**
```json
{
  "creative_direction": { "...情绪弧线..." },
  "shots": [ "...所有镜头及其旁白/对话..." ],
  "style_reference": "吉卜力风格，钢琴+弦乐为主，温馨治愈"
}
```

**输出 Schema：**
```json
{
  "bgm": {
    "prompt": "Warm orchestral, piano and strings, gentle adventure theme, 
              Studio Ghibli inspired, healing atmosphere, 4 minutes",
    "suno_track_url": "https://img.aistar.work/bgm_full.mp3",
    "suno_track_id": "suno_abc123"
  },
  "narrations": [
    {
      "shot_number": 1,
      "text": "在森林边缘，住着一只叫小橘的猫咪",
      "voice_id": "narrator_warm_female",
      "audio_url": "https://img.aistar.work/narration_001.mp3",
      "duration_seconds": 4.2
    }
  ],
  "dialogues": [
    {
      "shot_number": 1,
      "character": "小橘",
      "text": "那是谁在呼救？",
      "voice_id": "character_young_brave",
      "audio_url": "https://img.aistar.work/dialogue_001.mp3",
      "duration_seconds": 2.1
    }
  ],
  "sound_effects": [
    {
      "shot_number": 1,
      "type": "ambient",
      "description": "森林环境音：微风+鸟鸣",
      "audio_url": "https://img.aistar.work/sfx_ambient_01.mp3"
    }
  ]
}
```

**实现逻辑：**
1. **BGM**：LLM 生成 Suno 提示词 → 调用 Suno API 生成配乐 → 下载音频
2. **旁白/对话**：文本直接传给 MiniMax TTS → 选择合适音色 → 生成音频
3. **音效**：MVP 阶段使用预设音效库（ freesound.org 免费 CC0 音效），或 LLM 生成音效描述 → Suno 生成

**TTS 音色映射：**
| 角色/类型 | MiniMax 音色 | 特点 |
|-----------|-------------|------|
| 旁白（温馨） | `narrator_warm_female` | 温柔女声，语速适中 |
| 旁白（活泼） | `narrator_lively_male` | 活泼男声 |
| 少年角色 | `character_young` | 清亮少年音 |
| 老者 | `character_elder` | 沉稳老者音 |

**人工确认：** 不强制（配乐可快速重新生成）

---

### 3.7 后期合成 (Composer Agent)

**职责：** 将所有视频片段、音频、字幕合成为最终视频，支持多平台适配。

**输入 Schema：**
```json
{
  "shots": [ "...所有镜头视频URL..." ],
  "audio": { "...BGM + 旁白 + 对话 + 音效..." },
  "narration": [ "...旁白文本..." ],
  "output_config": {
    "aspect_ratio": "9:16",
    "resolution": "1080x1920",
    "format": "mp4",
    "codecs": { "video": "h264", "audio": "aac" },
    "subtitle": {
      "enabled": true,
      "style": "bottom_center",
      "font": "思源黑体",
      "font_size": 36,
      "color": "white",
      "outline": true
    }
  }
}
```

**输出 Schema：**
```json
{
  "final_video": "https://img.aistar.work/final_drama_xiaoju.mp4",
  "duration_seconds": 238,
  "file_size_mb": 85,
  "variants": [
    {
      "platform": "douyin",
      "aspect_ratio": "9:16",
      "resolution": "1080x1920",
      "url": "https://img.aistar.work/final_douyin.mp4"
    },
    {
      "platform": "youtube",
      "aspect_ratio": "16:9",
      "resolution": "1920x1080",
      "url": "https://img.aistar.work/final_youtube.mp4"
    },
    {
      "platform": "wechat",
      "aspect_ratio": "1:1",
      "resolution": "1080x1080",
      "url": "https://img.aistar.work/final_wechat.mp4"
    }
  ],
  "subtitle_file": "https://img.aistar.work/subtitles.srt",
  "timeline": [ "...时间线数据，供前端编辑器使用..." ]
}
```

**实现逻辑（ffmpeg）：**

```bash
# 1. 拼接所有镜头视频
ffmpeg -f concat -safe 0 -i shots_list.txt -c copy video_only.mp4

# 2. 合并所有音频轨道
ffmpeg -i video_only.mp4 \
       -i bgm.mp3 \
       -i narration_mix.mp3 \
       -filter_complex "[1:a]volume=0.3[bgm];[2:a]volume=1.0[narr];[0:a][bgm][narr]amix=inputs=3:duration=longest[aout]" \
       -map 0:v -map "[aout]" video_with_audio.mp4

# 3. 烧录字幕
ffmpeg -i video_with_audio.mp4 -vf "subtitles=subs.srt:force_style='FontName=Source Han Sans,FontSize=36,PrimaryColour=&HFFFFFF,OutlineColour=&H000000,Outline=2'" final.mp4

# 4. 多比例裁剪（9:16 → 16:9 / 1:1）
ffmpeg -i final.mp4 -vf "crop=ih*16/9:ih" output_16x9.mp4
ffmpeg -i final.mp4 -vf "crop=ih:ih" output_1x1.mp4
```

**人工确认：** 不强制（合成是确定性操作，结果可预期）

---

### 3.8 上下文传递机制（总结）

```
用户输入
  │
  ▼
Director ──→ creative_direction
  │
  ▼
Writer ──→ script (characters + scenes + dialogue + narration)
  │         │
  │         ├──→ Character Designer ──→ character_visuals (reference_images + prompt_templates)
  │         │
  │         └──→ Scene Designer ──→ scene_visuals (background_images + prompt_templates)
  │                    │
  │                    └──→ Storyboard ──→ shots (video_urls + veo_prompts)
  │                                     │
  │                                     └──→ Audio Agent ──→ audio_tracks
  │                                                       │
  │                                                       └──→ Composer ──→ final_video
  ▼
用户确认 & 下载
```

**上下文传递规则：**
1. 每个 Agent 接收 `creative_direction` + 所有前序 Agent 的输出
2. 大文件（图片/视频）只传 URL，不嵌入 JSON
3. LLM 对话历史按需截断（保留最近 20 轮）
4. 人工修改反馈通过 `revision_notes` 字段传入

---

## 4. 技术选型

### 4.1 后端框架

| 选项 | 推荐 | 理由 |
|------|------|------|
| **Python FastAPI** | ✅ 推荐 | 异步支持好，AI/ML 生态丰富，类型提示完善 |
| Node.js (NestJS) | 备选 | TypeScript 类型安全，但 AI 库生态弱于 Python |
| Go (Gin) | 不推荐 | AI SDK 生态差，开发效率低 |

**最终选择：Python 3.11 + FastAPI + uvicorn**

理由：
- 所有 AI SDK（openai、httpx）原生 Python
- ffmpeg 调用用 subprocess，Python 更方便
- 即梦 CLI 本身是 Python
- MiniMax TTS SDK 是 Python
- 异步支持（asyncio + FastAPI）足够应对高并发

### 4.2 前端框架

| 选项 | 推荐 | 理由 |
|------|------|------|
| **Next.js 14 (React)** | ✅ 推荐 | SSR + CSR，生态丰富，适合复杂交互 |
| Vue 3 (Nuxt) | 备选 | 学习曲线低，但动画/时间线组件生态弱 |
| Svelte | 不推荐 | 生态太小 |

**核心前端组件：**
- **对话面板**：与 Agent 对话、确认/修改输出
- **时间线编辑器**：拖拽排列镜头、调整时长、预览视频
- **角色/场景画廊**：展示设计图、对比预览
- **视频预览器**：播放最终视频、多比例切换

**推荐 UI 库：**
- Tailwind CSS（样式）
- Radix UI（基础组件）
- Fabric.js / Konva.js（时间线画布）
- Video.js / Plyr（视频播放）

### 4.3 数据库

| 用途 | 选型 | 理由 |
|------|------|------|
| **主数据库** | PostgreSQL 16 | JSONB 存 Agent 输出，成熟稳定 |
| **缓存/会话** | Redis 7 | 实时进度推送（Pub/Sub）、任务队列 |
| **对象存储** | MinIO（自部署）| 存储图片/视频/音频，S3 兼容 |
| **向量检索** | pgvector（PG 插件）| 风格/模板相似度搜索（Phase 3）|

### 4.4 消息队列 / 任务调度

| 选型 | 推荐场景 |
|------|----------|
| **Celery + Redis** | ✅ MVP 首选：异步任务队列、重试、定时任务 |
| **ARQ** | 轻量级替代：原生 asyncio，更简单 |
| **Temporal** | Phase 3 考虑：复杂工作流编排、长时间运行任务 |

**MVP 用 Celery + Redis：**
- 视频生成任务（耗时 2-10 分钟）
- 图片批量生成任务
- BGM/TTS 生成任务
- ffmpeg 合成任务

**队列优先级：**
```
P0: 人工审批等待（立即响应）
P1: 用户交互（生成预览、重新生成单镜头）
P2: 批量生成（全片渲染）
P3: 后台任务（缩略图生成、日志归档）
```

### 4.5 视频处理 (ffmpeg)

**版本要求：** ffmpeg 6.x+（支持硬件加速）

**核心操作：**

| 操作 | 命令 | 用途 |
|------|------|------|
| 视频拼接 | `concat` demuxer | 多镜头合并 |
| 音频混合 | `amix` filter | BGM + 旁白 + 音效 |
| 字幕烧录 | `subtitles` filter | 硬字幕 |
| 比例裁剪 | `crop` filter | 多平台适配 |
| 关键帧提取 | `select` filter | 视频缩略图 |
| 视频缩放 | `scale` filter | 分辨率调整 |
| 音频提取 | `-vn` | 提取音轨 |
| GIF 生成 | `palettegen` | 预览动图 |

**硬件加速（可选）：**
```bash
# NVIDIA GPU 加速（如有）
ffmpeg -hwaccel cuda -i input.mp4 -c:v h264_nvenc output.mp4
```

### 4.6 外部 API 对接

#### 聚鑫平台 Veo 3.1（视频生成）

```python
class VeoAdapter:
    BASE_URL = "https://api.juxin.example.com"  # 替换为实际地址
    
    async def text2video(self, prompt: str, duration: int = 8, 
                         negative_prompt: str = "") -> str:
        """文生视频"""
        # POST /v1/video/generate
        # 返回任务 ID，轮询等待完成
        ...
    
    async def image2video(self, image_url: str, prompt: str, 
                          duration: int = 8) -> str:
        """图生视频（主要使用）"""
        # 1. 确认图片已上传图床（公网 URL）
        # 2. POST /v1/video/image2video
        # 3. 轮询任务状态
        ...
    
    async def poll_task(self, task_id: str, timeout: int = 600) -> dict:
        """轮询任务状态，返回视频 URL"""
        # GET /v1/video/task/{task_id}
        # 间隔 10s 轮询，超时 10 分钟
        ...
```

**注意事项：**
- 图生视频必须先上传图床，只接受公网 URL
- 每个视频生成耗时约 2-5 分钟
- 需要实现指数退避重试（429 限流）
- 建议并发控制：最多 3 个同时生成

#### 即梦 Dreamina CLI（图片生成）

```python
class DreaminaAdapter:
    CLI_PATH = "/home/luo/.local/bin/dreamina"
    
    async def text2image(self, prompt: str, ratio: str = "9:16",
                         resolution: str = "2k") -> str:
        """文生图"""
        cmd = f"{self.CLI_PATH} text2image --prompt='{prompt}' " \
              f"--ratio={ratio} --resolution_type={resolution} --poll=30"
        result = await run_command(cmd)
        # 解析输出，获取图片路径
        return local_path
    
    async def image2image(self, image_path: str, prompt: str,
                          resolution: str = "2k") -> str:
        """图生图"""
        cmd = f"{self.CLI_PATH} image2image --images {image_path} " \
              f"--prompt='{prompt}' --resolution_type={resolution} --poll=30"
        ...
```

#### MiniMax TTS（语音合成）

```python
class MiniMaxTTSAdapter:
    BASE_URL = "https://api.minimax.chat/v1/t2a_v2"
    API_KEY = os.environ["MINIMAX_API_KEY"]
    
    async def synthesize(self, text: str, voice_id: str, 
                         speed: float = 1.0) -> str:
        """文本转语音"""
        payload = {
            "model": "speech-01-turbo",
            "text": text,
            "voice_setting": {"voice_id": voice_id},
            "audio_setting": {"speed": speed}
        }
        # POST 请求，返回音频二进制
        # 保存为 MP3 文件
        ...
```

#### Suno API（配乐生成）

```python
class SunoAdapter:
    BASE_URL = "https://api.suno.ai/v1"
    
    async def generate_music(self, prompt: str, duration: int,
                             style: str = "instrumental") -> str:
        """生成背景音乐"""
        # POST /generate
        # 轮询获取音频 URL
        ...
```

#### 图床服务

```python
class ImageBedAdapter:
    BASE_URL = "http://127.0.0.1:40061"
    TOKEN = "70a54e1d8ef958d2fff727e9af8c8d8e"
    
    async def upload(self, file_path: str) -> str:
        """上传文件到图床，返回公网 URL"""
        sign = int(time.time())
        files = {"file": open(file_path, "rb")}
        data = {"sign": sign, "token": self.TOKEN}
        async with aiohttp.ClientSession() as session:
            resp = await session.post(
                f"{self.BASE_URL}/app/upload.php",
                files=files, data=data
            )
        result = await resp.json()
        return result["url"]  # https://img.aistar.work/xxx.png
```

---

## 5. 关键技术难点与解决方案

### 5.1 角色一致性

**问题：** AI 生成视频中同一角色在不同镜头中外观不一致。

**解决方案（分层策略）：**

**Level 1 — 参考图锚定（MVP）：**
- 角色设计阶段生成高质量参考图（正面、侧面各一张）
- 每个镜头的视频生成都传入角色参考图
- Veo 提示词中强调角色外观描述

**Level 2 — 风格前缀注入（MVP）：**
```
所有图片/视频生成的 prompt 都以相同的风格前缀开头：
"Studio Ghibli style watercolor illustration, [角色描述], [场景描述], ..."
```

**Level 3 — LoRA 训练（Phase 3）：**
- 收集角色设计图 + 变体
- 训练轻量 LoRA（~20 张图，30 分钟）
- 推理时加载角色 LoRA，保证外观一致
- 技术：kohya_ss / sd-scripts

**Level 4 — 角色一致性评估（Phase 3）：**
- 生成后自动评估：提取角色面部特征向量
- 计算与参考图的特征距离
- 不一致时自动重新生成

### 5.2 风格一致性

**问题：** 不同 Agent 生成的图片/视频风格不统一。

**解决方案：**

1. **全局 Style Lock**：项目创建时锁定风格配置
   ```json
   {
     "style_prefix": "Studio Ghibli style watercolor illustration",
     "negative_prompt": "3D, realistic, photorealistic, dark, horror, anime",
     "color_palette": ["#FF9F43", "#2ED573", "#70A1FF"],
     "quality_tags": "masterpiece, best quality, hand-drawn texture"
   }
   ```

2. **强制注入**：所有即梦/Veo 调用自动拼接 style_prefix
3. **负向提示词白名单**：所有生成操作使用统一的 negative_prompt
4. **色彩一致性校验**：生成后提取主色调，与目标色板对比，偏差大则重新生成

### 5.3 批量视频生成的并发与队列管理

**问题：** 单集漫剧需要 30-60 个镜头视频，生成耗时长，需管理并发和失败重试。

**解决方案：**

**队列设计：**
```python
# Celery 任务定义
@celery.task(bind=True, max_retries=3, default_retry_delay=60)
def generate_shot_video(self, shot_id: str, image_url: str, prompt: str):
    try:
        task_id = veo_client.image2video(image_url, prompt)
        result = poll_with_backoff(task_id, max_wait=600)
        if result.status == "failed":
            self.retry(exc=Exception("Veo generation failed"))
        return result.video_url
    except RateLimitError:
        self.retry(countdown=120)  # 限流后等待 2 分钟
```

**并发控制：**
```
总并发上限：3（Veo API 限流）
├── 高优先级队列：1（用户正在等待的单镜头重新生成）
├── 普通优先级队列：2（批量生成中的镜头）
└── 低优先级队列：0（空闲时处理）
```

**断点续传：**
- 每个镜头的视频生成结果存入数据库
- 失败的镜头标记为 `failed`，可单独重试
- 支持从任意断点恢复整个 pipeline

**预估时间：** 40 个镜头 × 3 分钟平均 / 3 并发 = ~40 分钟全片渲染

### 5.4 成本控制

**策略：**

1. **智能缓存**：相同角色/场景描述的生成结果缓存 7 天
2. **渐进式生成**：先用低分辨率快速预览，确认后再生成高清版本
3. **批量折扣**：一次性提交多镜头，减少 API 调用开销
4. **用户配额**：按订阅等级限制月度生成量
5. **成本预估器**：在开始生成前显示预估成本，用户确认后执行

```python
def estimate_cost(scene_count: int, has_narration: bool = True) -> dict:
    """预估单集成本"""
    shots_per_scene = 4
    total_shots = scene_count * shots_per_scene
    
    return {
        "image_generation": total_shots * 0.02,     # 即梦 2K 免费，此处为 0
        "video_generation": total_shots * 0.15,       # Veo 估算
        "tts": scene_count * 3 * 0.01 if has_narration else 0,  # MiniMax
        "bgm": 0.05,                                   # Suno
        "total": total_shots * 0.15 + scene_count * 0.03 + 0.05
    }
```

### 5.5 分镜节奏控制

**问题：** AI 生成的分镜节奏单一，缺乏影视感。

**解决方案：**

1. **节奏模板库**：内置常见节奏模式
   ```
   温馨开场：远景(6s) → 中景(4s) → 特写(3s) → 全景(5s)
   紧张高潮：快切(2s×4) → 慢镜头(5s) → 快切(2s×3) → 全景(4s)
   情感高潮：慢推(6s) → 特写(4s) → 慢推(5s) → 渐黑(3s)
   ```

2. **情绪-节奏映射**：
   ```
   平静 → 长镜头(5-8s)，缓慢运镜
   紧张 → 短镜头(2-3s)，快切
   惊喜 → 特写(3-4s)，推镜头
   感动 → 慢镜头(6-8s)，渐变过渡
   ```

3. **LLM 节奏指导**：在分镜 Agent 的系统提示词中注入节奏规则
4. **时长自动计算**：根据旁白/对话音频时长自动调整镜头时长
5. **用户可调**：前端时间线编辑器支持拖拽调整每个镜头的时长

---

## 6. MVP 开发路线图

### Phase 1：命令行版（验证 Pipeline）

**目标：** 验证 7 个 Agent 的串行流水线是否跑通，产出第一个完整漫剧。

**工期：** 4-6 周（1 人全职）

**交付物：**
1. CLI 工具 `drama create "故事描述"`
2. 7 个 Agent 的 Python 实现
3. Agent 编排引擎（串行 + 人工确认卡点）
4. 外部 API 适配器（Veo、即梦、TTS、图床）
5. ffmpeg 合成脚本
6. 第一个示例漫剧视频

**周计划：**

| 周 | 任务 |
|----|------|
| W1 | 项目骨架、Director + Writer Agent、LLM 对接 |
| W2 | Character + Scene Designer Agent、即梦 CLI 对接 |
| W3 | Storyboard Agent、Veo API 对接、图床上传 |
| W4 | Audio Agent（TTS + Suno）、ffmpeg 合成 |
| W5 | 端到端联调、Bug 修复、第一个完整漫剧 |
| W6 | 优化、文档、缓存/重试机制 |

**里程碑：** 产出第一个 2 分钟漫剧视频

---

### Phase 2：Web UI（基础交互）

**目标：** 提供 Web 界面，用户可以通过浏览器创建项目、确认/修改 Agent 输出、预览视频。

**工期：** 6-8 周

**交付物：**
1. 用户注册/登录
2. 项目创建与管理
3. Agent 对话界面（确认/修改每步输出）
4. 视频/图片预览
5. 最终视频下载（多比例）
6. 基础用量统计

**周计划：**

| 周 | 任务 |
|----|------|
| W1-2 | 后端 API（FastAPI）、数据库设计、用户系统 |
| W3-4 | 前端框架搭建、项目列表页、Agent 对话界面 |
| W5-6 | 图片/视频预览、下载功能、WebSocket 进度推送 |
| W7 | 多比例输出、用量统计、Bug 修复 |
| W8 | 测试、部署（Docker + Cloudflare Tunnel）、文档 |

**里程碑：** 用户可通过 Web 界面完成完整漫剧制作流程

---

### Phase 3：完整产品（Agent 对话、时间线编辑）

**目标：** 产品级体验，支持高级功能和商业化。

**工期：** 8-12 周

**交付物：**
1. **时间线编辑器**：拖拽排列镜头、调整时长、预览播放
2. **角色库**：保存角色设计，跨项目复用
3. **模板系统**：风格模板、分镜模板、角色模板
4. **团队协作**：多人编辑同一项目
5. **订阅付费系统**：Stripe/微信支付集成
6. **角色 LoRA 训练**（可选）：提升角色一致性
7. **批量生产**：一次创建多集，队列管理
8. **API 开放**：RESTful API，第三方集成

**里程碑：** 上线公测，开始商业化运营

---

### 总工期估算

| 阶段 | 工期 | 累计 |
|------|------|------|
| Phase 1 (CLI) | 4-6 周 | 6 周 |
| Phase 2 (Web) | 6-8 周 | 14 周 |
| Phase 3 (完整产品) | 8-12 周 | 26 周 |
| **总计** | **18-26 周** | **~6 个月** |

---

## 7. 成本模型

### 7.1 单集 5 分钟漫剧成本估算

**假设：** 5 分钟（300 秒），约 10 个场景，40 个镜头。

| 环节 | 数量 | 单价 | 小计 | 备注 |
|------|------|------|------|------|
| LLM 调用（编剧+提示词） | ~20 次 | ¥0.05 | ¥1.0 | GLM-5 定价 |
| 角色设计图（即梦 2K） | 3-5 张 | ¥0（免费） | ¥0 | 会员免费额度 |
| 场景背景图（即梦 2K） | 10 张 | ¥0（免费） | ¥0 | 会员免费额度 |
| 镜头关键帧（即梦 2K） | 40 张 | ¥0（免费） | ¥0 | 会员免费额度 |
| 视频生成（Veo 图生视频） | 40 个 | ¥0.15 | ¥6.0 | 8 秒/个，估算 |
| 旁白 TTS | ~15 段 | ¥0.01 | ¥0.15 | MiniMax |
| 对话 TTS | ~20 段 | ¥0.01 | ¥0.20 | MiniMax |
| BGM 生成 | 1 首 | ¥0.05 | ¥0.05 | Suno |
| ffmpeg 合成 | 1 次 | ¥0 | ¥0 | 本机计算 |
| **总计** | | | **¥7.4** | |

**注意：** 以上为 API 直接成本。即梦 2K 免费（会员），Veo 价格需按实际聚鑫定价确认。

### 7.2 API 调用费用明细（按月）

**假设：** 月产 50 集 5 分钟漫剧。

| 项目 | 单集用量 | 月用量 | 单价 | 月费用 |
|------|----------|--------|------|--------|
| LLM | 20 次 | 1000 次 | ¥0.05 | ¥50 |
| 即梦图片 | 55 张 | 2750 张 | ¥0 | ¥0 |
| Veo 视频 | 40 个 | 2000 个 | ¥0.15 | ¥300 |
| TTS | 35 段 | 1750 段 | ¥0.01 | ¥17.5 |
| Suno BGM | 1 首 | 50 首 | ¥0.05 | ¥2.5 |
| 服务器 | - | - | - | ¥200（云服务器） |
| 存储 | - | ~50GB | - | ¥20 |
| **总计** | | | | **~¥590/月** |

### 7.3 定价建议

| 套餐 | 价格 | 包含 | 超出 |
|------|------|------|------|
| 免费体验 | ¥0 | 1 集/月（带水印） | - |
| 基础版 | ¥99/月 | 10 集/月 | ¥12/集 |
| 专业版 | ¥299/月 | 50 集/月 | ¥8/集 |
| 团队版 | ¥999/月 | 200 集/月 + 5 席位 | ¥6/集 |

**毛利率估算：**
- 基础版：成本 ¥74（10 集），售价 ¥99，毛利 25%
- 专业版：成本 ¥370（50 集），售价 ¥299，**亏损** → 需优化成本或调整定价
- **调整后专业版：** ¥399/月，50 集，成本 ¥370，毛利 7%
- 团队版：成本 ¥1480（200 集），售价 ¥999，**亏损** → 批量生成需成本优化

**建议：**
1. MVP 阶段按量计费，不设固定套餐
2. 通过优化 Veo 调用（减少冗余生成、智能缓存）降低视频成本
3. 当 LoRA 角色一致性上线后，可减少重拍率，降低视频成本 30%+

---

## 8. 风险评估

### 8.1 技术风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| Veo 视频质量不稳定 | 高 | 高 | 多次生成取最优；增加人工筛选卡点 |
| 角色一致性不足 | 高 | 高 | 参考图锚定 + LoRA（Phase 3） |
| API 限流/不可用 | 中 | 高 | 队列缓冲 + 重试 + 备用模型（即梦视频） |
| ffmpeg 合成质量 | 低 | 中 | 丰富的滤镜参数调优；参考 FFmpeg 社区方案 |
| 大文件传输/存储 | 中 | 中 | MinIO 自部署；按需转码压缩 |
| LLM 输出不稳定 | 中 | 中 | 结构化输出 + Schema 验证 + 重试 |

### 8.2 竞争风险

| 竞争对手 | 威胁等级 | 差异化 |
|----------|----------|--------|
| OiiOii.ai | 高 | 我们垂直中文漫剧，深度优化 |
| 可灵 (Kling) | 中 | 他们是通用视频生成，非制作工具 |
| Runway Gen-3 | 中 | 通用视频生成，非制作流程 |
| 即梦/字节系 | 高 | 如果字节做类似工具，资源碾压 |
| 开源项目 (ComfyUI 等) | 低 | 需要技术能力，非用户友好 |

**核心防御：** 快速迭代 + 中文漫剧场景的深度积累（模板库、最佳实践、用户反馈循环）

### 8.3 法律/版权风险

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| 生成内容版权不明确 | 高 | 高 | 用户协议明确版权归属；声明 AI 生成 |
| AI 模型版权争议 | 中 | 高 | 使用商业授权 API（Veo、即梦等） |
| 用户生成侵权内容 | 中 | 中 | 内容审核机制；用户协议免责条款 |
| 音乐版权 | 中 | 中 | Suno 商用授权；或使用免版权音效库 |
| 角色设计侵权 | 低 | 中 | 不内置知名 IP 角色；用户自定义 |
| GDPR/数据隐私 | 低 | 低 | 中国用户为主，暂不涉及 |

**关键法律措施：**
1. 用户协议：明确 AI 生成内容的版权归属用户
2. 免责声明：平台不对生成内容的版权做保证
3. 内容审核：敏感词过滤 + 人工抽查
4. API 授权：确保所有调用的 AI API 有商业使用授权

---

## 附录

### A. 项目目录结构

```
ai-drama/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI 入口
│   │   ├── config.py            # 配置管理
│   │   ├── models/              # 数据库模型
│   │   ├── api/                 # API 路由
│   │   ├── agents/              # 7 个 Agent
│   │   │   ├── base.py          # Agent 基类
│   │   │   ├── director.py
│   │   │   ├── writer.py
│   │   │   ├── character.py
│   │   │   ├── scene.py
│   │   │   ├── storyboard.py
│   │   │   ├── audio.py
│   │   │   └── composer.py
│   │   ├── adapters/            # 外部 API 适配器
│   │   │   ├── veo.py
│   │   │   ├── dreamina.py
│   │   │   ├── tts.py
│   │   │   ├── suno.py
│   │   │   └── imagebed.py
│   │   ├── orchestrator/        # Agent 编排引擎
│   │   │   ├── pipeline.py
│   │   │   ├── context.py
│   │   │   └── scheduler.py
│   │   └── services/            # 业务服务
│   │       ├── project.py
│   │       ├── media.py
│   │       └── user.py
│   ├── tasks/                   # Celery 任务
│   │   ├── video_gen.py
│   │   ├── image_gen.py
│   │   └── compose.py
│   ├── tests/
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   │   ├── AgentChat.tsx
│   │   │   ├── Timeline.tsx
│   │   │   ├── VideoPlayer.tsx
│   │   │   └── Gallery.tsx
│   │   └── lib/
│   ├── package.json
│   └── Dockerfile
├── docker-compose.yml
├── .env.example
└── README.md
```

### B. 环境变量模板

```bash
# .env.example
DATABASE_URL=postgresql://user:pass@localhost:5432/ai_drama
REDIS_URL=redis://localhost:6379/0
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin

# AI APIs
VEO_API_KEY=xxx
VEO_BASE_URL=https://api.juxin.example.com
MINIMAX_API_KEY=xxx
MINIMAX_GROUP_ID=xxx
SUNO_API_KEY=xxx

# Image Bed
IMAGE_BED_URL=http://127.0.0.1:40061
IMAGE_BED_TOKEN=70a54e1d8ef958d2fff727e9af8c8d8e

# LLM
LLM_API_KEY=xxx
LLM_BASE_URL=https://open.bigmodel.cn/api/paas/v4
LLM_MODEL=glm-5-turbo

# Dreamina CLI
DREAMINA_CLI_PATH=/home/luo/.local/bin/dreamina
```

### C. 关键依赖

**Python (backend):**
```
fastapi>=0.110
uvicorn[standard]>=0.29
sqlalchemy>=2.0
alembic>=1.13
celery[redis]>=5.3
redis>=5.0
httpx>=0.27
aiofiles>=23.0
python-multipart>=0.0.9
pydantic>=2.0
minio>=7.2
websockets>=12.0
```

**Node.js (frontend):**
```json
{
  "dependencies": {
    "next": "14.x",
    "react": "18.x",
    "tailwindcss": "3.x",
    "@radix-ui/react-dialog": "1.x",
    "video.js": "8.x",
    "zustand": "4.x",
    "socket.io-client": "4.x"
  }
}
```

---

> **文档结束** | 下一步：确认技术方案后，开始 Phase 1 开发
