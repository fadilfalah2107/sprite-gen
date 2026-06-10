<p align="center">
  <img src="docs/claudecy-idle.gif" width="110" alt="claudecy idle" />
  <img src="docs/claudecy-running.gif" width="110" alt="claudecy running" />
  <img src="docs/claudecy-success.gif" width="110" alt="claudecy success" />
  <img src="docs/claudecy-talking.gif" width="110" alt="claudecy talking" />
  <img src="docs/howl-idle.gif" width="110" alt="howl idle" />
  <img src="docs/howl-running.gif" width="110" alt="howl running" />
  <img src="docs/howl-success.gif" width="110" alt="howl success" />
</p>

<h1 align="center">sprite-gen</h1>

<p align="center"><b>Un dibujo de entrada. Un atlas de sprites listo para juegos de salida.</b></p>

<p align="center">

**English** · [한국어](README.ko.md) · [日本語](README.ja.md) · [简体中文](README.zh-Hans.md) · [Español](README.es.md) · [Français](README.fr.md)

</p>

---

Pides a un modelo de imagen una “hoja de sprites” y sabes lo que obtienes: un personaje cuyo rostro cambia en cada fotograma, un fondo que no se puede recortar por color, poses que se superponen y se salen de la cuadrícula, y un PNG que tu motor de juego no puede consumir de verdad. Demo adorable, activo inútil.

`sprite-gen` es una skill de Codex/Claude que cierra esa brecha. Dale **una imagen base** y una lista de acciones —avanza la generación fila por fila, bloquea la identidad del personaje, elimina el fondo croma a alfa real, extrae cada pose como un fotograma transparente limpio y compila un atlas en runtime **con un `manifest.json.frame_layout` legible por máquina**. Cada sprite anterior se hizo de esta manera.

Y para ese último 10% que la generación nunca acierta bien, existe una **webview de curación**: compara fotogramas lado a lado, rechaza los rotos, ajusta rotación/escala/posición de forma no destructiva, observa el ciclo en vivo y luego compila. La canalización hace el trabajo pesado; tú conservas el criterio.

```text
sprite-request.json → pautas de layout + prompts → filas de estado de image-gen
→ alfa croma → componentes conectados → fotogramas transparentes
→ sprite-sheet-alpha.png + manifest.json.frame_layout
```

## Lo que realmente obtienes

- **Un atlas de sprites transparente** (`sprite-sheet-alpha.png`) — alfa real, sin restos de croma, verificado contra fondos blancos.
- **Un manifiesto de runtime** (`manifest.json.frame_layout`) — rectángulos de fotograma absolutos, fps por estado y banderas de bucle. Tu motor toma muestras de rectángulos; nunca adivina una cuadrícula.
- **Control de calidad visible** — GIFs por estado y hojas de contacto, para juzgar el movimiento antes de lanzar nada.
- **Etiquetas honestas** — acciones cortas y legibles (idle, jump, attack, wave) son el camino estable; la locomoción cíclica (walk/run) se marca como experimental salvo que la QA de movimiento realmente pase. Sin promesas silenciosas excesivas.

## Webview de curación

La generación te da un 90%. La webview es donde un humano lo lleva a *listo para publicar* — independiente, sin dependencia de Studio ni framework, funciona donde esté instalada la skill (Claude Code Desktop, la app de Codex, una terminal simple).

![curation webview — characters](docs/demo-character.gif)

- **Compara fotogramas lado a lado** por estado y **selecciona/rechaza** fotogramas individuales.
- **Transformación no destructiva** por fotograma: arrastrar = mover, rueda = escalar, control superior = rotar, control inferior izquierdo = sesgar. Las ediciones viven en un sidecar `curation.json`: los PNG de origen nunca se reescriben, y el paso de composición “hornea” el resultado de forma determinista. La vista previa y el horneado comparten una misma matriz afín, así que lo que alineas es lo que obtienes.
- **Vista previa en vivo** anima los fotogramas seleccionados al fps del estado.
- No solo para sprites: apúntalo a cualquier carpeta de candidatos de imagen (iconos, logos, borradores generados) con `unpack_atlas_run.py --pngs-dir` y úsalo como vista general de “elegir al ganador”.

### Cuadrícula de suelo isométrica

Para sets isométricos, la webview superpone la cuadrícula del suelo (desde `meta.json` tile/anchor) para que puedas acoplar muebles a los ejes en diamante con el control de sesgo.

![curation webview — isometric furniture](docs/demo-furniture.gif)

<img src="docs/curator-iso.png" width="520" alt="isometric ground grid overlay" />

### Idiomas

La webview se distribuye con inglés y coreano. Usa `--lang en|ko` al iniciar, o el alternador dentro de la app:

```bash
python3 scripts/serve_curation.py --run-dir <run-dir> --lang en   # o ko
```

## Guía rápida

```bash
# 0. instalar dependencias (Pillow) en un virtualenv nuevo
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# 1. preparar una ejecución desde una imagen base
python3 scripts/prepare_sprite_run.py --out-dir <run-dir> --character-id <id> --base-image base.png

# 2. generar una imagen de fila por estado con image-gen, guardar como raw/<state>.png
# 3. extraer fotogramas
python3 scripts/extract_sprite_row_frames.py --run-dir <run-dir>

# 4. (opcional) curar fotogramas en la webview
python3 scripts/serve_curation.py --run-dir <run-dir>

# 5. compilar el atlas de runtime
python3 scripts/compose_sprite_atlas.py --run-dir <run-dir>
```

### Editar una hoja terminada

Cuando solo sobrevive la hoja combinada, reconstruye un directorio de ejecución listo para curar, luego edita y exporta:

```bash
# reconstruir fotogramas: --grid explícito, rectángulos --manifest, o detección automática de alfa (por defecto)
python3 scripts/unpack_atlas_run.py --atlas sheet.png            # auto-detect
python3 scripts/unpack_atlas_run.py --manifest manifest.json     # exact rectangles
python3 scripts/unpack_atlas_run.py --pngs-dir furniture/        # importar un conjunto suelto de PNG

# después de curar, hornea las correcciones a PNGs con nombre
python3 scripts/export_curated_pngs.py --run-dir <run-dir>
```

El resultado por defecto va a una carpeta `<source>-curator` fácilmente localizable junto a la entrada.

El flujo completo orientado a agente y los contratos viven en [`SKILL.md`](SKILL.md).

## Instalación

Desde los flujos de instalación de skills de Codex, instala este repositorio como una skill raíz:

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo aldegad/sprite-gen --path .
```

## Atribución

El flujo de filas de componentes se inspira en la skill `hatch-pet` con licencia Apache-2.0, pero apunta a atlas de sprites de juego genéricos e incluye solo los paquetes de recursos de juego de mascotas ni activos visuales de mascotas.

## Licencia

Apache-2.0