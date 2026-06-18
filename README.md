# Agent Skills

[中文](README.zh-CN.md)

This repository is a personal collection of reusable Agent Skills.

Each skill lives in its own directory and uses `SKILL.md` as its entrypoint. New skills can be added without changing the existing structure.

## Layout

```text
.
├── README.md
├── README.zh-CN.md
└── ssh-remote-ops/
    └── SKILL.md
```

Future skills should follow the same pattern:

```text
skill-name/
└── SKILL.md
```

## Skills

- `ssh-remote-ops`: Guides an agent to connect to remote servers from the local machine through SSH or rsync, while asking for missing connection details instead of hardcoding credentials or host information.
