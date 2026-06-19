<p align="center">
  <img src="docs/claudecy-idle.gif" width="110" alt="claudecy en attente" />
  <img src="docs/claudecy-running.gif" width="110" alt="claudecy en course" />
  <img src="docs/claudecy-success.gif" width="110" alt="claudecy succès" />
  <img src="docs/claudecy-talking.gif" width="110" alt="claudecy en train de parler" />
  <img src="docs/howl-idle.gif" width="110" alt="howl en attente" />
  <img src="docs/howl-running.gif" width="110" alt="howl en course" />
  <img src="docs/howl-success.gif" width="110" alt="howl succès" />
</p>

<h1 align="center">sprite-gen</h1>

<p align="center"><b>Un dessin en entrée. Un atlas de sprites prêt pour le jeu en sortie.</b></p>

<p align="center">

**English** · [한국어](README.ko.md) · [日本語](README.ja.md) · [简体中文](README.zh-Hans.md) · [Español](README.es.md) · [Français](README.fr.md)

</p>

---

Demander à un modèle d'image une « sprite sheet » et vous obtenez généralement : un personnage dont le visage change à chaque image, un arrière-plan qui refuse la suppression de chroma, des poses qui se chevauchent et sortent de la grille, et un PNG que votre moteur de jeu ne sait pas vraiment utiliser. Démo mignonne, asset inutilisable.

`sprite-gen` est une skill Codex/Claude qui comble ce fossé. Donnez-lui **une image de base** et une liste d’actions — il génère ligne par ligne, verrouille l’identité du personnage, supprime le fond en chroma pour obtenir un vrai alpha, extrait chaque pose sous forme de frame transparente propre, et génère un atlas runtime avec un `manifest.json.frame_layout` lisible par machine. Chaque sprite ci-dessus a été réalisé ainsi.

Et pour les 10 % que la génération ne fait jamais correctement, il existe une **vue de curation** : comparez les frames côte à côte, rejetez celles qui sont cassées, ajustez rotation/échelle/position de façon non destructive, observez la boucle en direct — puis générez. Le pipeline effectue le travail; vous gardez la décision finale.

```text
sprite-request.json → layout guides + prompts → image-gen state rows
→ chroma alpha → connected components → transparent frames
→ sprite-sheet-alpha.png + manifest.json.frame_layout
```

<p align="center">
  <img src="docs/architecture-diagram.png" width="640" alt="architecture sprite-gen — pipeline par ligne de composant" />
</p>

> Architecture complète : [`docs/architecture.md`](docs/architecture.md) · source du diagramme : [`docs/architecture-diagram.html`](docs/architecture-diagram.html)

## Ce que vous obtenez en réalité

- **Un atlas de sprites transparent** (`sprite-sheet-alpha.png`) — vrai alpha, sans bordure chroma résiduelle, vérifié sur fonds blancs.
- **Un manifeste runtime** (`manifest.json.frame_layout`) — rectangles de frame absolus, fps par état et indicateurs de boucle. Votre moteur lit les rectangles; il ne devine jamais une grille.
- **Un QA visuel** — GIF et planches de contact par état, pour juger le mouvement avant de livrer quoi que ce soit.
- **Des labels honnêtes** — les actions courtes et lisibles (idle, jump, attack, wave) restent le chemin stable ; la locomotion cyclique (walk/run) est marquée expérimentale sauf si le QA motion la valide réellement. Pas de surpromesse silencieuse.

## Vue de curation

La génération vous donne 90 %. La curation est l’endroit où un humain transforme ça en état *livrable* — autonome, sans dépendance Studio ou framework, fonctionne partout où la skill est installée (Claude Code Desktop, l’application Codex, un terminal simple).

![curation webview — personnages](docs/demo-character.gif)

- **Deux rangées par état :** la **séquence de lecture** en haut et une **banque de candidats** en bas (par ex. une deuxième ou troisième prise générée). Faites glisser l’icône ⠿ d’une frame pour réordonner la séquence, ou récupérez une image depuis la banque — reconstruisez une boucle propre en sélectionnant les meilleures frames entre les prises. L’arrangement est sauvegardé, donc il est restauré au prochain chargement.
- **Transformation non destructive** par frame : glisser = déplacer, molette = mettre à l’échelle, poignée du haut = rotation, bas-gauche = cisaillement, plus un basculement `horizontal-flip` pour obtenir une sortie inversée gauche-droite. Les modifications vivent dans un sidecar `curation.json` — les PNG source ne sont jamais réécrits, et l’étape de composition applique le résultat de façon déterministe. L’aperçu et la génération partagent une même matrice affine, donc ce que vous alignez est ce que vous obtenez.
- **Prévisualisation live** qui anime la séquence à la fps de l’état, avec lecture/pause, pas à pas image par image, et un contrôle de vitesse de 0,25× à 4×.
- Pas seulement pour les sprites : pointez-le sur n’importe quel dossier de candidats image (icônes, logos, brouillons générés) avec `unpack_atlas_run.py --pngs-dir` et utilisez-le comme vue de sélection du meilleur résultat.

### Grille de sol isométrique

Pour les sets isométriques, la vue de curation superpose la grille de sol (depuis `meta.json` tile/anchor) afin que vous puissiez aligner les meubles sur les axes en losange avec la poignée de cisaillement.

![curation webview — mobilier isométrique](docs/demo-furniture.gif)

<img src="docs/curator-iso.png" width="520" alt="superposition de la grille de sol isométrique" />

### Langues

La vue de curation est disponible en anglais et en coréen. Passez `--lang en|ko` au lancement, ou utilisez le basculeur intégré :

```bash
python3 scripts/serve_curation.py --run-dir <run-dir> --lang en   # or ko
```

## Support Python

`sprite-gen` supporte CPython 3.10+. La CI exécute la version minimale prise en charge (3.10) et la plus récente couverte (3.14) sur des runners GitHub-hosted.

Le démarrage rapide nécessite une installation Python avec `venv`/`ensurepip` fonctionnels. Si `python3 -m venv` échoue avant l’installation des paquets dans une distribution locale, utilisez une build CPython standard d’une version supportée et relancez les mêmes commandes.

## Démarrage rapide

```bash
# 0. install dependencies (Pillow) into a fresh virtualenv
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# 1. prepare a run from a base image
python3 scripts/prepare_sprite_run.py --out-dir <run-dir> --character-id <id> --base-image base.png

# 2. generate one row image per state with image-gen, save as raw/<state>.png
# 3. extract frames
python3 scripts/extract_sprite_row_frames.py --run-dir <run-dir>

# 4. (optional) curate frames in the webview
python3 scripts/serve_curation.py --run-dir <run-dir>

# 5. bake the runtime atlas
python3 scripts/compose_sprite_atlas.py --run-dir <run-dir>
```

### Modifier une feuille terminée

Quand seule la feuille combinée reste, reconstruisez un run dir prêt pour la curation, puis sélectionnez et exportez :

```bash
# rebuild frames: explicit --grid, --manifest rectangles, or alpha auto-detect (default)
python3 scripts/unpack_atlas_run.py --atlas sheet.png            # auto-detect
python3 scripts/unpack_atlas_run.py --manifest manifest.json     # exact rectangles
python3 scripts/unpack_atlas_run.py --pngs-dir furniture/        # import a loose PNG set

# after curating, bake corrections back to named PNGs
python3 scripts/export_curated_pngs.py --run-dir <run-dir>
```

La sortie par défaut est un dossier `<source>-curator` repérable à côté de l’entrée.

Le workflow complet orienté agent et les contrats sont décrits dans [`SKILL.md`](SKILL.md).

## Installation

Depuis les flux d’installation de skill Codex, installez ce dépôt comme skill racine :

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo aldegad/sprite-gen --path .
```

## Attribution

Le workflow en lignes de composant s’inspire de la skill `hatch-pet` sous licence Apache-2.0, mais cible des atlas de sprites de jeu génériques et n’inclut aucun package de pet ni assets visuels de pet.

## Licence

Apache-2.0