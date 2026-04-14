# Installation

## Prerequisites

- **Claude Code** — the skill runs as a Claude Code plugin
- **Python 3.10+** — for validation and prep scripts (stdlib only, no pip installs)
- **pdflatex + bibtex** — optional, for final PDF compilation

### Installing LaTeX (optional)

macOS:
```bash
brew install --cask mactex-no-gui
```

Ubuntu/Debian:
```bash
sudo apt-get install texlive-full
```

## Install the Skill Pack

### Option A: Clone and symlink

```bash
git clone https://github.com/athan-dial/paperorchestra.git
ln -s "$(pwd)/paperorchestra" ~/.claude/skills/paper-orchestra
```

### Option B: Copy into skills directory

```bash
git clone https://github.com/athan-dial/paperorchestra.git
cp -r paperorchestra ~/.claude/skills/paper-orchestra
```

## Verify Installation

In Claude Code, run:
```
/paper-orchestra
```

The skill should prompt for your paper idea and materials path.

## Updating

```bash
cd /path/to/paperorchestra
git pull
```

If you symlinked, updates apply immediately. If you copied, re-copy after pulling.
