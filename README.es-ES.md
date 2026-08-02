

# darwinism — un mundo para agentes
<img src="highlights/demo.gif" alt="Demo" width="800">

<p><em>(puntos blancos = ovejas, puntos rojos = zorros, verde = vegetación, las entidades con un punto negro en la parte superior derecha son machos; las que no lo tienen son hembras)</em></p>

Un **mundo vivo para agentes**, `darwinism` hace crecer un planeta completo —
**terrenos y biomas** (océano, ríos, bosque, pradera, montañas), **hidrología** y
**clima y estaciones** que ciclan a medida que pasan los días virtuales — luego permite que los agentes
cazuen, huyan, beban, se reproduzcan y evolucionen.

Diseñado para IA:

- **Los agentes ven a través de sus propios ojos.** Cada animal percibe solo su entorno **local**
  circundante — comida, amenazas, parejas y agua dentro de su propio rango de visión heredable, centrado
  en sí mismo — como **cuadrículas de percepción egocéntricas**: números puros, listos para alimentar directamente a una
  red neuronal. Sin un "objetivo más cercano" global, sin omnisciencia.
- **Los cerebros son intercambiables.** Cada decisión fluye a través de un
  `brain.decide(observation) -> action` contrato. El mundo predeterminado funciona con reglas codificadas,
  pero un **cerebro de PyTorch se integra bajo el mismo contrato sin reescribir la simulación**
- **La evolución es real, no programada.** Cada animal lleva un **genoma heredable** — velocidad,
  rango de visión, metabolismo, tamaño, vida útil, agresión y más — **muestreados de distribuciones de probabilidad**
  en lugar de estar codificados. Cuando dos parejas se reproducen, los genes de cada cría provienen del
  **cruce gen a gen de ambos padres más mutación gaussiana**, por lo que los rasgos derivan y
  las poblaciones se adaptan a lo largo de las generaciones.
- **El tiempo es solo otro parámetro.** Configura `dt` para que un segundo real pueda representar un segundo, un día o incluso años de tiempo simulado. Observa cada momento de forma interactiva, o avanza a toda velocidad a través de generaciones de evolución en simulaciones de alta velocidad sin interfaz gráfica.
- **Sin interfaz gráfica y determinista.** Ejecútalo como un simulador de números puros sin pantalla, o
  obsérvalo en vivo en una ventana de Arcade — el núcleo es el mismo en cualquier caso. Dos semillas (mundo + ejecución) hacen que
  **cada ejecución sea reproducible byte por byte**.

**Amigable con la IA por diseño.** Los bloques de construcción son del tipo que el aprendizaje automático entiende nativamente:
la percepción llega como **tensores** (canales egocéntricos listos para CNN), las decisiones salen como
**rumbos continuos y puertas de probabilidad** en lugar de bifurcaciones rígidas, y los genes están **muestreados
de distribuciones de probabilidad** y recombinados con cruce + mutación. En cualquier lugar donde hoy
exista una regla codificada, un modelo aprendido y diferenciable puede insertarse sin reescribir nada.

Y es un **marco de trabajo, no solo una aplicación**: `import darwinism`, compone un `Config`, y extiende
alrededor de cuatro puntos — **especies**, **cerebros**, **sistemas** y **rasgos** heredables —
sin tocar el núcleo.

## Inicio rápido

```python
import darwinism as dw

cfg = dw.make_config(world_seed=12345, seed=7)   # world seed + run seed (both reproducible)
sim = dw.Simulation(cfg)                          # default RuleBrain drives every species
for _ in range(5000):
    stats = sim.step()
print(sim.populations)                            # {'sheep': ..., 'fox': ...}
```

**Extenderlo** — añade una especie, un sistema de tick, un rasgo o un cerebro, todo como composición. Consulta
**[EXTENDING.md](EXTENDING.md)** y los ejemplos ejecutables en **[`examples/`](examples/)**:

```python
RABBIT = 2
cfg.species[RABBIT] = dw.SpeciesConfig(
    name="rabbit", species_id=RABBIT, init_count=90,
    diet=[dw.FieldFood("vegetation", eat_value=0.7)],       # herbivore
    gene_ranges={"max_speed": dw.GeneRange(0.8, 2.2), "burrow_depth": dw.GeneRange(0, 1), ...},
)
sim = dw.Simulation(cfg, systems=[*dw.default_pipeline(cfg), MyDiseaseSystem()],
                    brain={RABBIT: MyBrain()})
```

## Instalación

`darwinism` se instala desde GitHub. El núcleo (simulador sin interfaz gráfica) solo necesita `numpy` +
`opensimplex`; las piezas pesadas son extras opcionales que reflejan los límites de importación perezosa del código.

```bash
pip install "git+https://github.com/afreediz/darwinism.git"           # core, headless
pip install "darwinism[render] @ git+https://github.com/afreediz/darwinism.git"   # + Arcade viewer
pip install "darwinism[torch]  @ git+https://github.com/afreediz/darwinism.git"   # + learned PolicyBrain
```

Extras: `analysis` (pandas + matplotlib para el informe CSV), `render` (visor de Arcade),
`torch` (políticas aprendidas), `dev` (pytest, ruff, import-linter), `all`. Para desarrollo local:

```bash
python -m venv venv
venv/Scripts/activate          # Windows;  source venv/bin/activate on Unix
pip install -e ".[all,dev]"
```

La instalación agrega dos scripts de consola, `darwinism-run` (sin interfaz) y `darwinism-live` (visor);
`python -m darwinism [run|live]` y los adaptadores raíz `run_experiment.py` / `run_live.py` son
equivalentes.

## Ejecución

**Observar en vivo** (ventana observador de Arcade):

```bash
# default world, random run
darwinism-live

# with world configs
darwinism-live --world-seed 12345 --seed 7 --scale 5 --spf 4

# with trained pytorch brains
darwinism-live --world-seed 1 --seed 7 \
 --sheep-brain notebooks/imitation_learning/sheep.pt \
 --fox-brain notebooks/live_learning/offline/fox_offline_ppo.pt \
 --device cuda
```

**Cerebros entrenados.** Los puntos de control de cerebros listos para usar están disponibles para descarga — consulta
[scratchpad/MODELS.md](scratchpad/MODELS.md). Colócalos detrás de `--sheep-brain` /
`--fox-brain` (necesita el extra `[torch]`).

Banderas CLI: `--world-seed N` (terreno/ríos; misma semilla de mundo ⇒ mapa idéntico) · `--seed N`
(dinámica de ejecución; omítela para una ejecución aleatoria en ese mundo) · `--scale N` (píxeles por celda del mundo) ·
`--spf N` (pasos de simulación por frame renderizado; fracciones válidas, ej. `0.25` = 1 paso cada 4 frames) ·
`--log-csv PATH` (también registrar esta ejecución en vivo a un CSV) · `--monitor` (abrir una ventana de análisis en vivo separada —
consulta [Monitor en vivo](#live-monitor); establece `--log-csv` por defecto en `runs/live.csv`).

### Controles del visor en vivo

| Tecla / entrada | Acción |
|---|---|
| `SPACE` | pausar / reanudar |
| `↑` / `↓` | acelerar / desacelerar simulación (×2 / ÷2 pasos por frame) |
| `+` / `-` / `=` | acercar / alejar (centrado en la pantalla) |
| rueda del ratón | acercar / alejar (centrado en el cursor) |
| arrastrar con botón central | desplazar el mapa |
| `0` | restablecer vista (ajustar todo el mapa) |
| `V` | alternar superposición de vegetación |
| `Ctrl+V` | congelar / descongelar regeneración de vegetación (el pastoreo sigue agotándola) |
| `S` | avanzar estación (+0.1 año) |
| `Ctrl+S` | pausar / reanudar progresión estacional (día y clima siguen corriendo) |
| `Shift+S` | generar una oveja en el cursor |
| `Shift+F` | generar un zorro en el cursor |
| clic izquierdo | inspeccionar la percepción de un animal — rodearlo y mostrar sus canales de cuadrícula egocéntrica |
| `ESC` | salir |

El visor es un **observador únicamente** — estos controles nunca retroalimentan la simulación medida,
excepto la generación manual (`Shift+S` / `Shift+F`), que utiliza el generador de números aleatorios (RNG) de la ejecución
y por lo tanto rompe la reproducibilidad de la ejecución (la ruta sin interfaz nunca genera manualmente).

Marcadores en pantalla: un pequeño punto negro = macho · un tinte rosa = se reprodujo en los últimos ticks ·
atenido = durmiendo · toda la escena se oscurece de noche. Haz clic izquierdo en cualquier animal para abrir
un panel en la parte superior derecha que muestra sus cuadrículas de percepción en vivo (terreno / agua / comida / amenaza / pareja).

**Experimento sin interfaz** (avanzar rápido, escribe CSV):

```bash
darwinism-run --ticks 20000 --world-seed 12345 --seed 7 --out runs/run.csv
darwinism-run --ticks 20000 --world-seed 12345    # random run on a fixed world
darwinism-run --ticks 20000 --plot                # also render a PNG report
darwinism-run --ticks 20000 --monitor             # watch the plots update live
```

`--world-seed` fija el mapa; `--seed` fija la ejecución (omítela para una ejecución aleatoria — la semilla
resuelta se imprime al inicio para que puedas repetirla). `--log-every N` establece con qué frecuencia se escribe una fila CSV
(por defecto cada 10 ticks). `--monitor` abre una ventana de análisis en vivo separada (más abajo).

**Análisis** (curvas de población, deriva de rasgos, diagrama de fase):

```bash
python -m darwinism.analysis.plots runs/run.csv --out analysis/out
```

Cada ejecución produce un informe de 4 paneles — población vs tiempo, biomasa de vegetación, deriva de rasgos
de ovejas (la señal de evolución) y el diagrama de fase ovejas-zorros (el ciclo Lotka–Volterra):

![Sample analysis report](highlights/analysisOut.png)

### Monitor en vivo

`analysis.monitor` abre una **ventana separada** (independiente de la sim) que sigue un CSV de ejecución
y redibuja el mismo informe de 4 paneles a intervalos — para que puedas ver evolucionar las curvas de población,
la deriva de rasgos y el diagrama de fase mientras una ejecución aún está en curso.

```bash
# automatic: launch the monitor alongside a run (spawned as its own process)
darwinism-run --ticks 20000 --world-seed 12345 --seed 7 --monitor
darwinism-live --world-seed 12345 --seed 7 --monitor

# standalone: point it at ANY run CSV — one still being written, or a finished one
python -m darwinism.analysis.monitor runs/run.csv --interval 1.0
```

`--interval` establece el período de actualización en segundos (por defecto `1.0`). El monitor tolera un
archivo vacío o solo con encabezados, por lo que puede iniciarse antes de que se escriba la primera fila. Necesita un entorno gráfico
(backend `TkAgg` de matplotlib), por lo que no puede ejecutarse en un shell sin interfaz. Con `--monitor`, la
ventana de simulación se cierra de forma independiente y la ventana del monitor permanece abierta mostrando los datos finales.

## Arquitectura (los aspectos inegociables)

- **`sim/` son números puros y nunca importa `render/`.** Ambos puntos de entrada comparten
  el mismo núcleo `sim/`. El renderizador de Arcade es un observador opcional de solo lectura.
- **El contrato cerebro↔mundo es la columna vertebral.** Cada decisión fluye a través de
  `Brain.decide(obs_by_species, idx) -> act`: cada especie recibe cuadrículas de percepción egocéntricas
  `(N, C, K, K)` + un vector escalar, el cerebro devuelve la matriz de acción `(len(idx), ACT_DIM)`
  alineada al orden global de entidades vivas. El `RuleBrain` de la v1 es desechable; los esquemas de cuadrícula/
  escalar por especie (`sim/perception.py`, `sim/brain.py`) son el diseño real.
- **El estado de la entidad es Structure-of-Arrays** (`sim/entities.py`) — arrays de NumPy paralelos, no
  un objeto por entidad.
- **La percepción es solo local** — cada agente percibe comida/amenazas/parejas/agua solo dentro
  de su `sensory_range` heredable, como cuadrículas egocéntricas enmascaradas; nunca un "más cercano" global. El tiempo ciego se convierte en exploración.
- **Determinismo, dos semillas**: la **semilla del mundo** impulsa la generación del mundo (terreno + ríos);
  la **semilla de ejecución** impulsa toda la dinámica a través de un `numpy` Generator (desde `config.py`) enhebrado
  en cada sistema. `dt` fijo, iteración por índice de ranura. Misma semilla de mundo + config + semilla de ejecución
  ⇒ ejecución idéntica; una semilla de ejecución diferente ⇒ una ejecución diferente en el mismo mundo.

## Estructura

```
pyproject.toml       packaging (hatchling; core + [render]/[torch]/[analysis]/[dev] extras)
darwinism/           the package (flat layout)
  __init__.py        public API (Simulation, Config, Brain, System, ...) + __version__
  config.py          all tunables + world seed + run seed + declarative SpeciesConfig/diet
  sim/               headless core (world, hydrology, environment, entities, genome,
                     perception, brain, grid, systems/ incl. the pipeline registry)
  render/viewer.py   Arcade observer (never mutates the sim)
  analysis/          CSV logger + matplotlib plots + live monitor window
  cli/               console-script entry points (experiment, live)
examples/            runnable extension examples (species, system, brain)
tests/               golden-master determinism suite + extension tests
run_experiment.py / run_live.py   thin back-compat shims -> darwinism.cli
```

---

## Licencia

**Licencia de Investigación No Comercial darwinism v1.0** — ver [LICENSE](LICENSE).

Libre para usar, modificar y compartir con fines **personales, académicos, de investigación y educativos**
con atribución ("darwinism by Afreedi Z. — github.com/afreediz"). Las derivaciones públicas del núcleo de simulación deben
compartirse nuevamente bajo la misma licencia.

**El uso comercial requiere un acuerdo por escrito separado** — contactar a
afreedisulfiker@gmail.com.

---

⭐ Si te gusta este proyecto, dale una estrella — ¡ayuda mucho y es muy apreciado!

### Hecho con ❤️ por Afreedi Z.
