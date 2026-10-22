# 02. Datos y preparación

Archery Performance Analytics es una aplicación centrada en el arquero: permite ver su posición en una competición, en qué dianas rinde mejor o peor respecto a su grupo y cómo evoluciona entre competiciones. Este documento describe los datos reales con los que ya funciona el flujo, de dónde salen, cómo se tratan y qué contiene la capa Gold.

## 1. Qué datos necesita el producto

**Qué representa una observación.** Hay dos niveles, ambos ya disponibles con datos reales:

- **Resultado de un arquero en una competición** (una fila por arquero, división, sexo y categoría de edad).
- **Resultado de un arquero en una diana** (una fila por arquero y diana, en las modalidades en las que se tira por dianas, como campo).

**Datos imprescindibles:** competición (nombre, modalidad, fecha, lugar), arquero (seudonimizado), división (R recurvo, C compuesto, B desnudo, L longbow, T tradicional), sexo, puntuación total y posición. **Deseables:** categoría de edad, club, puntuación por diana y por flecha (incluidos los fallos), los conteos de desempate propios de cada modalidad (10 y 9 en sala, X y 10 en aire libre, 6 y 5 en campo, 11 y 10 en 3D), distancia a la que el arquero queda del primero.

**Granularidad e histórico.** Se parte de dos competiciones reales de 2026, de modalidades distintas (campo y sala), y el flujo admite añadir más sin cambios de código. Para ver evolución harán falta varias tiradas del mismo arquero; la Liga de Sala de Cantabria (varias tiradas por temporada) es la fuente prevista.

## 2. Fuentes reales y viabilidad

| Fuente | Qué aporta | Cómo se accede | Estado |
|---|---|---|---|
| Ianseo, Campeonato de España de Campo 2026 (Hoyos del Espino, 6-7 junio 2026) | 157 hojas de puntuación, 3.672 filas de dianas, 11.016 flechas | PDF descargado manualmente de la página de la competición | Usada |
| Resultados de la 1ª Tirada de la Liga de Sala 26-27 (Cantabria, 11-10-2026) | 63 arqueros en 18 grupos y 10 clubes: posición, dos rondas de 18 m, total, 10 y 9 | PDF "Clasificación por Clase y División", obtenido manualmente | Usada |
| World Archery, Book 2 (versión 2026-01-27) | Reglas de rondas y puntuación; no son resultados | PDF oficial descargable (`--download-rulebook`) | Referencia |
| RFETA (live.rfeta.es, clasificaciones RAUS) | Clasificaciones y rankings nacionales y autonómicos | Páginas públicas | Localizada, pendiente de explorar |
| Datos propios del usuario | Material, sesiones de práctica, impactos | Entrada en la aplicación | Previsto |

**Cobertura y estabilidad.** Cada organizador nombra y maqueta sus PDF a su manera, por eso el tipo de documento se detecta por su contenido y no por el nombre del archivo. Cada formato nuevo (otra modalidad u otro organizador) necesita comprobarse con un ejemplo real antes de darlo por soportado.

**Alternativa si una fuente falla.** Documentos oficiales descargables de la federación (circulares, clasificaciones finales) o incorporación controlada de resultados manuales conservando su procedencia.

## 3. Uso, privacidad y datos personales

**Condiciones de uso.** En la revisión de octubre de 2026, los términos de uso de Ianseo prohíben la recogida sistemática o automática de datos y la republicación de su material sin consentimiento escrito, y permiten la consulta, impresión y almacenamiento para uso personal. En consecuencia, los PDF se descargan manualmente y se procesan en local; la descarga automática está implementada solo como esquema desactivado (`src/data/ianseo_download.py`) a la espera de autorización. Los datos originales no se suben al repositorio ni se muestran en abierto. (Esta revisión es una lectura de los términos publicados, no asesoramiento jurídico.)

**Datos personales y minimización.** Los resultados contienen nombres de deportistas y clubes. Medidas aplicadas:

- El nombre real no entra en Silver, Gold ni en la base de datos. Cada arquero recibe un `archer_id` (siglas del nombre y un número secuencial que no se reutiliza, por ejemplo `ABC-0001`), y la correspondencia nombre ↔ identificador se guarda solo en `data/private/archer_registry.xlsx`, fuera del repositorio y con permisos restringidos.
- El mismo nombre normalizado (sin tildes ni mayúsculas) recibe siempre el mismo identificador, también entre fuentes distintas.
- No se guardan el código de la hoja (su prefijo numérico no está verificado y podría identificar), números de licencia ni la lista de jueces de las cabeceras.
- Función de supresión: borra el nombre de un arquero y conserva el número para no reutilizarlo.
- La búsqueda de una ficha por nombre no muestra datos si hay varias coincidencias; se pide el nombre completo o el identificador.

**Limitaciones de la seudonimización.** No es anonimización: con el registro se puede reidentificar, por eso el registro es el único fichero sensible. Dos personas con exactamente el mismo nombre compartirían identificador (se corregiría a mano). Sin inicio de sesión, cualquiera que conozca un nombre completo podría pedir esa ficha: es una mejora prevista, no un control resuelto.

**Perfiles previstos.** Solo está implementado el de arquero. Entrenador (acceso con invitación del arquero, revocable), juez (libros de World Archery con versión y fecha) y organizador o club están descritos en `src/security/roles.py` como diseño. El acceso a la ficha de otra persona se plantea con el consentimiento del propio arquero, que puede retirarlo en cualquier momento; mientras el permiso siga vigente se renueva sin fricción (por ejemplo, al cambiar de móvil basta con volver a entrar en la cuenta) y si se retira, el acceso se cierra.

## 4. Del dato bruto al dato utilizable

Flujo reproducible con un solo comando, `python main.py`, que procesa todos los PDF de `data/raw/`:

| Capa | Contenido | Dónde |
|---|---|---|
| Bronze / Raw | PDF originales tal como se descargan (o PDF ficticios de `sample/` en la demostración), más el registro de procedencia (fuente, URL, documento, fecha de adquisición) | `data/raw/`, `data/reference/sources.xlsx` |
| Silver | Tablas limpias y tipadas: `competitions`, `results`, `scorecards`, `arrows` | `data/silver/silver_tables.xlsx` y base SQLite `data/archery.db` |
| Gold | Tablas de análisis creadas con SQL (funciones de ventana) sobre la base de datos | `data/gold/gold_tables.xlsx` y las mismas tablas en SQLite |

**Bronze/Raw.** El programa clasifica cada PDF por su contenido (hojas de puntuación, resultados u otro) y extrae el texto con `pypdf`. Las flechas se conservan como texto para no perder la "M" (fallo).

**Comprobaciones de calidad (sobre el dato bruto).** En las hojas se recalcula cada suma de diana, el acumulado, el número de 6 y de 5, y el total de cada hoja, y se comparan con lo impreso; además se exige que cada hoja tenga las dianas sin repetir, que todas las de una competición tengan el mismo número de dianas (varía según el campo y la asistencia: 24 es lo habitual en campo y 3D, pero hay competiciones de 20) y que no haya hojas duplicadas. En los resultados se comprueba que no queden filas sin interpretar, que el total sea la suma de las rondas, que el puesto corresponda a la puntuación (desempatando por 10 y 9) y que no haya arqueros repetidos en un grupo. Con los datos reales, todas las comprobaciones dan 0 problemas. Cada PDF se comprueba por separado: el que falla se excluye y se informa, sin impedir que se procesen los demás.

**Silver.** Transformaciones realizadas:

- Cabecera de región separada en código y nombre; "1B" o "24bis-C" separado en diana de salida y turno de tiro (A-D); la M (fallo) se guarda a la vez como etiqueta (`score_label` = M) y como número (`points` = 0); todos los cálculos usan la columna numérica y la etiqueta solo sirve para mostrar y para distinguir la X del 10 en aire libre.
- Flechas en formato largo (una fila por flecha: 11.016).
- Fechas a tipo fecha; división, sexo y categoría de edad normalizados; modalidad deducida del nombre de la competición ("Campo", "Sala", "Aire libre", "3D"), sin asignarla a mano; si no es inequívoca se marca como desconocida.
- Seudonimización del nombre y descarte de campos sin valor para el producto.
- Validaciones: sin nulos en flechas, diana de salida igual a la primera diana tirada, totales recalculados con las flechas iguales a los impresos, integridad referencial (claves primarias y foráneas) en SQLite.

**Gold.** Tres tablas, cada una con un significado claro:

| Tabla | Una fila es… | Columnas principales | Para qué sirve |
|---|---|---|---|
| `gold_scorecard_performance` | un arquero en una competición por dianas (153 filas) | `archer_id`, `division_code`, `sex_code`, `total_score`, `avg_points_per_arrow`, `miss_pct`, `total_n6`, `total_n5`, `group_rank`, `group_size` | Posición y rendimiento del arquero frente a su grupo |
| `gold_target_performance` | un arquero en una diana (3.672 filas) | `scorecard_id` (enlaza con la hoja y su `archer_id`), `target`, `row_order`, `target_points`, `misses`, `group_avg_points`, `diff_vs_group` | Detectar en qué dianas rinde mejor o peor que su grupo; base para la fase de Machine Learning |
| `gold_results` | un arquero en un resultado oficial (63 filas) | `archer_id`, `competition`, `start_date`, `division_code`, `sex_code`, `age_class`, `club_name`, `position`, `total_score`, `n10`, `n9`, `group_size`, `gap_to_first` | Posición oficial y evolución entre tiradas de una liga |

Las comparaciones se hacen siempre dentro de la misma competición, división y sexo (y categoría de edad en los resultados oficiales). El puesto calculado en `gold_scorecard_performance` **no es una clasificación oficial**: se calcula con la puntuación de las hojas disponibles.

**Muestra de datos.** Los datos reales (PDF de Ianseo y sus derivados) no se suben al repositorio. En su lugar, `sample/` contiene dos PDF **ficticios** con el mismo formato que los reales (dos competiciones de campo con 23 hojas de puntuación cada una, una de 24 dianas y otra de 20, y una liga de sala con 22 arqueros, con los mismos arqueros ficticios repartidos entre ellas), con nombres, clubes y competiciones inventados. Los genera `tools/make_demo_pdfs.py`, y el flujo completo se ejecuta sobre ellos con `python main.py --demo`, que copia esos PDF a `data_demo/raw/` y recorre el mismo flujo que con datos reales (Raw → Silver → Gold) dentro de `data_demo/`, sin tocar los datos reales.

**Cómo reproducirlo.**

```
pip install -r requirements.txt
# copiar los PDF a data/raw/
python main.py                          # con los PDF reales
python main.py --demo                   # con los PDF ficticios de sample/
python main.py --demo --archer-id ADA-0001   # ficha de un arquero de la demostración
```

## 5. Riesgos y limitaciones

- **Datos que faltan:** la categoría de edad no figura en las hojas de campo (sí en los resultados de sala). Puede deducirse de otras competiciones del mismo arquero, pero no se da por fija: los menores pueden subir de categoría hasta senior pero no bajar, los veteranos pueden pasar a senior y los senior no cambian; por eso se tomaría de la misma temporada y se marcaría como deducida. Ninguna fuente aporta el tipo de diana, la distancia ni la posición del impacto; estos dos últimos solo se podrán obtener de datos introducidos por el propio arquero.
- **Cobertura y sesgo:** hoy hay dos competiciones; la comparación solo es válida dentro de un mismo grupo y, con 4 a 21 arqueros por grupo, los porcentajes por diana son poco robustos. En los resultados, la clase sin indicación en el título se registra como "General" sin asumir que equivalga a senior.
- **Identidad:** la unión entre fuentes se hace por nombre normalizado; puede haber homónimos y variaciones de escritura entre documentos. Si en el futuro los arqueros suben sus propios resultados, habrá que detectar la misma competición subida varias veces aunque el archivo tenga otro nombre (por nombre, fecha y puntuaciones).
- **Formatos de los PDF:** el lector depende del texto que extrae `pypdf` (se comprobó que cambia entre versiones) y cada formato nuevo exige una muestra real y sus comprobaciones. Un PDF que no se reconoce o no supera las comprobaciones se excluye y se avisa, y el resto de los archivos se procesan igualmente.
- **Si una fuente principal deja de estar disponible:** se recurriría a documentos oficiales descargables de la federación o a la incorporación controlada de resultados manuales con su procedencia registrada.
