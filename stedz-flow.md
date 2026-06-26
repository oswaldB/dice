# Flow de développement stedz

Enchaînement des skills `stedz-*` pour passer d'un brief brut à du code validé.

## PHASE 1 — Setup (1 fois par projet)

### 01. `stedz-project-init`
Crée l'arborescence : `specs/`, `specs/_app/{frontend,backend}/`, `frontend/`, `backend/`.

- Niveaux : `min` (dossiers + brief) | `standard` (+ styleguide + gitignore) | `max` (+ exemples)
- Type : `frontend` | `backend` | `fullstack` (défaut)
- Produit : `specs/brief.md` (vide)
- Cwd obligatoire : racine du projet (PAS dans `specs/`)

## PHASE 2 — Spécification (brief → features)

### 02. `stedz-spec-styleguide`
Produit `specs/styleguide.md` (Tailwind CDN, générique, 15 sections).
Peut tourner dès la phase 1 (niveau `standard`).

### 03. `stedz-spec-feature-extraction`
Lit `specs/brief.md` → produit `specs/features/F-XXX-*.md` (1 fichier par feature).

- Identifie personas, features explicites/implicites/transverses (`F-T-LOG`, `F-T-ERROR`, `F-T-CONFIG`, etc.)
- + `specs/features/README.md` (index)
- Pour les stacks non-standard (extension Chrome, mobile, CLI) : ne pas adapter silencieusement, mapper les concepts stedz à l'architecture réelle

## PHASE 3 — Inventaires (features → écrans + workflows)

### 04. `stedz-spec-screen-listing`
Lit `specs/features/` → produit `specs/screens.md`.

- Inventaire : écrans frontend + leurs états (`nominal`/`empty`/`loading`/`error`/`success`/...)
- Distingue **écrans liés** vs **workflows globaux** (auth, logger, nav)

### 05. `stedz-spec-workflow-listing`
Lit `specs/features/` + `specs/screens.md` → produit `specs/workflows.md`.

- Classe chaque workflow : `frontend-écran` | `frontend-global` | `backend` (avec justification)
- Frontend par défaut, backend seulement si justifié (DB, SMTP, intégrations externes)

+> pour moi il faut rajouter une phase qui est la construction de la base de données. il faut un skill qui list le data model à partir des workflows. regarde sur /rules/flat-files-db-backend.md pour voir le fonctionnement de la db.

## PHASE 4 — Mockups (screens → HTML statiques)

### 06. `stedz-spec-mockup-generator`
Lit `specs/screens.md` + `specs/styleguide.md`.

- Produit `specs/_app/frontend/<écran>/mockups/<état>.html`
- Tailwind CDN, pas de JS, standalone (ouvrable `file://`)

## PHASE 5 — Spécifications de workflows (par workflow)

### 07. `stedz-workflow-spec-frontend` (si type = frontend, lié ou global)
Produit `specs/_app/frontend/<écran|global>/workflows/<nom>/specs/spec.md`.

- Format : frontmatter (avec `screen` ou `global`, `mockup_entry`) + blocs JSDoc `@action` + `@checkpoint`

### 07b. `stedz-workflow-spec-backend` (si type = backend)
Produit `specs/_app/backend/workflows/<nom>/specs/spec.md`.

- Même format, justification backend obligatoire dans frontmatter

+ il faut impérativement que ces spec.md rappel le data model qui les concernent.

## PHASE 6 — Validation (par workflow : N scénarios)

### 08. `stedz-workflow-validator`
Lit `specs/spec.md` → produit `scenarios-to-validate/<scenario>-validate.sh`.

- Frontend : `nominal` / `empty` / `loading` / `error`
- Backend : `nominal` / `full-load` / `api-down` / `db-down`
- Scripts bash autonomes, vérifient checkpoints via CLI `hermes`

### 09. `stedz-workflow-checkpoint-runner`
Exécute 1 scénario → produit `scenarios-to-validate/<scenario>/{rapport.log,analyse.txt}`.

- Vérifie **émission** + **ordre** + **data** des checkpoints
- Verdict : `succès` ou `erreur: <raison>`

## PHASE 7 — Code (par workflow : itératif)

### 10. `stedz-workflow-coder`
Code `code/index.js` (mega-fonction `main()`, **hors `/specs/`**).

Pour chaque étape de `spec.md` :
1. Coder l'action
2. Émettre `console.log('[CHECKPOINT]', nom, data)`
3. Lancer scenario → corriger → recommencer

---

## Règles structurelles

- `/specs/` ne contient QUE des `.md` et scripts bash (jamais de JS/HTML, sauf mockups)
- Le code vit à côté : `frontend/<écran>/code/index.js` ou `backend/workflows/<nom>/code/index.js`
- 1 mega-fonction `main()` async par workflow, point d'entrée unique
- Format checkpoint strict : `console.log('[CHECKPOINT]', '<nom>', { ...data })`
- 1 scénario = 1 fichier exécutable, autonome
- Tout est idempotent (peut tourner plusieurs fois)