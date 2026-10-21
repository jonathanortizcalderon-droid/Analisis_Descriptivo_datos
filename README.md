# Informe de la Práctica — Análisis Descriptivo de Datos

**Curso:** Business Analytics — Especialización II
**Entregable:** Notebook — Informe de la práctica: Análisis Descriptivo de Datos
**Autores:** Jonathan Ortiz Calderón · Cindy Carolina Suárez Castro
**Repositorio:** https://github.com/jonathanortizcalderon-droid/Analisis_Descriptivo_datos

---

## 1. Descripción general

Este repositorio contiene el notebook [`Analisis_Descriptivo_Datos_Jonathan Ortiz_Carolina_Suarez.ipynb`](./Analisis_Descriptivo_Datos_Jonathan Ortiz_Carolina_Suarez.ipynb), que implementa un **Análisis Descriptivo** completo (la pregunta clave que responde es *¿qué ha sucedido?*) sobre un conjunto de datos transaccionales de una empresa ficticia, compuesto por tres fuentes heterogéneas:

| Archivo | Formato | Contenido | Registros |
|---|---|---|---|
| [`clientes.json`](./clientes.json) | JSON | Maestro de clientes | 15 |
| [`productos.xlsx`](./productos.xlsx) | Excel | Maestro de productos | 10 |
| [`ventas.csv`](./ventas.csv) | CSV (separador `;`) | Transacciones de venta | 515 (515 filas crudas) |

El trabajo se fundamenta en:
- El documento del curso *"Diferencias entre el Análisis Descriptivo, Predictivo y Prescriptivo"*.
- El artículo de Harvard Business School Online: [*4 Types of Data Analysis to Improve Decision-Making*](https://online.hbs.edu/blog/post/types-of-data-analysis).

El notebook se centra **exclusivamente** en análisis descriptivo: no construye modelos predictivos ni recomendaciones de optimización (eso corresponde a trabajos de análisis predictivo/prescriptivo); únicamente resume, limpia y visualiza lo que ya ocurrió con las ventas.

---

## 2. Estructura del notebook

El notebook tiene **61 celdas** (26 de código + 35 de markdown explicativo) organizadas en las siguientes secciones. Todas las celdas ya fueron ejecutadas de principio a fin sin errores; los resultados (tablas y gráficos) quedan guardados dentro del `.ipynb`.

### Portada y marco conceptual
Presenta autores, objetivo del trabajo y la tabla conceptual Descriptivo / Predictivo / Prescriptivo tomada del documento de apoyo, relacionándola con los ejemplos de HBS Online.

### 0. Configuración del entorno
Importa `pandas`, `numpy`, `matplotlib`, `seaborn` y `json`; define opciones de presentación (ancho de columnas, estilo de gráficos con `seaborn.set_theme`) y fija el directorio de trabajo.

### 1. Carga de datos desde fuentes heterogéneas
Carga cada archivo con el lector apropiado:
- `json.load` + `pd.DataFrame` para `clientes.json`.
- `pd.read_excel` para `productos.xlsx`.
- `pd.read_csv(..., sep=";")` para `ventas.csv`.

Resultado verificado: `clientes (15, 5)` · `productos (10, 4)` · `ventas (515, 7)`.

### 2. Exploración inicial
Inspecciona tipos de dato (`.info()`), dimensiones y valores únicos de las columnas categóricas (`ciudad`, `categoria`, `canal`, `descuento`) y una muestra de los formatos de fecha presentes.

### 3. Perfilado y diagnóstico de calidad de datos
Cuantifica de forma programática los problemas de calidad del dataset crudo. Hallazgos documentados en el notebook:

| Problema detectado | Dónde | Cantidad |
|---|---|---|
| Fechas en 3 formatos distintos (`YYYY-MM-DD`, `YYYY/MM/DD`, `DD/MM/YYYY`) | `ventas.fecha` | 515 filas afectadas |
| Filas 100% duplicadas | `ventas` | 15 |
| Canal con capitalización inconsistente (`Web`, `web`, `TIENDA`, `Tienda`) | `ventas.canal` | 5 variantes → 3 valores reales |
| Categoría de producto inconsistente (`Computo`, `accesorios`, `Accesorios `) | `productos.categoria` | 8 variantes → 6 valores reales |
| Ciudad con variantes de mayúsculas/acentos/espacios (`bogota`, `Bogotá `, `Bogotá`) | `clientes.ciudad` | 9 variantes → 6 ciudades reales |
| Nombres en mayúscula sostenida (`LAURA MARTÍNEZ`) | `clientes.nombre` | varios |
| Valores faltantes en `cantidad` | `ventas` | 9 |
| Valores faltantes en `descuento` | `ventas` | 135 |
| `cantidad` negativa (posibles devoluciones) | `ventas` | 13 |
| Llave foránea huérfana: `id_cliente = 16` sin maestro de cliente | `ventas` | 26 ventas |

### 4. Limpieza, estandarización e integración
Aplica reglas de negocio documentadas y justificadas en el propio notebook:

- **Clientes:** nombres a formato *Título*; ciudades normalizadas mediante un diccionario de equivalencias (`bogota` / `Bogotá ` → `Bogotá`); bandera `email_informado` para no perder trazabilidad de los correos faltantes; `fecha_registro` convertida a `datetime`.
- **Productos:** `categoria` recortada y normalizada a formato *Título*.
- **Ventas:**
  1. Eliminación de las 15 filas 100% duplicadas.
  2. Parseo flexible de fechas probando los 3 formatos detectados hasta encontrar el válido (`parse_fecha_flexible`).
  3. Normalización de `canal` a *Título* (`Web`, `App`, `Tienda`).
  4. Conversión de `descuento` de texto porcentual (`"15%"`) a proporción decimal (`0.15`); los 135 valores faltantes se interpretan como "sin descuento informado" y se imputan en `0`, conservando la bandera `descuento_informado`.
  5. Las 13 cantidades negativas se interpretan como **devoluciones** (no errores de digitación) y se marcan en la columna `tipo_movimiento` (`Venta` / `Devolución`), conservando el signo para que el monto neto refleje el efecto real en los ingresos.
  6. Las 9 filas sin `cantidad` se excluyen del análisis cuantitativo (no hay base confiable para imputarlas) y la exclusión queda documentada.
- **Integración:** se agrega un registro **"Cliente Desconocido"** (`id_cliente = 16`) al maestro de clientes para no descartar las 26 ventas huérfanas, siguiendo la práctica estándar de *miembro desconocido* en modelado dimensional. Luego se integran `ventas` + `clientes` + `productos` mediante `merge`.
- **Enriquecimiento:** se derivan `anio`, `mes`, `nombre_mes`, `trimestre`, `dia_semana`, `anio_mes`, y las métricas `monto_bruto`, `valor_descuento`, `monto_neto`, `cantidad_abs`.

Tras la limpieza, el dataset de ventas pasa de **515 a 491 filas** válidas para el análisis.

### 5. Modelo en estrella (Star Schema)
Construye un esquema dimensional a partir de los datos ya limpios e integrados:
- `dim_clientes` (16 registros, incluye "Cliente Desconocido"), `dim_productos` (10 registros) y `dim_tiempo` (279 fechas únicas), cada una con llave subrogada (`sk_*`).
- `fact_ventas` (491 registros) con las llaves subrogadas y las métricas de negocio (`cantidad`, `descuento_pct`, `monto_bruto`, `valor_descuento`, `monto_neto`).

### 6. Conciliación de la carga
Valida que la carga al modelo en estrella no haya alterado la información, comparando `df_integrado` contra `fact_ventas`. Resultado obtenido en la ejecución:

```
[OK] N° de filas: esperado=491.00 | obtenido=491.00
[OK] Suma monto_neto: esperado=797,463,000.00 | obtenido=797,463,000.00
[OK] Suma cantidad: esperado=1,417.00 | obtenido=1,417.00
[OK] N° clientes distintos: esperado=16.00 | obtenido=16.00
[OK] N° productos distintos: esperado=10.00 | obtenido=10.00
[OK] N° fechas distintas: esperado=279.00 | obtenido=279.00
Llaves subrogadas sin resolver: sk_cliente=0, sk_producto=0, sk_tiempo=0
>>> Conciliación exitosa: True
```

### 7. Estadísticas descriptivas
Calcula medidas de tendencia central y dispersión (media, mediana, moda, desviación estándar, varianza, cuartiles, asimetría) sobre `monto_neto`, y agrupa los ingresos netos por `canal`, `categoria` y `ciudad` para identificar dónde se concentra el negocio.

### 8. Visualizaciones (7 gráficos generados)
Cada gráfico incluye una celda de interpretación en markdown inmediatamente después:

1. **Evolución mensual del ingreso neto** (línea) — tendencia temporal / estacionalidad.
2. **Ingreso neto por canal de venta** (barras) — Web vs. App vs. Tienda.
3. **Ingreso neto por categoría de producto** (barras horizontales).
4. **Top 10 clientes por ingreso neto** (barras horizontales) — concentración de clientes.
5. **Distribución del monto neto por transacción** (histograma + boxplot por canal).
6. **Matriz de correlación** entre `cantidad_abs`, `descuento_pct`, `precio_unitario`, `monto_bruto`, `monto_neto` (heatmap).
7. **Ingreso neto por ciudad del cliente** (barras).

### 9–10. Interpretación general y conclusiones
Cierra el notebook relacionando los hallazgos con el marco conceptual (qué ha sucedido, no qué sucederá ni qué hacer), resume las decisiones de limpieza tomadas, y propone líneas futuras de análisis predictivo/prescriptivo que podrían construirse sobre esta base descriptiva. Incluye el enlace al repositorio y los autores.

---

## 3. Cómo ejecutar el notebook

### Requisitos
- Python 3.10+ (probado con Python 3.13 / 3.14 vía el *launcher* `py`).
- Librerías: `pandas`, `numpy`, `matplotlib`, `seaborn`, `openpyxl` (lectura de `.xlsx`), `jupyter`/`jupyterlab` o la extensión Jupyter de VS Code.

Instalación rápida de dependencias:

```powershell
py -m pip install pandas numpy matplotlib seaborn openpyxl jupyterlab
```

### Ejecución
1. Clonar o descargar este repositorio, asegurando que `clientes.json`, `productos.xlsx` y `ventas.csv` queden en la **misma carpeta** que el notebook (el notebook usa rutas relativas con `Path.cwd()`).
2. Abrir `Analisis_Descriptivo_Datos_Jonathan Ortiz_Carolina_Suarez.ipynb` en Jupyter Lab, Jupyter Notebook o VS Code.
3. Ejecutar todas las celdas en orden (*Run All*). El notebook es determinista: cada celda depende de las anteriores (carga → perfilado → limpieza → integración → modelo en estrella → conciliación → estadísticas → visualizaciones).
4. También puede regenerarse/ejecutarse por línea de comandos:

```powershell
py -m jupyter nbconvert --to notebook --execute --inplace Analisis_Descriptivo_Datos_Jonathan Ortiz_Carolina_Suarez.ipynb
```

---

## 4. Principales hallazgos (resumen ejecutivo)

- De las **515 transacciones crudas**, quedaron **491 válidas** tras eliminar duplicados exactos (15) y registros sin `cantidad` informada (9).
- **26 ventas (≈5%)** referenciaban un cliente inexistente en el maestro (`id_cliente = 16`); se preservaron mediante un miembro dimensional "Cliente Desconocido" en lugar de descartarlas.
- **13 transacciones** tenían cantidad negativa y se reclasificaron como **devoluciones**, no como errores de digitación.
- El ingreso neto total conciliado entre los datos integrados y el modelo en estrella es de **$797.463.000**, sobre **1.417 unidades** vendidas/devueltas.
- Se identificaron patrones de concentración de ingresos por canal, categoría de producto, ciudad y en un grupo reducido de clientes (Top 10), además de la tendencia mensual de ingresos a lo largo de 2025.

---

## 5. Contenido del repositorio

```
├── Analisis_Descriptivo_Datos_Jonathan Ortiz_Carolina_Suarez.ipynb   # Notebook principal (análisis descriptivo completo)
├── clientes.json                          # Maestro de clientes (fuente JSON)
├── productos.xlsx                         # Maestro de productos (fuente Excel)
├── ventas.csv                             # Transacciones de venta (fuente CSV, separador ;)
└── README.md                              # Este documento
```

## Autores
- Jonathan Ortiz Calderón
- Cindy Carolina Suárez Castro
