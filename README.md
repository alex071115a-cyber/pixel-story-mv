# pixel-story-mv · 像素叙事 MV

一个 Claude Skill：给你自己的歌（音频 + 带时间轴的 `.lrc` 歌词）做一支**有剧情的像素风音乐视频**。
每一帧都由 Python / Pillow 用代码绘制，不使用任何图像或视频生成模型。

A Claude skill that turns a song you already have (audio + synced `.lrc`) into a story-driven
pixel-art music video. Every frame is drawn in code (Python / Pillow); no image or video models are used.

## 效果 / What it makes

- 320×180 原生像素画面，24 fps，整数倍放大到 1920×1080 MP4
- 原创 Q 版角色（约 37 px 高，大头，深色描边，2–3 阶明暗）
- 米色歌词字幕框，字幕直接来自你的 LRC，镜头切换对齐歌词和节拍
- 相册式开场 / 结尾，片尾带 MV 署名卡
- 也可以**评审**一支已有的 MV：抽帧、查黑边、查卡点、查响度，并给出按性价比排序的修改建议

## 依赖 / Requirements

- Python 3 + [Pillow](https://pypi.org/project/pillow/)
- `ffmpeg` / `ffprobe`
- `librosa`（节拍检测，缺失时 skill 会提示安装）
- 一个支持中文的字体（默认 Noto Sans CJK，路径写在 skill 里的 `engine.py` 中，需要时修改 `FONT`）

## 安装 / Install

把整个文件夹放进你的 skills 目录，目录名保持 `pixel-story-mv`：

```bash
# Claude Code（个人级）
git clone https://github.com/<你的用户名>/pixel-story-mv.git ~/.claude/skills/pixel-story-mv

# 或只用于某个项目
git clone https://github.com/<你的用户名>/pixel-story-mv.git <项目目录>/.claude/skills/pixel-story-mv
```

其他支持 Skills 的产品，请按其文档把该文件夹作为 skill 上传或放入对应目录。

## 使用 / Usage

准备好歌曲音频和同名 `.lrc`，然后对 Claude 说：

- `给这首歌做像素风 MV`
- `/pixel-story-mv`
- `帮我评审一下这支 MV`（上传 .mp4）

流程：分析歌曲 → 逐句编排剧情 → 角色设定确认 → 先出约 30 秒片段 → 全曲渲染 → 质检 → 交付 MP4 与封面。

## 目录 / Layout

```
pixel-story-mv/
├── SKILL.md    # skill 本体：工作流、画面规范、角色规范，以及 chars.py / engine.py / demo.py 的完整代码
└── README.md
```

`chars.py`、`engine.py`、`demo.py` 以代码块形式内嵌在 `SKILL.md` 末尾，运行时由 Claude 写出到项目目录。

## 说明 / Notes

- 视频无法由 Claude "听"，音画同步需要你自己确认。
- 请只用你有权使用的歌曲和歌词。
