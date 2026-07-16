# Databricks Medallion — Weather Pipeline

Mini-proyecto de práctica en Azure Databricks: arquitectura medallion (bronze/silver/gold) con Unity Catalog, usando datos de clima de [Open-Meteo](https://open-meteo.com/) (CDMX).

## Arquitectura

- **2 workspaces** (`dbx-weather-dev`, `dbx-weather-prod`) compartiendo un mismo metastore de Unity Catalog
- **2 catálogos aislados por ambiente** (`weather_dev`, `weather_prod`), cada uno con su propia ubicación externa en ADLS Gen2, y **workspace-catalog bindings** para que cada catálogo solo sea accesible desde su workspace correspondiente
- **3 schemas por catálogo**: `bronze`, `silver`, `gold`

## Pipeline

| Notebook | Capa | Qué hace |
|---|---|---|
| `01_bronze_ingest_weather` | Bronze | Llama a la API de Open-Meteo y guarda la respuesta cruda (JSON) tal cual, con metadata de ingesta |
| `02_silver_transform_weather` | Silver | Parsea el JSON, explota los arreglos horarios en filas individuales, tipa cada columna |
| `03_gold_daily_summary` | Gold | Agrega a nivel día: temperatura promedio/máx/mín, precipitación total, viento |

## Decisiones de diseño

- **Bronze se particiona por `ingestion_date`** (cuándo se jaló el dato) y usa `append` — funciona como log histórico de cada corrida
- **Silver y gold se particionan por `forecast_date`** (a qué fecha corresponde el dato), que es como se consulta en la práctica
- **Gold usa `overwrite`, no `append`** — cada corrida recalcula el resumen completo desde silver, así que acumular filas generaría duplicados
- **Linaje**: Unity Catalog detecta automáticamente bronze→silver→gold sin documentación manual, solo por el uso de `spark.table()`/`saveAsTable()`

## Gobierno de datos

- Catálogos aislados por ambiente vía workspace-catalog bindings
- Storage credentials + external locations en vez de claves de storage hardcodeadas
- Comentarios a nivel tabla/columna en las tablas gold