# Hito 1: Integración de fuentes de datos públicas del Perú

Proyecto de análisis de datos que integra dos fuentes oficiales peruanas a nivel distrital
para describir cómo se reparten los recursos que el Estado transfiere a las municipalidades.

---

## 1. Problema

El Estado peruano transfiere cada año recursos a las municipalidades por distintos rubros
(FONCOMUN, canon, regalías, programas sociales). Esa información la publica el MEF, mientras
que las características de las municipalidades que reciben ese dinero las publica el INEI.
Están en sistemas separados, así que no es posible responder directamente:

> ¿Cómo se distribuyen las transferencias entre los distritos del país, qué tan concentrado
> está ese reparto, y qué municipalidades quedan en los extremos?

## 2. Organización y usuario

| | |
|---|---|
| **Contexto** | Gobiernos locales del Perú |
| **Usuario de los resultados** | MEF – Dirección General del Tesoro Público; Contraloría General de la República; gerencias municipales |
| **Decisión que habilita** | Identificar municipalidades atípicas en el reparto y sustentar ajustes en los criterios de distribución |

## 3. Objetivo

**Describir y comparar** las transferencias recibidas por las municipalidades del Perú en 2024,
integrando los registros del MEF con el marco censal de municipalidades del INEI, e
identificar los casos atípicos.

Es un objetivo **descriptivo con detección de anomalías**. No se plantea predicción ni se
afirma causalidad en ninguna conclusión.

## 4. Fuentes de datos

### Fuente A: principal

- **Nombre:** Transferencias de fondos y asignaciones financieras (ejercicio 2024)
- **Entidad:** Ministerio de Economía y Finanzas (MEF) — Dirección General del Tesoro Público
- **Página del dataset:** https://www.datosabiertos.gob.pe/dataset/transferencias-de-fondos-y-asignaciones-financieras
- **Descarga directa:** https://fs.datosabiertos.mef.gob.pe/datastorefiles/2024-Transferencias.csv
- **Diccionario oficial:** https://fs.datosabiertos.mef.gob.pe/datastorefiles/Transferencias_Diccionario.csv
- **Cobertura:** 2004 a 2026, Gobierno Nacional, Regional y Local
- **Contenido:** cada fila es una transferencia a nivel de unidad ejecutora × mes × rubro × concepto.
- **Variables usadas:** `ANO_PRC`, `MES_PRC`, `TIPO_ENTIDAD_NOMBRE`, `EJECUTORA_NOMBRE`,
  `DEPARTAMENTO_EJECUTORA`, `PROVINCIA_EJECUTORA`, `DISTRITO_EJECUTORA`, `RUBRO_NOMBRE`,
  `FUENTE_NOMBRE`, `RECURSO_NOMBRE`, `MONTO_AUTORIZADO`, `MONTO_ACREDITADO`.

**Ruta manual de descarga:** Plataforma Nacional de Datos Abiertos → buscar
*"Transferencias de fondos y asignaciones financieras"* → organización
*Ministerio de Economía y Finanzas* → recurso `2024-Transferencias.csv` → Descargar.

> **Nota sobre la elección del archivo.** El primer candidato fue
> `2024-Gasto.csv` del dataset *Presupuesto y Ejecución de Gasto*, del mismo ministerio.
> Se descartó tras comprobar que pesa **9 764 MB**, inviable de descargar en Colab. Se optó
> por el dataset de Transferencias, que tiene los mismos códigos geográficos y por tanto la
> misma clave de integración, pero un volumen manejable.

### Fuente B: a integrar

- **Nombre:** Registro Nacional de Municipalidades (RENAMU) 2025
- **Entidad:** Instituto Nacional de Estadística e Informática (INEI)
- **Página del dataset:** https://www.datosabiertos.gob.pe/dataset/registro-nacional-de-municipalidades-renamu-2025-instituto-nacional-de-estadística-e
- **Descarga directa (ZIP, ~2 MB):** https://proyectos.inei.gob.pe/iinei/srienaho/descarga/CSV/984-Modulo1963.zip
- **Diccionario oficial (PDF):** https://proyectos.inei.gob.pe/iinei/srienaho/Descarga/DocumentosMetodologicos/2025-142/Diccionario_Anexo01.pdf
- **Licencia:** ODbL
- **Contenido:** censo de municipalidades provinciales, distritales y de centros poblados;
  módulos de datos generales, equipamiento y TIC, personal, competencias y servicios locales.
- **Variables de identificación:** `Año`, `idmunici`, `ccdd`, `ccpp`, `ccdi`, `Ubigeo`,
  `Departamento`, `Provincia`, `Distrito`, `Tipomuni` (1 = provincial, 2 = distrital, 3 = centro poblado).

**Ruta manual de descarga:** Plataforma Nacional de Datos Abiertos → buscar *"RENAMU 2025"* →
recurso *"Data completa del Registro Nacional de Municipalidades (RENAMU) 2025"* → Descargar.

### Fuente auxiliar (validación de la clave)

- **INEI: Ubigeos, 1 891 distritos:**
  https://www.datosabiertos.gob.pe/dataset/ubigeos-códigos-de-ubicación-geográfica-instituto-nacional-de-estadística-e-informática-inei

## 5. Unidad de análisis

**Una municipalidad (un distrito) en el ejercicio 2024.** Cada fila de la base final
representa un distrito con el total de transferencias del año y sus atributos de identificación.

Esto obliga a dos transformaciones antes del cruce:

- **Fuente A:** viene a nivel ejecutora × mes × rubro → se **agrega** sumando por UBIGEO.
- **Fuente B:** viene a nivel municipalidad, incluidos centros poblados → se **filtra**.

## 6. Clave de integración

**`UBIGEO` de 6 dígitos** = departamento (2) + provincia (2) + distrito (2).

| | Fuente A (MEF) | Fuente B (RENAMU) |
|---|---|---|
| Cómo se obtiene | Concatenar `DEPARTAMENTO_EJECUTORA` + `PROVINCIA_EJECUTORA` + `DISTRITO_EJECUTORA` | Columna `Ubigeo` |
| Unicidad | No en la base cruda; sí tras agregar | No: los centros poblados repiten el ubigeo de su distrito |
| Tipo | Texto — crítico conservar los ceros a la izquierda | Alfanumérico de 6 |

**Tipo de JOIN: `LEFT JOIN` con RENAMU como tabla izquierda.** RENAMU es el marco censal
completo de municipalidades; el MEF solo trae las que recibieron transferencias. Con LEFT se
conservan los distritos sin transferencias, que forman parte del hallazgo y no son un error.
El notebook reporta además qué queda fuera por cada lado usando `indicator=True`.

## 7. Problemas de calidad que el notebook comprueba

El diagnóstico se produce **al ejecutar**, no está escrito de antemano. Se revisa:

- valores faltantes por columna y porcentaje;
- duplicados en la clave, en cada fuente por separado;
- tipos mal detectados, en particular el ubigeo leído como número;
- montos no numéricos, convertidos con `errors="coerce"` y contados;
- valores fuera de rango en el porcentaje de acreditación;
- categorías no uniformes en las variables de texto;
- registros huérfanos: presentes en una fuente y ausentes en la otra.

## 8. Decisiones de limpieza

| # | Decisión | Justificación |
|---|---|---|
| 1 | Eliminar municipalidades de centro poblado (`Tipomuni = 3`) | Repiten el UBIGEO de su distrito, rompen la unicidad de la clave y no reciben transferencias propias |
| 2 | Eliminar duplicados residuales de UBIGEO conservando el primer registro | Garantiza una fila por unidad de análisis; se reporta cuántos fueron |
| 3 | **Conservar** los montos negativos | Corresponden a rectificaciones y devoluciones reales de la DGTP; se marcan pero no se borran |
| 4 | Forzar la clave a texto con `zfill(6)` | Evita perder el cero inicial de los departamentos 01–09, que rompería el cruce |
| 5 | Calcular `ACREDITACION_PCT` solo cuando el monto autorizado es mayor que cero | Evita divisiones por cero; los casos sin autorización quedan como nulos explícitos |

## 9. Visualizaciones

| # | Gráfico | Variables | Por qué ese gráfico |
|---|---|---|---|
| 1 | Barras | resultado del cruce | Categórica: audita la integración antes de interpretar nada |
| 2 | Histograma en log | monto acreditado | Numérica muy asimétrica; la escala log la hace legible |
| 3 | Barras horizontales | departamento × monto | Categórica con muchos niveles + numérica |
| 4 | Dispersión log-log | autorizado vs. acreditado | Dos numéricas; la diagonal marca el 100 % |
| 5 | Boxplot | tipo de municipalidad × monto | Compara nivel y dispersión entre dos grupos |
| 6 | Barras | top 15 distritos | Identifica los casos extremos |
| 7 | Curva de Lorenz + Gini | monto acumulado | Mide la concentración del reparto con un número citable |
| 8 | Barras | rubro × monto | Explica de dónde viene el dinero |

Todas las lecturas se redactan como **asociaciones descriptivas**. En ningún caso se afirma
causalidad.

## 10. Cómo ejecutar

### Opción A: Google Colab (recomendada)

1. Abrir https://colab.research.google.com
2. `Archivo` → `Subir cuaderno` → seleccionar `hito1_integracion_mef_renamu.ipynb`
3. `Entorno de ejecución` → `Ejecutar todas`

El notebook descarga las dos fuentes por sí solo. No requiere credenciales.

**Parámetros configurables** en la primera celda:

```python
ANIO = 2024              # ejercicio a analizar; bajarlo si el archivo pesa demasiado
GUARDAR_EN_DRIVE = False # True guarda las descargas en Drive y sobreviven a una desconexión
```

La celda de descarga consulta primero el tamaño de cada archivo y luego muestra progreso.
Si un archivo ya está completo en la carpeta, no lo vuelve a bajar.

### Opción B: Local

```bash
git clone <URL-DEL-REPOSITORIO>
cd <carpeta-del-repositorio>
pip install pandas numpy matplotlib requests jupyter
jupyter notebook hito1_integracion_mef_renamu.ipynb
```

## 11. Estructura del repositorio

```
.
├── README.md
├── hito1_integracion_mef_renamu.ipynb   #notebook principal
├── .gitignore
├── datos/                               #se crea al ejecutar (no se versiona)
│   ├── 2024-Transferencias.zip          #debido a su tamaño lo comprimimos a zip
│   ├── Transferencias_Diccionario.csv
│   └── renamu2025.zip
└── base_integrada_hito1.csv             #salida del notebook
```

Los archivos de datos **no se suben al repositorio** por su tamaño; el notebook los descarga
desde las fuentes oficiales y aquí se documentan los enlaces exactos.

## 12. Resultado esperado

Tipo de solución analítica: **descripción + detección de anomalías (outliers)**.

Tiene sentido para este problema porque el usuario institucional no necesita una predicción,
sino saber **qué municipalidades se apartan del patrón** para revisarlas. Con una sola foto
anual no hay base para un modelo predictivo, y afirmar lo contrario sería metodológicamente
insostenible.

Extensiones para el Hito 2:

- incorporar población distrital para calcular transferencias per cápita comparables;
- sumar variables de servicio del RENAMU (personal, residuos sólidos, equipamiento);
- segmentación de municipalidades por perfil de recursos y capacidad.

## 13. Limitaciones

- El análisis es **descriptivo**. Las diferencias entre grupos son asociaciones, no relaciones causales.
- RENAMU 2025 y las transferencias 2024 son cortes temporales distintos aunque contiguos.
- El monto transferido no mide la calidad de la gestión municipal ni del gasto posterior.
- Parte de la desigualdad observada responde a criterios legales de distribución (canon,
  regalías), por lo que un Gini alto no equivale por sí solo a un reparto injusto.

## 14. Uso de IA
Este proyecto utilizó **Claude (Anthropic)** como herramienta de soporte técnico y co-piloto de desarrollo. 
Su aplicación se centró en las siguientes actividades:

* **Desarrollo de Código:** Optimización, refactorización y depuración de los scripts de Python en Google Colab.
* **Exploración Metodológica:** Evaluación de alternativas para la integración de datos y justificación técnica de diseño.
* **Documentación:** Estructuración, redacción y revisión editorial de los informes técnicos.

> ⚠️ **Nota de responsabilidad:** La validación de los datos oficiales (MEF e INEI), la lógica detrás de la agregación de
> las claves de unión (UBIGEO), la interpretación crítica de los gráficos y la toma de decisiones analíticas fueron supervisadas
> y ejecutadas íntegramente por los autores del proyecto.

---

*Fuentes: Ministerio de Economía y Finanzas del Perú e Instituto Nacional de Estadística e
Informática, a través de la Plataforma Nacional de Datos Abiertos.*
