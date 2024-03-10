# Dos formas de obtener el reporte de IPC: API del FMI y Excel del BCRD

## Para ver el jupyter notebook con el detalle: [entra aquí](ipc_do.ipynb)

## TL;DR

### Usando el API del FMI

```python
>>> import requests
>>> import polars as pl
>>> url = "http://dataservices.imf.org/REST/SDMX_JSON.svc/\
... CompactData/CPI/M.DO.PCPI_IX"
>>> data = (
...    requests.get(url).json()
...    ['CompactData']['DataSet']['Series']
... )
>>> obs_list_of_dicts = data['Obs']
>>> df = (
...    pl.DataFrame(
...        [
...            [obs.get("@TIME_PERIOD") for obs in obs_list_of_dicts],
...            [obs.get("@OBS_VALUE") for obs in obs_list_of_dicts]
...        ], schema=['time_period', 'obs_value']
...    )
...    .drop_nulls()
...    .with_columns(
...        pl.col('time_period').str.to_date("%Y-%m"),
...        pl.col('obs_value').cast(pl.Float64)
...    )
... )
>>> print(df)
shape: (301, 2)
+-------------+-----------+
| time_period | obs_value |
| ---         | ---       |
| date        | f64       |
+=========================+
| 1999-01-01  | 21.434212 |
| 1999-02-01  | 21.37634  |
| 1999-03-01  | 21.487797 |
| 1999-04-01  | 21.543526 |
| 1999-05-01  | 21.532809 |
| …           | …         |
| 2023-09-01  | 125.3654  |
| 2023-10-01  | 125.6376  |
| 2023-11-01  | 125.8094  |
| 2023-12-01  | 126.4866  |
| 2024-01-01  | 126.9796  |
+-------------+-----------+
```

### Descargando el Excel del BCRD

```python
>>> import polars as pl
>>> url = "https://cdn.bancentral.gov.do/documents/estadisticas/\
... precios/documents/ipc_base_2019-2020.xls?v=1687458753373"
>>> r = requests.get(url)
>>> with open('ipc_base_2019-2020.xls', 'wb') as f:
...    f.write(r.content)
>>> df = pl.read_excel('ipc_base_2019-2020.xls')
>>> df = df[4:-1, 0:3]
>>> df.columns = ['Año', 'Mes', 'Indice']
>>> df = (
...    df
...    .with_columns(
...        pl.when(pl.col('Año').str.len_bytes() == 0)
...        .then(None)
...        .otherwise(pl.col('Año'))
...        .alias('Año')
...    )
... )
>>> df = (
...    df
...    .with_columns(
...        pl.col('Año')
...        .str.strip_chars()
...        .fill_null(strategy="forward")
...        .cast(pl.Int64, strict=False)
...        .fill_null(strategy="forward"),
...        pl.col('Indice').cast(pl.Float64),
...        pl.col('Mes')
...        .str.strip_chars()
...    )
...    .drop_nulls()
... )
>>> meses = ['Enero','Febrero','Marzo','Abril','Mayo','Junio','Julio',\
... 'Agosto','Septiembre','Octubre','Noviembre','Diciembre']
>>> df = (
...    df
...    .join(
...        pl.DataFrame(
...            {'Mes': meses,'mes_num': range(1,13)}
...        ), on = 'Mes', how='left'
...    )
...    .with_columns(
...        pl.date('Año', 'mes_num', 1).alias('time_period')
...    )
...    .rename({'Indice': 'obs_value'})
...    .select(['time_period', 'obs_value'])
... )
>>> print(df)
shape: (482, 2)
+-------------+-----------+
| time_period | obs_value |
| ---         | ---       |
| date        | f64       |
+=========================+
| 1984-01-01  | 1.37936   |
| 1984-02-01  | 1.418138  |
| 1984-03-01  | 1.435622  |
| 1984-04-01  | 1.458127  |
| 1984-05-01  | 1.475611  |
| …           | …         |
| 2023-10-01  | 125.6376  |
| 2023-11-01  | 125.8094  |
| 2023-12-01  | 126.4866  |
| 2024-01-01  | 126.9796  |
| 2024-02-01  | 127.0946  |
+-------------+-----------+
```
