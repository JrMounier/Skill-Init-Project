---
name: init-project
description: Initialise ou bootstrap un projet avec documentation structuree (CLAUDE.md, MEMORY.md, PRD, ADR), configuration Claude Code (permissions, hooks, hookify) et conventions Git. Fonctionne en mode standalone ou hub.
trigger: /init-project
auto_detect: false
---

# Skill : /init-project — Initialisation projet

## Description

Ce skill orchestre le setup documentaire et organisationnel complet d'un projet. Il genere la documentation technique (CLAUDE.md, PRD, ADR), configure Claude Code (permissions, hooks, hookify) et etablit les conventions Git.

Deux modes :
- **`new`** : Projet vierge — cree toute la structure depuis zero
- **`bootstrap`** : Projet existant — analyse l'environnement, detecte les existants et les manquements, complete sans ecraser

Deux environnements (detection automatique) :
- **Mode Hub** : Infrastructure partagee detectee — hooks, agents, MCP, hookify depuis le hub
- **Mode Standalone** : Projet autonome sans dependance externe

## Declenchement

```
/init-project [nom du projet]
/init-project --check [nom du projet]   # Mode audit (aucune ecriture)
```

## Templates

Les templates sont co-distribues avec le skill dans `~/.claude/commands/init-project-templates/` :

| Template | Destination | Description |
|----------|-------------|-------------|
| `CLAUDE_MD.template.md` | `./CLAUDE.md` | Doc technique avec Matrice documentaire, Standards de Documentation et 6 directives |
| `MEMORY_MD.template.md` | `~/.claude/projects/.../memory/MEMORY.md` | Memoire persistante (50 lignes max) |
| `settings.local.json.template` | `./.claude/settings.local.json` | Permissions adaptees a la stack |
| `settings.json.template` | `./.claude/settings.json` | Hooks projet |
| `eslint.config.mjs.template` | `./eslint.config.mjs` | Config ESLint avec jsdoc (TypeScript) |
| `gitignore.template` | `./.gitignore` | Gitignore multi-stack |
| `PRD/*.template` | `docs/prd/` | 10 fichiers PRD |
| `marketing-context-template.md` | `docs/marketing-context.md` | Contexte marketing (optionnel) |

## Variables de substitution

| Variable | Source | Obligatoire | Description |
|----------|--------|-------------|-------------|
| `{{PROJECT_NAME}}` | Collecte | Oui | Nom du projet |
| `{{PROJECT_DESCRIPTION}}` | Collecte | Oui | Description courte |
| `{{PROJECT_PATH}}` | Detection | Oui | Chemin absolu du projet (repertoire courant) |
| `{{DATE}}` | Automatique | Oui | Date du jour (YYYY-MM-DD) |
| `{{HUB_PATH}}` | Detection hub | Non | Chemin absolu du hub (si detecte) |
| `{{HUB_MODE}}` | Detection hub | Non | `hub` ou `standalone` |
| `{{STACK}}` | Detection | Non | Stack technique — omise si non detectee/renseignee |
| `{{DATABASE}}` | Detection | Non | Type de base de donnees — omise si non detectee |
| `{{PORT}}` | Detection | Non | Port de dev — omis si non detecte |
| `{{PACKAGE_MANAGER}}` | Detection lockfile | Non | npm/pnpm/bun/yarn — omis si non detecte |
| `{{STACK_TABLE}}` | Genere selon deps reelles | Non | Tableau stack — "A completer" si rien detecte |
| `{{INFRASTRUCTURE}}` | Detection | Non | Description infra (Docker, services, etc.) |
| `{{INFRASTRUCTURE_SUMMARY}}` | Genere | Non | Resume infra pour MEMORY.md |
| `{{ENV_VARS}}` | Detection .env | Non | Variables d'environnement (sans valeurs sensibles) |
| `{{COMMANDS}}` | Genere selon outils detectes | Non | Commandes utiles — "A completer" si rien detecte |
| `{{PROJECT_STRUCTURE}}` | Detection | Non | Arborescence du projet |
| `{{MEMORY_STACK_LINE}}` | Conditionnel | Non | Ligne Stack pour MEMORY.md (omise si non detectee) |
| `{{MEMORY_PORT_LINE}}` | Conditionnel | Non | Ligne Port pour MEMORY.md (omise si non detecte) |
| `{{MEMORY_PKG_LINE}}` | Conditionnel | Non | Ligne Package manager pour MEMORY.md (omise si non detecte) |
| `{{MEMORY_DB_LINE}}` | Conditionnel | Non | Ligne Database pour MEMORY.md (omise si non detectee) |

> Les variables marquees "Non" obligatoire produisent une valeur vide ou "A completer" si l'information n'est pas disponible. Aucune supposition n'est faite.

---

## Workflow d'execution

### Etape 0 — Detection, confirmation et collecte

#### 0.1 — Detection du mode

Inspecter le repertoire courant :
- Si `package.json` ou `src/` ou `.git` existe → mode suggere `bootstrap`
- Sinon → mode suggere `new`

Afficher le resultat et **demander confirmation explicite** via AskUserQuestion :
```
Mode detecte : **[new/bootstrap]**

Quel mode souhaitez-vous executer ?
- new : Creer toute la structure depuis zero
- bootstrap : Analyser l'existant et completer les manquants
```

L'utilisateur peut choisir un mode different de celui detecte. Attendre sa reponse avant de continuer.

#### 0.2 — Auto-detection (factuelle uniquement)

Inspecter le projet courant et **ne collecter que ce qui est effectivement present**. Ne rien inventer, ne rien supposer.

- **Stack** : lire `package.json` dependencies (si le fichier existe)
  - `next` → `nextjs`
  - `svelte` ou `@sveltejs/kit` → `sveltekit`
  - `astro` → `astro`
  - `express` ou `fastify` ou `hono` → `node-api`
  - `django` ou `flask` (via `requirements.txt`/`pyproject.toml`) → `python`
  - Aucun fichier de dependances ou aucun match → `non detectee`
- **Package manager** : detecter lockfile
  - `package-lock.json` → npm
  - `pnpm-lock.yaml` → pnpm
  - `bun.lockb` → bun
  - `yarn.lock` → yarn
  - `requirements.txt` ou `pyproject.toml` → pip/poetry
  - Aucun lockfile → `non detecte`
- **Database/ORM** : detecter dans les dependances et fichiers existants
  - `prisma` dans devDependencies + dossier `prisma/` → Prisma detecte
  - `@supabase/supabase-js` dans dependencies → Supabase detecte
  - `mongoose` ou `mongodb` dans dependencies → MongoDB detecte
  - `docker-compose.yml` contenant `postgres` → PostgreSQL detecte
  - Rien de detecte → `non detectee`
- **Git** : verifier `.git`, lire remote si existe (`git remote -v`)
- **Port** : chercher dans (ordre de priorite) :
  1. `.env` ou `.env.local` (variable `PORT` ou `NEXT_PUBLIC_PORT`)
  2. `package.json` scripts.dev (`--port` ou `-p`)
  3. Config framework (`next.config.*`, `vite.config.*`, `astro.config.*`)
  4. Rien de detecte → `non detecte`

> **Regle absolue** : Si une information n'est pas detectee, elle est marquee `non detectee`.
> Aucune valeur par defaut, aucune recommandation, aucun choix arbitraire ne doit etre fait.
> L'utilisateur sera invite a completer les informations manquantes s'il le souhaite.

#### 0.3 — Audit de l'existant (mode bootstrap uniquement)

En mode bootstrap, **avant d'afficher le resume**, effectuer un audit complet :

**Audit documentaire** :
- Lister les fichiers de documentation existants (CLAUDE.md, README.md, docs/, .claude/)
- Detecter les configurations presentes (settings.json, settings.local.json, hooks)
- Verifier la presence de PRD (compter les fichiers dans docs/prd/), ADR, .gitignore
- Preparer le resume : "Existants detectes : X | Manquants : Y"

**Scan des sources de pre-remplissage** :

Scanner le projet pour identifier les fichiers contenant des informations exploitables pour pre-remplir les PRD. Pour chaque source trouvee, noter son chemin et le type d'information qu'elle contient.

Sources a scanner (dans l'ordre) :

| Source | Quoi chercher |
|--------|---------------|
| `CLAUDE.md` | Personas, objectifs/KPIs, stack, features, historique sessions, decisions techniques, architecture, commandes |
| `README.md` | Description projet, objectifs, instructions, architecture |
| `docs/*.md` (hors prd/) | PRD legacy, specs, phases, charte graphique, identite visuelle |
| `docs/**/*.md` (hors prd/) | Documentation approfondie (features, integration, specs detaillees) |
| `package.json` | Description, scripts, dependances (versions reelles) |
| `.env` / `.env.example` | Variables d'environnement (noms uniquement, pas de valeurs sensibles) |
| `docker-compose.yml` | Services, ports, infrastructure |

Pour chaque source trouvee, extraire un inventaire structure :

```
## Sources de pre-remplissage detectees

| Source | Contenu exploitable | PRD cibles |
|--------|--------------------|-----------|
| CLAUDE.md | 2 personas, 4 KPIs, stack complete, 8 sessions historique | 01, 02, 03, 06, 08, 09 |
| docs/PRD-legacy.md | Vision, objectifs, personas detailles | 01, 02 |
| docs/Phase1-*.md | Decisions architecture, design system | 07 |
```

> **Regle absolue** : Ne lister que les fichiers **effectivement presents**. Ne pas inventer de contenu.
> Le contenu sera extrait et propose a l'utilisateur a l'etape 0.6.

> En mode `new`, cette sous-etape est ignoree.

#### 0.4 — Affichage du resume de detection

Afficher un tableau recapitulatif de tout ce qui a ete detecte :

```
## Resume de detection

| Element | Valeur detectee |
|---------|-----------------|
| Stack | nextjs / non detectee |
| Package manager | npm / non detecte |
| Database/ORM | Prisma + PostgreSQL / non detectee |
| Port | 4500 / non detecte |
| Git | Oui (remote: xxx) / Non |
| Environnement | Hub ([chemin]) / Standalone |
```

En mode bootstrap, ajouter le resume de l'audit documentaire et des sources de pre-remplissage :
```
## Audit documentaire

| Element | Statut |
|---------|--------|
| CLAUDE.md | Present / Absent |
| .claude/settings.local.json | Present / Absent |
| .claude/settings.json | Present / Absent |
| docs/prd/ | X/10 fichiers |
| docs/adr/ | Present / Absent |
| docs/guides/ | Present / Absent |
| docs/assets/ | Present / Absent |
| .gitignore | Present / Absent |
| MEMORY.md | Present / Absent |

Existants : X | Manquants : Y

## Sources de pre-remplissage

| Source | Contenu exploitable | PRD cibles |
|--------|--------------------|-----------|
| [fichier] | [resume du contenu] | [numeros PRD] |
| ... | ... | ... |

Pre-remplissage disponible pour X/10 fichiers PRD.
```

Si aucune source de pre-remplissage n'est detectee, afficher :
```
Aucune source de pre-remplissage detectee. Les PRD seront generes comme templates vides.
```

#### 0.5 — Collecte des informations manquantes

Collecter via AskUserQuestion **uniquement les informations obligatoires** :
- Nom du projet (obligatoire, toujours demande)
- Description courte (obligatoire, toujours demande)

Pour les informations techniques non detectees, demander a l'utilisateur **s'il souhaite les renseigner** (optionnel) :
- Stack : proposer `nextjs` / `sveltekit` / `astro` / `node-api` / `python` / `autre` / `ne pas renseigner` — uniquement si non detectee
- Base de donnees : proposer `PostgreSQL` / `Supabase` / `SQLite` / `MongoDB` / `ne pas renseigner` — uniquement si non detectee
- Port de dev : demander uniquement si non detecte

**Option marketing** (toujours proposee) :
- Generer un `marketing-context.md` ? → `Oui` / `Non`
- Si Oui : le template `_shared/templates/marketing-context-template.md` sera copie dans `docs/marketing-context.md` a l'etape 7b
- Ce fichier sert de contexte aux skills marketing du hub (`marketing:*`, `brand-voice`, `content-creation`)

> Les informations non renseignees seront **omises** des fichiers generes (sections vides ou marquees "A completer").
> Elles pourront etre ajoutees ulterieurement en editant les fichiers concernes.

En mode bootstrap, ne poser des questions que pour les informations **non deductibles de l'existant**.

#### 0.6 — Pre-remplissage intelligent (mode bootstrap uniquement)

> **Objectif** : Rendre la documentation generee immediatement operationnelle en transferant
> le contenu existant dans les fichiers PRD, au lieu de produire des coquilles vides.
> En mode `new`, cette sous-etape est ignoree (aucune source disponible).

Si des sources de pre-remplissage ont ete detectees a l'etape 0.3, proceder comme suit :

**Principe** : Pour chaque fichier PRD, lire les sources pertinentes, extraire les informations factuelles, et les presenter a l'utilisateur pour validation **avant toute ecriture**.

**Regle absolue** : Ne transferer que du contenu **effectivement present** dans les sources.
Ne jamais inventer, reformuler de maniere speculative ou extrapoler. Si une information
est ambigue ou incomplete dans la source, la transferer telle quelle avec la mention
`*(source: [fichier], a verifier)*`.

**Matrice d'extraction** :

| PRD cible | Quoi extraire | Ou chercher (par priorite) |
|-----------|--------------|---------------------------|
| `01-VISION.md` | Problem statement, objectifs, proposition de valeur, scope, contraintes | CLAUDE.md (description, objectifs), README.md, docs/*PRD*.md, docs/*vision*.md |
| `02-PERSONAS.md` | Personas (nom, role, age, pain points, besoins, comportements) | CLAUDE.md (section personas), docs/*PRD*.md, docs/*persona*.md |
| `03-FEATURES.md` | Liste de features avec ID, nom, priorite, statut (done/in-progress/planned) | CLAUDE.md (composants, historique sessions, prochaines etapes), docs/*phase*.md |
| `04-BACKLOG.md` | Taches a faire, en cours, terminees, avec priorite MoSCoW | CLAUDE.md (prochaines etapes, TODO), docs/*phase*.md, docs/*backlog*.md |
| `05-ROADMAP.md` | Phases du projet, milestones, dates | CLAUDE.md (historique sessions, phases), docs/*phase*.md, docs/*roadmap*.md |
| `06-SPECIFICATIONS.md` | Architecture, stack (versions reelles), APIs, schema de donnees, infrastructure | CLAUDE.md (stack, structure, commandes), package.json, docker-compose.yml, .env |
| `07-DECISIONS.md` | Index des ADR — decisions techniques documentees | CLAUDE.md (decisions, conventions, design system), docs/*decision*.md, docs/*phase*.md |
| `08-CHANGELOG.md` | Historique des changements, sessions de travail, releases | CLAUDE.md (historique sessions), git log (10 derniers commits) |
| `09-METRICS.md` | KPIs, objectifs chiffres, metriques cibles | CLAUDE.md (objectifs, KPIs), docs/*PRD*.md, docs/*metrics*.md |
| `00-INDEX.md` | Genere automatiquement apres les autres — refleter le statut reel de chaque fichier | Resultat du pre-remplissage des 9 autres fichiers |

**Processus d'extraction pour chaque fichier PRD** :

1. **Lire** les sources pertinentes (colonne "Ou chercher")
2. **Extraire** le contenu factuel correspondant au PRD cible
3. **Structurer** le contenu extrait dans le format du template PRD
4. **Presenter** a l'utilisateur via un resume clair :

```
## Pre-remplissage : [NOM_PRD].md

Sources analysees : [liste des fichiers lus]

### Contenu extrait :

[Apercu du contenu qui sera injecte dans le fichier]

### Sections restant a completer :

- [Liste des sections du template qui restent vides]

Valider ce pre-remplissage ?
- Oui : integrer tel quel
- Oui avec modifications : [l'utilisateur peut ajuster]
- Non : generer le template vide
```

5. **Attendre** la validation utilisateur avant de passer au PRD suivant

**Regroupement** : Si le nombre de PRD a pre-remplir est >= 5, proposer a l'utilisateur
de valider par lot (tous en une fois) plutot qu'un par un, en affichant un resume global :

```
## Resume du pre-remplissage (X/10 fichiers)

| PRD | Sections remplies | Sections vides | Sources |
|-----|-------------------|----------------|---------|
| 01-VISION | 3/5 | Contraintes, Scope | CLAUDE.md, docs/PRD-legacy.md |
| 02-PERSONAS | 2/3 | Persona 3 | CLAUDE.md |
| ... | ... | ... | ... |

Valider l'ensemble du pre-remplissage ?
- Oui pour tous
- Revoir un par un
- Non, generer des templates vides
```

**Stockage** : Les contenus valides par l'utilisateur sont conserves en memoire pour etre
injectes a l'etape 6 (generation PRD) a la place des placeholders du template.

**Suivi** : Maintenir un registre de ce qui a ete pre-rempli vs ce qui reste vide.
Ce registre sera utilise a l'etape 10 pour generer des preconisations contextuelles.

```
prefill_registry = {
  "01-VISION":     { filled: ["problem_statement", "objectifs", "valeur"], empty: ["scope", "contraintes"] },
  "02-PERSONAS":   { filled: ["persona_1", "persona_2"], empty: ["persona_3"] },
  ...
  "09-METRICS":    { filled: ["kpis"], empty: ["metriques_actuelles"] }
}
```

#### 0.7 — Detection environnement hub

Remonter l'arborescence depuis le repertoire courant pour detecter un hub :
- Chercher `_shared/hooks/` → si trouve, hub avec hooks disponible
- Chercher `_shared/agents/` → si trouve, agents disponibles
- Chercher `_shared/config/` → si trouve, configuration multi-agents

Si hub detecte :
  - Mode Hub actif — les etapes conditionnelles hub seront executees
  - Chemin hub : [chemin detecte]
  - Hooks disponibles : [liste]
  - Agents : [nombre]

Si aucun hub detecte :
  - Mode Standalone — generation autonome sans dependance externe

Afficher le resultat dans le resume de detection (etape 0.4).

### Etape 1 — Structure repertoires

**Mode `new`** :
```bash
mkdir -p src .claude docs/prd docs/adr docs/guides docs/assets
```

**Mode `bootstrap`** :
```bash
# Creer uniquement les repertoires manquants
mkdir -p .claude docs/prd docs/adr docs/guides docs/assets
```

Ne **jamais** supprimer ou ecraser de fichiers existants.

### Etape 2 — `.claude/settings.local.json`

Partir du template `settings.local.json.template` (permissions de base : git, lecture, ecriture, systeme).

**Ajouter les permissions specifiques uniquement si la stack a ete detectee ou renseignee** :

| Stack | Permissions supplementaires |
|-------|----------------------------|
| `nextjs` | `Bash(npm run:*)`, `Bash(npx:*)`, `Bash(npx prisma:*)`, `Bash(npx tsc:*)` |
| `sveltekit` | `Bash(npm run:*)`, `Bash(npx:*)`, `Bash(npx svelte-kit:*)` |
| `astro` | `Bash(npm run:*)`, `Bash(npx:*)`, `Bash(npx astro:*)` |
| `node-api` | `Bash(npm run:*)`, `Bash(npx:*)`, `Bash(node:*)` |
| `python` | `Bash(python:*)`, `Bash(pip:*)`, `Bash(pytest:*)` |
| `non detectee` | Aucun ajout — ne garder que les permissions de base |

**Ajustements package manager** (uniquement si detecte) :
| Package manager | Remplacements |
|-----------------|---------------|
| `bun` | Remplacer `npm run` par `bun run`, ajouter `Bash(bun:*)` |
| `pnpm` | Remplacer `npm run` par `pnpm run`, ajouter `Bash(pnpm:*)` |
| `yarn` | Remplacer `npm run` par `yarn`, ajouter `Bash(yarn:*)` |
| `non detecte` | Aucun ajout specifique |

Si `.claude/settings.local.json` existe deja, **ne pas ecraser**. Proposer de fusionner les permissions manquantes.

### Etape 3 — `.claude/settings.json` (hooks)

Generer depuis `settings.json.template`. Le contenu varie selon l'environnement detecte.

**Mode Hub** (hub detecte a l'etape 0.7) :

Utiliser le script Python `scope_check.py` du hub et ajouter `additionalDirectories` :

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "python {{HUB_PATH}}/_shared/hooks/scope_check.py {{PROJECT_NAME}}",
        "timeout": 5000
      }]
    }]
  },
  "permissions": {
    "additionalDirectories": ["{{HUB_PATH}}"]
  }
}
```

**Mode Standalone** (aucun hub detecte) :

Utiliser un echo inline basique, SANS `additionalDirectories` :

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "echo {\"systemMessage\": \"Vous modifiez un fichier hors du repertoire projet. Confirmez l'action.\"}",
        "timeout": 5000
      }]
    }]
  }
}
```

Si `.claude/settings.json` existe deja :
- Lire le contenu existant
- Proposer d'ajouter les hooks manquants **sans ecraser les hooks existants**
- Attendre confirmation utilisateur

### Etape 3b — Regles Hookify (mode hub uniquement)

> En mode standalone, cette etape est ignoree.

Si un hub a ete detecte (etape 0.7), copier les regles hookify depuis le hub.

Chercher les fichiers `hookify.*.local.md` dans le projet le plus proche du hub
(par exemple dans `{{HUB_PATH}}/FindObjects/.claude/` ou le premier projet avec des hookify).

Regles a copier :

- `hookify.jsdoc-required.local.md`
- `hookify.no-typescript-any.local.md`
- `hookify.protect-main-branch.local.md`
- `hookify.anti-overengineering.local.md`
- `hookify.dangerous-commands.local.md`
- `hookify.french-docx-accents.local.md`
- `hookify.checklist-before-stop.local.md`
- `hookify.update-claudemd-session.local.md`

Pour chaque fichier :
- Si le fichier n'existe pas dans le projet cible `.claude/` → copier
- Si le fichier existe deja → `[SKIP]`

Afficher le nombre de regles copiees : `[HOOKIFY] X/8 regles copiees`

### Etape 4 — CLAUDE.md

**Mode `new`** : Generer depuis `CLAUDE_MD.template.md` en substituant toutes les variables.
La Matrice documentaire est generee avec les 10 fichiers PRD en statut "A completer" (mode new = aucun pre-remplissage).

**Mode `bootstrap`** :
- Si CLAUDE.md existe, **ne pas ecraser**
- Verifier si les sections suivantes sont presentes :
  - "Matrice documentaire"
  - "Standards de Documentation"
  - "Directives Claude"
  - "Conventions Git"
- Pour chaque section manquante, proposer de l'ajouter (avec confirmation utilisateur)
- Conserver l'integralite du contenu existant

**Matrice documentaire — generation contextuelle (les deux modes)** :

La Matrice documentaire dans CLAUDE.md doit referencer tous les documents generes avec leur **statut reel** :

```markdown
## Matrice documentaire

> Reference unique — indique ou trouver, creer et mettre a jour chaque type de documentation.

| Document | Emplacement | Quand le creer/modifier | Comment s'y referer |
|----------|-------------|-------------------------|---------------------|
| Doc technique projet | CLAUDE.md (racine) | A chaque session | Lu automatiquement par Claude Code |
| Presentation projet | README.md (racine) | A chaque changement majeur | Point d'entree pour tout lecteur |
| Variables d'env | .env.example (racine) | A chaque ajout/suppression de variable | Synchronise avec .env |
| Vision produit | docs/prd/01-VISION.md | Au cadrage, puis si pivot | Contexte et objectifs |
| Personas | docs/prd/02-PERSONAS.md | Au cadrage, puis si nouveau segment | Profils utilisateurs |
| Features | docs/prd/03-FEATURES.md | A chaque feature ou changement priorite | Inventaire fonctionnel |
| Backlog | docs/prd/04-BACKLOG.md | A chaque sprint/iteration | Taches priorisees (MoSCoW) |
| Roadmap | docs/prd/05-ROADMAP.md | A chaque jalon ou replannification | Vue temporelle |
| Specifications | docs/prd/06-SPECIFICATIONS.md | A chaque decision technique | Architecture et contrats |
| Decisions (PRD) | docs/prd/07-DECISIONS.md | Index des ADR | Lien vers docs/adr/ |
| Changelog | docs/prd/08-CHANGELOG.md | A chaque version ou livraison | Historique changements |
| Metriques | docs/prd/09-METRICS.md | Apres chaque mesure | Suivi performance |
| Index PRD | docs/prd/00-INDEX.md | Auto quand un PRD change de statut | Navigation PRD |
| Decisions archi | docs/adr/ADR-NNN-*.md | A chaque decision structurante | Format MADR |
| Guides techniques | docs/guides/*.md | Quand un processus doit etre documente | Procedures operationnelles |
| Assets visuels | docs/assets/* | Schemas et diagrammes | Referencables depuis les docs |
```

Regles de statut :
- **Actif** : Le fichier a ete pre-rempli avec du contenu factuel (au moins une section non-vide)
- **A completer** : Le fichier est un template vide ou n'a aucun contenu substantiel

En mode `bootstrap`, si d'autres documents existent deja dans le projet (ex: `docs/claude.md`, `docs/PRD-legacy.md`, `docs/Phase*.md`), les ajouter a la Matrice documentaire pour maintenir une cartographie complete :

```markdown
| Identite visuelle | `docs/claude.md` | Reference identite | Actif |
| PRD legacy | `docs/PRD-Site-Swel.md` | PRD original (pre-init) | Actif |
```

> L'objectif est que Claude dispose, des le debut de chaque session, de la cartographie
> complete de toute la documentation disponible dans le projet.

**Standards de Documentation** — generation dans CLAUDE.md :

```markdown
## Standards de Documentation

### JSDoc

- **Langue** : francais obligatoire sur tous les exports (fonctions, classes, types, constantes)
- **Tags obligatoires** : `@description` + `@param` + `@returns` + `@throws` + `@example`
- **Pas de types dans JSDoc** : TypeScript les gere — ne pas dupliquer avec `@type` ou `@param {string}`
- **Commentaires inline** : expliquer le POURQUOI, pas le QUOI
- **TODO normalise** : `// TODO(JrM): description — YYYY-MM-DD`
- **Code commente interdit** : supprimer plutot que commenter (Git conserve l'historique)

Exemple :
\`\`\`typescript
/**
 * Recherche les objets trouves correspondant aux criteres.
 *
 * @param criteres - Filtres de recherche (categorie, date, lieu)
 * @returns Liste des objets correspondants, triee par date decroissante
 * @throws {ValidationError} Si les criteres sont invalides
 * @example
 * const objets = await rechercherObjets({ categorie: 'telephone', lieu: 'ligne1' });
 */
\`\`\`
```

**Section hub conditionnelle** — generee UNIQUEMENT en mode hub :

```markdown
## Ressources Partagees (Hub)

> Ce projet est integre a un hub de developpement (`{{HUB_PATH}}`).

### Agents recommandes

| Agent | Expertise | Quand l'utiliser |
|-------|-----------|------------------|
| [selon la stack detectee — voir matrice de dispatch du hub CLAUDE.md] |

### MCP disponibles

| MCP | Usage |
|-----|-------|
| Context7 | Documentation librairies a jour |
| Playwright | Self-QA navigateur |
| PostgreSQL | Acces direct BDD (DSN a configurer) |

### Hooks hub

Les hooks du hub (`{{HUB_PATH}}/_shared/hooks/`) s'executent automatiquement :
- `security_check.py` — bloque ecriture .env/secrets
- `code_review.py` — review avant git commit/push
- `route_skills.py` — routage automatique par famille de skills
- `context_loader.py` — expertise agent selon le repertoire
- `session_stop.py` — rappel Git + tests en fin de session
```

En mode standalone, cette section n'est PAS generee.

Pour les variables de generation :

**`{{STACK_TABLE}}`** — Generation conditionnelle, basee **exclusivement sur les informations detectees ou confirmees** :

- Si la stack est detectee/renseignee : generer le tableau a partir des **dependances reelles** lues dans `package.json` (ou equivalent), avec les **versions reelles** installees (pas de versions generiques "15.x")
- Si la stack est `non detectee` et non renseignee : inscrire `*A completer*` dans la section
- Pour chaque technologie, ne l'ajouter au tableau que si elle est **effectivement presente** dans les dependances (ex: ne pas ajouter Tailwind si `tailwindcss` n'est pas dans les deps)

Exemple de lignes a generer **uniquement si la dependance est detectee** :
```
| **Frontend** | Next.js | [version reelle] | Framework React fullstack |
| **ORM** | Prisma | [version reelle] | Acces base de donnees |
| **Database** | PostgreSQL | [version detectee ou ""] | Stockage relationnel |
```

> **Regle** : Aucune technologie ne doit apparaitre dans le tableau si elle n'a pas ete detectee
> dans les fichiers du projet ou explicitement confirmee par l'utilisateur.
> Pas de version generique — lire la version reelle ou laisser vide.

**Lignes ORM/DB** — Ajouter uniquement si detectees (etape 0.2) :
- Prisma detecte → ajouter la ligne ORM avec version reelle depuis `package.json`
- PostgreSQL detecte (docker-compose ou .env) → ajouter la ligne Database
- Supabase detecte → ajouter la ligne Backend-as-a-Service
- MongoDB detecte → ajouter la ligne Database NoSQL
- Rien detecte et rien renseigne → omettre ces lignes

**`{{COMMANDS}}`** — Generation conditionnelle, **uniquement les blocs dont les outils sont detectes** :

```bash
# Developpement (si package.json avec scripts detecte)
{{pkg}} run dev          # Lancer en mode dev
{{pkg}} run build        # Build production
{{pkg}} run lint         # Verifier le code

# Base de donnees (uniquement si Prisma detecte dans les dependances)
{{pkg}} run db:generate  # Generer le client Prisma
{{pkg}} run db:push      # Pousser le schema

# Base de donnees (uniquement si @supabase/supabase-js detecte)
npx supabase start       # Demarrer Supabase local
npx supabase db push     # Appliquer les migrations

# Docker (uniquement si docker-compose.yml ou compose.yml existe)
docker compose up -d     # Demarrer les services
docker compose down      # Arreter les services
```

> **Regle** : Ne generer que les blocs dont les outils sont effectivement presents dans le projet.
> Si aucun outil n'est detecte, la section Commandes affiche : `*A completer selon votre stack*`
> `{{pkg}}` est remplace par le package manager detecte. Si non detecte, utiliser `npm` comme
> placeholder et ajouter un commentaire `# Adaptez a votre package manager`.

### Etape 4b — ESLint et documentation (projets TypeScript uniquement)

> Si la stack detectee n'est pas `nextjs`, `sveltekit`, `astro` ou `node-api`, cette etape est ignoree.
> En mode standalone sans stack TypeScript, cette etape est ignoree.

Si la stack detectee est `nextjs`, `sveltekit`, `astro` ou `node-api` :

1. **Verifier** si `eslint-plugin-jsdoc` est installe (dans `package.json` devDependencies)
2. **Si non installe**, l'installer :
   - `npm install -D eslint-plugin-jsdoc` (ou equivalent selon le package manager detecte : `pnpm add -D`, `bun add -D`, `yarn add -D`)
3. **Generer** `eslint.config.mjs` depuis le template `eslint.config.mjs.template` avec les regles :
   - `jsdoc/require-jsdoc` : `warn`
   - `jsdoc/require-param-description` : `warn`
   - `jsdoc/check-param-names` : `error` avec `checkDestructured: false`
   - `jsdoc/no-types` : `warn`
4. **Si `eslint.config.mjs` existe deja**, proposer fusion (ajouter les regles JSDoc manquantes sans ecraser les regles existantes)

### Etape 5 — MEMORY.md

Determiner le chemin memoire Claude Code :
- Lister les repertoires dans `~/.claude/projects/`
- Identifier celui correspondant au projet courant (par convention, le hash du chemin)
- Si aucun repertoire n'existe, en creer un avec `mkdir -p`

Generer depuis `MEMORY_MD.template.md` en substituant les variables. Le fichier doit rester **sous 50 lignes**.

Pour les variables conditionnelles du template MEMORY.md :
- `{{MEMORY_STACK_LINE}}` → `- **Stack** : [valeur]` si detectee, sinon ligne omise (vide)
- `{{MEMORY_PORT_LINE}}` → `- **Port** : [valeur]` si detecte, sinon ligne omise
- `{{MEMORY_PKG_LINE}}` → `- **Package manager** : [valeur]` si detecte, sinon ligne omise
- `{{MEMORY_DB_LINE}}` → `- **Database** : [valeur]` si detectee, sinon ligne omise

> Ne pas ecrire de lignes avec "non detecte" — omettre completement la ligne si l'info est absente.

Si MEMORY.md existe deja, **ne pas ecraser**. Afficher : `[SKIP] MEMORY.md existe deja`

### Etape 6 — PRD (10 fichiers)

Pour chaque template dans `~/.claude/commands/init-project-templates/PRD/` :
1. Substituer les variables (`{{PROJECT_NAME}}`, `{{DATE}}`, `{{STACK}}`, etc.)
2. **Si du contenu pre-rempli est disponible** (valide a l'etape 0.6) : injecter le contenu extrait dans les sections correspondantes du template, en remplacement des placeholders vides. Les sections sans contenu pre-rempli conservent leurs placeholders "A completer".
3. Ecrire dans `docs/prd/` (retirer le suffixe `.template`)

**Mode `new`** : Les templates sont generes avec les placeholders vides (aucun pre-remplissage disponible).

**Mode `bootstrap`** :
- Si un fichier PRD existe deja, **ne pas ecraser**. Afficher : `[SKIP] docs/prd/XX-NOM.md existe deja`
- Si du contenu pre-rempli est disponible pour ce fichier, l'injecter dans le template avant ecriture
- Afficher pour chaque fichier : `[CREE] docs/prd/XX-NOM.md (Y sections pre-remplies)` ou `[CREE] docs/prd/XX-NOM.md (template vide)`

**`00-INDEX.md` — generation en dernier** :

Le fichier index est genere **apres** tous les autres PRD. Son tableau de statut reflete l'etat reel de chaque fichier :
- Fichier pre-rempli → statut `Actif`
- Fichier template vide → statut `A completer`
- Fichier deja existant (skip) → statut `Existant`

Fichiers generes :
- `docs/prd/01-VISION.md`
- `docs/prd/02-PERSONAS.md`
- `docs/prd/03-FEATURES.md`
- `docs/prd/04-BACKLOG.md`
- `docs/prd/05-ROADMAP.md`
- `docs/prd/06-SPECIFICATIONS.md`
- `docs/prd/07-DECISIONS.md`
- `docs/prd/08-CHANGELOG.md`
- `docs/prd/09-METRICS.md`
- `docs/prd/00-INDEX.md` ← genere en dernier

### Etape 7 — Dossier ADR

Creer `docs/adr/` avec un fichier `README.md` s'il n'existe pas :

```markdown
# Architecture Decision Records (ADR)

Ce dossier contient les decisions architecturales du projet au format MADR
(Markdown Any Decision Records).

## Format MADR

Chaque ADR suit cette structure :

\`\`\`
# ADR-NNN : Titre court de la decision

## Statut

[Propose | Accepte | Deprecie | Remplace par ADR-XXX]

## Contexte

Quel est le probleme ou la situation qui motive cette decision ?

## Decision

Quelle decision a ete prise ?

## Alternatives envisagees

### Alternative 1 — [Nom]
- Avantages : ...
- Inconvenients : ...

### Alternative 2 — [Nom]
- Avantages : ...
- Inconvenients : ...

## Consequences

### Positives
- ...

### Negatives
- ...

### Neutres
- ...
\`\`\`

## Convention de nommage

- Fichier : `ADR-NNN-titre-court.md` (ex: `ADR-001-choix-framework.md`)
- NNN : numero sequentiel a 3 chiffres
- Titre court : en minuscules, mots separes par des tirets

## Creer un nouvel ADR

1. Identifier le prochain numero disponible
2. Copier la structure ci-dessus dans un nouveau fichier
3. Remplir toutes les sections
4. Mettre le statut a "Propose"
5. Ajouter une reference dans `docs/prd/07-DECISIONS.md`

## Index des ADR

| ADR | Titre | Statut | Date |
|-----|-------|--------|------|
| *(Aucun ADR enregistre)* | | | |
```

Si `docs/adr/README.md` existe deja, `[SKIP]`.

### Etape 7b — Marketing context (optionnel)

> Cette etape n'est executee que si l'utilisateur a choisi "Oui" a l'option marketing (etape 0.5).

1. Copier le template `_shared/templates/marketing-context-template.md` vers `docs/marketing-context.md`
2. Substituer `{{PROJECT_NAME}}` et `{{DATE}}` dans le fichier
3. Afficher : `[CREE] docs/marketing-context.md (template a completer)`

Si `docs/marketing-context.md` existe deja, **ne pas ecraser**. Afficher : `[SKIP] docs/marketing-context.md existe deja`

> Ce fichier est consomme par les skills marketing du hub. Il permet aux skills `marketing:*`
> de produire du contenu adapte au projet (brand voice, SEO, campagnes, contenu).
> Voir `_shared/skills/knowledge-work/brand-voice.md` et `content-creation.md` pour les frameworks.

### Etape 8 — Git

**Mode `new`** :
1. Ecrire le `.gitignore` depuis le template
2. `git init`
3. Lister les fichiers a stager et les presenter a l'utilisateur
4. Apres confirmation : `git add <fichiers listes>` puis `git commit -m "chore: initialize project with /init-project"`

**Important** : Ne jamais utiliser `git add -A` ou `git add .` sans confirmation. Toujours lister explicitement les fichiers.

**Mode `bootstrap`** :
- Verifier si `.gitignore` existe, sinon le creer depuis le template
- Proposer un commit : `docs: initialize project documentation`
- Attendre confirmation utilisateur avant de committer

### Etape 8b — Archivage des sources pre-remplissage (mode bootstrap uniquement)

> Cette etape n'est executee qu'en mode `bootstrap` et uniquement si du pre-remplissage a eu lieu (etape 0.6).
> En mode `new`, cette etape est ignoree (aucune source a archiver).

**Objectif** : Apres restructuration de la documentation dans les nouveaux PRD/ADR, les fichiers sources
epars n'ont plus lieu d'etre dans l'arborescence de travail. Les archiver evite la confusion entre
ancienne et nouvelle documentation.

**Fichiers eligibles a l'archivage** : Uniquement les fichiers de documentation qui ont ete utilises
comme source de pre-remplissage (etape 0.6) et dont le contenu a ete absorbe dans les nouveaux fichiers.

**Fichiers JAMAIS archives** (exclusions absolues) :
- `CLAUDE.md` — fichier central, enrichi mais jamais archive
- `README.md` — convention Git, toujours a la racine
- `.env`, `.env.example`, `.env.local` — fichiers operationnels
- `package.json`, `tsconfig.json` — fichiers techniques actifs
- `docker-compose.yml`, `compose.yml` — infrastructure active
- Tout fichier en dehors de `docs/` — seule la documentation est concernee
- Tout fichier dans `docs/prd/`, `docs/adr/`, `docs/guides/`, `docs/assets/` — ce sont les nouveaux fichiers

**Processus** :

1. **Lister** les fichiers source utilises depuis le `prefill_registry` (etape 0.6)
2. **Filtrer** en excluant les fichiers proteges ci-dessus
3. **Presenter** la liste a l'utilisateur pour validation :

```
## Archivage des documents restructures

Les fichiers suivants ont ete utilises pour pre-remplir les nouveaux PRD.
Leur contenu a ete restructure dans les documents standards du projet.

Fichiers a archiver dans `_archive_pre-init/` :

| Fichier | Contenu absorbe dans |
|---------|---------------------|
| docs/PRD-legacy.md | 01-VISION, 02-PERSONAS, 03-FEATURES |
| docs/Phase1-specs.md | 06-SPECIFICATIONS, 07-DECISIONS |
| docs/charte-graphique.md | 06-SPECIFICATIONS |

Fichiers conserves (operationnels ou enrichis) :
- CLAUDE.md (enrichi, pas archive)
- package.json (fichier technique actif)

Archiver ces fichiers ?
- Oui : deplacer dans _archive_pre-init/
- Non : conserver en l'etat
```

4. **Si Oui** :
   - Creer le repertoire `_archive_pre-init/` a la racine du projet
   - Deplacer chaque fichier eligible (conserver l'arborescence relative : `docs/PRD-legacy.md` → `_archive_pre-init/docs/PRD-legacy.md`)
   - Afficher : `[ARCHIVE] X fichier(s) deplaces dans _archive_pre-init/`
   - Ajouter `_archive_pre-init/` au `.gitignore` si ce n'est pas deja le cas
5. **Si Non** : ne rien faire, afficher : `[SKIP] Archivage ignore — les fichiers sources restent en place`

> **Regle** : L'archivage est toujours propose, jamais impose. L'utilisateur garde le controle total.
> Les fichiers ne sont pas supprimes — ils sont deplaces dans un repertoire d'archive accessible.

### Etape 9 — Verification

Executer la checklist suivante et afficher le resultat :

```
[x/X] .claude/settings.local.json — existe et JSON valide
[x/X] .claude/settings.json — hooks configures
[x/X] CLAUDE.md — contient Matrice documentaire
[x/X] CLAUDE.md — contient Standards de Documentation
[x/X] CLAUDE.md — contient Directives Claude
[x/X] CLAUDE.md — contient Conventions Git
[x/X] docs/prd/ — 10 fichiers presents
[x/X] docs/adr/README.md — existe
[x/X] docs/guides/ — existe
[x/X] docs/assets/ — existe
[x/X] docs/marketing-context.md — present (si option marketing activee)
[x/X] MEMORY.md — cree ou deja present
[x/X] .gitignore — present
[x/X] Git — propre (commit effectue)
```

**Checks conditionnels mode hub** :
```
[x/X] Hookify rules presentes (X/8)
```

**Checks conditionnels TypeScript** :
```
[x/X] eslint-plugin-jsdoc installe
[x/X] eslint.config.mjs present avec regles JSDoc
```

**Checks conditionnels bootstrap** :
```
[x/X] Archivage sources pre-remplissage — effectue / ignore / non applicable
```

### Etape 10 — Rapport de fin d'execution

Generer un rapport structure en 3 sections et l'afficher a l'utilisateur :

#### Section 1 — Resume d'execution

```markdown
## Rapport /init-project — {{PROJECT_NAME}}

**Mode** : [new/bootstrap]
**Environnement** : [Hub (chemin) / Standalone]
**Date** : {{DATE}}
**Duree** : [temps d'execution]

### Actions realisees

| Action | Statut | Detail |
|--------|--------|--------|
| Structure repertoires | Cree / Existe | X dossiers crees |
| settings.local.json | Cree / Fusionne / Existe | X permissions |
| settings.json | Cree / Enrichi / Existe | X hooks |
| Hookify rules (hub) | Copie / N/A | X/8 regles |
| CLAUDE.md | Cree / Enrichi / Existe | X sections |
| ESLint JSDoc (TypeScript) | Configure / N/A | eslint-plugin-jsdoc |
| MEMORY.md | Cree / Existe | |
| PRD | Cree | X/10 fichiers (Y pre-remplis) |
| ADR | Cree / Existe | README.md |
| Marketing context | Cree / Existe / Non demande | docs/marketing-context.md |
| .gitignore | Cree / Existe | |
| Git commit | Oui / Non | [message] |
| Archivage sources (bootstrap) | Effectue / Ignore / N/A | X fichiers dans _archive_pre-init/ |

**Fichiers crees** : X
**Fichiers ignores (existants)** : Y
**Fichiers archives** : Z (dans `_archive_pre-init/`)
```

#### Section 2 — Environnement du projet

```markdown
### Votre environnement

| Element | Valeur |
|---------|--------|
| Projet | {{PROJECT_NAME}} |
| Stack | {{STACK}} ou "non detectee" |
| Database | {{DATABASE}} ou "non detectee" |
| Port dev | {{PORT}} ou "non detecte" |
| Package manager | {{PACKAGE_MANAGER}} ou "non detecte" |
| Git remote | [url ou "non configure"] |
| Environnement | [Hub (chemin) / Standalone] |

> Les elements marques "non detecte" peuvent etre renseignes en editant directement CLAUDE.md et MEMORY.md.

### Arborescence documentaire

docs/
├── prd/
│   ├── 00-INDEX.md          <- Point d'entree PRD
│   ├── 01-VISION.md         <- Objectifs et scope
│   ├── 02-PERSONAS.md       <- Utilisateurs cibles
│   ├── 03-FEATURES.md       <- Inventaire fonctionnel
│   ├── 04-BACKLOG.md        <- Backlog priorise (MoSCoW)
│   ├── 05-ROADMAP.md        <- Phases et jalons
│   ├── 06-SPECIFICATIONS.md <- Architecture technique
│   ├── 07-DECISIONS.md      <- Index des ADR
│   ├── 08-CHANGELOG.md      <- Journal des changements
│   └── 09-METRICS.md        <- KPIs et metriques
├── adr/
│   └── README.md             <- Format MADR et index des ADR
├── guides/
│   └── (guides techniques a creer selon les besoins)
└── assets/
    └── (schemas, diagrammes, images)
```

#### Section 3 — Preconisations contextuelles

Les preconisations sont generees **dynamiquement** en fonction du `prefill_registry` (etape 0.6)
et de l'etat reel des fichiers generes. Ne recommander que ce qui reste a faire.

**Structure de generation** :

```markdown
### Preconisations
```

**Bloc 1 — "A completer en priorite"** (genere uniquement si des fichiers PRD ont des sections vides) :

Lister uniquement les fichiers PRD qui ont des sections restant a completer, par ordre de priorite :

```markdown
#### A completer en priorite

| PRD | Sections a completer | Pourquoi |
|-----|---------------------|----------|
| [fichier] | [sections vides] | [importance pour le projet] |
```

Regles :
- Si `01-VISION.md` est entierement pre-rempli → ne PAS le mentionner
- Si `02-PERSONAS.md` a ete pre-rempli avec 2 personas mais le template en prevoit 3 → mentionner "Persona 3 (optionnel)"
- Si `06-SPECIFICATIONS.md` a la stack mais pas les APIs → mentionner "Endpoints API, schema de donnees"
- Si TOUS les PRD sont pre-remplis → remplacer ce bloc par : "Tous les fichiers PRD ont ete pre-remplis a partir des sources existantes. Verifiez et ajustez le contenu selon vos besoins."

**Bloc 2 — "Pendant le developpement"** (toujours present — regles comportementales) :

```markdown
#### Pendant le developpement

1. **Respectez les directives** documentees dans CLAUDE.md :
   - Apres chaque commit significatif → MAJ `docs/prd/08-CHANGELOG.md`
   - Apres chaque feature terminee → MAJ `docs/prd/03-FEATURES.md` (changer statut)
   - Apres chaque decision technique → Creer un ADR dans `docs/adr/`
     et mettre a jour l'index dans `docs/prd/07-DECISIONS.md`

2. **Documentez l'architecture** : Pour chaque decision structurante,
   creez un ADR dans `docs/adr/` en suivant le format MADR.

3. **Suivez les conventions Git** : Branches depuis main, commits conventionnels,
   PRs avec summary + test plan.
```

**Bloc 3 — "Bonnes pratiques"** (toujours present) :

```markdown
#### Bonnes pratiques

4. **CLAUDE.md est votre fichier principal** : C'est le premier fichier lu par Claude
   a chaque session. La Matrice documentaire vous donne une vue d'ensemble de toute
   la documentation disponible.

5. **MEMORY.md est votre aide-memoire** : Notez-y les pieges, ports, commandes
   specifiques et tout ce qui doit survivre entre les sessions (50 lignes max).
```

**Bloc 4 — "Contenu transfere"** (genere uniquement en mode bootstrap si du pre-remplissage a eu lieu) :

```markdown
#### Contenu transfere depuis les sources existantes

| PRD | Sections pre-remplies | Source(s) |
|-----|----------------------|-----------|
| [fichier] | [sections remplies] | [fichiers source] |

> Ce contenu a ete extrait tel quel des sources existantes. Verifiez son exactitude
> et completez les sections manquantes.
```

Proposer a l'utilisateur de sauvegarder ce rapport dans `docs/INIT-REPORT.md`.

---

## Mode audit : /init-project --check

Quand lance avec `--check`, le skill n'ecrit aucun fichier. Il effectue uniquement :

1. **Detection de l'environnement** (etape 0.2)
2. **Detection hub** (etape 0.7)
3. **Audit de l'existant** (comme etape 0.3 en mode bootstrap)
4. **Verification de conformite** :

```
## Rapport d'audit — {{PROJECT_NAME}}

### Conformite documentaire

| Element | Attendu | Statut | Detail |
|---------|---------|--------|--------|
| CLAUDE.md | Present | [OK/ABSENT] | |
| CLAUDE.md > Matrice documentaire | Section presente | [OK/ABSENT] | |
| CLAUDE.md > Standards de Documentation | Section presente | [OK/ABSENT] | |
| CLAUDE.md > Conventions Git | Section presente | [OK/ABSENT] | |
| settings.json | Present + scope-check | [OK/ABSENT/INCOMPLET] | |
| Hookify rules (hub) | 8/8 regles | [X/8] | Mode hub uniquement |
| ESLint JSDoc | Configure | [OK/ABSENT/N/A] | TypeScript uniquement |
| docs/prd/ | 10/10 fichiers | [X/10] | |
| docs/adr/ | Present | [OK/ABSENT] | |
| docs/guides/ | Present | [OK/ABSENT] | |
| docs/assets/ | Present | [OK/ABSENT] | |
| README.md | Present | [OK/ABSENT] | |
| .env.example | Present | [OK/ABSENT] | |
| .gitignore | Present | [OK/ABSENT] | |

### Ecarts detectes

| Ecart | Severite | Preconisation |
|-------|----------|---------------|
| [description] | [Critique/Majeur/Mineur] | [action corrective] |
```

5. **Rapport d'ecarts** avec preconisations de mise en conformite

Le mode `--check` est utile pour verifier qu'un projet existant respecte les conventions
ou pour identifier ce qui manque avant un bootstrap.

---

## Gestion des erreurs

| Erreur | Action |
|--------|--------|
| Pas de permission ecriture | Afficher message et interrompre |
| Template manquant | Afficher avertissement, continuer avec les autres |
| Fichier existant (mode bootstrap) | Ne jamais ecraser, afficher `[SKIP] fichier existe deja` |
| Git non installe | Ignorer etapes Git, afficher avertissement |
| Chemin MEMORY.md introuvable | Creer le repertoire, afficher le chemin utilise |
| Hub detecte mais script manquant | Fallback mode standalone pour l'etape concernee, afficher avertissement |

## Notes

- Le skill ne fait **aucune installation de dependances** (pas de `npm install`), sauf `eslint-plugin-jsdoc` a l'etape 4b si applicable.
- Les templates utilisent des placeholders `{{VARIABLE}}` qui sont substitues a l'execution.
- En mode `bootstrap`, le principe est : **ne jamais casser, toujours completer**.
- Le skill est **adaptatif** : il fonctionne en mode standalone (sans hub) ou en mode hub
  (avec infrastructure partagee). En mode standalone, il genere un projet autonome.
  En mode hub, il s'integre a l'environnement existant (hooks, agents, MCP, hookify).
- La detection hub est automatique — l'utilisateur n'a rien a configurer.
- Pour partager ce skill : utiliser le package distribuable dans `init-project-skill/` (README + scripts d'installation + fichiers).
