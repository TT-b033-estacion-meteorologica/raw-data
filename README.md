# Datos Meteorológicos en Crudo - Estación ESCOM IPN

Registros meteorológicos **sin transformar** capturados cada 15 minutos por la estación meteorológica Davis Vantage Vue
dentro la Escuela Superior de Cómputo (ESCOM) del Instituto Politécnico Nacional,
Ciudad de México. Este repositorio conserva los datos tal como los entrega la consola de la
estación, para trazabilidad, auditoría y reutilización como datos abiertos.

> ¿Buscas datos listos para analizar? Usa el repositorio de datos limpios:
> [clean-data](https://github.com/TT-b033-estacion-meteorologica/clean-data).

## Resumen

| Campo | Valor |
|---|---|
| Institución | Escuela Superior de Cómputo - IPN |
| Ubicación | Ciudad de México (19.5046, -99.1467) |
| Altitud | ≈2,240 msnm |
| Resolución temporal | cada 15 minutos |
| Registros esperados por día | 96 (la mayoría de los días completos tiene 95) |
| Cobertura temporal | desde el 18 de abril de 2026; se agrega un archivo nuevo por cada día con datos |
| Formato | CSV, un archivo por día |
| Licencia | CC BY 4.0 |
| Repositorio de datos limpios | [clean-data](https://github.com/TT-b033-estacion-meteorologica/clean-data) |

## ¿Datos en crudo o datos limpios?

| | **raw-data** (este repositorio) | **clean-data** |
|---|---|---|
| Contenido | Datos originales de la estación | Datos con valores inválidos convertidos a nulos y tipos corregidos |
| Granularidad | 1 archivo por día | 1 archivo mensual por vista temática |
| Se sobrescribe | No (un archivo nuevo por día) | Sí (el archivo del mes crece a diario) |
| Úsalo para | Verificar la captura, auditar o hacer tu propia limpieza | Análisis, visualización y modelado |

## Estructura del repositorio

```
raw-data/
├── README.md
├── metadata.yml
└── data/
    └── AAAA/
        └── MM/
            ├── weather_AAAAMMDD.csv
            └── ...
```

Los archivos se nombran `weather_AAAAMMDD.csv`, donde `AAAAMMDD` es la fecha **de las mediciones**
(no la de publicación). Por ejemplo, `data/2026/10/weather_20261018.csv` contiene las mediciones
del 18 de octubre de 2026. No existe archivo para los días en que no se obtuvieron datos.

## Formato de los archivos

- CSV separado por comas, con una línea de encabezado `Date` y una fila por
  registro de 15 minutos. 
- Fecha en formato `DD:MM:AAAA` y hora local en formato `HH:MM`.
- Los valores faltantes o no disponibles se representan con `---`.
- Los valores no se modifican: cada archivo es el contenido que entregó la estación para ese día.

### Diccionario de campos

Los nombres corresponden al esquema con el que estos datos se almacenan en la base de datos del
proyecto. 

| Campo | Descripción | Unidad |
|---|---|---|
| `date_raw` | Fecha del registro | DD:MM:AAAA |
| `time_raw` | Hora local del registro | HH:MM |
| `temp_out` | Temperatura exterior | °C |
| `hi_temp` | Temperatura máxima del intervalo | °C |
| `low_temp` | Temperatura mínima del intervalo | °C |
| `in_temp` | Temperatura interior | °C |
| `out_hum` | Humedad relativa exterior | % |
| `in_hum` | Humedad relativa interior | % |
| `bar` | Presión barométrica | mmHg |
| `wind_speed` | Velocidad del viento | km/h |
| `hi_speed` | Velocidad máxima (ráfaga) | km/h |
| `wind_dir` | Dirección del viento | rumbo |
| `hi_dir` | Dirección de la ráfaga | rumbo |
| `rain` | Precipitación acumulada | mm |
| `rain_rate` | Tasa de precipitación | mm/h |
| `temp_units`, `bar_units`, `wind_units`, `rain_units` | Unidades declaradas por la consola | texto |
| `solar_rad`, `uv_Index`, `et` | Radiación solar, índice UV y evapotranspiración | sin datos |

## Procedencia

Un proceso automático en Google Cloud consulta cada día, a la 01:00 (hora de la Ciudad de
México), la consola de la estación y obtiene los registros del día anterior; además revisa los
últimos 7 días y recupera los registros que falten. Los registros se cargan a una base de datos
PostgreSQL y se agregan aquí sin transformación; en la base de datos se limpian y se publican a
diario (13:00) en [clean-data](https://github.com/TT-b033-estacion-meteorologica/clean-data).
Los metadatos de este repositorio, según la norma NTM-INEGI 2024 (v2.0), están en
`metadata.yml`.

## Calidad y limitaciones conocidas

Estos datos **no están limpios ni validados**. Antes de analizarlos, considera:

- Los valores `---` indican dato no disponible y deben tratarse como nulos.
- Pueden existir días incompletos por cortes en la transmisión de la estación.
- El registro de las 00:00 h no se captura, por lo que la mayoría de los días completos tiene 95
  registros en lugar de 96.
- Hay días sin datos que ya no pueden recuperarse: 31 de mayo; 19, 23, 24 y 29 de junio; 3, 18 y
  19 de julio; 11 de agosto, y 7, 15 y 16 de septiembre de 2026. Los días de la puesta en marcha
  (24 de abril al 2 de mayo de 2026) y algunos días aislados están incompletos.
- **Presión barométrica:** el campo `bar` está en mmHg, no en hPa. Del 3 de julio de 2026 a las
  00:00 h al 20 de octubre de 2026 a las 09:15 h la consola tuvo una calibración errónea del
  barómetro y reportó la presión 192.1 mmHg por debajo de su valor (≈572–582 mmHg). Aquí se
  conserva sin corregir; la corrección (+192.1 mmHg) solo se aplica en
  [clean-data](https://github.com/TT-b033-estacion-meteorologica/clean-data).
- Temperatura y humedad relativa se reportan sin decimales (resolución de 1 °C y 1 %).
- Radiación solar, índice UV y evapotranspiración no están disponibles (sensores no instalados).
- Viento: la mayoría de los intervalos registra calma (0 km/h); el anemómetro podría estar
  resguardado, por lo que los datos reflejan la circulación local.
- Los valores atípicos no se marcan ni se eliminan. En cada captura el sistema revisa los registros
  nuevos (rangos físicos, saltos de presión y sensores sin lectura) y avisa al equipo.

## Cómo usar los datos

Ejemplo en Python para unir todos los archivos:

```python
import glob
import pandas as pd

archivos = sorted(glob.glob("data/*/*/weather_*.csv"))
df = pd.concat(
    (pd.read_csv(f, na_values=["---"], skipinitialspace=True) for f in archivos),
    ignore_index=True,
)
df.columns = df.columns.str.strip()
df["timestamp"] = pd.to_datetime(
    df.iloc[:, 0].astype(str).str.strip() + " " + df.iloc[:, 1].astype(str).str.strip(),
    format="%d:%m:%Y %H:%M",
)
df = df.drop_duplicates().sort_values("timestamp")
```

## Cómo citar

> Estación Meteorológica ESCOM-IPN (2026). *Datos meteorológicos en crudo - Estación ESCOM IPN*
> [conjunto de datos]. GitHub. https://github.com/TT-b033-estacion-meteorologica/raw-data

## Licencia

Estos datos se publican bajo la licencia
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.es): puedes usarlos, compartirlos y
adaptarlos citando la fuente. Se proporcionan tal como son capturados, sin garantía de exactitud
ni de continuidad del servicio.

