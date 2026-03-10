# {{PROJECT_NAME}} — Documentation Technique

> Document de reference pour le developpement du projet.
> Derniere mise a jour : {{DATE}}
>
> **Emplacement** : `{{PROJECT_PATH}}`

---

## Documentation Matrix

| Document | Chemin | Description | Statut |
|----------|--------|-------------|--------|
| **CLAUDE.md** | `./CLAUDE.md` | Ce fichier — reference technique | Actif |
| PRD Index | `docs/PRD/00-INDEX.md` | Navigation PRD | Actif |
| Vision | `docs/PRD/01-VISION.md` | Problem statement, objectifs | A completer |
| Personas | `docs/PRD/02-PERSONAS.md` | Utilisateurs cibles | A completer |
| Features | `docs/PRD/03-FEATURES.md` | Inventaire fonctionnel | Actif |
| Backlog | `docs/PRD/04-BACKLOG.md` | Backlog priorise | Actif |
| Roadmap | `docs/PRD/05-ROADMAP.md` | Phases et milestones | A completer |
| Specifications | `docs/PRD/06-SPECIFICATIONS.md` | Architecture, APIs | A completer |
| Decisions | `docs/PRD/07-DECISIONS.md` | ADRs | Actif |
| Changelog | `docs/PRD/08-CHANGELOG.md` | Journal des changements | Actif |
| Metriques | `docs/PRD/09-METRICS.md` | KPIs et suivi | A completer |
| Features docs | `docs/features/` | Documentation par feature | Actif |

---

## Directives Claude — Regles de mise a jour obligatoires

> **IMPORTANT** : Ces 6 regles doivent etre suivies systematiquement.

1. **Apres chaque commit significatif** → MAJ `docs/PRD/08-CHANGELOG.md`
2. **Apres chaque feature terminee** → MAJ `docs/PRD/03-FEATURES.md` (changer statut)
3. **Apres chaque decision technique** → Ajouter ADR dans `docs/PRD/07-DECISIONS.md`
4. **Lors de modification d'une feature documentee** → MAJ `docs/features/[feature].md`
5. **Lors de l'ajout d'un nouveau document** → MAJ Documentation Matrix ci-dessus
6. **MAJ PRD complete** (CHANGELOG, FEATURES, BACKLOG) apres chaque changement significatif

---

## Workflow Implementation Feature

```
1. git checkout main && git pull
2. git checkout -b feat/nom-feature
3. Implementer + tests
4. Mettre a jour docs (FEATURES, CHANGELOG, feature doc si existant)
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

*Ce document est maintenu a jour au fil du developpement du projet.*
