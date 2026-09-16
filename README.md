# Documentación técnica — Prueba técnica Científico de Datos (Austro)

Las explicaciones y demás hallazgos están dentro de la carpeta **notebooks** [prueba_tecnica_CienciaDatos](notebooks/prueba_tecnica_CienciaDatos.ipynb)

## 1. Estructura del repositorio

| Carpeta / archivo   | Contenido                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `data/raw/`       | `evaluacion.json` (fuente original) y su extracción 1:1 a parquet: `cliente.parquet`, `demografico.parquet`, `histRecuperacion.parquet`, `gestion.parquet`.                                                                                                                                                                                                                                           |
| `data/processed/` | `base_consolidada.parquet` (17,813 clientes, salida del Ejercicio 2); `quarantine.parquet` (147 casos irregulares excluidos del entrenamiento del Ejercicio 3, con su `razon_exclusion`); `scoring_simulado_ejemplo.csv` y `scoring_simulado_cuarentena.csv` (salidas de ejemplo de `src/aplicar_modelo.py`).                                                                                        |
| `notebooks/`      | `prueba_tecnica_CienciaDatos.ipynb`: desarrollo, visualizaciones, métricas y las 4 respuestas de la prueba.                                                                                                                                                                                                                                                                                                   |
| `src/`            | Scripts de producción:`consolidar_datos.py` (Ejercicio 2), `entrenar_modelo.py` (Ejercicio 3, con cuarentena), `aplicar_modelo.py` (scoring sobre cartera nueva).                                                                                                                                                                                                                                         |
| `utils/`          | Funciones/clases reutilizables:`ids.py` (normalización de identificadores), `recuperacion.py` (pivoteo de histórico de pagos), `edad.py` (cálculo de edad/fecha de corte), `pipeline_ml.py` (`CapFloorTransformer`), `calidad_datos.py` (reglas de cuarentena), `metricas.py` (KS, lift/gain, resumen de métricas), `segmentacion.py` (segmentación estratégica, tamaño de muestra, PSI). |
| `models/`         | Artefactos del modelo campeón persistidos por`entrenar_modelo.py`: `modelo_priorizacion_recuperacion.joblib` (pipeline completo) y `modelo_priorizacion_metadata.json` (metadatos de trazabilidad: hiperparámetros, métricas, reglas de cuarentena, fecha de entrenamiento).                                                                                                                            |
| `docs/`           | `plan_estrategia.md` (plan detallado previo, con la justificación extendida del Ejercicio 1 y decisiones de diseño del Ejercicio 3); este documento (`documentacion_tecnica.md`); PDF original de la prueba.                                                                                                                                                                                               |
| `scripts/`        | `extraer_bases_json.py`: extracción inicial de `evaluacion.json` a parquet en `data/raw/`                                                                                                                                                                                                                                                                                                                 |

## 2. Cómo ejecutar los scripts

Todos los scripts se ejecutan con un intérprete de Python que tenga
instaladas las dependencias de `requirements.txt` (`pip install -r requirements.txt`), desde la raíz del repositorio, en este orden. Los
comandos usan `python` de forma genérica: sustitúyelo por la ruta a tu
propio intérprete/entorno virtual o de conda si `python` no apunta al
correcto (ej. `ruta\a\tu\entorno\python.exe` en Windows, o la salida de
`which python` / `where python` en tu sistema).

1. **Extracción inicial** — separa el JSON crudo en 4 parquet dentro de
   `data/raw/`:

   ```Shell
   python scripts/extraer_bases_json.py
   ```
2. **Consolidación (Ejercicio 2)** — une `cliente` + `demografico` +
   `histRecuperacion`, normaliza identificadores, calcula edad y el target
   agregado de recuperación a 6 meses; genera
   `data/processed/base_consolidada.parquet`:

   ```Shell
   python src/consolidar_datos.py
   ```
3. **Entrenamiento (Ejercicio 3)** — identifica y separa los casos en
   cuarentena, arma el pipeline reproducible, compara 4 algoritmos con CV +
   tuneo de hiperparámetros, selecciona y persiste el modelo campeón.
   Genera `data/processed/quarantine.parquet`,
   `models/modelo_priorizacion_recuperacion.joblib` y
   `models/modelo_priorizacion_metadata.json`:

   ```Shell
   python src/entrenar_modelo.py
   ```
4. **Aplicación del modelo a cartera nueva** — carga el pipeline
   persistido, simula "cartera nueva" (muestra de `base_consolidada.parquet`
   sin columnas derivadas del target), separa los registros que cumplan
   alguna regla de cuarentena (no se puntúan, se reportan aparte en
   `data/processed/scoring_simulado_cuarentena.csv`) y genera scores de
   priorización para el resto en `data/processed/scoring_simulado_ejemplo.csv`:

   ```Shell
   python src/aplicar_modelo.py
   ```
5. **Notebook** — contiene el desarrollo completo, las visualizaciones y
   las respuestas de los 4 ejercicios. Se abre con Jupyter (`jupyter lab`
   o `jupyter notebook` desde el entorno de Anaconda) o se re-ejecuta de
   punta a punta con:

## 3. Cómo aplicar el modelo a cartera nueva

1. Asegurarse de que exista el pipeline entrenado en
   `models/modelo_priorizacion_recuperacion.joblib` (ejecutar primero
   `src/entrenar_modelo.py` si no existe o si se quiere reentrenar).
2. Preparar un DataFrame con las columnas de entrada del Modelo
   (`id_cliente_canonico`, `valorCapitalCompra`, `valorCompra`,
   `diasMoraCompra`, `edad`, `RelacionDependencia`, `Genero`,
   `EstadoCivil`, `NivelEstudio`, `regionCedenteInicial`, `es_fallecido`) —
   el mismo esquema que produce `src/consolidar_datos.py`.
3. Ejecutar `src/aplicar_modelo.py` (por defecto simula "cartera nueva"
   muestreando `base_consolidada.parquet`; para datos genuinamente nuevos,
   sustituir `simular_datos_nuevos()` por la carga del archivo real con el
   mismo esquema de columnas):
   ```Shell
   python src/aplicar_modelo.py
   ```
4. El script aplica primero las reglas de cuarentena
   (`utils/calidad_datos.identificar_casos_irregulares`) sobre los datos
   nuevos: los registros que las cumplan **no se puntúan** y se
   guardan aparte en `data/processed/scoring_simulado_cuarentena.csv` para
   revisión de calidad de datos.
5. Los registros válidos se puntúan con el pipeline persistido (mismos
   límites de cap-and-floor, imputador, transformación de sesgo, encoder,
   escalador y selector de variables aprendidos en entrenamiento, sin
   reajustar ningún parámetro) y el resultado (score de probabilidad de
   recuperación, ranking de prioridad y flag de prioridad de gestión según
   el umbral operativo top-30% aprendido en entrenamiento) se guarda en
   `data/processed/scoring_simulado_ejemplo.csv`.
