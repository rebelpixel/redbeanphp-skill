# redbeanphp-skill

A [Claude skill](https://docs.claude.com) that teaches Claude how to write, review and debug code using [RedBeanPHP](https://redbeanphp.com), the zero-config PHP ORM.

It covers setup, CRUD, finders, relations, FUSE models, production hardening (freezing the schema), and the gotchas that most often trip people up. Examples were checked against RedBeanPHP 5.7 on PHP 8.3 with SQLite.

## Layout

```
redbeanphp/
├── SKILL.md                  # core workflow, API cheat sheet, gotchas
└── references/
    ├── relations.md          # own/shared/xown, aliases, via, tagging
    ├── production.md         # freeze, migrations, input handling, transactions
    └── models.md             # FUSE models and hooks
```

## Install

**Claude Code** (all projects):

```bash
git clone https://github.com/<you>/redbeanphp-skill.git
cp -r redbeanphp-skill/redbeanphp ~/.claude/skills/
```

For a single project, copy the `redbeanphp` folder into `.claude/skills/` inside that project.

**Claude.ai:** zip the `redbeanphp` folder (so `SKILL.md` sits inside it) and upload it under Settings > Capabilities > Skills.

## Contributing

Fixes welcome, especially for behaviour that differs on MySQL or PostgreSQL. Please keep `SKILL.md` under 500 lines and move detail into `references/`.

## License

MIT. RedBeanPHP itself is licensed separately (BSD/GPLv2) by its authors; this repository contains no RedBeanPHP code.
