# /init-project — Skill d'initialisation projet pour Claude Code

Skill Claude Code qui orchestre le setup documentaire, organisationnel et qualite complet d'un projet.

## Ce que fait ce skill

- Genere un **CLAUDE.md** structure avec matrice documentaire, standards de documentation et conventions Git
- Cree un **PRD complet** (10 fichiers : vision, personas, features, backlog, roadmap, specs, decisions, changelog, metriques)
- Configure **Claude Code** (permissions par stack, hooks, regles hookify)
- Initialise un **MEMORY.md** pour la persistance entre sessions
- Prepare la structure **ADR** (Architecture Decision Records) au format MADR
- Configure **ESLint + eslint-plugin-jsdoc** pour la documentation obligatoire du code (TypeScript)
- Genere un `.gitignore` adapte et fait le commit initial
- Produit un **rapport de fin** avec preconisations d'usage

## Deux modes d'execution

| Mode | Declenchement | Comportement |
|------|---------------|--------------|
| `new` | Projet vierge (pas de package.json, src/, .git) | Cree tout depuis zero |
| `bootstrap` | Projet existant detecte | Analyse l'existant, identifie les manquements, complete sans ecraser |

> Le mode est auto-detecte puis **soumis a confirmation** — vous pouvez toujours choisir un mode different de celui suggere.

## Mode adaptatif : Hub vs Standalone

Le skill detecte automatiquement s'il est lance dans un environnement hub (infrastructure partagee) ou de facon autonome.

| Mode | Detection | Comportement |
|------|-----------|--------------|
| **Hub** | `_shared/hooks/`, `_shared/agents/`, `_shared/config/` trouves en remontant l'arborescence | Integre les hooks hub, copie les hookify rules, reference les agents et MCP pertinents |
| **Standalone** | Aucune infrastructure hub detectee | Genere un projet autonome et complet sans dependance externe |

La detection est automatique — l'utilisateur n'a rien a configurer.

## Mode audit : --check

```
/init-project --check
```

N'ecrit aucun fichier. Effectue un audit de conformite du projet et produit un rapport d'ecarts avec preconisations de mise en conformite. Utile pour mettre a jour un projet existant.

## Installation

1. Copier le dossier `init-project/` dans votre repertoire de skills Claude Code :
   ```
   ~/.claude/skills/init-project/
   ```
   Ou dans un repertoire partage si vous en avez un.

2. Verifier que le dossier contient :
   ```
   init-project/
   ├── skill.md              # Le skill (workflow)
   ├── README.md             # Ce fichier
   └── templates/
       ├── CLAUDE_MD.template.md
       ├── MEMORY_MD.template.md
       ├── settings.local.json.template
       ├── settings.json.template
       ├── eslint.config.mjs.template
       ├── adr-README.template.md
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

Dans Claude Code, a la racine de votre projet :

```
/init-project MonProjet
```

Le skill va :
1. Detecter le mode (new/bootstrap) et vous **demander confirmation**
2. Analyser le projet pour detecter stack, package manager, database, port
3. Detecter l'environnement (hub ou standalone)
4. Afficher un resume de detection et vous demander de completer les infos manquantes (optionnel)
5. Generer les fichiers de documentation, configuration et outillage qualite
6. Afficher un rapport final avec preconisations d'usage

## Structure generee

```
projet/
├── CLAUDE.md                    # Doc technique (matrice documentaire + standards)
├── README.md                    # Presentation projet
├── .env.example                 # Variables d'environnement
├── .gitignore                   # Gitignore multi-stack
├── eslint.config.mjs            # ESLint + JSDoc (si TypeScript)
├── .claude/
│   ├── settings.json            # Hooks (scope-check)
│   ├── settings.local.json      # Permissions par stack
│   └── hookify.*.local.md       # Regles qualite (mode hub)
└── docs/
    ├── prd/                     # Product Requirements (10 fichiers)
    │   ├── 00-INDEX.md
    │   ├── 01-VISION.md
    │   ├── ...
    │   └── 09-METRICS.md
    ├── adr/                     # Architecture Decision Records
    │   └── README.md            # Template MADR + index
    ├── guides/                  # Guides techniques
    └── assets/                  # Images, schemas, diagrammes
```

## Detection automatique

Le skill detecte les informations **factuellement presentes** dans le projet. Il ne fait aucune supposition — si une information n'est pas trouvee, elle est marquee "non detectee" et l'utilisateur peut la renseigner ou la laisser vide.

| Element | Methode de detection |
|---------|---------------------|
| Stack | Dependances dans `package.json`, `requirements.txt`, `pyproject.toml` |
| Package manager | Lockfile (`package-lock.json`, `pnpm-lock.yaml`, `bun.lockb`, `yarn.lock`) |
| Database/ORM | Dependances (prisma, supabase, mongoose) + fichiers (docker-compose, .env) |
| Port | `.env`, scripts dans `package.json`, config framework |
| Git | Presence de `.git` et remote configure |
| Hub | Presence de `_shared/hooks/`, `_shared/agents/`, `_shared/config/` dans l'arborescence parente |

### Stacks reconnues

| Stack | Dependance detectee |
|-------|---------------------|
| Next.js | `next` |
| SvelteKit | `svelte` ou `@sveltejs/kit` |
| Astro | `astro` |
| Node API | `express`, `fastify` ou `hono` |
| Python | `requirements.txt` ou `pyproject.toml` |

## Personnalisation

### Modifier les templates

Les templates dans `./templates/` sont des fichiers Markdown avec des variables `{{VARIABLE}}`. Vous pouvez les modifier pour adapter le contenu genere a vos conventions.

### Variables disponibles

| Variable | Obligatoire | Description |
|----------|-------------|-------------|
| `{{PROJECT_NAME}}` | Oui | Nom du projet |
| `{{PROJECT_DESCRIPTION}}` | Oui | Description courte |
| `{{PROJECT_PATH}}` | Oui | Chemin absolu |
| `{{DATE}}` | Oui | Date du jour |
| `{{STACK}}` | Non | Stack technique (omise si non detectee) |
| `{{DATABASE}}` | Non | Type de BDD (omise si non detectee) |
| `{{PORT}}` | Non | Port de dev (omis si non detecte) |
| `{{PACKAGE_MANAGER}}` | Non | npm/pnpm/bun/yarn (omis si non detecte) |
| `{{HUB_PATH}}` | Non | Chemin du hub (omis en mode standalone) |

### Ajouter une stack

Pour supporter une nouvelle stack, modifier dans `skill.md` :
1. L'auto-detection (etape 0.2, section "Stack")
2. Les permissions (etape 2, table des stacks)
3. La generation de `{{STACK_TABLE}}` et `{{COMMANDS}}` (etape 4)

## Prerequis

- Claude Code installe
- Git installe (optionnel — les etapes Git sont ignorees si absent)
- npm/pnpm/bun/yarn pour l'installation de eslint-plugin-jsdoc (optionnel — ignoree si pas de package manager)
- Aucune autre dependance externe

## Licence

Libre d'utilisation et de modification.
