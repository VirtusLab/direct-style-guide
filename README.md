# Claude Code Skills

A collection of skills for [Claude Code](https://claude.ai/claude-code). Written by AI, for AI and humans alike.

## Available skills

- **[direct-style-scala](direct-style-scala/)** — Use-case driven guide to
  writing direct-style Scala 3 applications with virtual threads (Java 21+),
  using [Tapir](https://tapir.softwaremill.com),
  [Ox](https://ox.softwaremill.com), and
  [sttp](https://sttp.softwaremill.com).

## Installation

### Add the marketplace

```
/plugin marketplace add VirtusLab/direct-style-guide
```

### Install the skill

```
/plugin install direct-style-scala@virtuslab-scala-skills
```

### Manual installation

```bash
git clone https://github.com/VirtusLab/direct-style-guide.git /tmp/direct-style-guide
cp -r /tmp/direct-style-guide/direct-style-scala ~/.claude/skills/
```
