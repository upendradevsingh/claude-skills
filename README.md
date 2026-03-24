# Claude Skills

Language-agnostic testing skills for Claude Code. Principles are universal; code examples are loaded per-language (Python, Java, Node.js/TypeScript).

## Skills

| Skill | Description |
|-------|-------------|
| **writing-effective-tests** | Combines TDD process (RED-GREEN-REFACTOR) with test craft (type selection, mocking boundaries, naming, testability patterns). Language-specific examples loaded automatically. |
| **reviewing-tests** | Reviewer-focused skill — catches bad tests, overcomes reviewer biases (Green Bar, Coverage Number, Mock Confidence), and systematically evaluates test suites. |

## Supported Languages

| Language | Test Framework | Mocking | DB Testing | API Testing |
|----------|---------------|---------|------------|-------------|
| Python | pytest | unittest.mock | Testcontainers + SQLAlchemy | httpx TestClient |
| Java | JUnit 5 | Mockito | Testcontainers + Spring | MockMvc |
| Node.js/TS | Jest, Vitest | jest.mock, vi.mock | Testcontainers | supertest |

Adding a new language? See [GENERATION-GUIDE.md](GENERATION-GUIDE.md).

## Install

```bash
git clone https://github.com/upendradevsingh/claude-skills.git
cd claude-skills
chmod +x install.sh
./install.sh
```

This symlinks skills into `~/.claude/skills/` so Claude Code picks them up globally across all projects.

## Update

```bash
cd claude-skills
git pull
# Symlinks auto-update — no reinstall needed
```

## Uninstall

```bash
cd claude-skills
./uninstall.sh
```

## How It Works

The `install.sh` script creates symlinks from `~/.claude/skills/<skill-name>` to the corresponding directory in this repo. Because they're symlinks, `git pull` immediately updates all skills with no extra step.

```
~/.claude/skills/
    writing-effective-tests -> /path/to/claude-skills/skills/writing-effective-tests/
    reviewing-tests -> /path/to/claude-skills/skills/reviewing-tests/
```

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` with frontmatter:
   ```yaml
   ---
   name: skill-name
   description: Use when [triggering conditions]
   ---
   ```
2. Run `./install.sh` to link it
3. Test with Claude Code in any project

## License

MIT
