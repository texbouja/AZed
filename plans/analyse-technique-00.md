# Analyse Technique : Architecture AZed (Fork Éditeur Zed)

**Version :** 1.0  
**Date :** 2026-07-07  
**Statut :** Validé  
**Basé sur :** [architecture-00.md](./architecture-00.md) + Retours d'expérience AZprose

---

## 1. Résumé Exécutif

AZed est un fork de l'éditeur Zed ciblant la rédaction scientifique multi-langage (Markdown, Typst, LaTeX). L'approche consiste à exploiter l'architecture existante de Zed (performante, Rust native, GPUI) tout en comblant ses lacunes : rendu HTML limité, absence de viewer PDF, et configuration exclusivement textuelle.

L'expérience du projet AZprose (Tauri + Svelte + CodeMirror) a démontré la viabilité fonctionnelle mais a révélé des fragilités d'intégration : éditeur non natif, duplication des pipelines de rendu, synchronisation imparfaite Typst, et architecture monolithique. **AZed repart d'une base plus solide : Zed lui-même**, en y ajoutant des composants modulaires via son système d'extensions et quelques modifications du noyau.

---

## 2. Analyse de l'Architecture Zed

### 2.1 Vue d'ensemble (d'après la documentation publique)

```
crates/
├── gpui/              # Framework GUI propre à Zed
│   ├── platform/      # Backends (macOS, Linux, Windows)
│   ├── elements/      # Éléments graphiques de base
│   └── text/          # Rendu texte
├── editor/            # Éditeur de texte central
│   ├── movements/     # Navigation curseur
│   ├── display/       # Rendu visuel
│   └── items/         # Items d'éditeur
├── project/           # Gestion de projet/workspace
├── workspace/         # Gestion des fenêtres et tabs
├── lsp/               # Intégration Language Server Protocol
├── languages/         # Support des langages + serveurs associés
│   └── markdown/      # Parseur tree-sitter + serveur Markdown
├── settings/          # Système de configuration (JSON)
├── extensions/        # Système d'extensions (WASM)
├── markdown_preview/  # Aperçu HTML du Markdown (GPUI)
└── collab/            # Fonctionnalités collaboratives
```

### 2.2 Points d'extension critiques pour AZed

| Point d'entrée | Usage prévu | Fichier clé |
|----------------|-------------|-------------|
| `languages::LanguageRegistry` | Enregistrer marksman comme LSP Markdown par défaut | `crates/languages/src/markdown.rs` |
| `lsp::LspAdapterDelegate` | Interface LSP extensible | `crates/lsp/src/lsp_adapter.rs` |
| `workspace::Workspace` | Ajouter des panneaux (preview, PDF) | `crates/workspace/src/workspace.rs` |
| `settings::Settings` | Ajouter des sections de config | `crates/settings/src/default_settings.json` |
| `gpui::Window` | Créer des vues embarquées | `crates/gpui/src/window.rs` |
| `extensions::ExtensionApi` | API pour extensions WASM | `crates/extensions/` |

### 2.3 Contraintes identifiées

1. **GPUI ne supporte pas le HTML complet** — Le renderer GPUI est conçu pour des éléments natifs, pas un moteur de rendu web. La crate `wry` (webview) doit être intégrée comme un élément GPUI personnalisé.

2. **Le système d'extensions Zed est WASM-based** — Les modifications profondes (remplacement du LSP par défaut, ajout de composants graphiques) nécessitent des modifications du noyau. Une approche hybride est recommandée : modifications du noyau pour l'infrastructure + extensions WASM pour les fonctionnalités optionnelles.

3. **Pas de viewer PDF natif** — Le PDF doit être rendu dans une webview (PDF.js via wry).

4. **Pas d'API de configuration graphique** — Le module de configuration doit être construit de toutes pièces.

---

## 3. Analyse Comparative : Approches Possibles

| Critère | Fork pur Zed | Extension WASM | Sidecar (comme AZprose) |
|---------|-------------|----------------|-------------------------|
| **Profondeur d'intégration** | Maximale | Limitée par l'API | Faible (processus externe) |
| **Maintenabilité** | Complexe (merge) | Simple | Moyenne (IPC) |
| **Performances** | Optimales | Bonnes | Limitée par IPC |
| **Accès au noyau** | Total | API restreinte | Aucun (communication process) |
| **Risque de régression** | Élevé | Faible | Nul |

**Décision :** Fork pur pour l'infrastructure de base (webview, PDF, LSPs), avec les modules additionnels en sidecar si nécessaire (indexation, OpenCode).

---

## 4. Architecture Cible : Composants et Dépendances

```
┌──────────────────────────────────────────────────────────────────────┐
│                        AZed (Fork Zed)                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Core (Zed inchangé ou modifié minimalement)                 │   │
│  │  ┌────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │   │
│  │  │ GPUI   │ │ Editor   │ │ Terminal │ │ Collaboration   │ │   │
│  │  └────────┘ └──────────┘ └──────────┘ └──────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                   │                                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Modules AZed (nouveaux composants)                          │   │
│  │                                                              │   │
│  │  ┌─────────────────┐    ┌─────────────────┐                  │   │
│  │  │  wry-webview     │    │  Configuration  │                  │   │
│  │  │  (composant GPU  │◄──►│  Graphique       │                  │   │
│  │  │   pour HTML/JS) │    │  (panneau de     │                  │   │
│  │  └──────┬──────────┘    │   réglages)     │                  │   │
│  │         │               └─────────────────┘                  │   │
│  │         ▼                                                    │   │
│  │  ┌──────────────────────────────────────────┐                │   │
│  │  │  Preview HTML (wry + KaTeX)               │                │   │
│  │  │  Viewer PDF  (wry + PDF.js)               │                │   │
│  │  └──────────────────────────────────────────┘                │   │
│  │                                                              │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                    │   │
│  │  │ Mksman   │ │ Tinymist │ │ TeXlab   │ ← LSPs              │   │
│  │  │ (Markdwn)│ │ (Typst)  │ │ (LaTeX)  │   (sidecar ou       │   │
│  │  └──────────┘ └──────────┘ └──────────┘    embeddé Rust)    │   │
│  │                                                              │   │
│  │  ┌──────────────────────────────────────────┐                │   │
│  │  │  Module Configuration Projet              │                │   │
│  │  │  (dossier .azed/ à la racine du projet,  │                │   │
│  │  │   like Obsidian vaults)                  │                │   │
│  │  └──────────────────────────────────────────┘                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Services externes (sidecars)                                │   │
│  │  ┌──────────────────┐  ┌──────────────────┐                  │   │
│  │  │  Indexation/     │  │  OpenCode        │                  │   │
│  │  │  Recherche       │  │  (IA/MCP)        │                  │   │
│  │  └──────────────────┘  └──────────────────┘                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.1 Ordre d'intégration (par dépendance)

```
Phase 1 : wry-webview (fondation)
  └─► Phase 2 : Preview HTML + KaTeX
      └─► Phase 3 : Viewer PDF
          └─► Phase 4 : LSPs (marksman, tinymist, texlab)
              └─► Phase 5 : Configuration graphique
                  └─► Phase 6 : Configuration projet (.azed/)
                      └─► Phase 7 (future) : Indexation, OpenCode
```

---

## 5. Analyse Technique par Composant

### 5.1 Composant wry-webview (Phase 1)

**Problème :** GPUI ne permet pas d'afficher du HTML/JS/CSS complet.  
**Solution :** Intégrer `wry` (la webview utilisée par Tauri) comme un élément GPUI personnalisé.  
**Précédent :** AZprose utilise déjà wry via Tauri (son backend de rendu natif). Le pattern est validé.

**Approche d'intégration :** Créer une crate `gpui_wry` ou intégrer directement dans `gpui::platform` :
- Sur macOS : `WKWebView`
- Sur Linux : `webkit2gtk`
- Sur Windows : `WebView2`

```rust
// Concept : Élément GPUI encapsulant une webview wry
struct WebViewElement {
    webview: wry::WebView,
    bounds: Bounds,
}

impl Element for WebViewElement {
    fn render(&mut self, cx: &mut RenderContext) {
        // Redimensionner la webview pour correspondre au bounds GPUI
        self.webview.set_bounds(self.bounds);
    }
}
```

**Risques :** 
- Synchronisation des cycles de vie GPUI/wry (redimensionnement, focus)
- Gestion des entrées clavier entre GPUI et la webview
- Performances sur Windows (WebView2 peut être lent au démarrage)

### 5.2 Preview HTML + KaTeX (Phase 2)

**Problème :** La preview Markdown de Zed utilise `pulldown-cmark` sans support mathématique.  
**Solution :** Pipeline de rendu personnalisé avec KaTeX.

**Pipeline proposé :**
```
Markdown source
    │
    ▼
[Prétraitement : Wiki-links → liens MD]
    │
    ▼
[Compilation MD → HTML (pulldown-cmark)]
    │
    ▼
[Injection préambule LaTeX (.latex_preamble.tex)]
    │
    ▼
[Assemblage template HTML + KaTeX]
    │
    ▼
[Affichage dans webview wry]
```

**Tests AZprose :** Le pipeline markdown-it + MathJax a fonctionné. KaTeX est plus performant et plus prévisible. Le préambule LaTeX global a été validé.

**Fichiers à modifier :**
- `crates/markdown_preview/src/preview_engine.rs` — Remplacer le renderer
- `crates/markdown_preview/src/markdown_preview.rs` — Intégrer la webview

### 5.3 Viewer PDF (Phase 3)

**Problème :** Aucun viewer PDF natif dans Zed.  
**Solution :** PDF.js dans une webview wry, avec pont IPC pour Synctex.

**Architecture :**
```
Éditeur LaTeX
    │
    ├── Synctex (analyse du fichier .synctex.gz)
    │      │
    │      ▼
    └── IPC → PDF.js (scrollTo, highlight)
    
PDF.js
    │
    ├── click → position → Synctex → éditeur (inverse search)
    └── scroll → Synctex → éditeur (forward search tracking)
```

**Précédent AZprose :** L'intégration PDF.js + Synctex a été développée et testée. Le pattern est directment réutilisable :
- `src-tauri/src/latex_engine/synctex.rs` → logique Synctex en Rust
- `src/components/pdf/PdfViewer.svelte` → configuration PDF.js

**Fichiers à créer :**
- `crates/pdf_viewer/` — Nouvelle crate avec webview wry + PDF.js
- `crates/synctex/` — Parser Synctex (port de la logique AZprose)

### 5.4 LSPs : marksman, tinymist, texlab (Phase 4)

#### 5.4.1 marksman (Markdown)

**Remplacement du LSP Markdown par défaut.**

Zed utilise actuellement un serveur Markdown dérivé de VS Code. Il faut le remplacer par marksman.

**Points d'intégration :**
```rust
// Dans crates/languages/src/markdown.rs
languages::Language::new("Markdown", tree_sitter_markdown::language())
    .with_language_server(LanguageServerId::new("marksman"))
```

**Précédent AZprose :** Le rapport `marksman-katex.md` détaille l'intégration. Marksman s'intègre facilement via LSP standard.

**Attention :** Marksman propose un mode "omniscience" (comme Obsidian) qui nécessite que tous les fichiers du projet soient connus du serveur. Il faut configurer le workspaceFolders correctement.

#### 5.4.2 tinymist (Typst)

**Intégration LSP + Preview typst.**

tinymist offre :
1. Un serveur LSP standard (diagnostics, completion, hover)
2. Un serveur de preview HTTP (compilation incrémentale)

**Architecture proposée :**
```
tinymist lsp          → Zed LSP (diagnostics, completion)
tinymist preview      → wry webview dans un panneau latéral
```

**Précédent AZprose :** L'approche `tinymist preview` sidecar avec affichage dans une webview a été validée (AZprose utilise Tauri WebviewWindow pointant vers `http://localhost:<port>`). Le diagnostic parsing du preview server a été implémenté.

#### 5.4.3 texlab (LaTeX)

**Serveur LSP LaTeX avec synchronisation Synctex.**

**Architecture :**
```
texlab LSP          → Zed LSP (complétion, signature, document symbols)
Compilation latexmk → processus externe (log parsing, diagnostic)
Synctex             → bridge entre éditeur et PDF viewer
```

**Précédent AZprose :** Le module `latex_engine/` gère la compilation et Synctex. La communication avec PDF.js est fonctionnelle.

### 5.5 Configuration Graphique (Phase 5)

**Problème :** La configuration de Zed est en JSON pur. Le public cible (enseignants, chercheurs) n'est pas familier avec ce format.  
**Solution :** Un panneau de configuration graphique intégré, avec une API pour ajouter des sous-modules.

**Architecture :**
```
SettingsModule (trait Rust)
├── id: &'static str
├── title: &'static str
├── icon: IconName
├── schema: JsonSchema (validation des champs)
├── render(config: &Config, cx: &mut ViewContext) -> SettingsPane
└── on_change(callback: fn(&mut Config))
```

**Implémentation :** Le panneau de configuration s'affiche dans une webview wry avec un formulaire généré à partir du schéma JSON. Chaque module enregistre son schéma et son handler.

**Précédent AZprose :** `SettingsOverlay.svelte` (668 lignes) — monolithique et difficile à maintenir. Le pattern `ModuleRegistry` résout ce problème.

### 5.6 Configuration Projet (Phase 6)

**Système de "vault" à la Obsidian.**

Chaque projet contient un dossier `.azed/` :
```
mon-projet/
├── .azed/
│   ├── config.json       # Configuration du projet
│   ├── session.json      # État de session (tabs ouverts)
│   └── .latex_preamble.tex  # Préambule LaTeX partagé
├── chapitre-01.md
├── chapitre-02.typ
└── article.tex
```

**Précédent AZprose :** Le système `.azprose/config.json` + `session.json` est opérationnel. Leçons apprises :
- Passer le chemin racine de manière synchrone (URL param) plutôt que par events asynchrones
- Écritures atomiques pour éviter la corruption
- Scoper le state par workspace (pas de localStorage partagé)

---

## 6. Risques et Atténuations

| Risque | Impact | Probabilité | Atténuation |
|--------|--------|-------------|-------------|
| **Divergence avec upstream Zed** — Difficulté à merger les mises à jour | Élevé | Élevée | Isolation des modifications dans des crates séparées ; merge réguliers |
| **Performances webview** — Latence entre GPUI et wry | Moyen | Moyenne | Utiliser wry en mode transparent ; limiter les IPC ; préférer le chargement direct URL |
| **Stabilité tinymist** — API en évolution rapide | Moyen | Élevée | Version pin ; fallback sur la compilation typst native |
| **Windows WebView2** — Dépendance runtime non disponible | Faible | Faible | Bundle WebView2 installer ; fallback sur message clair |
| **Complexité Synctex** — Fichier .synctex.gz volumineux | Faible | Faible | Parsing lazy ; cache de pages |
| **Marksman omniscience** — Perf sur grands projets | Faible | Moyenne | WorkspaceFolders filtrés ; exclusion node_modules/.git |

---

## 7. Retours d'Expérience AZprose Applicables

| Leçon AZprose | Application dans AZed |
|---------------|----------------------|
| App.svelte monolithique (1518 lignes) → ModuleRegistry | Architecture modulaire dès le départ ; un trait par composant |
| Deux pipelines Markdown (ProseMark + markdown-it) → unification impossible | Un seul pipeline (pulldown-cmark + KaTeX) ; pas de WYSIWYG |
| tinymist preview évite la compilation Typst directe | Adopter `tinymist preview` en sidecar |
| Handshake race condition sur le workspace | Passer le workspace path de manière synchrone |
| localStorage partagé entre projets | State projet dans `.azed/` fichiers, pas de localStorage |
| Atomic writes pour config | Implementer atomic saves (write → rename) |
| Duplication des registres d'extensions | Central registry pattern (un seul fichier de mapping) |

---

## 8. Compatibilité Extensions Zed

### 8.1 Principe

Les extensions Zed sont des modules WASM stockées dans `extensions_dir()` = `data_dir().join("extensions")`. Pour qu'elles restent compatibles, **AZed conserve `APP_NAME = "Zed"`** dans `crates/paths/src/paths.rs`. Tous les chemins système restent identiques.

### 8.2 Chemins

| Rôle | Chemin (Linux) | Note |
|------|---------------|------|
| Config partagée | `~/.config/zed/` | settings.json, keymap.json, thèmes |
| Data partagée | `~/.local/share/zed/` | extensions, db, langages |
| **Extensions** | `~/.local/share/zed/extensions/` | **Compatibilité totale** |
| Projet (Zed) | `<projet>/.zed/` | Ned settings legacy |
| **Projet (AZed)** | `<projet>/.azed/` | Settings AZed (vault) |
| **Config AZed** | `~/.config/az/` | UI state, préférences AZed |
| **Modules AZed** | `~/.local/share/az/` | Composants additionnels AZed |

### 8.3 Implications

- **Extensions installées par Zed** : visibles et utilisables dans AZed immédiatement
- **Settings.json partagé** : un utilisateur Zed/AZed a la même configuration de base
- **Ajouts AZed** : stockés dans des dossiers séparés (`~/.config/az/`, `~/.local/share/az/`, `.azed/`)
- **`local_settings_folder_name()`** : conserve `.zed` pour la rétrocompatibilité ; AZed lit aussi `.azed/`

### 8.4 Mise en garde

Si un jour le projet doit diverger au point de ne plus supporter les extensions Zed, on pourra changer `APP_NAME` (c'est une constante unique) mais ce n'est pas recommandé dans cette phase.

---

## 9. Impact sur l'Architecture Zed Existante

### Modifications du noyau nécessaires

1. **GPUI** : Ajout du support wry (élément webview personnalisé)
2. **Languages** : Remplacement du LSP Markdown par défaut
3. **Workspace** : Ajout du système de panneaux configurables
4. **Settings** : Ajout du module de configuration graphique

### Crates à créer

| Crate | Description | Dépend de |
|-------|-------------|-----------|
| `gpui_wry` | Intégration wry comme élément GPUI | gpui, wry |
| `markdown_preview_az` | Preview scientifique (KaTeX) | markdown_preview, gpui_wry |
| `pdf_viewer` | Viewer PDF avec PDF.js | gpui_wry, synctex |
| `synctex` | Parsing Synctex | — |
| `settings_ui` | Interface de config graphique | gpui_wry |
| `project_config` | Gestion configuration projet | — |
| `az_core` | Orchestrateur des modules AZed | tout ce qui précède |

### Crates à modifier

| Crate | Modification |
|-------|-------------|
| `paths/src/paths.rs` | **Aucune** — on garde `APP_NAME = "Zed"` pour la compatibilité |
| `languages/src/markdown.rs` | Remplacer LSP par marksman |
| `settings/src/default_settings.json` | Ajouter les entrées marksman, tinymist, texlab |
| `workspace/src/workspace.rs` | Ajouter les panneaux configurables |
| `lsp/src/lsp_adapter.rs` | Adapter pour tinymist preview |

---

## 10. Conclusion Technique

L'architecture proposée est réalisable. Les points critiques sont :

1. **L'intégration wry/GPUI** est le fondement dont tout dépend — c'est la première étape et la plus risquée. Tauri a déjà résolu ce problème au niveau de son framework ; AZed doit le résoudre au niveau GPUI.

2. **Les LSPs** (marksman, tinymist, texlab) s'intègrent via le protocole LSP standard. Aucune modification majeure du noyau n'est nécessaire pour les faire fonctionner.

3. **PDF.js + Synctex** est un problème déjà résolu par de nombreux projets (LaTeX-Workshop, AZprose). Le port en Rust est direct.

4. **La configuration graphique** est le composant le plus indépendant — peut être développé en parallèle.

5. **Le risque principal** n'est pas technique mais stratégique : la divergence avec le upstream Zed. Des merges réguliers et une isolation des modifications dans des crates séparées sont essentiels.

**Recommandation :** Procéder par phases (1→6) avec validation fonctionnelle à chaque étape avant de passer à la suivante.
