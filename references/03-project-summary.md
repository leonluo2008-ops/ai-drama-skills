# AI漫剧制作工具 · 开发存档

> 存档时间：2026-04-16
> 当前阶段：场景设计（阶段3）完成，多角度场景图完成；待进入分镜导演（阶段4）

---

## 一、项目概述

### 定位
AI漫剧/短视频制作工具，将小说/剧本自动转化为分镜图+视频片段+音频的完整制作流程。

### 差异化
- 垂直聚焦：中文漫剧短剧，非通用动画平台
- 画风一致：角色参考图锚定 + 风格前缀注入
- 三层音频：TTS台词 + 音效（嵌入视频prompt） + BGM（Music 2.6）
- 多平台适配：竖屏9:16为主，支持横屏1:1/16:9

### 技术栈
- Agent框架：Python + FastAPI + SQLAlchemy（Async）
- 图片生成：即梦 Dreamina CLI（2K免费，会员）
- 视频生成：聚鑫平台 Veo 3.1（图生视频）
- 语音合成：MiniMax TTS（speech-2.6-turbo）
- 配乐生成：MiniMax Music 2.6
- 视频合成：ffmpeg

---

## 二、Skill架构（已验证）

### 目录位置
```
~/.openclaw/workspace/skills/ai-drama/          # 总控
~/.openclaw/workspace/skills/ai-drama-script/  # 编剧
~/.openclaw/workspace/skills/ai-drama-character/ # 角色设计
~/.openclaw/workspace/skills/ai-drama-scene/   # 场景设计
~/.openclaw/workspace/skills/ai-drama-storyboard/ # 分镜导演
~/.openclaw/workspace/skills/ai-drama-audio/    # 音频设计
```

### 流水线

```
阶段1：编剧（ai-drama-script）
  输入：原始剧本文本
  输出：视频脚本 + 角色表 + 场景表 + 场次表 + 风格锁定

阶段2：角色设计（ai-drama-character）
  输入：角色表 + 风格锁定
  输出：角色卡 + 六视图定妆照（图生图）

阶段3：场景设计（ai-drama-scene）
  输入：场景表 + 风格锁定
  输出：场景卡 + 大场景全景图 + 多角度场景图

阶段4：分镜导演（ai-drama-storyboard）
  输入：视频脚本 + 角色卡 + 场景卡 + 风格锁定
  输出：分镜图prompt + 视频prompt + 音效prompt + 台词prompt

阶段5：音频设计（ai-drama-audio）
  输入：视频脚本 + 分镜方案 + 风格锁定
  输出：TTS脚本 + BGM提示词 + 音效补充

阶段6：视频生成 + 合成（外部工具）
  即梦 → 分镜图 → 聚鑫 → 视频片段 → ffmpeg合成
```

---

## 三、已验证的核心技术点

### 3.1 即梦CLI · 任务状态轮询（R-010 铁律）

**问题**：误用 `--poll=30` 反复提交相同prompt，导致后台大量垃圾任务。

**正确做法**：
```bash
# 1. 提交任务，--poll=0 立即返回 submit_id
result=$(dreamina text2image --prompt="..." --ratio=1:1 --resolution_type=2k --poll=0)
submit_id=$(echo "$result" | python3 -c "import json,sys; print(json.load(sys.stdin)['submit_id'])")

# 2. while 循环轮询 query_result
while true; do
  status=$(dreamina query_result --submit_id=$submit_id | python3 -c "import json,sys; print(json.load(sys.stdin)['gen_status'])")
  if [ "$status" = "success" ]; then
    break
  elif [ "$status" = "failed" ]; then
    exit 1
  fi
  sleep 10
done

# 3. 成功后下载
dreamina query_result --submit_id=$submit_id --download_dir=/tmp

# 4. 通过 message 工具发送到飞书（只在成功后发一次，禁止在循环内发）
message(action="send", channel="feishu", message="...", filePath="/path/to/图.png")
```

**CLI能力**：dreamina 不支持 batch 模式，binary 有 batch 符号但 CLI 不开放。多个任务需分次提交。

### 3.2 场景图 · 空场景铁律

**问题**：场景图中出现故事角色（已生成的场景2/3），导致角色外观与场景合成后重复穿帮。

**规则**：场景图 = 空场景，角色在后续分镜阶段通过 image2image 合成。
- ✅ 正确：生成纯场景背景，无人
- ❌ 错误：场景中出现了主角/配角/故事角色
- ⚠️ 唯一例外：场景氛围所需的背景群众（如服务员）可以出现但不能占主导

**提示词约束**：`NO HUMANS NO CHARACTERS, no people, empty scene`

### 3.3 角色定妆照 · 图生图链路

**链路**：面部图 + 服装参考图 → image2image → 六视图定妆照

**服装参考图规则**：
- 必须包含全身穿搭（上衣+下装+鞋+配饰+道具），缺一不可
- 每件物品必须完整出镜、不裁切
- 用"俯视平铺+完整展示不裁切"，AI理解更准确

### 3.4 分镜图 · 生成链路

**链路**：角色定妆照 + 场景多角度图（指定角度） → image2image → 分镜图

### 3.5 视频生成

**只用聚鑫，不用即梦**：即梦视频排队 10万+，基本不可用（R-006）。

---

## 四、测试剧本（已跑通）

### 剧本内容
都市情感剧《婆婆五十岁生日》，核心冲突：儿媳沈晚宁 vs 孝子顾砚 vs 恶女助理林浅浅

### 角色（5人）

| 角色 | 身份 | 外观全称 |
|------|------|----------|
| 沈晚宁 | 女主角 | 待生成 |
| 顾砚 | 男主角 | 待生成 |
| 林浅浅 | 女二，助理 | 待生成 |
| 顾思敏 | 豪门千金 | 待生成 |
| 张经理 | 中层管理 | 待生成 |

### 场景（4个）

| 场景 | 地点 | 氛围 | 状态 |
|------|------|------|------|
| 场景1 | 林浅浅餐厅后厨 | 阴暗残忍 | ✅大场景+✅多角度 |
| 场景2 | 私密电话室 | 压抑孤独 | ✅大场景+✅多角度 |
| 场景3 | 顾砚办公室 | 冷峻疏离 | ✅大场景+✅多角度 |
| 场景4 | 万豪宴会厅 | 奢华讽刺 | ✅大场景+✅多角度 |

### 风格锁定
- 画风：现代都市风格，写实现代感
- 色调：莫兰迪色调（低饱和灰调）
- 光影：电影感强明暗对比
- 整体氛围：压抑中带克制的愤怒

---

## 五、Skill文件清单

| 文件 | 行数 | 状态 | 说明 |
|------|------|------|------|
| ai-drama/SKILL.md | 91 | ✅ 稳定 | 总控，流水线调度 |
| ai-drama-script/SKILL.md | 124 | ✅ 稳定 | 编剧，已验证 |
| ai-drama-character/SKILL.md | 199 | ✅ 已更新 | 角色设计，含自动生成流程+R-010 |
| ai-drama-scene/SKILL.md | 186 | ✅ 已更新 | 场景设计，含自动生成流程+R-010+空场景铁律 |
| ai-drama-storyboard/SKILL.md | 149 | ✅ 稳定 | 分镜导演，待实际测试 |
| ai-drama-audio/SKILL.md | 127 | ✅ 稳定 | 音频设计，待实际测试 |

---

## 六、待验证阶段（未实际跑通）

### 高优先级
- [ ] 分镜图生成（阶段4）：角色定妆照+场景图→分镜图，尚未实际测试
- [ ] 角色六视图定妆照生成（阶段2补充）：面部图+服装图→六视图，尚未实际测试
- [ ] 聚鑫Veo视频生成：分镜图→视频片段，尚未实际测试
- [ ] TTS台词生成（阶段5）：尚未实际测试
- [ ] BGM生成（阶段5）：尚未实际测试

### 中优先级
- [ ] 端到端联调：从剧本到最终视频完整跑通
- [ ] Seedance 2.0 视频prompt格式验证

---

## 七、Python后端（ai-drama app）

位于 `apps/ai-drama/`，架构：

```
ai-drama/
├── app/
│   ├── agents/           # 7个Agent的Python实现（Director/Writer/Character/Scene/Storyboard/Audio/Composer）
│   ├── adapters/         # 外部API适配器（dreamina/tts/veo/imagebed）
│   ├── api/              # FastAPI路由（health/projects）
│   ├── core/             # 核心（config/database/style）
│   ├── models/           # SQLAlchemy模型
│   ├── orchestrator/     # 流水线引擎
│   └── schemas/         # Pydantic schema
├── alembic/              # 数据库迁移
├── tests/                # 测试
└── requirements.txt
```

**注意**：当前agent实际执行依赖LLM+即梦CLI，Python后端主要用于Web UI和数据库管理。

---

## 八、核心参考文档

| 文档 | 路径 | 说明 |
|------|------|------|
| 即梦CLI使用规范 | `references/dreamina-cli-usage.md` | R-010铁律详解，轮询脚本模板 |
| 技术方案 | `references/ai-drama-tool-tech-plan.md` | 产品/系统/Agent详细设计 |
| OiiOii调研 | `references/oiioii-deep-analysis.md` | 参考竞品分析 |

---

## 九、已知坑位（Error Patterns）

| 坑位 | 原因 | 解决方案 |
|------|------|----------|
| 重复提交任务 | `--poll=30` 误用为反复轮询 | R-010：--poll=0+while轮询 |
| 场景图出现人物 | prompt缺少空场景约束 | R-010补充：NO HUMANS NO CHARACTERS |
| 即梦视频不可用 | 排队10万+ | 只用聚鑫Veo |
| httpx代理问题 | 系统代理导致外网API失败 | R-005：proxy=None |

---

## 十、下一步工作

### 立即可做
1. 角色六视图定妆照生成（阶段2实际测试）
2. 分镜图生成（阶段4实际测试）

### 待做
1. 端到端联调（剧本→最终视频）
2. 聚鑫Veo视频生成测试
3. TTS/BGM生成测试

---

_Last updated: 2026-04-16 by Claw_
