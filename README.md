# /init-project — Slash command Claude Code

Initialise ou bootstrap n'importe quel projet avec une documentation structuree (CLAUDE.md, MEMORY.md, PRD, feature docs), une configuration Claude Code (permissions, hooks) et des conventions Git.

## Fonctionnalites

- **Deux modes** : `new` (projet vierge) ou `bootstrap` (projet existant — detecte et complete)
- Auto-detection de la stack, du package manager, de la base de donnees, du port et de Git
- Pre-remplissage intelligent des PRD a partir des sources existantes (mode bootstrap)
- Generation de 10 fichiers PRD, CLAUDE.md, MEMORY.md, settings, .gitignore
- Ne jamais ecraser les fichiers existants en mode bootstrap
- Interactif — demande confirmation a chaque etape

## Installation

1. Creer le dossier `~/.claude/commands/` s'il n'existe pas :
   ```bash
   mkdir -p ~/.claude/commands
   ```
   Sous Windows (PowerShell) :
   ```powershell
   New-Item -ItemType Directory -Path "$env:USERPROFILE\.claude\commands" -Force
   ```

2. Copier `init-project.md` dans `~/.claude/commands/`

3. Copier le dossier `init-project-templates/` (avec tout son contenu) dans `~/.claude/commands/`

La structure finale doit etre :
```
~/.claude/commands/
├── init-project.md
└── init-project-templates/
    ├── skill.md
    ├── CLAUDE_MD.template.md
    ├── MEMORY_MD.template.md
    ├── settings.local.json.template
    ├── settings.json.template
    ├── feature-doc.template.md
    ├── gitignore.template
    └── PRD/
        ├── 00-INDEX.md.template
        ├── 01-VISION.md.template
        ├── 02-PERSONAS.md.template
        ├── 03-FEATURES.md.template
        ├── 04-BACKLOG.md.template
        ├── 05-ROADMAP.md.template
        ├── 06-SPECIFICATIONS.md.template
        ├── 07-DECISIONS.md.template
        ├── 08-CHANGELOG.md.template
        └── 09-METRICS.md.template
```

## Utilisation

Dans n'importe quelle session Claude Code :

```
/init-project MonProjet
```

Ou sans argument (le nom du projet sera demande) :

```
/init-project
```

## Ce qui est genere

| Fichier | Description |
|---------|-------------|
| `CLAUDE.md` | Documentation technique avec Documentation Matrix et 6 directives |
| `MEMORY.md` | Memoire persistante entre sessions (50 lignes max) |
| `.claude/settings.local.json` | Permissions adaptees a la stack detectee |
| `.claude/settings.json` | Hooks projet |
| `docs/PRD/` | 10 fichiers PRD (INDEX, VISION, PERSONAS, FEATURES, BACKLOG, ROADMAP, SPECIFICATIONS, DECISIONS, CHANGELOG, METRICS) |
| `docs/features/README.md` | Index de la documentation par feature |
| `.gitignore` | Gitignore multi-stack |

## Pre-remplissage intelligent (mode bootstrap)

En mode bootstrap, le skill scanne les sources existantes du projet (CLAUDE.md, README.md, docs/*.md, package.json, etc.) et extrait le contenu factuel pour pre-remplir les fichiers PRD. Chaque extraction est presentee pour validation avant ecriture.

Les preconisations de fin d'execution sont contextuelles : elles ne mentionnent que les sections restant reellement a completer.
