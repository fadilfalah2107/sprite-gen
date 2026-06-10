<p align="center">
  <img src="docs/claudecy-idle.gif" width="110" alt="claudecy au repos" />
  <img src="docs/claudecy-running.gif" width="110" alt="claudecy en mouvement" />
  <img src="docs/claudecy-success.gif" width="110" alt="claudecy réussi" />
  <img src="docs/claudecy-talking.gif" width="110" alt="claudecy qui parle" />
  <img src="docs/howl-idle.gif" width="110" alt="howl au repos" />
  <img src="docs/howl-running.gif" width="110" alt="howl en mouvement" />
  <img src="docs/howl-success.gif" width="110" alt="howl réussi" />
</p>

<h1 align="center">sprite-gen</h1>

<p align="center"><b>Un dessin en entrée. Un atlas de sprites prêt pour le jeu en sortie.</b></p>

<p align="center">

**Anglais** · [한국어](README.ko.md) · [日本語](README.ja.md) · [简体中文](README.zh-Hans.md) · [Español](README.es.md) · [Français](README.fr.md)

</p>

---

Demandez à un modèle d’image une "sprite sheet" et vous savez ce que vous obtenez : un personnage dont le visage change à chaque image, un arrière-plan qui ne se supprime pas proprement, des poses qui se chevauchent et sortent de la grille, et un PNG que votre moteur de jeu ne peut pas réellement consommer. Démo mignonne, ressource inutilisable.

`sprite-gen` est une skill Codex/Claude qui comble ce fossé. Fournissez-lui **une image de base** et une liste d’actions — il pilote la génération ligne par ligne, verrouille l’identité du personnage, élimine le fond chroma en alpha réel, extrait chaque pose en image transparente propre, et génère un atlas runtime **avec un `manifest.json.frame_layout` lisible par machine**. Chaque sprite ci-dessus a été créé ainsi.

Et pour le dernier 10 % que la génération n’obtient jamais parfaitement, il y a une **webview de curation** : comparez les images côte à côte, rejetez celles qui sont cassées, ajustez rotation/échelle/position de manière non destructive, regardez la boucle en direct, puis générez le résultat final. La pipeline fait le travail lourd ; vous gardez la finition.

```text
sprite-request.json → guides de mise en page + prompts → lignes d’état image-gen
→ alpha chroma → composantes connexes → cadres transparents
→ sprite-sheet-alpha.png + manifest.json.frame_layout
```

## Ce que vous obtenez réellement

- **Un atlas de sprites transparent** (`sprite-sheet-alpha.png`) — vrai alpha, sans frange chroma résiduelle, vérifié sur fonds blancs.
- **Un manifeste runtime** (`manifest.json.frame_layout`) — rectangles de cadres absolus, fps par état et drapeaux de boucle. Votre moteur lit les rectangles, jamais une grille en devinant.
- **Une QA visuelle** — GIFs par état et planches contact, pour juger le mouvement avant tout déploiement.
- **Des libellés honnêtes** — des actions courtes et lisibles (idle, jump, attack, wave) forment le chemin stable ; la locomotion cyclique (walk/run) est marquée expérimentale tant que la QA de mouvement ne passe pas réellement. Pas de promesse implicite.

## Curation webview

La génération vous donne 90 %. La webview est l’étape où un humain amène le résultat au niveau **livrable** — autonome, sans dépendance Studio ou framework, fonctionne partout où la skill est installée (Claude Code Desktop, l’application Codex, un terminal simple).

![curation webview — personnages](docs/demo-character.gif)

- **Comparez les cadres côte à côte** par état, et **sélectionnez / rejetez** des cadres individuels.
- **Transformation non destructive** par cadre : glisser = déplacer, molette = échelle, poignée supérieure = rotation, poignée bas-gauche = cisaillement. Les modifications sont enregistrées dans un fichier `curation.json` annexe — les PNG sources ne sont jamais réécrits, et l’étape de composition applique le rendu de manière déterministe. L’aperçu et la cuisson partagent la même matrice affine, donc ce que vous alignez est ce que vous obtenez.
- **Aperçu en direct** avec animation des cadres sélectionnés au fps de l’état.
- Pas réservé aux sprites : pointez-le vers n’importe quel dossier de candidats d’images (icônes, logos, brouillons générés) avec `unpack_atlas_run.py --pngs-dir` et utilisez-le comme vue de sélection du meilleur visuel.

### Grille de sol isométrique

Pour les sets isométriques, la webview superpose la grille du sol (depuis `meta.json` tile/anchor) afin de snapper le mobilier sur les axes en losange avec la poignée de cisaillement.

![curation webview — mobilier isométrique](docs/demo-furniture.gif)

<img src="docs/curator-iso.png" width="520" alt="superposition de grille de sol isométrique" />

### Langues

La webview est livrée en anglais et coréen. Passez `--lang en|ko` au lancement, ou utilisez le basculement intégré :

```bash
python3 scripts/serve_curation.py --run-dir <run-dir> --lang en   # ou ko
```

## Démarrage rapide

```bash
# 0. installer les dépendances (Pillow) dans un nouveau virtualenv
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# 1. préparer un run à partir d’une image de base
python3 scripts/prepare_sprite_run.py --out-dir <run-dir> --character-id <id> --base-image base.png

# 2. générer une image par état ligne par ligne avec image-gen, enregistrer sous raw/<state>.png
# 3. extraire les cadres
python3 scripts/extract_sprite_row_frames.py --run-dir <run-dir>

# 4. (optionnel) faire la curation des cadres dans la webview
python3 scripts/serve_curation.py --run-dir <run-dir>

# 5. générer l’atlas runtime
python3 scripts/compose_sprite_atlas.py --run-dir <run-dir>
```

### Éditer une feuille finalisée

Quand seule la feuille combinée subsiste, reconstruisez un dossier de run prêt pour le curator, puis faites la curation et exportez :

```bash
# reconstruire les cadres : --grid explicite, rectangles --manifest, ou détection alpha automatique (par défaut)
python3 scripts/unpack_atlas_run.py --atlas sheet.png            # détection automatique
python3 scripts/unpack_atlas_run.py --manifest manifest.json     # rectangles exacts
python3 scripts/unpack_atlas_run.py --pngs-dir furniture/        # importer un lot PNG indépendant

# après curation, réintégrer les corrections dans les PNG nommés
python3 scripts/export_curated_pngs.py --run-dir <run-dir>
```

La sortie est par défaut un dossier `<source>-curator` détectable à côté de l’entrée.

Le flux complet orienté agent et les contrats associés se trouvent dans [`SKILL.md`](SKILL.md).

## Installation

Depuis les workflows d’installation de skill Codex, installez ce dépôt en tant que skill racine :

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo aldegad/sprite-gen --path .
```

## Attribution

Le workflow par lignes de composants s’inspire de la skill `hatch-pet` sous licence Apache-2.0, mais cible des atlas de sprites de jeu génériques et n’inclut aucun paquet ni actif visuel de type pet.

## Licence

Apache-2.0