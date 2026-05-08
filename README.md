# Knowledge Base

算法工程师个人知识库，由 Claude 自动维护，人工只读。

## 目录结构

```
knowledge-base/
├── papers/          # 论文笔记（paper-digest 生成）
│   ├── LLM-蒸馏/
│   ├── Agent/
│   ├── 强化学习/
│   └── _待整理/
└── weekly/          # 周级论文扫描记录（weekly-papers 生成）
```

## 使用方式

- 论文笔记由 `paper-digest` skill 自动生成并 commit
- 周级扫描由 `weekly-papers` skill 自动生成并 commit
- 每次操作后自动 push 到远端
