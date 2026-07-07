# Appliction d'édition de texte scientifique multi-langage (Markdwon-Typst-LaTeX)
Une application Rust based offrant une architecture commune pour pouvoir rédiger des documents scientifques en Markdown, Typst ou LaTeX. 
L'application repose sur l'éditeur établit Zed et profite de son infrastructure pour ajouter les fonctionnalités désirées en tant qu'extensions (par défaut).
Des serveurs LSP particuliers seront utilisés et devront s'intégrer avec l'architecture LSP déjà présente dans Zed. 

## Lacunes de Zed à combler 
Zed est pensé pour être un éditeur de texte performant. 
N'offre que des capacités limitées de rendu HTML et pas de support du tout pour le PDF.

## Modifications d'architecture : 
- **consolider l'architecture gestion par projets** : un dossier de configuration par projet comme les vaults Obsidian.
- **Rendu HTML** : le renderer HTML de Zed (GPUI based) n'est pas assez complet. S'inspirer de Tauri : prévoir un container webview à base de wry pour un support plus complet de JS, CSS, SVG... Le composant graphique doit s'intégrer dans la logique des tab Zed. 
- **Viewer PDF** : indispensable pour le projet, un viewer PDF intégré capable de communiquer avec les autres composants (inverse/forward search pour latex). PDFjs est un bon candidat (confiramation du besoin d'un moteur de rendu HTML plus solide). 
- **Module de configuration** : ajouter un module de configuration graphique à Zed. Un container avec API pour facilement ajouter des sous-modules de configuration. Le module ne concernera pas la configuration de Zed lui même (par json dans éditeur). Le public cible est assez novice et risque d'être dérouté par une interface de configuration par texte pur. 

## Markdwon 
Utiliser marksman en tant que serveur par défaut (à la place de celui de Zed). Exploiter tout son potentiel "omniscience" à la Obsidian.
Ajouter un support Katex dans le pipeline de rendu avec la possibilité d'intégrer un preámbule LaTeX centralisé par projet (fourni par configuration).
Marksman s'intègre facilement dans Zed : https://github.com/artempyanykh/marksman, mais il faut en faire le serveur par défaut,  diriger le rendu HTML vers le composant wry et ajouter Katex dans le pipeline. 
 
## Typst 
tinymist comme serveur LSP. Doit offrir les mêmes fonctionnalités que l'extension tinymist pour VScode : live view, compilation et preview PDF.

## Latex 
TeXlab comme serveur LSP. la synchronisation Synctex/PDFjs doit probablement être ajoutée manuellement.

## Remarque  
Tinymist et TexLab sont écrit en Rust. L'auteur ne serait pas contre une intégration native dans le backend. Sinon en "sidecar" si c'est suffisant. 

## Validation
Si l'architecture est validée, écrire un rapport d'analyse technique et un plan d'action détaillé dans le dossier plans du projet. 
**Fork Zed et rebrandig en AZed**. Les composants seront ajoutés un par un selon leur ordre de priorité. 

## Fonctionnalité à venir 
- Ajouter un moteur d'indexation/recherche (comme une extension Zed avec un backend "sidecar" multi-plateforme et bien établi) ;
- Ajouter le support d'OpenCode en profitant de la couche IA/MCP de Zed.
