# Clasificación de la causa de incendios forestales en EE.UU.

Seminario Final Licenciatura en Ciencia de Datos
Universidad Nacional Guillermo Brown

Facundo Tarizzo

---

## Tema a tratar

Este trabajo surge sobre la base FPA-FOD, que tiene 1.880.465 incendios forestales registrados en
Estados Unidos entre 1992 y 2015. Cada incendio cuenta con ubicación, fecha, tamaño, quién lo
reportó y cuál fue la causa. Con estas variables lo que busca este trabajo es reponder a la pregunta de
¿se puede predecir si un incendio fue causado por causas naturales o directa/indirectamente por una persona? 


Empecé queriendo predecir la causa exacta, que en el dataset son 13 categorías. Los resultados fueron bajos,
me quedé en un macro-F1 de 0,37 y no lograba subirlo con ningún ajuste probablemente porque el dataset no enriquecia
lo suficiente a mi investigación. Antes de descartar la idea busqué qué habían hecho otros con esta misma base. Encontré un
paper de 2025 (Pourmohamad et al., en Earth's Future) que hace exactamente eso pero con
casi 270 variables extra de clima, vegetación y densidad poblacional. Aún
con toda esa información llegan al 55% de exactitud discriminando entre las causas humanas.

## Los datos

| | |
|---|---|
| Registros totales | 1.880.465 |
| Con causa conocida | 1.713.742 |
| Sin causa reportada | 166.723 (8,87%) |
| Causados por rayo | 278.468 (16,25% de los conocidos) |
| Período | 1992–2015 |
| Formato | SQLite, tabla `Fires` |

De las 39 columnas originales me quedé con ocho:

| Variable | Qué es |
|---|---|
| `FIRE_YEAR` | Año |
| `DISCOVERY_DOY` | Día del año en que se descubrió (1 a 366) |
| `LATITUDE` | Latitud |
| `LONGITUDE` | Longitud |
| `FIRE_SIZE` | Superficie quemada en acres |
| `STATE` | Estado |
| `OWNER_DESCR` | Tipo de propietario del terreno |
| `NWCG_REPORTING_AGENCY` | Agencia que reportó el incendio |

Descarté las columnas con más del 40% de valores faltantes (hora de descubrimiento, fecha de
contención, condado), los identificadores y los códigos numéricos que repetían información
que ya estaba en texto.

## Segmentación para la validación de modelos

La idea es usar una validación temporal progresiva en cuatro lotes. Cada lote entrena con los años
anteriores y prueba con los que siguen:

| Lote | Entrena | Prueba |
|---|---|---|
| 1 | 1992–2001 | 2002–2004 |
| 2 | 1992–2004 | 2005–2007 |
| 3 | 1992–2007 | 2008–2011 |
| 4 | 1992–2011 | 2012–2015 |

No uso split aleatorio a propósito por pruebas anteriores. Los incendios cercanos en tiempo y espacio se parecen
mucho entre sí, así que si mezclo todo al azar el conjunto de prueba termina teniendo casos
casi idénticos a los de entrenamiento y el resultado sale inflado. Roberts et al. (2017)
tratan este problema en detalle.

Con cuatro lotes tengo cuatro mediciones en vez de una, así que puedo reportar un promedio y
un desvío en lugar de un número suelto que puede haber salido por suerte del corte.

Dentro de cada lote reservo los últimos dos años del entrenamiento para validación. Ahí elijo
el umbral de decisión y el número de iteraciones. El conjunto de prueba no se toca hasta el
final.

## Resultados

Comparé tres modelos. Los números son el promedio de los cuatro lotes con su desvío:

| Modelo | AUC-ROC | AUC-PR | Bal. Accuracy | F1 | MCC |
|---|---|---|---|---|---|
| Regresión logística | 0,875 ± 0,029 | 0,614 ± 0,072 | 0,815 | 0,632 | 0,560 |
| Random Forest | 0,922 ± 0,018 | 0,751 ± 0,056 | 0,832 | 0,710 | 0,655 |
| **CatBoost** | **0,929 ± 0,018** | **0,780 ± 0,063** | **0,842** | **0,726** | **0,674** |

CatBoost gana en los cuatro lotes, sin excepción. El desvío del AUC-ROC es bajo (0,018), o
sea que el resultado no depende del corte temporal que elegí.

AUC-PR lote por lote:

| Lote (período de prueba) | Reg. logística | Random Forest | CatBoost |
|---|---|---|---|
| 1 (2002–2004) | 0,708 | 0,825 | 0,856 |
| 2 (2005–2007) | 0,619 | 0,740 | 0,771 |
| 3 (2008–2011) | 0,535 | 0,688 | 0,703 |
| 4 (2012–2015) | 0,594 | 0,749 | 0,788 |

Matriz de confusión del último lote (umbral 0,392):

| | Predice antrópico | Predice natural |
|---|---|---|
| **Antrópico real** | 210.942 (94,9%) | 11.264 (5,1%) |
| **Natural real** | 10.462 (26,4%) | 29.172 (73,6%) |

Uso AUC-PR además de AUC-ROC porque las clases están desbalanceadas: solo el 16% de los
incendios son naturales. Con ese desbalance la exactitud común engaña, porque un modelo que
prediga siempre "antrópico" saca 84% sin haber aprendido nada.

## Peso de las variables dentro del modelo

Importancia de variables según CatBoost:

| Variable | Importancia |
|---|---|
| `DISCOVERY_DOY` | 35,8% |
| `LONGITUDE` | 18,9% |
| `LATITUDE` | 14,2% |
| `STATE` | 11,1% |
| `FIRE_YEAR` | 7,2% |
| `OWNER_DESCR` | 5,7% |
| `NWCG_REPORTING_AGENCY` | 4,2% |
| `FIRE_SIZE` | 2,9% |

Lo que más me convenció de que el modelo aprendió algo real y no un atajo estadístico es el
análisis SHAP de `DISCOVERY_DOY`. La curva reproduce la temporada de tormentas eléctricas de
Estados Unidos: empuja hacia antrópico de enero a abril, cruza el cero a fines de abril,
llega al máximo en julio y agosto, y vuelve a bajar en octubre.

`FIRE_SIZE` muestra que los incendios grandes tienden a ser naturales, lo cual tiene sentido:
los rayos caen en zonas remotas donde el fuego se detecta tarde y se combate menos.

Los errores se concentran en verano, que es la única época donde conviven rayos y actividad
humana. El resto del año casi no hay error porque casi no hay realmente causas naturales de peso.

En incendios de más de 1000 acres el 12% son falsos positivos: son grandes, y como los rayos
tienden a producir incendios grandes, el modelo los marca como naturales. Pero hay incendios
humanos enormes también.

## Los registros sin causa

Los 166.723 registros sin causa quedaron afuera del modelos. Pero no son una muestra al azar, y eso puede meter un sesgo, así que los
analicé aparte (Figura 5 del notebook de exploración).

Hay 62 combinaciones de estado y año donde más del 80% de los incendios no tiene causa
anotada. Esas 62 concentran el 60,6% de todos los registros sin causa del dataset. No es un
goteo constante, son bloques.

Y depende muchísimo de quién reporta:

| Agencia | Sin causa | Registros |
|---|---|---|
| FS (Forest Service) | 0,03% | 220.497 |
| BIA | 0,40% | 119.943 |
| BLM | 5,86% | 97.034 |
| ST/C&L (Estado/Condado/Local) | 9,92% | 1.377.090 |
| IA (Interagencial) | 99,88% | 21.841 |

Probé modelarlo con el mismo diseño que el modelo principal y no funcionó: el AUC-ROC salió
inestable entre lotes (0,84 / 0,74 / 0,59 / 0,75) y el entrenamiento se frenaba a las pocas
iteraciones. Mi lectura es que el sub-reporte responde a decisiones administrativas puntuales
y no a un patrón que se pueda aprender del pasado.

## Limitaciones

- Descarté los 166.723 registros sin causa.
- Las agencias DOD (81 registros), BOR (14) y DOE (2) tienen muestras demasiado chicas. Sus
  porcentajes son reales pero no se pueden generalizar.
- El modelo se degrada un poco con el tiempo. El lote 3 (2008–2011) es el más difícil para
  los tres algoritmos, así que es algo del período y no del método.


## Bibliografía

Short, K. C. (2017). *Spatial wildfire occurrence data for the United States, 1992–2015
[FPA_FOD_20170508]* (4ª ed.). Forest Service Research Data Archive.
https://doi.org/10.2737/RDS-2013-0009.4

Pourmohamad, Y., Abatzoglou, J. T., Fleishman, E., Short, K. C., Shuman, J., AghaKouchak, A.,
Williamson, M., Seydi, S. T., & Sadegh, M. (2025). Inference of wildfire causes from their
physical, biological, social and management attributes. *Earth's Future*, 13(1).
https://doi.org/10.1029/2024EF005187

Coffield, S. R., Graff, C. A., Chen, Y., Smyth, P., Foufoula-Georgiou, E., & Randerson, J. T.
(2019). Machine learning to predict final fire size at the time of ignition. *International
Journal of Wildland Fire*, 28(11), 861–873. https://doi.org/10.1071/WF19023

Roberts, D. R., Bahn, V., Ciuti, S., Boyce, M. S., Elith, J., Guillera-Arroita, G.,
Hauenstein, S., Lahoz-Monfort, J. J., Schröder, B., Thuiller, W., Warton, D. I., Wintle,
B. A., Hartig, F., & Dormann, C. F. (2017). Cross-validation strategies for data with
temporal, spatial, hierarchical, or phylogenetic structure. *Ecography*, 40(8), 913–929.
https://doi.org/10.1111/ecog.02881

Prokhorenkova, L., Gusev, G., Vorobev, A., Dorogush, A. V., & Gulin, A. (2018). CatBoost:
unbiased boosting with categorical features. *Advances in Neural Information Processing
Systems*, 31. https://arxiv.org/abs/1706.09516

Breiman, L. (2001). Random Forests. *Machine Learning*, 45(1), 5–32.
https://doi.org/10.1023/A:1010933404324

Lundberg, S. M., & Lee, S. I. (2017). A unified approach to interpreting model predictions.
*Advances in Neural Information Processing Systems*, 30.
https://arxiv.org/abs/1705.07874
