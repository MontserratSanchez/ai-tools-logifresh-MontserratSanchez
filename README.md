PRUEBA-123# LogiFresh México · Tablero de Desempeño Operativo

Dashboard estático e interactivo sobre desempeño operativo y cadena de frío, construido a partir del archivo `Datos_sinteticos_LogiFresh_dashboard.xlsx` (240 embarques, abril–junio 2026).

**Sitio publicado:** _(ver sección Publicación)_

> ⚠️ **Los datos son sintéticos y didácticos.** No representan operaciones reales de ninguna empresa y no deben usarse para decisiones de negocio. No contienen datos personales ni secretos.

---

## 1. Propósito y alcance

**Qué resuelve.** Responde cuatro preguntas analíticas con una sola vista filtrable:

1. ¿Cuánto se aleja el cumplimiento de SLA de la meta institucional de **90%** y **cuándo** se rompe?
2. ¿Cómo se distribuye el retraso entre los embarques tardíos?
3. ¿La cadena de frío (excursiones > 8 °C) se asocia con incumplimiento o con reclamaciones?
4. ¿Dónde se concentra el impacto económico (reclamaciones en MXN)?

**Qué NO cubre.**

- No identifica **causas**. El dataset no contiene variables de proceso, personal, clima, tráfico real ni causa raíz del incidente.
- No permite comparar contra un periodo anterior ni contra un *benchmark* de industria.
- No modela pronósticos ni escenarios.
- No es una fuente operativa: es un ejercicio con datos generados artificialmente.

---

## 2. Entradas

| Elemento | Detalle |
|---|---|
| Archivo fuente | `Datos_sinteticos_LogiFresh_dashboard.xlsx` |
| Hoja de datos | `Datos` — 240 filas × 18 columnas |
| Hoja de control | `Diccionario_y_control` — definiciones y 7 métricas de comprobación |
| Periodo | 1 de abril – 28 de junio de 2026 |
| Moneda | Pesos mexicanos (MXN) |
| Actualización | 3 de septiembre de 2026 |

### Campos usados

| Campo | Tipo | Uso en el tablero |
|---|---|---|
| `id_embarque` | Texto | Identificador único (240 valores, sin duplicados) |
| `fecha_salida` | Fecha | Filtro por mes y eje temporal del scatter |
| `sla_entrega` | Categórica (Cumple / No cumple) | KPI de SLA, filtro, color de barras |
| `retraso_min` | Numérica (min) | KPI de retraso promedio e histograma |
| `tipo_incidente` | Categórica (6 valores) | KPI de incidentes, gráfica de frecuencia, filtro |
| `excursion_temp_mayor_8c` | Categórica (Sí / No) | KPI de excursiones |
| `temperatura_max_c` | Numérica (°C) | Scatter de cadena de frío contra umbral de 8 °C |
| `reclamacion_mxn` | Numérica (MXN) | KPI y gráfica de concentración (Pareto) |
| `satisfaccion_1_10` | Numérica | KPI de percepción |
| `transportista`, `tipo_ruta`, `producto`, `origen`, `destino` | Categóricas | Filtros y tabla ejecutiva |

Los campos `unidad`, `horas_transito`, `distancia_km` y `ocupacion_unidad` se perfilaron pero **no se visualizan**: no aportan a ninguna de las cuatro preguntas analíticas y habrían añadido componentes decorativos.

---

## 3. Definiciones de las métricas

| Métrica | Fórmula exacta | Unidad |
|---|---|---|
| Embarques | `count(filas)` | registros |
| Cumplimiento de SLA | `count(sla_entrega = "Cumple") / count(filas) × 100` | % |
| Meta de SLA | Valor institucional fijo | **90%** |
| Retraso promedio | `mean(retraso_min)` **solo** donde `retraso_min > 0` | minutos |
| Incidentes | `count(tipo_incidente ≠ "Sin incidente")` | registros |
| Excursiones > 8 °C | `count(excursion_temp_mayor_8c = "Sí")` | eventos |
| Reclamaciones | `sum(reclamacion_mxn)` | MXN |
| Satisfacción | `mean(satisfaccion_1_10)` | escala 1–10 |

Todas las métricas se recalculan sobre el subconjunto filtrado. Los ocho filtros se combinan con lógica **Y** (AND).

---

## 4. Arquitectura

Sitio estático de **un solo archivo**, sin dependencias externas ni build step.

```
index.html            Marcado, estilos y lógica; los 240 registros van embebidos como JSON
README.md             Este documento
REPORTE_VALIDACION.md Reporte de pruebas, hallazgos y trazabilidad
.nojekyll             Desactiva el procesamiento Jekyll de GitHub Pages
```

**Decisiones de arquitectura y su justificación:**

| Decisión | Razón |
|---|---|
| Un solo `index.html` autocontenido | 61 KB, **una sola petición de red**, carga instantánea y cero riesgo de rutas rotas en GitHub Pages |
| Sin librerías de gráficas (Chart.js, D3, etc.) | Cinco gráficas simples no justifican 150–300 KB de dependencias externas ni una petición a un CDN |
| Gráficas en **SVG generado a mano** | Escala sin pixelarse, hereda los colores del tema, y cada `<svg>` lleva `role="img"` y `aria-label` |
| Datos embebidos como JSON, no `fetch()` de CSV | Evita CORS, latencia y el problema de que la página funcione en local pero no publicada |
| Sin `localStorage` ni cookies | No hay nada que persistir; evita avisos de privacidad |
| Panel Hechos/Hipótesis **no** se filtra | Son conclusiones sobre el dataset completo; recalcularlas por filtro produciría afirmaciones sin sustento |

**Paleta.** Tierra sobre blanco: terracota `#B4552D` (acento y valores fuera de meta), terracota oscuro `#8A3F20`, arena `#F5EDE6` (fondos secundarios), tinta `#33251E` (texto), olivo `#4F7A54` (dentro de meta), blanco `#FFFFFF` (lienzo). El color codifica **estado respecto a la meta**, nunca decoración: verde = cumple 90%, terracota = no cumple.

---

## 5. Accesibilidad y responsividad

- HTML semántico: un solo `<h1>`, `lang="es"`, secciones y encabezados jerárquicos.
- Enlace «Saltar al contenido» y anillo de foco visible en todo control interactivo.
- Cada `<select>` tiene su `<label for>` asociado; la tabla usa `<th scope>` y `<caption>`.
- Región `aria-live="polite"` que anuncia el conteo de registros al cambiar un filtro.
- Texto principal sobre blanco con contraste ≥ 7:1; blanco sobre terracota `#B4552D` alcanza 5.1:1 (AA para texto normal).
- Verificado sin desbordamiento horizontal a 390 px, 768 px y 1440 px. La tabla hace scroll dentro de su propio contenedor.
- La información nunca se transmite solo por color: cada barra lleva su valor numérico y cada KPI su etiqueta de estado.

---

## 6. Cómo ejecutarlo en local

No requiere servidor ni instalación:

```bash
open index.html          # macOS
```

Para reproducir la batería de pruebas automatizadas se necesita Python con Playwright:

```bash
pip install playwright && playwright install chromium
python3 test.py
```

---

## 7. Publicación

El sitio se publica con **GitHub Pages** desde la rama `main`, carpeta raíz (`/`). El archivo `.nojekyll` evita que Jekyll ignore archivos o rutas.

---

## 8. Limitaciones conocidas

1. **Dataset balanceado por diseño.** Transportista, producto, tipo de ruta, origen y destino tienen un SLA prácticamente idéntico (~76.7%). Los cortes por segmento **no discriminan desempeño** y sirven solo para demostrar la mecánica de filtrado.
2. **Bloque artificial de temperatura.** Las 9 excursiones > 8 °C corresponden a IDs consecutivos `LF-0027`–`LF-0035`, lo que apunta a generación sintética por bloque y no a un patrón operativo.
3. **Discrepancia en un valor de control.** La suma real de `reclamacion_mxn` es **$882,549**; la hoja `Diccionario_y_control` indica **$882,649** ($100 de diferencia). El tablero muestra el valor calculado desde los datos. Ver `REPORTE_VALIDACION.md`.
4. **Satisfacción sin poder discriminante.** Vale 8.5 tanto en «Cumple» como en «No cumple» (correlación retraso–satisfacción ≈ 0.00), por lo que no sirve como indicador de resultado en este dataset.
5. **Sin causalidad.** Ninguna cifra del tablero identifica una causa. Toda relación mostrada es asociación.

---

## 9. Privacidad y seguridad

Sin datos personales, credenciales, tokens ni secretos. El repositorio contiene únicamente HTML estático y documentación en Markdown.

---

**Autoría:** Montserrat Sánchez · Actividad Work/Cowork — Dashboard en GitHub Pages
**Licencia de uso:** material académico. Datos sintéticos, sin valor operativo.
