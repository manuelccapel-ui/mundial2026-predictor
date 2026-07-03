# Mundial 2026 — Predictor

Proyecto de predicción de partidos del Mundial 2026 (48 selecciones). Combina
datos de Kaggle, Transfermarkt y (opcionalmente) FBref con un modelo híbrido
**Poisson + XGBoost** para predecir goles esperados y probabilidades de
resultado (1X2 y marcador) de cualquier cruce del torneo, incluidos los que
todavía no se han disputado.

## Estructura del repositorio

```
mundial2026_datos.ipynb    # Pipeline de datos (hace peticiones de red)
mundial2026_modelo.ipynb   # Features, modelo y predicciones (100% offline)
data/
  world_cup_matches.csv      # Partidos del Mundial: marcador, xG, stats de equipo
  pre_world_cup_form.csv     # Últimos 5 partidos pre-Mundial de las 48 selecciones
  teams.csv                  # Elo, ranking FIFA y grupo de cada selección (snapshot de Kaggle)
  bracket_structure.json     # Estructura del cuadro de eliminatorias (grupos, dieciseisavos, progresión)
  features.csv                # Dataset de entrenamiento (generado por el notebook de modelo)
cache/                      # HTML cacheado de los scrapers (no se sube al repo)
requirements.txt
```

### `mundial2026_datos.ipynb` — pipeline de datos

1. Descarga la última versión de [`mominullptr/fifa-world-cup-2026-dataset`](https://www.kaggle.com/datasets/mominullptr/fifa-world-cup-2026-dataset)
   (Kaggle, CC0, se actualiza a diario) e integra marcador, xG, posesión,
   tiros, córners, faltas, fueras de juego, paradas y tarjetas en
   `world_cup_matches.csv`.
2. Incluye una celda de diagnóstico que mide empíricamente cuánto tarda Kaggle
   en reflejar un partido recién jugado (comparando la fecha del partido con
   `last_updated` de sus estadísticas).
3. **Complemento opcional de FBref** (`complement_with_fbref()`): rellena
   columnas que Kaggle no trae (centros, intercepciones, entradas ganadas,
   penaltis de tandas, asistencia, árbitro) scrapeando FBref con un navegador
   Edge real (librería `nodriver`, necesaria porque FBref bloquea clientes
   HTTP normales con un desafío de Cloudflare). **No se ejecuta con "Run
   All"** — abre una ventana de navegador real y hay que lanzarla a mano,
   descomentando la celda marcada con ⚠️ en el notebook.
4. Scrapea Transfermarkt (peticiones HTTP normales, sin bypass) para obtener
   los últimos 5 partidos pre-Mundial de las 48 selecciones.
5. Construye `bracket_structure.json` (estructura oficial del cuadro de
   eliminatorias) y expone `get_bracket_state()`, que resuelve cada casilla
   del cuadro cruzando la estructura con los resultados reales — sin inventar
   nunca un cruce que los datos no confirmen.
6. Exporta también `data/teams.csv`, para que el notebook de modelo no
   necesite volver a llamar a `kagglehub`.

### `mundial2026_modelo.ipynb` — features, modelo y predicciones

No hace **ninguna** petición de red: solo lee los ficheros que deja listos el
notebook de datos.

1. Construye `data/features.csv` (Elo, ranking FIFA, forma pre-Mundial
   ponderada, fase del torneo, días de descanso) para cada partido ya jugado.
2. Entrena un modelo **híbrido**: Poisson lineal para los goles del **local**
   y XGBoost para los goles del **visitante** (ver "Estado actual" — es una
   decisión basada en los datos disponibles ahora mismo, no una preferencia
   de diseño).
3. Expone `predict_match(home_team, away_team, phase)`, que da las lambdas de
   ambos modelos y una matriz de probabilidad de marcador vía Poisson.
4. Sección visual con `display_prediction()`: heatmap de marcadores, tabla de
   los 5 más probables, barras 1X2 y predicción destacada.

## Requisitos e instalación

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
pip install -r requirements.txt
```

El scraper complementario de FBref necesita además Microsoft Edge o Google
Chrome instalados en el sistema (`nodriver` los controla directamente).

### Credenciales de Kaggle

El dataset usado es público, así que `kagglehub` puede descargarlo sin
autenticarse. Si Kaggle te pide login en tu cuenta (o si usas un dataset
privado en el futuro), coloca tu token en la ruta estándar que espera la
librería:

- **Windows:** `%USERPROFILE%\.kaggle\kaggle.json`
- **Linux/Mac:** `~/.kaggle/kaggle.json`

Descarga ese fichero desde *Kaggle → Account → Create New API Token*.
**Nunca lo subas al repositorio** — está excluido en `.gitignore`.

## Cómo ejecutar

1. Abre y ejecuta **`mundial2026_datos.ipynb`** de arriba a abajo (Run All).
   Deja `data/` actualizado con los últimos resultados.
   - Opcional: ejecuta manualmente `complement_with_fbref()` (sección 6) si
     quieres completar centros/intercepciones/entradas/árbitro/asistencia.
2. Abre y ejecuta **`mundial2026_modelo.ipynb`** de arriba a abajo. Entrena
   los modelos y genera las predicciones visuales.

## Ejemplo de uso

```python
result = predict_match("Portugal", "Croatia", "Round of 32")
```

```
Goles esperados (λ): Portugal 1.64 - 1.31 Croatia

P(1) Portugal gana : 45.1%
P(X) empate         : 24.2%
P(2) Croatia gana   : 30.7%

Marcadores más probables:
  1-1  ->  11.3%
  2-1  ->   9.2%
  1-0  ->   8.6%
  1-2  ->   7.4%
  2-0  ->   7.0%
```

`display_prediction("Portugal", "Croatia", "Round of 32")` da la misma
información en forma de heatmap + tabla + barras 1X2 + texto destacado.

## Estado actual del proyecto (3-jul-2026)

**Completado:**
- Scraping e integración de datos (Kaggle + Transfermarkt + FBref complementario)
- Estructura del cuadro de eliminatorias corregida y validada contra los 6
  octavos confirmados por FIFA (3-jul-2026): Canada-Morocco, Paraguay-France,
  Brazil-Norway, Mexico-England, USA-Belgium, Portugal-Spain
- Feature engineering (Elo, ranking FIFA, forma ponderada por importancia de
  competición, fase, descanso)
- Modelo híbrido Poisson (local) + XGBoost (visitante), validado con split
  temporal train/test (83 partidos jugados a 3-jul-2026)
- `predict_match()` y visualización (`display_prediction()`)
- `predict_round()` — tabla de predicciones de una ronda completa
- **Simulación Monte Carlo** (`simulate_tournament()`, sección 8 del notebook
  de modelo): 10 000 repeticiones del cuadro, probabilidades de avance por
  selección hasta campeón. Resultados guardados en `data/monte_carlo_probs.csv`
- Caché con TTL (schedule FBref: 6 h; match reports recientes: 4 h)
- `complement_with_fbref()` como fuente primaria de marcadores recientes
  (Kaggle puede tardar ~1 día; FBref los tiene antes)
- `clear_recent_cache(days=N)` para refrescar solo la caché de partidos recientes

**Pendiente / manual:**
- Ejecutar `complement_with_fbref()` (requiere Edge/Chrome real — no se lanza
  en *Run All*) cuando haya resultados nuevos en FBref que Kaggle todavía no
  tenga. Actualmente Portugal-Croatia y los partidos del 3-jul todavía
  aparecen como `scheduled`.
- Reevaluar XGBoost vs. Poisson para el modelo del local cuando haya más
  rondas jugadas: con los datos actuales (66 partidos de grupos en train) la
  ventaja sigue siendo de Poisson (MAE 0.802 vs. 0.894 de XGBoost). Ver
  sección 4 de `mundial2026_modelo.ipynb`.
