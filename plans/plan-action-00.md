# Plan d'Action AZed — Implémentation par Phases

**Version :** 1.1  
**Date :** 2026-07-07  
**Basé sur :** [architecture-00.md](./architecture-00.md), [analyse-technique-00.md](./analyse-technique-00.md)  
**Correction v1.1 :** Webviews wry en fenêtres OS séparées, pas d'embedding GPUI.

---

## Structure du Projet

```
AZed/
├── assets/
│   ├── binaries/         # Binaires sidecar (marksman, tinymist, texlab)
│   ├── katex/            # Fichiers KaTeX (CSS, JS)
│   └── pdfjs/            # Fichiers PDF.js
├── crates/
│   ├── gpui/             # [INCHANGÉ]
│   ├── editor/           # [INCHANGÉ]
│   ├── project/          # [INCHANGÉ]
│   ├── workspace/        # [MODIFIÉ] Gestion fenêtres webview
│   ├── languages/        # [MODIFIÉ] LSP par défaut
│   ├── lsp/              # [MODIFIÉ] Support tinymist preview
│   ├── az_wry/           # [NOUVEAU] Gestionnaire fenêtres wry
│   ├── markdown_preview/ # [MODIFIÉ] Pipeline KaTeX
│   ├── pdf_viewer/       # [NOUVEAU] Viewer PDF.js + Synctex
│   ├── synctex/          # [NOUVEAU] Parsing Synctex
│   ├── typst_preview/    # [NOUVEAU] Preview SVG tinymist
│   ├── settings_ui/      # [NOUVEAU] Interface config graphique
│   ├── project_config/   # [NOUVEAU] Config projet (.azed/)
│   └── az_core/          # [NOUVEAU] Orchestrateur AZed
├── extensions/           # Extensions WASM (futur)
├── docs/                 # Documentation
└── plans/                # Documents d'architecture
```

---

## Phase 0 : Fork et Rebranding

**Durée estimée :** 2-3 jours  
**Priorité :** CRITIQUE (prérequis à tout le reste)

### Règle fondamentale : compatibilité extensions Zed

**`APP_NAME` reste `"Zed"`** dans `crates/paths/src/paths.rs`. Tous les chemins système (`~/.config/zed/`, `~/.local/share/zed/`, etc.) ne changent pas. Le rebranding est cosmétique (affichage, binaires) mais pas structurel.

### Étapes

| # | Tâche | Détails | Fichiers |
|---|-------|---------|----------|
| 0.1 | **Fork du dépôt Zed** | Cloner le dépôt officiel, créer un nouveau remote `origin` vers le dépôt AZed | — |
| 0.2 | **Rebranding cosmétique** | Changer le nom d'affichage, pas `APP_NAME` | `Cargo.toml` (description, metadata), `crates/zed/src/main.rs` (titre fenêtre) |
| 0.3 | **Identité visuelle** | Nouveau logo, icônes, nom d'application | `assets/icons/`, `crates/zed/resources/` |
| 0.4 | **CI/CD** | Adapter GitHub Actions pour builder AZed | `.github/workflows/` |
| 0.5 | **README** | Rédiger la présentation du projet | `README.md` |
| 0.6 | **Vérification `APP_NAME`** | S'assurer que `paths.rs` est untouched | `crates/paths/src/paths.rs` |

### Validation
- `cargo build` réussi
- L'application se lance et affiche "AZed" dans le titre
- `~/.config/zed/` est toujours utilisé (pas `~/.config/az/`)
- Les extensions Zed installées sont chargées

---

## Phase 1 : Fenêtres Webview wry (Fondation)

**Durée estimée :** 2 semaines  
**Priorité :** CRITIQUE (prérequis pour les phases 2, 3, 4)  
**Dépendances :** Phase 0

### Description
Créer la fondation : ouvrir des fenêtres OS séparées avec `wry` pour le rendu HTML/JS/CSS. Chaque preview (KaTeX, PDF.js, SVG Typst) sera une fenêtre wry attachée à un workspace. **GPUI n'est pas modifié.**

### Étapes

| # | Tâche | Détails | Fichiers |
|---|-------|---------|----------|
| 1.1 | **Créer la crate `az_wry`** | Dépendance wry + wrapper simple | `crates/az_wry/Cargo.toml` |
| 1.2 | **`PreviewWindow` struct** | Fenêtre wry standalone (titre, taille, position) | `crates/az_wry/src/preview_window.rs` |
| 1.3 | **IPC : Rust → Webview** | `evaluate_script()` pour envoyer données | `crates/az_wry/src/ipc.rs` |
| 1.4 | **IPC : Webview → Rust** | `ipc_handler` pour recevoir events | `crates/az_wry/src/ipc.rs` |
| 1.5 | **Intégration workspace** | Créer/supprimer une webview quand un workspace s'ouvre/se ferme | `crates/workspace/src/workspace.rs` |
| 1.6 | **Test : afficher "Hello AZed"** | Fenêtre wry avec titre "AZed Preview" | Test manuel |
| 1.7 | **Test : IPC roundtrip** | Webview envoie "ping" → Rust répond "pong" | `crates/az_wry/tests/` |
| 1.8 | **Positionnement fenêtre** | Placer la preview à côté de la fenêtre GPUI | `crates/az_wry/src/preview_window.rs` |

### Validation
- Une fenêtre webview s'ouvre à côté de l'éditeur
- `evaluate_script("hello()")` fonctionne
- `ipc_handler` reçoit les messages JS
- La fenêtre se ferme quand le workspace se ferme
- Fenêtre fonctionnelle sur Linux (webkit2gtk) et macOS (WKWebView)

### Multi-session
Chaque workspace possède sa propre `Vec<PreviewWindow>`. Deux projets ouverts = deux groupes de fenêtres indépendants. Aucun état partagé.

---

## Phase 2 : Preview HTML Scientifique (Markdown + KaTeX)

**Durée estimée :** 3-4 semaines  
**Priorité :** ÉLEVÉE (fonctionnalité phare)  
**Dépendances :** Phase 1

### Étapes

| # | Tâche | Détails | Fichiers |
|---|-------|---------|----------|
| 2.1 | **Pipeline de rendu** | Compiler MD → HTML avec pulldown-cmark + KaTeX | `crates/markdown_preview/src/preview_engine.rs` |
| 2.2 | **Wiki-links** | Prétraitement `[[lien]]` → `<a href>` pour marksman | `crates/markdown_preview/src/wiki_links.rs` |
| 2.3 | **Préambule LaTeX** | Injection du contenu de `.latex_preamble.tex` | `crates/markdown_preview/src/latex_preamble.rs` |
| 2.4 | **Template HTML** | Page HTML complète avec KaTeX CSS/JS embarqué | `crates/markdown_preview/src/template.html` |
| 2.5 | **Bundle KaTeX** | Copier les assets KaTeX dans AZed | `assets/katex/` |
| 2.6 | **Panneau de preview** | Remplacer le panneau GPUI par la webview wry | `crates/markdown_preview/src/markdown_preview.rs` |
| 2.7 | **Mise à jour temps réel** | Re-rendu à chaque modification du buffer | `crates/markdown_preview/src/markdown_preview.rs` |
| 2.8 | **Support `$...$` et `$$...$$`** | Détection et conversion pour KaTeX | `crates/markdown_preview/src/math.rs` |

### Validation
- Preview en direct d'un fichier .md avec formules mathématiques
- Les wiki-links marksman sont cliquables
- Le préambule LaTeX est chargé et les macros sont disponibles dans KaTeX
- La preview se met à jour automatiquement

---

## Phase 3 : Viewer PDF (PDF.js + Synctex)

**Durée estimée :** 4-5 semaines  
**Priorité :** ÉLEVÉE (indispensable pour LaTeX)  
**Dépendances :** Phase 1

### Étapes

| # | Tâche | Détails | Fichiers |
|---|-------|---------|----------|
| 3.1 | **Bundle PDF.js** | Intégrer PDF.js dans les assets | `assets/pdfjs/` |
| 3.2 | **Crate `pdf_viewer`** | Webview wry chargée avec PDF.js | `crates/pdf_viewer/` |
| 3.3 | **Panneau PDF** | Ajouter le panneau dans workspace | `crates/pdf_viewer/src/pdf_panel.rs` |
| 3.4 | **Navigation** | Scroll, zoom, recherche dans PDF | Via PDF.js API |
| 3.5 | **Crate `synctex`** | Parser les fichiers `.synctex.gz` en Rust | `crates/synctex/src/lib.rs` |
| 3.6 | **IPC Synctex→PDF** | Position éditeur → page PDF correspondante | `crates/pdf_viewer/src/synctex_forward.rs` |
| 3.7 | **IPC PDF→Synctex** | Click PDF → position éditeur | `crates/pdf_viewer/src/synctex_inverse.rs` |
| 3.8 | **Compilation LaTeX (latexmk)** | Déclencher `latexmk` sur le fichier .tex courant | `crates/pdf_viewer/src/latex_build.rs` |
| 3.9 | **Auto-reload PDF** | Recharger le PDF après compilation | `crates/pdf_viewer/src/auto_reload.rs` |

### Validation
- Ouverture d'un fichier .tex → compilation → PDF affiché
- `Cmd+Click` sur PDF → curseur placé dans l'éditeur (inverse search)
- Clic dans l'éditeur → scroll de la page PDF correspondante (forward search)
- Recompilation automatique au save

### Précédent AZprose
Les modules `latex_engine/synctex.rs` et `PdfViewer.svelte` sont réutilisables comme référence.

---

## Phase 4 : Intégration des LSPs

**Durée estimée :** 3-4 semaines  
**Priorité :** ÉLEVÉE  
**Dépendances :** Phase 0 (fork minimal)

### 4.1 marksman (Markdown)

| # | Tâche | Détails | Fichiers |
|---|-------|---------|----------|
| 4.1.1 | **Télécharger binaire marksman** | Bundle pour les 3 OS | `assets/binaries/marksman-*` |
| 4.1.2 | **Enregistrer marksman comme LSP** | Remplacer le LSP Markdown par défaut | `crates/languages/src/markdown.rs` |
| 4.1.3 | **Config workspaceFolders** | Pour le mode omniscience | `crates/languages/src/markdown.rs` |
| 4.1.4 | **default_settings.json** | Ajouter marksman aux settings | `crates/settings/src/default_settings.json` |

### 4.2 tinymist (Typst)

| # | Tâche | Détails | Fichiers |
|---|-------|---------|----------|
| 4.2.1 | **Télécharger binaire tinymist** | Bundle pour les 3 OS | `assets/binaries/tinymist-*` |
| 4.2.2 | **Enregistrer tinymist LSP** | Ajouter support du langage Typst | `crates/languages/src/typst.rs` |
| 4.2.3 | **Tree-sitter Typst** | Ajouter le parseur syntaxique si disponible | `crates/languages/src/typst.rs` |
| 4.2.4 | **Preview tinymist** | Lancer `tinymist preview` en sidecar, afficher dans webview | `crates/lsp/src/tinymist_preview.rs` |
| 4.2.5 | **Diagnostics** | Afficher les erreurs de compilation Typst | `crates/lsp/src/tinymist_preview.rs` |

### 4.3 texlab (LaTeX)

| # | Tâche | Détails | Fichiers |
|---|-------|---------|----------|
| 4.3.1 | **Télécharger binaire texlab** | Bundle pour les 3 OS | `assets/binaries/texlab-*` |
| 4.3.2 | **Enregistrer texlab LSP** | Ajouter support LaTeX | `crates/languages/src/latex.rs` |
| 4.3.3 | **Complétion BibTeX** | Support bibliographie | `crates/languages/src/latex.rs` |
| 4.3.4 | **Formatage** | latexindent ou intégré | `crates/languages/src/latex.rs` |

### Validation
- Autocomplétion, diagnostics, hover pour les 3 langages
- Preview Typst fonctionnelle dans un panneau webview
- Navigation vers les définitions (bibliographie, labels)

---

## Phase 5 : Configuration Graphique

**Durée estimée :** 3-4 semaines  
**Priorité :** MOYENNE  
**Dépendances :** Phase 1

### Étapes

| # | Tâche | Détails | Fichiers |
|---|-------|---------|----------|
| 5.1 | **API des modules de config** | Trait `SettingsModule` | `crates/settings_ui/src/module.rs` |
| 5.2 | **Générateur de formulaire** | JSON Schema → formulaire HTML dans webview | `crates/settings_ui/src/form_generator.rs` |
| 5.3 | **Panneau de configuration** | Nouveau panneau workspace listant les modules | `crates/settings_ui/src/settings_panel.rs` |
| 5.4 | **Module : Général** | Thème, langue, police | `crates/settings_ui/src/modules/general.rs` |
| 5.5 | **Module : Markdown** | Préambule, moteur math, formatage | `crates/settings_ui/src/modules/markdown.rs` |
| 5.6 | **Module : LaTeX** | Compilateur, options latexmk | `crates/settings_ui/src/modules/latex.rs` |
| 5.7 | **Module : Typst** | Options de compilation | `crates/settings_ui/src/modules/typst.rs` |
| 5.8 | **Sauvegarde** | Écriture dans le fichier de config | `crates/settings_ui/src/storage.rs` |

### Validation
- Un panneau "Paramètres" accessible depuis le menu ou un raccourci
- Chaque module de langage affiche ses options
- Les changements sont persistés et appliqués en temps réel

---

## Phase 6 : Configuration Projet (.azed/)

**Durée estimée :** 2-3 semaines  
**Priorité :** MOYENNE  
**Dépendances :** Phase 5

### Étapes

| # | Tâche | Détails | Fichiers |
|---|-------|---------|----------|
| 6.1 | **Détection projet** | Rechercher `.azed/` à l'ouverture d'un dossier | `crates/project_config/src/detection.rs` |
| 6.2 | **Config projet** | Lecture/écriture de `.azed/config.json` | `crates/project_config/src/config.rs` |
| 6.3 | **Session projet** | Sauvegarde des tabs ouverts | `crates/project_config/src/session.rs` |
| 6.4 | **Préambule partagé** | `.azed/.latex_preamble.tex` chargé par tous les fichiers MD | `crates/project_config/src/preamble.rs` |
| 6.5 | **Atomic saves** | Écriture atomique des fichiers de config | `crates/project_config/src/storage.rs` |
| 6.6 | **UI : sélecteur de projet** | Liste des projets récents, ouverture rapide | `crates/project_config/src/project_selector.rs` |

### Précédent AZprose
Le système `.azprose/` est fonctionnel et testé. Reprendre le pattern :
- Lecture au démarrage (synchrone, via URL param)
- Écriture atomique (write → rename)
- Session scope par workspace

---

## Phase 7 : Fonctionnalités Futures

| Fonctionnalité | Description | Priorité | Dépendances |
|----------------|-------------|----------|-------------|
| **Indexation/Recherche** | Moteur de recherche full-text style Obsidian | Basse | Phase 4 |
| **OpenCode** | Intégration IA/MCP via OpenCode | Basse | Phase 1 |
| **Présentation (slides)** | Mode diaporama depuis Markdown | Basse | Phase 2 |
| **Graphe de connaissances** | Visualisation des liens entre notes | Basse | Phase 4 (marksman) |
| **Collaboration** | Édition temps réel (Zed collab) | Très basse | — |

---

## Résumé des Durées et Dépendances

```
Phase 0 : Fork + Rebranding                  [terminé]
    │
    ▼
Phase 1 : Fenêtres webview wry (fondation)  [2 semaines]  ─── Dépend de Phase 0
    │
    ├───────────┬───────────┐
    ▼           ▼           ▼
Phase 2 :      Phase 3 :   Phase 4 :        [2-4 semaines chacune]
Preview HTML   Viewer PDF  Preview SVG       ─── Dépendent de Phase 1
KaTeX          + Synctex   Typst/tinymist    ─── EN PARALLÈLE
    │           │           │
    └───────────┼───────────┘
                ▼
Phase 5 : LSPs (marksman, tinymist, texlab) [3-4 semaines] ◄─── Dépend de Phase 0
    │
    ▼
Phase 6 : Configuration graphique           [3-4 semaines] ◄─── Dépend de Phase 1
    │
    ▼
Phase 7 : Configuration projet (.azed/)     [2-3 semaines] ◄─── Dépend de Phase 6
    │
    ▼
Phase 8 : Fonctionnalités futures           [continu]

Total estimé phases 1-7 : ~16-22 semaines (4-5 mois)
```

Les phases 2, 3 et 4 peuvent être développées en parallèle après la Phase 1.
Les phases 6 et 7 sont les moins urgentes et peuvent être reportées si nécessaire.

---

## Recommandation de Démarrage Immédiat

1. **Phase 0 terminée** — fork + rebranding OK
2. **Phase 1 : fenêtres wry** — le socle de tout le reste. Objectif : une fenêtre webview qui s'ouvre, affiche du HTML, et communique avec l'éditeur en Rust
3. **Phase 1 ne touche pas à GPUI** — wry s'utilise en fenêtre OS native, aucune modification du moteur de rendu

La plus grande leçon d'AZprose est que l'architecture modulaire doit être pensée dès le départ. Chaque nouveau composant doit suivre le pattern `ModuleRegistry` sans compromis.
