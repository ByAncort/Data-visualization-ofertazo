# StreamView Analytics · Dashboard interactivo (EV2)

Solución de Visual Analytics en Power BI para el catálogo de StreamView: 31.991 títulos (16.000 películas y 15.991 series), con estrenos entre 2010 y 2025.

## 1. Archivos

| Archivo | Uso |
|---|---|
| `../StreamView_Dashboard_EV2.pbix` | Dashboard para entregar y presentar (incluye los datos). |
| `StreamView.pbip` + carpetas `.Report` / `.SemanticModel` | El mismo informe en formato proyecto (texto), útil para versionar cambios. |
| `../Datos/*.csv` | Fuentes oficiales, sin modificar (extraídas de `Data_Movies.zip`). |
| `capturas/` | Imagen de cada página. |

**Si mueves la carpeta EV2:** en Power BI ve a *Transformar datos → Editar parámetros*, actualiza `RutaDatos` con la nueva ruta de la carpeta `Datos\` y presiona *Actualizar*.

## 2. Modelo de datos

Modelo en estrella con una tabla de hechos y tablas puente para los campos multivalor:

```
              Calendario (Fecha, Año, Mes, Trimestre)
                    │ 1:*
Genero  ──*:1──  Contenido  ──1:*──  Pais
(Género)         (1 fila = 1 título)   (País)
Reparto ──*:1──┘         └──1:*── Direccion
(Actor)                             (Director)
```

- **Contenido**: une películas y series en una sola tabla (requerimiento funcional 1). La clave `ContenidoID` lleva el prefijo `P-` o `S-` porque el `show_id` de TMDB puede repetirse entre películas y series.
- **Tablas puente** (Genero, Pais, Reparto, Direccion): `genres`, `country`, `cast` y `director` traen varios valores separados por comas. Separarlos permite contar cada género o país sin duplicar títulos. Las relaciones son bidireccionales para que un filtro de género o país filtre todo el informe.
- **Calendario**: tabla de fechas continua y marcada como tabla de fechas.
- **Medidas**: 48 medidas DAX organizadas en carpetas (Catálogo, Popularidad, Valoración, Finanzas, Textos, Títulos dinámicos).

## 3. Calidad de datos y transformaciones (Power Query)

Todas las transformaciones están en Power Query y comentadas paso a paso. Los CSV originales no se modifican.

| Hallazgo | Decisión |
|---|---|
| `rating` es idéntica a `vote_average` en el 100% de los registros (no es la clasificación por edad que indica el diccionario). | Se descarta. **No es posible construir los KPIs de clasificación etaria**, y esa limitación se declara. |
| `duration` está 100% vacía en películas, y en series siempre dice "1 Seasons". | Se descarta por no aportar información. |
| 9 `show_id` duplicados en series (mismo título, distinta popularidad). | Se conserva la primera fila. |
| `budget` / `revenue` = 0 significan dato faltante. Hay presupuestos atípicos (p. ej. US$1). | Se convierten a vacío. Los KPIs financieros usan solo películas con presupuesto e ingresos ≥ US$10.000: 3.492 de 16.000 (22%). |
| `vote_average` = 0 con 0 votos significa "sin evaluar", no una nota de 0. | La calificación queda vacía cuando no hay votos. |
| Notas con pocos votos son poco confiables. | Los rankings de "mejor evaluados" exigen ≥100 votos (columna `Evaluación Confiable`). |
| `date_added` coincide con el año de estreno y el extracto trae exactamente 1.000 películas y 1.000 series por año. | Se declara que el **volumen anual no muestra tendencia real** (es un extracto estratificado). Las tendencias se analizan con popularidad, calificación y composición. |
| Géneros, países e idiomas vienen en inglés o como códigos ISO. | Se traducen al español con diccionarios en Power Query. "Unknown" se trata como dato faltante. |
| TMDB usa taxonomías de género distintas para cine y TV (p. ej. "Action" vs "Action & Adventure"). | Se mantienen tal cual y se avisa en los gráficos que comparan ambos tipos (regla de negocio 2). |

## 4. KPIs principales (medidas DAX)

- **Catálogo**: Total Contenidos, Total Películas, Total Series, Países Representados, Idiomas Disponibles, Géneros Distintos, Contenidos Acumulados.
- **Popularidad**: Popularidad Mediana (la principal, porque la popularidad tiene valores extremos), Popularidad Promedio.
- **Valoración**: Calificación Promedio, Calificación Ponderada por votos, Calificación Confiable (≥100 votos), Total Votos, % Evaluación Confiable.
- **Finanzas (solo películas)**: Presupuesto Total, Ingresos Totales, ROI = Ingresos / Presupuesto (definición del caso), ROI Mediano, % Películas Rentables (ROI > 1), Cobertura Financiera.
- **Textos dinámicos**: pie de página con el período y el alcance de cada vista (regla de negocio 7).

## 5. Títulos con hallazgo clave

Cada gráfico se titula con su **hallazgo principal** en vez de con una descripción (por ejemplo, "Estados Unidos participa en el 34% de los títulos, 3,8 veces el volumen de Japón"). El subtítulo explica qué muestra el gráfico y su alcance.

Los títulos son **medidas DAX** (carpeta *6. Títulos dinámicos*): se recalculan con los filtros. Por ejemplo, con el filtro "Serie" el mismo título dice "Estados Unidos participa en el 20% de los títulos, 1,7 veces el volumen de China". Así el titular siempre coincide con lo que se ve. Para calcularlos se quita el filtro Top N del propio gráfico y se mantienen los segmentadores, de modo que los porcentajes se calculan sobre todo el catálogo filtrado.

## 6. Páginas y justificación de diseño

Todas las páginas comparten la misma estructura: título con audiencia y pregunta guía, navegador de páginas, una fila de segmentadores sincronizados entre páginas, KPIs, gráficos y pie de alcance. La consistencia reduce el esfuerzo de lectura (principios de consistencia y jerarquía visual).

### 6.1 Resumen Ejecutivo — Directorio y Gerencia General
*¿Cómo está compuesto el catálogo y cómo ha evolucionado?*
- **8 tarjetas KPI**: entregan la visión consolidada que pide el Problema 1 de un solo vistazo.
- **Barras horizontales Top 10 (géneros y países)**: las barras horizontales comparan magnitudes con precisión y dejan leer etiquetas largas. Se ordenan de mayor a menor y llevan el valor en la barra, sin eje redundante.
- **Áreas apiladas acumuladas**: muestran la evolución del total y el aporte de cada tipo.
- Hallazgos: Drama está presente en el 46% de los títulos. EE. UU. participa en el 34%, 3,8 veces el volumen de Japón. El catálogo crece de forma lineal, ~2.000 títulos por año.

### 6.2 Análisis del Catálogo — Gerencia de Contenidos y Producto
*¿Qué géneros, idiomas y talentos componen la oferta?*
- **Treemap tipo → género**: muestra la composición parte-todo jerárquica en poco espacio, coloreado por tipo.
- **Línea de incorporaciones por año**: el eje parte en 0 para no exagerar variaciones. Evidencia el volumen constante del extracto.
- **Top 10 idiomas, directores y actores** en barras (preguntas de negocio de producción).
- Hallazgos: el inglés concentra el 44% de los títulos. La dirección está muy atomizada (14.025 directores y un máximo de 21 títulos). Min Do-yoon lidera el reparto con 111 apariciones.

### 6.3 Popularidad y Valoración — Gerencia de Contenidos y Marketing
*¿Qué contenidos destacan y qué los caracteriza?*
- **Top 10 más populares y mejor evaluados** (≥100 votos para no premiar notas con pocos votos).
- **Columnas agrupadas película vs serie** por género, para responder si hay diferencias entre ambos tipos.
- **Dispersión calificación vs popularidad** en escala logarítmica (la popularidad varía en varios órdenes de magnitud), con el coeficiente r en el título.
- Hallazgos: la popularidad está muy concentrada (el n.º 1 multiplica por 291 la mediana). Solo 157 de 11.805 títulos evaluados superan 8,5. Las series tienen 3,3 veces la popularidad mediana del cine. Nota y popularidad tienen una relación moderada (r = 0,46).

### 6.4 Rentabilidad de Películas — Directorio y Gerencia de Adquisición
*¿Qué películas retornan más que su inversión?*
- Filtro fijo Tipo = Película: budget y revenue solo existen para películas (regla de negocio 3). El pie declara la cobertura (22%).
- **Dispersión presupuesto vs ingresos** con ambos ejes logarítmicos: muestra la relación sin que las superproducciones aplasten al resto.
- **Top 10 por ingresos y por ROI**, con el presupuesto en el tooltip.
- Hallazgos: relación presupuesto-ingresos moderada (r = 0,66). 10 superproducciones (0,3% de las películas) generan el 5% de los ingresos. Los mayores ROI vienen de presupuestos bajos (mediana de US$113 mil vs US$15,8 M del total).

### 6.5 Estrategia y Marketing — Gerencia de Marketing y Adquisición
*¿Qué priorizar en adquisiciones y campañas?*
- **Mapa de oportunidades (burbujas)**: calificación vs popularidad por género. Arriba a la derecha quedan los géneros que conviene priorizar, y el tamaño indica el volumen.
- **Mercados con mayor potencial**: popularidad mediana de los 8 mayores países productores.
- **Géneros más rentables**: ROI mediano por género (películas).
- **Tabla de candidatos a campaña**: estrenos de los últimos 3 años, ≥100 votos y calificación ≥ 7, ordenados por popularidad.
- Hallazgos: 9 géneros superan la media en popularidad y nota, y destacan Infantil y Reality. China lidera en popularidad mediana (29,4), 1,6 veces Estados Unidos. Documental es el género más rentable (ROI mediano 2,8x vs 1,7x del total). 516 títulos recientes cumplen los criterios de campaña.

## 7. Decisiones visuales

- **Color con significado fijo**: azul = Película, naranja = Serie en todos los gráficos. El color está asignado a la categoría, así que no cambia al filtrar. La paleta fue validada para daltonismo (separación CVD ΔE ≥ 9). Los gráficos de una sola serie usan solo azul para no agregar colores sin significado.
- **Sin 3D, sin doble eje, sin decoración**: fondo neutro, tarjetas con borde suave y cuadrícula tenue.
- **Medianas en vez de promedios** donde hay valores extremos (popularidad, ROI). Se declara en cada subtítulo.
- **Escalas logarítmicas** solo en las dispersiones, y siempre indicadas en el subtítulo.
- **Números en formato chileno** (miles con punto, decimales con coma) y montos en US$.

## 8. Limitaciones declaradas

- No existe clasificación por edad en los datos (`rating` duplica `vote_average`).
- El volumen anual es fijo por diseño del extracto. No sirve para medir crecimiento real.
- Solo el 22% de las películas tiene datos financieros válidos. Las series no tienen.
- La popularidad es un índice relativo de TMDB, no equivale a reproducciones (regla de negocio 6).
