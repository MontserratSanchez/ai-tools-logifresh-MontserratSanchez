# Reporte de Validación · Tablero LogiFresh México

**Fecha de validación:** 3 de septiembre de 2026
**Archivo validado:** `index.html`
**Fuente:** `Datos_sinteticos_LogiFresh_dashboard.xlsx`, hoja `Datos` (240 registros)
**Método:** batería automatizada de 39 pruebas ejecutada con Playwright sobre Chromium headless (`test.py`), más inspección visual de capturas a 390 px, 768 px y 1440 px.
**Resultado global:** **39 / 39 pruebas aprobadas.**

---

## 1. Perfil de calidad de datos

| Verificación | Resultado |
|---|---|
| Filas | 240 |
| Columnas | 18 |
| Valores faltantes | **0** en las 18 columnas |
| `id_embarque` duplicados | **0** |
| Rango de fechas | 2026-04-01 a 2026-06-28, sin huecos mensuales |
| Cardinalidad de categóricas | 38 unidades · 6 orígenes · 8 destinos · 5 productos · 4 transportistas · 6 tipos de incidente · 3 tipos de ruta |
| Rangos numéricos | `retraso_min` 0–79 · `temperatura_max_c` 3.2–12.0 · `reclamacion_mxn` 0–145,000 · `satisfaccion_1_10` 8–9 · `ocupacion_unidad` 0.82–0.98 |
| Valores fuera de dominio | Ninguno |

**Anomalías estructurales detectadas** (no son errores del archivo, son propiedades del generador sintético):

| # | Anomalía | Evidencia | Consecuencia para el tablero |
|---|---|---|---|
| A | Diseño perfectamente balanceado: los 4 transportistas tienen exactamente 60 embarques y **76.7% de SLA cada uno**; los 5 productos tienen 48 cada uno | Agrupación de `sla_entrega` por `transportista` y `producto` | Los cortes por segmento no discriminan; se documenta explícitamente en el pie del tablero |
| B | Las 9 excursiones > 8 °C son los IDs consecutivos `LF-0027`–`LF-0035` | Filtro `temperatura_max_c > 8` ordenado por `id_embarque` | Se advierte que es un bloque generado, no un patrón operativo |
| C | `satisfaccion_1_10` toma solo 3 valores (8, 8.5, 9) y su media es 8.5 en todos los cortes | `describe()` y agrupación por `sla_entrega` | El KPI se etiqueta como «percepción declarada, no indicador duro» |
| D | `retraso_min` es 0 en **todos** los «Cumple» y ≥ 25 en **todos** los «No cumple» | `min/max` de `retraso_min` por `sla_entrega` | Confirma que SLA y retraso son la misma variable expresada dos veces; se reporta como definición, no como hallazgo |

---

## 2. Reconciliación de las métricas de control

Comparación entre la hoja `Diccionario_y_control` del archivo fuente y el cálculo realizado sobre la hoja `Datos`.

| Métrica de control | Valor esperado (hoja de control) | Valor calculado (hoja Datos) | Estado |
|---|---|---|---|
| Embarques | 240 | 240 | ✅ Cuadra |
| SLA | 0.7666666667 → 76.7% | 76.7% | ✅ Cuadra |
| Retraso promedio (solo tardíos) | 51.8 min | 51.8 min | ✅ Cuadra |
| Incidentes | 52 | 52 | ✅ Cuadra |
| Excursiones > 8 °C | 9 | 9 | ✅ Cuadra |
| Reclamaciones MXN | 882,649 | **882,549** | ⚠️ **Discrepancia de $100** |
| Satisfacción promedio | 8.5 | 8.5 | ✅ Cuadra |

**Tratamiento de la discrepancia.** Se recalculó `sum(reclamacion_mxn)` por tres vías independientes (pandas, suma de los 15 registros con reclamación > 0, y suma en JavaScript dentro del propio tablero); las tres arrojan **882,549**. Seis de las siete métricas de control cuadran exactamente, lo que sugiere un error de captura de $100 en la hoja de control y no en los datos.

**Decisión (autorizada por la usuaria el 3-sep-2026):** el tablero muestra el valor **calculado desde los datos ($882,549)**, no el valor de control. Forzar 882,649 habría roto la trazabilidad dato → cálculo → pantalla. La diferencia queda documentada en el pie del tablero y en el README.

---

## 3. Preguntas analíticas formuladas

| # | Pregunta | Componente que la responde |
|---|---|---|
| P1 | ¿Cuánto se aleja el SLA de la meta de 90% y **cuándo** se rompe? | KPI de SLA con barra de meta + gráfica «SLA por mes» |
| P2 | ¿Cómo se distribuye la magnitud del retraso? | Histograma «Distribución del retraso (solo tardíos)» |
| P3 | ¿La cadena de frío se asocia a incumplimiento o reclamación? | Scatter de temperatura con umbral de 8 °C + KPI de excursiones |
| P4 | ¿Dónde se concentra el impacto económico? | Gráfica de concentración (Pareto) + tabla ejecutiva |

---

## 4. Decisiones de diseño registradas

| # | Decisión | Justificación |
|---|---|---|
| D1 | Archivo único autocontenido, sin CDN | Una sola petición de red, 61 KB, cero rutas rotas al publicar en Pages |
| D2 | SVG generado a mano en lugar de librería de gráficas | Cinco gráficas simples no justifican 150–300 KB ni una dependencia externa |
| D3 | Filtro por **mes** en lugar de rango de fechas | El quiebre de servicio es mensual; un selector de rango añadía complejidad sin valor decisional |
| D4 | El panel Hechos/Hipótesis **no** responde a los filtros | Son conclusiones sobre el universo completo; recalcularlas por filtro generaría afirmaciones sin sustento |
| D5 | Barras verdes solo cuando el valor ≥ 90% | El color codifica estado respecto a la meta, no categoría |
| D6 | Se descartaron `unidad`, `horas_transito`, `distancia_km`, `ocupacion_unidad` | No aportan a P1–P4; visualizarlos habría sido decoración |
| D7 | Se descartó un mapa de rutas | Con 6 orígenes × 8 destinos y SLA idéntico entre ellos, un mapa no soporta ninguna decisión |
| D8 | Tabla ejecutiva por transportista ordenada por **reclamaciones**, no por SLA | El SLA es idéntico entre transportistas; el monto sí varía y es lo accionable |

---

## 5. Supuestos declarados

| # | Supuesto | Base |
|---|---|---|
| S1 | La meta de SLA es 90% | Indicada en la guía de la actividad; no aparece en el archivo de datos |
| S2 | «Retraso promedio» excluye los embarques con retraso 0 | Definición explícita en la hoja `Diccionario_y_control` |
| S3 | El umbral de cadena de frío es 8 °C | Derivado del nombre del campo `excursion_temp_mayor_8c` |
| S4 | Los meses se derivan de `fecha_salida`, no de una fecha de entrega | El dataset no contiene fecha de entrega |
| S5 | Un embarque = una fila | `id_embarque` es único en las 240 filas |

---

## 6. Tabla de pruebas de aceptación

Leyenda de bloques: **A** cálculos base · **F** filtros · **U** actualización de componentes · **E** estado sin datos · **R** restablecer · **V** responsividad · **X** accesibilidad · **P** carga y recursos.

|ID|Prueba|Esperado|Obtenido|Estado|
|---|---|---|---|---|
|A1|Total de embarques sin filtros|240|240|✅ Pasa|
|A2|SLA global|76.7%|76.7%|✅ Pasa|
|A3|Retraso promedio de tardíos|51.8 min|51.8 min|✅ Pasa|
|A4|Incidentes|52|52|✅ Pasa|
|A5|Excursiones >8 °C|9|9|✅ Pasa|
|A6|Reclamaciones MXN (valor de control 882,649)|882,549 (dato real)|882,549|✅ Pasa|
|A7|Satisfacción promedio|8.5/10|8.5|✅ Pasa|
|A8|KPI SLA visible en el DOM|contiene '76.7%'|sí|✅ Pasa|
|A9|Meta 90% etiquetada explícitamente|contiene 'meta institucional: 90%'|sí|✅ Pasa|
|F1|Filtro individual · Transportista = Norte|60 embarques|60|✅ Pasa|
|F2|KPIs recalculan con filtro individual|SLA 76.7%|76.7%|✅ Pasa|
|F3|Combinado · Norte + Junio 2026 (conteo)|20|20|✅ Pasa|
|F4|Combinado · Norte + Junio 2026 (SLA)|30.0%|30.0%|✅ Pasa|
|F5|Combinado triple · Berries + Prioritaria + No cumple|4|4|✅ Pasa|
|F6|SLA = 0% cuando se filtra solo 'No cumple'|0.0%|0.0%|✅ Pasa|
|U1|Tabla se actualiza al cambiar de mes|contenido distinto|distinto|✅ Pasa|
|U2|Gráfica se actualiza al cambiar de mes|contenido distinto|distinto|✅ Pasa|
|U3|Chips reflejan el filtro activo|muestra 'Abril 2026'|sí|✅ Pasa|
|E1|Caso sin resultados · Abril + No cumple|0 embarques|0|✅ Pasa|
|E2|Estado vacío con mensaje accionable|mensaje 'Sin datos'|sí|✅ Pasa|
|E3|Tabla en estado vacío no rompe|mensaje 'Sin registros'|sí|✅ Pasa|
|E4|Gráficas en estado vacío no rompen|mensaje de vacío|sí|✅ Pasa|
|R1|Restablecer devuelve el universo completo|240|240|✅ Pasa|
|R2|Restablecer limpia todos los selects|todos vacíos|sí|✅ Pasa|
|R3|Restablecer restaura los KPIs originales|76.7%|76.7%|✅ Pasa|
|V-390|Móvil 390px · sin desbordamiento horizontal|≤ 1 px|0 px|✅ Pasa|
|V-768|Tablet 768px · sin desbordamiento horizontal|≤ 1 px|0 px|✅ Pasa|
|V-1440|Escritorio 1440px · sin desbordamiento horizontal|≤ 1 px|0 px|✅ Pasa|
|X1|Todos los selects tienen <label for>|true|True|✅ Pasa|
|X2|Un solo <h1> y lang declarado|1 / es|1 / es|✅ Pasa|
|X3|Enlace 'Saltar al contenido'|presente|sí|✅ Pasa|
|X4|Gráficas SVG con role e aria-label|true|True|✅ Pasa|
|X5|Región aria-live para cambios de filtro|presente|sí|✅ Pasa|
|X6|Encabezados de tabla con scope|> 0|12|✅ Pasa|
|P1|Cero dependencias externas (autocontenido)|0 recursos externos|0|✅ Pasa|
|P2|Número total de peticiones de red|1 (solo el HTML)|1|✅ Pasa|
|P3|Sin errores de consola ni excepciones JS|0|0|✅ Pasa|
|P4|Peso de la página|< 150 KB|61 KB|✅ Pasa|
|P5|Título de página definido|no vacío|LogiFresh México · Tablero de Desempeño Opera|✅ Pasa|


### Notas sobre pruebas específicas

- **A6** es la única prueba cuyo valor esperado se apartó del criterio original de aceptación ($882,649). Se validó contra el **dato real** ($882,549) por la decisión documentada en la sección 2.
- **F3/F4** (Norte + Junio 2026) verifican dos filtros combinados: 20 embarques con 30.0% de SLA, coincidente con el cálculo independiente en pandas.
- **F5** verifica tres filtros combinados simultáneos (Berries + Prioritaria + No cumple → 4 embarques).
- **E1** verifica el caso sin resultados con una combinación lógicamente vacía (Abril + No cumple → 0 embarques, porque abril cerró en 100% de cumplimiento).
- **U1/U2** confirman que un cambio de filtro modifica el DOM de la tabla **y** de las gráficas, no solo el de los KPIs.
- **P1/P2** confirman que la página se sirve con **una sola petición** y sin ningún recurso externo, lo que elimina el riesgo de rutas rotas al publicar.

---

## 7. Fallas detectadas y correcciones aplicadas

| # | Falla | Detección | Corrección |
|---|---|---|---|
| C1 | En el Pareto, la etiqueta «100%» del eje acumulado se encimaba con el rótulo «acum.» | Inspección visual de la captura a 390 px | Se separaron verticalmente las etiquetas y se renombró el rótulo a «% acum.» |
| C2 | El intento inicial de instalar Playwright vía npm fue rechazado por la política de seguridad del registro | Salida del comando `npm i playwright-core` (403) | Se instaló el paquete de Python y se apuntó al Chromium preinstalado del entorno |
| C3 | El valor de control de reclamaciones no cuadraba | Reconciliación de la sección 2 | Se escaló a la usuaria antes de construir; se documentó y se usó el dato real |

No quedan fallas abiertas.

---

## 8. Tres hallazgos priorizados

### Hallazgo 1 — El incumplimiento de SLA no es crónico: es un quiebre puntual en junio

**Dato.** Los **56 incumplimientos ocurren todos en junio de 2026**. Abril cierra en 100% (80/80), mayo en 100% (80/80) y junio en **30%** (24/80).
**Evidencia.** Cruce de `sla_entrega` con `fecha_salida` agrupada por mes.
**Por qué importa.** Un SLA global de 76.7% leído como promedio anual sugeriría un problema estructural de capacidad. La descomposición temporal muestra lo contrario: dos meses perfectos y un colapso en el tercero.
**Acción correctiva.** Auditar con registros primarios qué cambió operativamente el 1 de junio, antes de invertir en capacidad.
**Indicador de seguimiento.** % de SLA **semanal** (no mensual), meta ≥ 90%.

### Hallazgo 2 — El 6.3% de los embarques concentra el 100% del impacto económico

**Dato.** Solo **15 de 240 embarques** registran reclamación; los **5 mayores suman $592,950, el 67.2%** del total de $882,549. La mayor individual es $145,000 (`LF-0003`).
**Evidencia.** Campo `reclamacion_mxn` filtrado a `> 0`, ordenado descendente.
**Por qué importa.** Un control uniforme sobre 240 embarques es caro e ineficiente frente a un control reforzado sobre las decenas de embarques de alto valor declarado.
**Acción correctiva.** Definir un umbral de «embarque de alto valor» y aplicarle checklist reforzado y confirmación de ventana de entrega.
**Indicador de seguimiento.** Monto de reclamación semanal y % de embarques de alto valor con checklist completo.

### Hallazgo 3 — Los indicadores de incidente y de satisfacción no tienen poder discriminante

**Dato.** Los **52 embarques con incidente registrado cumplen SLA al 100%**, mientras que los 188 «Sin incidente» cumplen apenas **70.2%**. Las **9 excursiones > 8 °C cumplieron SLA** y 8 de ellas no generaron reclamación. La satisfacción vale **8.5 tanto en «Cumple» como en «No cumple»** (correlación retraso–satisfacción ≈ 0.00).
**Evidencia.** Tabla cruzada `tipo_incidente` × `sla_entrega`; cruce `excursion_temp_mayor_8c` × `sla_entrega`; media de `satisfaccion_1_10` por `sla_entrega`.
**Por qué importa.** Estos tres campos, tal como están capturados, **no sirven hoy para anticipar ni explicar un incumplimiento**. Usarlos como alerta temprana produciría falsos negativos sistemáticos.
**Acción correctiva.** Revisar el protocolo de captura de incidentes antes de construir cualquier alerta sobre este campo.
**Indicador de seguimiento.** % de incumplimientos con incidente documentado (hoy: 0%).

---

## 9. Dos hipótesis por validar

**H1 — Cambio de régimen en junio, no degradación gradual.**
El salto de 100% a 30% de un mes al siguiente es consistente con un evento discreto: un cambio de proceso, de proveedor, de sistema de captura, o un error en la generación del dato.
*Cómo validarla:* obtener el SLA con granularidad diaria de mayo y junio y localizar la fecha exacta del quiebre; cruzarla con la bitácora de cambios operativos.
*No demostrado:* el dataset no contiene variables de proceso, personal ni clima.

**H2 — El campo de incidente refleja atención, no ocurrencia.**
Que el 100% de los embarques con incidente cumpla SLA sugiere un sesgo de registro: el incidente se documenta cuando alguien lo detecta **y** lo resuelve a tiempo, mientras que los incumplimientos silenciosos quedan como «Sin incidente».
*Cómo validarla:* auditar una muestra de los 56 incumplimientos contra evidencia primaria (bitácora de conductor, GPS, comunicación con el cliente) y medir cuántos tuvieron un incidente real no capturado.
*No demostrado:* también es compatible con generación sintética independiente de ambos campos.

---

## 10. Recomendación: piloto de 30 días

**Nombre.** Auditoría del quiebre de junio + control diferenciado de embarques de alto valor.

**Alcance.** Dos frentes en paralelo, sin cambios de flota ni de personal.

1. **Frente diagnóstico (semanas 1–2).** Reconstruir con registros primarios qué cambió el 1 de junio. Auditar una muestra aleatoria de 20 de los 56 incumplimientos contra bitácora, GPS y comunicación con el cliente, registrando si hubo incidente real no capturado.
2. **Frente de contención (semanas 1–4).** Aplicar checklist reforzado, verificación de temperatura al cargar y confirmación de ventana de entrega con el cliente a todo embarque que supere el umbral de alto valor declarado.

**Indicadores de seguimiento semanal.**

| Indicador | Fórmula | Meta a 30 días |
|---|---|---|
| SLA semanal | Cumple / total de la semana | ≥ 90% |
| Incumplimientos | Conteo semanal | Tendencia decreciente |
| Monto de reclamación | Suma semanal en MXN | Cero eventos > $100,000 |
| Cobertura de checklist | Embarques de alto valor con checklist completo / total de alto valor | 100% |
| Trazabilidad de incidentes | Incumplimientos con incidente documentado / incumplimientos | ≥ 80% |

**Criterio de éxito.** SLA semanal ≥ 90% en las dos últimas semanas del piloto y cero reclamaciones superiores a $100,000 MXN.
**Criterio de abandono.** Si al día 15 la auditoría muestra que el quiebre de junio proviene de un error de captura y no de la operación, se cancela el frente de contención y el esfuerzo se redirige a corregir el sistema de registro.
**Costo estimado.** Horas de analista y de supervisión operativa; no requiere inversión en activos.

---

## 11. Riesgos y datos faltantes

| # | Riesgo / dato faltante | Impacto | Mitigación |
|---|---|---|---|
| R1 | **Los datos son sintéticos.** Ninguna conclusión es trasladable a una operación real | Alto | Todo el tablero y esta documentación lo declaran de forma visible |
| R2 | **Falta la causa raíz del incidente.** El campo solo registra una categoría, no qué ocurrió | Alto | Solicitar el campo de causa raíz o texto libre de la bitácora |
| R3 | **No hay fecha ni hora de entrega real**, solo `fecha_salida` | Medio | Sin ella no se puede medir tiempo de ciclo ni cumplimiento de ventana |
| R4 | **No hay identificador de cliente ni política de SLA por cliente** | Medio | El SLA se trata como umbral único, lo cual puede ser incorrecto |
| R5 | **La discrepancia de $100 en el valor de control** sugiere que el archivo fuente no está bajo control de versiones | Bajo | Documentada; se recomienda versionar el archivo fuente |
| R6 | **Sin periodo comparable anterior** (solo 3 meses) | Medio | No se puede distinguir estacionalidad de deterioro real |
| R7 | **Riesgo de sobreinterpretación.** Un lector puede leer el panel de hipótesis como conclusión | Medio | El panel está rotulado por columnas y cada hipótesis lleva su cláusula «No demostrado» |
| R8 | La satisfacción tiene rango casi nulo (8–9), lo que impide segmentar por experiencia | Bajo | El KPI se etiqueta como percepción declarada |

---

## 12. Trazabilidad de la ejecución

| Paso | Realizado | Evidencia |
|---|---|---|
| 1. Inspección de carpeta y diccionario | ✅ | Lectura de la carpeta del proyecto, de la hoja `Diccionario_y_control` y de `diccionario_datos_actividades_1_2.csv` |
| 2. Perfil de calidad y reconciliación | ✅ | Secciones 1 y 2 de este reporte |
| 3. Formulación de preguntas analíticas | ✅ | Sección 3 |
| 4. Diseño previo a la programación | ✅ | Secciones 4 (arquitectura) y 5 (accesibilidad); decisiones D1–D8 |
| 5. Construcción | ✅ | `index.html`, 61 KB, una sola petición |
| 6. Pruebas | ✅ | 39 pruebas automatizadas, sección 6 |
| 7. Corrección de fallas | ✅ | Sección 7, correcciones C1–C3 |
| 8. Publicación | ⏳ | Pendiente de credenciales de GitHub |
| 9. Verificación de la URL publicada | ⏳ | Pendiente; la batería se reejecutará contra la URL pública |
| 10. README y reporte de validación | ✅ | `README.md` y este documento |

---

*Documento generado el 3 de septiembre de 2026. Datos sintéticos, sin valor operativo.*
