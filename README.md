# Claude Skills

Shared [Claude Code](https://claude.com/claude-code) skills for WordPress plugin work.

| Skill | What it does |
|-------|--------------|
| [`compare-code`](skills/compare-code/SKILL.md) | Compares the current branch with `master`/`main`, reviews only the changed code (WPCS, PSR-4/PSR-12, security, Gutenberg, REST, performance, upgrade path), gives a release decision, and writes a readme.txt-style changelog. |

## Install (once)

Clone the repo, then link each skill into your personal skills folder so `git pull` updates it:

```bash
git clone https://github.com/bayejid00/claude-skills.git ~/claude-skills
mkdir -p ~/.claude/skills
ln -s ~/claude-skills/skills/compare-code ~/.claude/skills/compare-code
```

Windows (PowerShell, as administrator or with Developer Mode on):

```powershell
git clone https://github.com/bayejid00/claude-skills.git $HOME\claude-skills
New-Item -ItemType SymbolicLink -Path $HOME\.claude\skills\compare-code -Target $HOME\claude-skills\skills\compare-code
```

Restart Claude Code. Use it by typing `/compare-code`, or ask "is this branch release ready?".

## Update

```bash
git -C ~/claude-skills pull
```

## Contributing

Edit `skills/<name>/SKILL.md`, bump `version:` in its frontmatter, and open a pull request.
