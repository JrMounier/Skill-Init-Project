# {{PROJECT_NAME}} — Documentation Technique

> Document de reference pour le developpement du projet.
> Derniere mise a jour : {{DATE}}
>
> **Emplacement** : `{{PROJECT_PATH}}`

---

## Matrice documentaire

> Reference unique — indique ou trouver, creer et mettre a jour chaque type de documentation.

| Document | Emplacement | Quand le creer/modifier | Comment s'y referer |
|----------|-------------|-------------------------|---------------------|
| **Doc technique projet** | `CLAUDE.md` (racine) | A chaque session, maintenir a jour | Lu automatiquement par Claude Code |
| **Presentation projet** | `README.md` (racine) | A chaque changement majeur (stack, install, usage) | Point d'entree pour tout nouveau lecteur |
| **Variables d'env** | `.env.example` (racine) | A chaque ajout/suppression de variable | Synchronise avec `.env` |
| Vision produit | `docs/prd/01-VISION.md` | Au cadrage, puis si pivot strategique | Contexte et objectifs du projet |
| Personas | `docs/prd/02-PERSONAS.md` | Au cadrage, puis si nouveau segment utilisateur | Profils utilisateurs cibles |
| Features | `docs/prd/03-FEATURES.md` | A chaque nouvelle feature ou changement de priorite | Inventaire fonctionnel du projet |
| Backlog | `docs/prd/04-BACKLOG.md` | A chaque sprint/iteration | Taches priorisees (MoSCoW) |
| Roadmap | `docs/prd/05-ROADMAP.md` | A chaque jalon atteint ou replannification | Vue temporelle des phases |
| Specifications | `docs/prd/06-SPECIFICATIONS.md` | A chaque decision technique (API, schema, securite) | Architecture et contrats techniques |
| Decisions (PRD) | `docs/prd/07-DECISIONS.md` | Index des ADR, mis a jour quand nouvel ADR | Lien vers docs/adr/ |
| Changelog | `docs/prd/08-CHANGELOG.md` | A chaque version ou livraison | Historique des changements |
| Metriques | `docs/prd/09-METRICS.md` | Apres chaque mesure ou revue de KPI | Suivi performance et qualite |
| Index PRD | `docs/prd/00-INDEX.md` | Auto — mis a jour quand un PRD change de statut | Navigation PRD |
| **Decisions archi** | `docs/adr/ADR-NNN-*.md` | A chaque decision structurante (stack, infra, pattern) | Format MADR, numerotation sequentielle |
| **Guides techniques** | `docs/guides/*.md` | Quand un processus merite d'etre documente | Procedures operationnelles |
| **Assets visuels** | `docs/assets/*` | Quand un schema ou diagramme est necessaire | Referencables depuis les autres docs |

## Standards de Documentation

> **OBLIGATOIRE** : Tout code ecrit ou modifie dans ce projet DOIT respecter ces standards.

### JSDoc — Fonctions et methodes

Toute fonction ou methode **exportee** DOIT avoir un commentaire JSDoc en francais.

**Tags obligatoires :**
| Tag | Quand | Exemple |
|-----|-------|---------|
| Description | Toujours | Premiere ligne du bloc JSDoc |
| `@param` | Toujours (avec description) | `@param id - Identifiant unique de l'objet` |
| `@returns` | Si non-void | `@returns L'objet trouve ou null si inexistant` |
| `@throws` | Si erreurs possibles | `@throws {Error} Si la connexion echoue` |
| `@example` | Pour utilitaires et APIs | Bloc de code illustrant l'usage |

**Regles :**
- Types non repetes dans la JSDoc (TypeScript les gere)
- Fonctions internes : documenter uniquement si logique non evidente
- Commentaires inline : POURQUOI, jamais le QUOI
- Tout workaround DOIT etre commente
- Format TODO : `// TODO(auteur): [description] — [date YYYY-MM-DD]`
- Code commente INTERDIT (utiliser Git)

### JSDoc — Composants et types

- Toute interface de Props : JSDoc sur l'interface ET sur chaque prop
- Tout composant exporte : description JSDoc d'une ligne minimum
- Tout type/interface/enum exporte : JSDoc descriptive

---

## Directives Claude — Regles de mise a jour obligatoires

> **IMPORTANT** : Ces 6 regles doivent etre suivies systematiquement.

1. **Apres chaque commit significatif** → MAJ `docs/prd/08-CHANGELOG.md`
2. **Apres chaque feature terminee** → MAJ `docs/prd/03-FEATURES.md` (changer statut)
3. **Apres chaque decision technique** → Ajouter ADR dans `docs/prd/07-DECISIONS.md`
4. **Apres chaque decision structurante (choix de stack, infrastructure, pattern d'architecture)** → creer un ADR dans `docs/adr/`
5. **Lors de l'ajout d'un nouveau document** → MAJ Matrice documentaire ci-dessus
6. **MAJ PRD complete** (CHANGELOG, FEATURES, BACKLOG) apres chaque changement significatif

---

## Workflow Implementation Feature

```
1. git checkout main && git pull
2. git checkout -b feat/nom-feature
3. Implementer + tests
4. Mettre a jour docs (FEATURES, CHANGELOG, ADR si decision structurante)
5. git add && git commit -m "feat(scope): description"
6. Creer PR (gh pr create)
```

---

## Conventions Git

- **Branches** : Toujours depuis `main` — `feat/xxx`, `fix/xxx`, `refactor/xxx`, `docs/xxx`
- **Commits** : Format conventionnel `type(scope): description`
  - `feat` : nouvelle fonctionnalite
  - `fix` : correction de bug
  - `refactor` : refactoring sans changement fonctionnel
  - `docs` : documentation
  - `chore` : maintenance, dependances
  - `test` : ajout/modification de tests
- **PRs** : Titre < 70 chars, body avec Summary + Test Plan

---

## Stack technique

| Couche | Technologie | Version | Role |
|--------|-------------|---------|------|
{{STACK_TABLE}}

> *Si ce tableau est vide ou marque "A completer", renseignez les technologies utilisees au fil du developpement.*

## Infrastructure

{{INFRASTRUCTURE}}

> *Si cette section est vide, documentez votre infrastructure (Docker, services, ports) quand elle sera en place.*

## Structure du projet

```
{{PROJECT_STRUCTURE}}
```

## Variables d'environnement

```env
{{ENV_VARS}}
```

> *Si cette section est vide, documentez vos variables d'environnement (sans valeurs sensibles) quand elles seront definies.*

## Commandes utiles

```bash
{{COMMANDS}}
```

> *Si cette section est vide, ajoutez les commandes de developpement, build et base de donnees au fur et a mesure.*

---

## Etat actuel

- **Phase** : Initialisation
- **Version** : v0.1.0
- **Derniere session** : {{DATE}}

### Historique des sessions

#### Session du {{DATE}}

1. Initialisation du projet avec `/init-project`
2. Creation de la structure PRD et documentation
3. Configuration Claude Code (settings, hooks)

---

{{#IF_HUB}}
## Ressources Partagees (Hub)

Ce projet fait partie du hub de developpement `{{HUB_PATH}}`. Les ressources partagees suivantes sont disponibles :

### Agents recommandes pour ce projet

{{AGENTS_TABLE}}

### Outils MCP projet

{{MCP_TABLE}}

> Les hooks hub (security_check, code_review, documentation_check, etc.) sont herites automatiquement.
{{/IF_HUB}}

*Ce document est maintenu a jour au fil du developpement du projet.*
