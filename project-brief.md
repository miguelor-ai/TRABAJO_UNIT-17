# Previsión de demanda horaria para el sistema de bicicletas compartidas de Seúl

Miguel Ochoa· **Fecha:** 15/10/2026 · **Tipo de solución:** motor analítico de previsión + dashboard de gestión

---

## 1. Resumen

El sistema público de bicicletas de Seúl decide cada día cuántas bicicletas dejar operativas y cuándo mandar a las cuadrillas. Tendra como objetivo preveer la demanda horaria del día siguiente con el histórico de alquileres y la meteorología, y la presenta en un dashboard con una recomendación de flota por hora.

---

## 2. Datos

### Ficha del dataset

| Campo                           | Respuesta                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Nombre                          | Seoul Bike Sharing Demand (copia en Kaggle: *Seoul Bike Sharing Demand Prediction*)                                                                                                                                                                                                                                                                                                                                                          |
| Publicador y URL                | Repositorio: UCI Machine Learning Repository (donado el 29/02/2020), https://archive.ics.uci.edu/dataset/560, DOI https://doi.org/10.24432/C5F62R. Autores: Sathishkumar V E, Park J., Cho Y. Según el artículo asociado, los datos proceden del Seoul Public Data Park (Gobierno Metropolitano de Seúl). Espejo en Kaggle: https://www.kaggle.com/datasets/joebeachcapital/seoul-bike-sharing                                               |
| Licencia                        | UCI: **CC BY 4.0** (https://creativecommons.org/licenses/by/4.0/). Copia de Kaggle: **ODbL** (http://opendatacommons.org/licenses/odbl/1-0/) para la base de datos y **DbCL** (http://opendatacommons.org/licenses/dbcl/1.0/) para los contenidos. Todas son abiertas                                                                                                                                                                        |
| Fecha de descarga y versión     | 15/10/2026. `SeoulBikeData.csv` del ZIP oficial de UCI (dataset 560); ficha sin cambios desde la donación (29/02/2020)                                                                                                                                                                                                                                                                                                                       |
| Granularidad (una fila)         | Una hora de la ciudad de Seúl, agregada para todo el sistema (no hay estaciones físicas ni viajes individuales)                                                                                                                                                                                                                                                                                                                              |
| Periodo cubierto                | 01/12/2017 – 30/11/2018 (365 días × 24 h)                                                                                                                                                                                                                                                                                                                                                                                                    |
| Tamaño                          | B8.760 filas × 14 columnas, 590 K                                                                                                                                                                                                                                                                                                                                                                                                            |
| Columnas clave                  | **Fecha:** `Date`, `Hour`. **Dimensiones:** `Seasons`, `Holiday`, `Functioning Day`. **Medidas:** `Rented Bike Count` (mín. 0, máx. 3.556, media 704,6), temperatura, humedad, viento, visibilidad, punto de rocío, radiación solar, lluvia, nieve. **Variable objetivo:** `Rented Bike Count`                                                                                                                                               |
| Problemas de calidad detectados | 0 nulos y 0 duplicados (tampoco por `(Date, Hour)`). 295 horas (13 días, 3,37 %) con `Functioning Day = No`, todas con 0 alquileres: son servicio parado, no demanda. El CSV solo se lee con `encoding="latin-1"` (UTF-8 falla en `°C`). `Date` es texto `dd/mm/aaaa`, aunque la ficha de UCI la describe como año-mes-día. `Visibility` va en unidades de 10 m. Temperatura y punto de rocío correlacionan (r = 0,91). 528 horas con lluvia |
| Otras fuentes a unir            | Previsión meteorológica de la KMA (agencia meteorológica coreana) y calendario oficial de festivos de Corea del Sur, para pasar de meteorología observada a prevista                                                                                                                                                                                                                                                                         |

---

## 3. Problema de negocio

### 3.1 Organización, departamento y stakeholder

- **Organización:** el operador del sistema público de bicicletas compartidas de Seúl, que ofrece alquiler por horas como complemento del transporte público. El dataset no nombra al operador, así que es el destinatario previsto.
- **Departamento:** el equipo de planificación de flota (escenario de partida: unas 6 personas), que decide cuántas bicicletas operativas hay en cada franja y cuándo salen las cuadrillas de reposición y mantenimiento.
- **Stakeholder:** la jefa de planificación de operaciones. Revisa el dashboard cada tarde, para fijar el plan del día siguiente, y cada semana para ver tendencias. Puede cambiar turnos de cuadrillas, nivel de flota por franja y ventanas de mantenimiento.

### 3.2 Decisión a apoyar

> Con esto, la jefa de planificación decidirá **cuántas bicicletas desplegar y en qué horas concentrar las cuadrillas de reposición al día siguiente, y en qué franjas de baja demanda programar el mantenimiento.**

**Restricciones:** presupuesto de cuadrillas fijo; solo hay un total ciudad-hora; la meteorología del día siguiente es una previsión; no hay datos personales.
### 3.3 Hipótesis

| | Afirmación | Métrica | Dirección esperada | Cómo se contrasta | Si cierta → | Si falsa → |
|---|---|---|---|---|---|---|
| **H1** | En días laborables la demanda tiene dos picos (mañana y tarde); en fin de semana y festivos, uno solo a media tarde | Media de `Rented Bike Count` por hora y tipo de día; ratio pico/valle | Picos en torno a las 8:00 y las 18:00 en laborables | Líneas por hora segmentadas por tipo de día | Dos turnos de reposición, antes de cada pico | Un único perfil horario para toda la semana |
| **H2** | Con lluvia, la demanda en horas punta de días laborables (7-9 h y 17-19 h) cae más de un 30 % frente a las mismas horas en seco | Ratio de alquileres medios con `Rainfall > 0` frente a seco, misma hora, en horas punta laborables | Caída notable, pero menor que la media global (−78 %: 163,5 frente a 739,3 alquileres/hora) | Comparación de medias por hora + test de diferencias | Reducir flota y cuadrillas en días de lluvia prevista, también en horas punta | Mantener el plan normal en horas punta aunque llueva |
| **H3** | La relación con la temperatura no es lineal: sube hasta unos 25-30 °C y cae con calor extremo | Demanda media por tramos de 5 °C, por franja horaria | Curva en campana | Gráfico por tramos de temperatura y estación del año | En olas de calor, mover la reposición a primera hora y tarde-noche | Tratar la temperatura como lineal en el modelo |
| **H4** | Añadir meteorología a una línea base de hora + tipo de día reduce el error de predicción al menos un 20 % | WAPE horario con y sin variables de clima, en test temporal | Error claramente menor con clima | Comparación de modelos en la fase de modelado | Invertir en integrar la previsión meteorológica | Usar solo perfiles horarios por calendario, más simples y baratos |

### 3.4 KPIs

| KPI | Definición y fórmula | Granularidad | Comparación | Objetivo |
|---|---|---|---|---|
| Alquileres por día | Suma de `Rented Bike Count` por fecha | Día | Estación del año, tipo de día | Referencia histórica |
| Demanda en horas pico | Media de alquileres en las 3 horas más cargadas de cada día | Día | Tipo de día (**H1**) | Guía la dotación de cuadrillas |
| Sensibilidad a la lluvia | Alquileres medios con lluvia ÷ alquileres medios en seco (misma hora) | Estación del año | Horas punta frente a resto (**H2**) | Valor de referencia |
| Error de predicción (WAPE) | Σ\|real − predicción\| ÷ Σ real | Hora | Modelo con/sin clima (**H4**) | ≥ 20 % mejor que la línea base |
| Horas sin servicio | % de horas con `Functioning Day = No` | Mes | Mes | Seguimiento (3,37 % en 2017-18); sirve para ubicar el mantenimiento |

Priorizar la cobertura en horas pico sube el coste de cuadrillas; por eso el motor devuelve una flota con margen sobre la previsión (por ejemplo, el percentil 80), ajustable por la jefa de planificación.

---

## 4. Diseño de la solución

### 4.1 Tipo de solución

**Principal: motor analítico**, mostrado en un **dashboard de gestión**.

- **Entradas:** histórico horario (demanda, clima, calendario) y previsión meteorológica de las próximas 24 horas.
- **Lógica:** línea base por perfil (hora × tipo de día) y un modelo de regresión que añade el clima.
- **Salidas:** demanda prevista hora a hora del día siguiente, nivel de flota recomendado por hora y alertas de horas de alta presión, junto con los KPIs y el resultado de las hipótesis.

### 4.2 Arquitectura

```mermaid
flowchart LR
    A[Fuente<br/>UCI / Kaggle CSV + previsión KMA] --> B[Ingesta<br/>script de descarga, pandas]
    B --> C[Limpieza y calidad<br/>tipos, fechas, horas sin servicio, outliers]
    C --> D[Modelo de datos<br/>hecho horario + dimensiones, parquet]
    D --> E[Motor analítico<br/>KPIs, línea base, modelo de demanda]
    E --> F[Dashboard<br/>Streamlit + plotly]
    F --> G((Jefa de planificación<br/>decisión diaria))
```

| Capa | Qué hace | Herramienta | Refresco |
|---|---|---|---|
| Fuente | CSV histórico (estático) y previsión meteorológica (opcional) | UCI / Kaggle, API de la KMA | Histórico: una vez. Previsión: diaria |
| Ingesta | Carga con la codificación correcta y valida el esquema | Python, pandas | Bajo demanda |
| Limpieza y calidad | Convierte la fecha, normaliza nombres de columna, marca horas sin servicio, revisa outliers | pandas | Con cada ingesta |
| Modelo de datos | Tabla de hechos horaria (`fact_hourly`) y dimensiones `dim_date` (día de la semana, mes, estación del año, festivo) y `dim_hour` (franja) | Parquet | Con cada ingesta |
| Motor analítico | KPIs, línea base y modelo de demanda | pandas, scikit-learn | Diario |
| Dashboard | Filtros por estación del año, mes, tipo de día y clima; previsión y flota recomendada | Streamlit, plotly | Diario |

**Riesgos principales:**

1. **Una sola temporada de cada tipo.** Puede sobreestimar la precisión; se mitiga con test temporal y error por estación del año.
2. **Meteorología observada frente a prevista.** En producción el error será mayor; se evaluará con previsiones o ruido añadido.
3. **Licencia.** Se atribuye siempre a UCI y a los autores. La copia de Kaggle (ODbL) exige además mantener la licencia abierta en bases de datos derivadas que se publiquen; se publicarán solo resultados agregados.
**Privacidad y ética.** Son totales por hora, sin usuarios ni rutas: riesgo de privacidad bajo. El riesgo ético es que optimizar solo para las horas de mayor demanda deje sin servicio a quienes usan el sistema en franjas valle; la recomendación debe garantizar un mínimo de cobertura.

---

## Fuentes

Joebeachcapital (n.d.) _Seoul Bike Sharing Demand Prediction_ [Dataset]. Kaggle. Available at: [https://www.kaggle.com/datasets/joebeachcapital/seoul-bike-sharing](https://www.kaggle.com/datasets/joebeachcapital/seoul-bike-sharing) (Accessed: 18 October 2026).

Sathishkumar, V.E., Park, J. and Cho, Y. (2020a) ‘Using data mining techniques for bike sharing demand prediction in metropolitan city’, _Computer Communications_, 153, pp. 353-366. doi: 10.1016/j.comcom.2020.02.007.

Sathishkumar, V.E., Park, J. and Cho, Y. (2020b) _Seoul Bike Sharing Demand_ [Dataset]. UCI Machine Learning Repository. Available at: [https://doi.org/10.24432/C5F62R](https://doi.org/10.24432/C5F62R) (Accessed: 18 October 2026).