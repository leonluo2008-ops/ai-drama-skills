# AI Drama Skills

AI漫剧制作的全流程Skill套件，驱动AI Agent自动完成从剧本到视频片段的完整制作。

## 功能概述

| 阶段 | Skill | 核心产出 |
|------|--------|---------|
| 总控 | `ai-drama` | 流水线调度 |
| 编剧 | `ai-drama-script` | 视频脚本 + 角色表 + 场景表 |
| 角色设计 | `ai-drama-character` | 角色卡 + 六视图定妆照 |
| 场景设计 | `ai-drama-scene` | 场景卡 + 大场景图 + 多角度图 |
| 分镜导演 | `ai-drama-storyboard` | 分镜图prompt + 视频prompt + 音效prompt |
| 音频设计 | `ai-drama-audio` | TTS脚本 + BGM提示词 |

## 快速开始

### 1. 安装Skill

```bash
# 克隆到本地
git clone https://github.com/leonluo2008-ops/ai-drama-skills.git

# 复制到OpenClaw的skill目录
cp -r skills/* ~/.openclaw/workspace/skills/
```

### 2. 配置外部工具

**即梦Dreamina CLI**（图片生成）：
```bash
# 安装
pip install dreamina

# 登录
dreamina login
```

**聚鑫平台**（视频生成）：申请API Key，配置到环境变量。

**MiniMax**（TTS/Music）：申请API Key，配置到环境变量。

### 3. 使用

在支持skill的AI Agent中发送剧本，触发总控skill即可自动进入流水线。

## 核心规则

### R-010 · 即梦CLI任务轮询

**所有图片生成任务必须遵循：**
```bash
# 1. --poll=0 立即获取submit_id
result=$(dreamina text2image --prompt="..." --ratio=1:1 --resolution_type=2k --poll=0)
submit_id=$(echo "$result" | python3 -c "import json,sys; print(json.load(sys.stdin)['submit_id'])")

# 2. while循环轮询query_result
while true; do
  status=$(dreamina query_result --submit_id=$submit_id | python3 -c "import json,sys; print(json.load(sys.stdin)['gen_status'])")
  [ "$status" = "success" ] && break
  [ "$status" = "failed" ] && exit 1
  sleep 10
done

# 3. 下载并发送图片（只发一次）
dreamina query_result --submit_id=$submit_id --download_dir=/tmp
```

详见 [references/01-dreamina-cli-usage.md](references/01-dreamina-cli-usage.md)

### 空场景铁律

场景图 = 纯背景空场景，角色在分镜阶段合成。禁止在场景图中出现故事角色。

## 外部依赖

| 工具 | 用途 | 说明 |
|------|------|------|
| [dreamina CLI](https://dreamina.cn) | 文生图/图生图 | 即梦官方CLI，2K免费 |
| 聚鑫 Veo 3.1 | 图生视频 | 8秒片段，视频主力工具 |
| MiniMax TTS | 语音合成 | 台词/旁白 |
| MiniMax Music | 配乐生成 | BGM |

## 参考文档

- [即梦CLI使用规范](references/01-dreamina-cli-usage.md) — R-010铁律详解
- [技术方案](references/02-tech-plan.md) — 系统架构、Agent设计、成本模型
- [项目存档](references/03-project-summary.md) — 开发进度、已知坑位、下一步

## 许可

MIT License

## 贡献

Issue和PR欢迎！如有多平台适配经验，欢迎分享。
