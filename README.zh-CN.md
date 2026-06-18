# Agent Skills

[English](README.md)

这个仓库用于保存个人可复用的 Agent Skills。

每个 skill 都放在独立目录中，并以 `SKILL.md` 作为入口文件。后续可以继续添加新的 skill，而不需要改变现有结构。

## 目录

```text
.
├── README.md
├── README.zh-CN.md
└── ssh-remote-ops/
    └── SKILL.md
```

后续新增 skill 时，建议继续使用相同结构：

```text
skill-name/
└── SKILL.md
```

## Skills

- `ssh-remote-ops`：指导智能体通过本地 SSH 或 rsync 连接远端服务器；不会写死凭据或主机信息，缺少连接参数时会先要求用户提供。
