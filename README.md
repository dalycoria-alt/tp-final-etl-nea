# Pipeline ETL de exportaciones del NEA

**TP Final — Unidad II · Fundamentos de la Programación**
Diplomatura en Data Analytics e IA Aplicada — UNNE / Extender
Autor: Dalila Coria

---

## Qué hace este pipeline

Es un programa en Python que arma, de forma automática, un dataset con las
exportaciones de las cuatro provincias del NEA (Chaco, Corrientes, Formosa y
Misiones) por país de destino, entre 1993 y 2024.

Trabaja en tres etapas, cada una en su propio archivo:

| Etapa | Archivo | Qué hace |
|---|---|---|
| **Extract** | `src/extract.py` | Pide los datos a la API de datos.gob.ar y guarda las respuestas crudas en `data/raw/` |
| **Transform** | `src/transform.py` | Pasa los datos de formato ancho a largo, calcula participación, variación interanual, década y ranking, y los une con los rubros de exportación |
| **Load** | `src/load.py` | Corre los controles de calidad y guarda el CSV, el resumen JSON y el log |

`src/main.py` llama a las tres etapas en orden, y `config.py` reúne todos los
parámetros (IDs de series, rutas, umbrales) para no tenerlos escritos dentro
de la lógica.

### Controles de calidad

Antes de guardar, el pipeline verifica cuatro cosas. Si alguna falla, se
detiene en lugar de generar un archivo con errores:

1. **Cantidad:** hay al menos 1.000 filas.
2. **Columnas:** todas las filas tienen las 13 columnas del contrato.
3. **Unicidad:** no hay filas repetidas por (provincia, año, destino).
4. **Rangos:** ningún valor es negativo ni supera los 10.000 millones de USD.

Además informa, como advertencia, cuántas filas quedaron sin variación
interanual o sin rubro.

---

## Instalación y ejecución

Requisitos: **Python 3.8 o superior** y conexión a internet. No hace falta
instalar librerías: el proyecto usa solo la biblioteca estándar.

```bash
# 1. Clonar el repositorio
git clone https://github.com/dalycoria-alt/tp-final-etl-nea.git
cd tp-final-etl-nea

# 2. (Opcional) Crear un entorno virtual
python -m venv .venv
.venv\Scripts\activate        # Windows
source .venv/bin/activate     # Mac / Linux

# 3. Ejecutar el pipeline
python src/main.py

# 4. Correr los tests
python tests/test_transform.py
```

Si ya descargaste los datos una vez, podés reprocesarlos sin internet:

```bash
python src/main.py --sin-internet
```

### Salidas

| Archivo | Contenido |
|---|---|
| `data/processed/exportaciones_nea.csv` | Dataset final: una fila por año, provincia y destino, con 13 columnas |
| `data/processed/resumen.json` | Ficha técnica: fuente, período, estadísticas y resultado de los controles |
| `logs/pipeline.log` | Una línea por cada corrida |

El pipeline es **idempotente**: correrlo varias veces deja siempre el mismo
CSV, y el log suma una línea por corrida.

---

## Fuente de los datos

- **API de Series de Tiempo** del portal de datos abiertos del Estado
  argentino: <https://apis.datos.gob.ar/series/api/>
- **Dataset 357.1:** Exportaciones por provincia y por país de destino.
- **Dataset 350.1:** Exportaciones por provincia y por rubro.
- **Origen:** INDEC. **Unidad:** millones de dólares FOB. **Período:** 1993–2024.

---

## Hallazgos

### Corrientes en 2020: un año fuera de serie

Mirando el total por provincia, 2020 es un año muy raro para Corrientes. Sus
exportaciones pasaron de **217 millones de USD en 2019 a 586 millones en
2020**, casi el triple, y al año siguiente volvieron a niveles normales.

El salto viene casi todo de un solo destino: **Brasil, que pasó de 25 a 399
millones de USD y llegó a representar el 68 % de todo lo que exportó la
provincia ese año.** Además, ese es el único año en que el rubro principal de
Corrientes no fue *Productos primarios* sino **Combustibles y Energía**, y
la participación de los productos primarios cayó del 63 % al 21 %.

Me llamó la atención y creo que puede ser por exportacion de energía, habría que investigarlo.

### China, el destino que creció en Chaco

En 1993 Chaco le vendía a China apenas 0,31 millones de USD. Desde 2001
China está casi todos los años entre los dos primeros destinos de la
provincia, solo detrás de "Resto". En 2010 llegó a ser el primero, y en 2024
recibió 110,93 millones de USD, el 27,6 % del total.

---

## Limitaciones conocidas

**Faltan datos de Misiones.** El dataset tiene **1.152 filas en lugar de las
1.408 esperadas.** La API devuelve vacías (`count: 0`) 8 de las 11 series de
destino de Misiones (Brasil, Estados Unidos, Bélgica, China, Francia, Chile,
Sudáfrica y Alemania) y también la serie del total provincial. Lo verifiqué
consultando la API directamente.

El pipeline saltea esas observaciones, como pide la consigna, en lugar de
inventar valores. Por eso Misiones solo aparece con Siria, Países Bajos y
Resto, y sin `participacion_pct`, porque falta el total para calcularla. Las
otras tres provincias están completas: 11 destinos × 32 años cada una.