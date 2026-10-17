# code-generated MV by AISZWL

> 用**纯代码**生成完整歌曲 MV 的技能：每一帧都是时间 T 的确定性纯函数，程序化绘制逐帧渲染 + ffmpeg 合成。
> 不调用视频生成模型，不使用图片/视频素材。

## 这是什么

一套被完整验证过的代码 MV 生产流程，源自《达拉崩吧》纸艺 RPG 代码 MV 的复现实践：

**取歌词时间轴 → 导演任务书 → 多智能体分镜设计（3 提案 → 2 评审 → 1 合成）→ 基础资产并行 → 接缝合同 → 场景块并行 → 自动化验收 → 逐帧渲染出片**

核心文档是 [`references/reproduction-handbook.md`](references/reproduction-handbook.md)（完全复现手册：引擎规格、
工具链、分块与接缝、验收方法、踩坑全集、复现清单），技能入口见 [`SKILL.md`](SKILL.md)。

## 配套示例：《不凡》（《凡人修仙传》片头曲）

[`examples/fanren-xiuxian-zhuan/`](examples/fanren-xiuxian-zhuan/) 给出一套可直接套用的模板素材：
角色表（主角/盟友/反派阵营与六位角色的视觉规格）、六张皮克斯风 Q 版角色定妆图、
按歌曲真实结构（经词级转写校准，含两段长器乐间奏）编排的分镜表。

| # | 角色 | 阵营 | 模板图 |
|---|---|---|---|
| 1 | 韩立（绿袍主角，镜头最多） | 主角 | [IMG_2003.jpeg](examples/fanren-xiuxian-zhuan/characters/IMG_2003.jpeg) |
| 2 | 红裙少女 | 主角阵营 | [IMG_2004.jpeg](examples/fanren-xiuxian-zhuan/characters/IMG_2004.jpeg) |
| 3 | 狐耳少女 | 主角阵营 | [IMG_2005.jpeg](examples/fanren-xiuxian-zhuan/characters/IMG_2005.jpeg) |
| 4 | 黑袍法杖反派 | 反派阵营 | [IMG_2006.jpeg](examples/fanren-xiuxian-zhuan/characters/IMG_2006.jpeg) |
| 5 | 青纱女反派 | 反派阵营 | [IMG_2007.jpeg](examples/fanren-xiuxian-zhuan/characters/IMG_2007.jpeg) |
| 6 | 山羊胡反派 | 反派阵营 | [IMG_2008.jpeg](examples/fanren-xiuxian-zhuan/characters/IMG_2008.jpeg) |

## 仓库结构

```
SKILL.md                                技能入口（给 AI agent 读的工作流）
references/reproduction-handbook.md     完全复现手册
examples/fanren-xiuxian-zhuan/
├── cast.md                             角色表：阵营、视觉规格、镜头配额
├── storyboard.md                       分镜表：按歌曲结构逐段编排
└── characters/                         六张角色模板图
```

## 使用

把本仓库作为 skill 交给你的 AI agent，并给出：目标歌曲音频、带时间戳歌词、角色参考图、想要的画风。
Agent 会按 SKILL.md 的流程执行，手册提供全部实现细节与验收标准。

## 版权说明

流程与文档可自由复用。示例角色图与所涉歌曲（《不凡》，王铮亮演唱，《凡人修仙传》动画片头曲）
版权归原权利人；生成的 MV 仅供个人观看，公开发布请署名原词曲/演唱并遵守平台二创规则。

*By AISZWL（AI塑造未来）*
