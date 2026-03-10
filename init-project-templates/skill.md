---
name: init-project
description: Initialise ou bootstrap un projet avec documentation structuree (CLAUDE.md, MEMORY.md, PRD, feature docs), configuration Claude Code (permissions, hooks) et conventions Git.
trigger: /init-project
auto_detect: false
---

# Skill : /init-project — Initialisation projet

## Description

Ce skill orchestre le setup documentaire et organisationnel complet d'un projet. Il genere la documentation technique (CLAUDE.md, PRD, feature docs), configure Claude Code (permissions, hooks) et etablit les conventions Git.

Deux modes :
- **`new`** : Projet vierge — cree toute la structure depuis zero
- **`bootstrap`** : Projet existant — analyse l'environnement, detecte les existants et les manquements, complete sans ecraser

## Declenchement

```
/init-project [nom du projet]
```

## Templates

Les templates sont co-distribues avec le skill dans `~/.claude/commands/init-project-templates/` :

| Template | Destination | Description |
|----------|-------------|-------------|
| `CLAUDE_MD.template.md` | `./CLAUDE.md` | Doc technique avec Documentation Matrix et 6 directives |
| `MEMORY_MD.template.md` | `~/.claude/projects/.../memory/MEMORY.md` | Memoire persistante (50 lignes max) |
| `settings.local.json.template` | `./.claude/settings.local.json` | Permissions adaptees a la stack |
| `settings.json.template` | `./.claude/settings.json` | Hooks projet |
| `feature-doc.template.md` | `docs/features/` | Template doc par feature |
| `gitignore.template` | `./.gitignore` | Gitignore multi-stack |
| `PRD/*.template` | `docs/PRD/` | 10 fichiers PRD |

## Variables de substitution

| Variable | Source | Obligatoire | Description |
|----------|--------|-------------|-------------|
| `{{PROJECT_NAME}}` | Collecte | Oui | Nom du projet |
| `{{PROJECT_DESCRIPTION}}` | Collecte | Oui | Description courte |
| `{{PROJECT_PATH}}` | Detection | Oui | Chemin absolu du projet (repertoire courant) |
| `{{DATE}}` | Automatique | Oui | Date du jour (YYYY-MM-DD) |
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
| `{{FEATURE_NAME}}` | Parametre | Non | Nom de la feature (feature-doc) |
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
- Verifier la presence de PRD (compter les fichiers dans docs/PRD/), feature docs, .gitignore
- Preparer le resume : "Existants detectes : X | Manquants : Y"

**Scan des sources de pre-remplissage** :

Scanner le projet pour identifier les fichiers contenant des informations exploitables pour pre-remplir les PRD. Pour chaque source trouvee, noter son chemin et le type d'information qu'elle contient.

Sources a scanner (dans l'ordre) :

| Source | Quoi chercher |
|--------|---------------|
| `CLAUDE.md` | Personas, objectifs/KPIs, stack, features, historique sessions, decisions techniques, architecture, commandes |
| `README.md` | Description projet, objectifs, instructions, architecture |
| `docs/*.md` (hors PRD/) | PRD legacy, specs, phases, charte graphique, identite visuelle |
| `docs/**/*.md` (hors PRD/) | Documentation approfondie (features, integration, specs detaillees) |
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
```

En mode bootstrap, ajouter le resume de l'audit documentaire et des sources de pre-remplissage :
```
## Audit documentaire

| Element | Statut |
|---------|--------|
| CLAUDE.md | Present / Absent |
| .claude/settings.local.json | Present / Absent |
| .claude/settings.json | Present / Absent |
| docs/PRD/ | X/10 fichiers |
| docs/features/ | Present / Absent |
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
| `07-DECISIONS.md` | Decisions techniques documentees (choix stack, design system, patterns, conventions) | CLAUDE.md (decisions, conventions, design system), docs/*decision*.md, docs/*phase*.md |
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

### Etape 1 — Structure repertoires

**Mode `new`** :
```bash
mkdir -p src .claude docs/PRD docs/features
```

**Mode `bootstrap`** :
```bash
# Creer uniquement les repertoires manquants
mkdir -p .claude docs/PRD docs/features
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

Generer depuis `settings.json.template`. Le template fournit un hook PreToolUse basique qui alerte lors de modifications de fichiers hors du repertoire projet.

Si `.claude/settings.json` existe deja :
- Lire le contenu existant
- Proposer d'ajouter les hooks manquants **sans ecraser les hooks existants**
- Attendre confirmation utilisateur

### Etape 4 — CLAUDE.md

**Mode `new`** : Generer depuis `CLAUDE_MD.template.md` en substituant toutes les variables.
La Documentation Matrix est generee avec les 10 fichiers PRD en statut "A completer" (mode new = aucun pre-remplissage).

**Mode `bootstrap`** :
- Si CLAUDE.md existe, **ne pas ecraser**
- Verifier si les sections suivantes sont presentes :
  - "Documentation Matrix"
  - "Directives Claude"
  - "Conventions Git"
- Pour chaque section manquante, proposer de l'ajouter (avec confirmation utilisateur)
- Conserver l'integralite du contenu existant

**Documentation Matrix — generation contextuelle (les deux modes)** :

La Documentation Matrix dans CLAUDE.md doit referencer tous les documents generes avec leur **statut reel** :

```markdown
## Documentation Matrix

| Document | Chemin | Description | Statut |
|----------|--------|-------------|--------|
| **CLAUDE.md** | `./CLAUDE.md` | Ce fichier — reference technique | Actif |
| PRD Index | `docs/PRD/00-INDEX.md` | Navigation PRD | Actif |
| Vision | `docs/PRD/01-VISION.md` | Problem statement, objectifs | [Actif/A completer] |
| Personas | `docs/PRD/02-PERSONAS.md` | Utilisateurs cibles | [Actif/A completer] |
| Features | `docs/PRD/03-FEATURES.md` | Inventaire fonctionnel | [Actif/A completer] |
| Backlog | `docs/PRD/04-BACKLOG.md` | Backlog priorise | [Actif/A completer] |
| Roadmap | `docs/PRD/05-ROADMAP.md` | Phases et milestones | [Actif/A completer] |
| Specifications | `docs/PRD/06-SPECIFICATIONS.md` | Architecture, APIs | [Actif/A completer] |
| Decisions | `docs/PRD/07-DECISIONS.md` | ADRs | [Actif/A completer] |
| Changelog | `docs/PRD/08-CHANGELOG.md` | Journal des changements | [Actif/A completer] |
| Metriques | `docs/PRD/09-METRICS.md` | KPIs et suivi | [Actif/A completer] |
| Features docs | `docs/features/` | Documentation par feature | Actif |
```

Regles de statut :
- **Actif** : Le fichier a ete pre-rempli avec du contenu factuel (au moins une section non-vide)
- **A completer** : Le fichier est un template vide ou n'a aucun contenu substantiel

En mode `bootstrap`, si d'autres documents existent deja dans le projet (ex: `docs/claude.md`, `docs/PRD-legacy.md`, `docs/Phase*.md`), les ajouter a la Documentation Matrix pour maintenir une cartographie complete :

```markdown
| Identite visuelle | `docs/claude.md` | Reference identite | Actif |
| PRD legacy | `docs/PRD-Site-Swel.md` | PRD original (pre-init) | Actif |
```

> L'objectif est que Claude dispose, des le debut de chaque session, de la cartographie
> complete de toute la documentation disponible dans le projet.

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
3. Ecrire dans `docs/PRD/` (retirer le suffixe `.template`)

**Mode `new`** : Les templates sont generes avec les placeholders vides (aucun pre-remplissage disponible).

**Mode `bootstrap`** :
- Si un fichier PRD existe deja, **ne pas ecraser**. Afficher : `[SKIP] docs/PRD/XX-NOM.md existe deja`
- Si du contenu pre-rempli est disponible pour ce fichier, l'injecter dans le template avant ecriture
- Afficher pour chaque fichier : `[CREE] docs/PRD/XX-NOM.md (Y sections pre-remplies)` ou `[CREE] docs/PRD/XX-NOM.md (template vide)`

**`00-INDEX.md` — generation en dernier** :

Le fichier index est genere **apres** tous les autres PRD. Son tableau de statut reflète l'etat reel de chaque fichier :
- Fichier pre-rempli → statut `Actif`
- Fichier template vide → statut `A completer`
- Fichier deja existant (skip) → statut `Existant`

Fichiers generes :
- `docs/PRD/01-VISION.md`
- `docs/PRD/02-PERSONAS.md`
- `docs/PRD/03-FEATURES.md`
- `docs/PRD/04-BACKLOG.md`
- `docs/PRD/05-ROADMAP.md`
- `docs/PRD/06-SPECIFICATIONS.md`
- `docs/PRD/07-DECISIONS.md`
- `docs/PRD/08-CHANGELOG.md`
- `docs/PRD/09-METRICS.md`
- `docs/PRD/00-INDEX.md` ← genere en dernier

### Etape 7 — Feature docs

Creer `docs/features/README.md` s'il n'existe pas :

```markdown
# Documentation par Feature

Chaque feature significative dispose de sa propre documentation.

## Convention de nommage

- Fichier : `[feature-id]-[nom-court].md` (ex: `F-001-authentification.md`)
- Un template de reference est disponible dans `~/.claude/commands/init-project-templates/feature-doc.template.md`

## Index des features

| Feature | Fichier | Statut |
|---------|---------|--------|
| *(Aucune feature documentee)* | | |

> Pour creer une nouvelle doc feature, copier le template et remplir les sections.
```

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

### Etape 9 — Verification

Executer la checklist suivante et afficher le resultat :

```
[x/✗] .claude/settings.local.json — existe et JSON valide
[x/✗] .claude/settings.json — hooks configures
[x/✗] CLAUDE.md — contient Documentation Matrix
[x/✗] CLAUDE.md — contient Directives Claude
[x/✗] CLAUDE.md — contient Conventions Git
[x/✗] docs/PRD/ — 10 fichiers presents
[x/✗] docs/features/README.md — existe
[x/✗] MEMORY.md — cree ou deja present
[x/✗] .gitignore — present
[x/✗] Git — propre (commit effectue)
```

### Etape 10 — Rapport de fin d'execution

Generer un rapport structure en 3 sections et l'afficher a l'utilisateur :

#### Section 1 — Resume d'execution

```markdown
## Rapport /init-project — {{PROJECT_NAME}}

**Mode** : [new/bootstrap]
**Date** : {{DATE}}
**Duree** : [temps d'execution]

### Actions realisees

| Action | Statut | Detail |
|--------|--------|--------|
| Structure repertoires | Cree / Existe | X dossiers crees |
| settings.local.json | Cree / Fusionne / Existe | X permissions |
| settings.json | Cree / Enrichi / Existe | X hooks |
| CLAUDE.md | Cree / Enrichi / Existe | X sections |
| MEMORY.md | Cree / Existe | |
| PRD | Cree | X/10 fichiers (Y pre-remplis) |
| Feature docs | Cree / Existe | README.md |
| .gitignore | Cree / Existe | |
| Git commit | Oui / Non | [message] |

**Fichiers crees** : X
**Fichiers ignores (existants)** : Y
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

> Les elements marques "non detecte" peuvent etre renseignes en editant directement CLAUDE.md et MEMORY.md.

### Arborescence documentaire

docs/
├── PRD/
│   ├── 00-INDEX.md          ← Point d'entree PRD
│   ├── 01-VISION.md         ← Objectifs et scope
│   ├── 02-PERSONAS.md       ← Utilisateurs cibles
│   ├── 03-FEATURES.md       ← Inventaire fonctionnel
│   ├── 04-BACKLOG.md        ← Backlog priorise (MoSCoW)
│   ├── 05-ROADMAP.md        ← Phases et jalons
│   ├── 06-SPECIFICATIONS.md ← Architecture technique
│   ├── 07-DECISIONS.md      ← Decisions (ADR)
│   ├── 08-CHANGELOG.md      ← Journal des changements
│   └── 09-METRICS.md        ← KPIs et metriques
└── features/
    └── README.md             ← Index des feature docs
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

1. **Respectez les 6 directives** documentees dans CLAUDE.md :
   - Apres chaque commit significatif → MAJ `docs/PRD/08-CHANGELOG.md`
   - Apres chaque feature terminee → MAJ `docs/PRD/03-FEATURES.md` (changer statut)
   - Apres chaque decision technique → Ajouter un ADR dans `docs/PRD/07-DECISIONS.md`

2. **Documentez les features complexes** : Pour chaque feature significative,
   creez un fichier dans `docs/features/` en suivant le template.

3. **Suivez les conventions Git** : Branches depuis main, commits conventionnels,
   PRs avec summary + test plan.
```

**Bloc 3 — "Bonnes pratiques"** (toujours present) :

```markdown
#### Bonnes pratiques

4. **CLAUDE.md est votre fichier principal** : C'est le premier fichier lu par Claude
   a chaque session. La Documentation Matrix vous donne une vue d'ensemble de toute
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

## Gestion des erreurs

| Erreur | Action |
|--------|--------|
| Pas de permission ecriture | Afficher message et interrompre |
| Template manquant | Afficher avertissement, continuer avec les autres |
| Fichier existant (mode bootstrap) | Ne jamais ecraser, afficher `[SKIP] fichier existe deja` |
| Git non installe | Ignorer etapes Git, afficher avertissement |
| Chemin MEMORY.md introuvable | Creer le repertoire, afficher le chemin utilise |

## Notes

- Le skill ne fait **aucune installation de dependances** (pas de `npm install`). C'est le role de l'utilisateur ou d'un autre workflow.
- Les templates utilisent des placeholders `{{VARIABLE}}` qui sont substitues a l'execution.
- En mode `bootstrap`, le principe est : **ne jamais casser, toujours completer**.
- Le skill est **auto-contenu** : il fonctionne independamment de tout environnement hub ou configuration externe. Les templates sont co-distribues dans le dossier `~/.claude/commands/init-project-templates/`.
- Pour partager ce skill : utiliser le package distribuable dans `init-project-skill/` (README + scripts d'installation + fichiers).
