# ¿A quién llega el dinero? Concentración de las transferencias del Estado a las municipalidades del Perú, 2024

Proyecto de análisis de datos que integra dos fuentes oficiales peruanas a nivel de municipalidad para medir qué tan
concentrado está el reparto de las transferencias del Estado e identificar las municipalidades en los extremos.

**Estado actual: Hito 2: preparación de datos y primera matriz analítica.**

---

## 1. Problema

En 2024 el Estado transfirió a las 1,891 municipalidades del país alrededor de S/ 27,500 millones por distintos rubros
(canon, FONCOMUN, impuestos municipales, entre otros), cada uno con reglas de reparto propias. Los montos los publica
el MEF y las características de las municipalidades el INEI, en sistemas separados. Por eso no existe una vista por
municipalidad que muestre qué tan concentrado está el reparto ni qué municipalidades reciben montos muy por encima o
muy por debajo del resto.

> **Pregunta:** ¿qué tan concentradas están las transferencias 2024 entre las municipalidades del Perú y cuáles se
> ubican en los extremos del reparto?

| | |
|---|---|
| **Usuario** | MEF – Dirección General del Tesoro Público; Contraloría General de la República; gerencias municipales |
| **Decisión que habilita** | Priorizar qué municipalidades revisar (montos atípicamente altos o bajos, distinguiendo el canon) |
| **Objetivo** | Construir una base integrada y una matriz analítica por municipalidad que permitan medir la concentración e identificar casos atípicos |
| **Tipo de análisis** | Descriptivo con detección de atípicos. No se afirma causalidad |

## 2. Fuentes de datos

| Fuente | Entidad | Descarga directa |
|---|---|---|
| A — Transferencias de fondos y asignaciones financieras 2024 | MEF | <https://fs.datosabiertos.mef.gob.pe/datastorefiles/2024-Transferencias.csv> |
| A — Diccionario | MEF | <https://fs.datosabiertos.mef.gob.pe/datastorefiles/Transferencias_Diccionario.csv> |
| B — RENAMU 2025 | INEI | <https://proyectos.inei.gob.pe/iinei/srienaho/descarga/CSV/984-Modulo1963.zip> |

**Ruta manual**, si los enlaces cambian: Plataforma Nacional de Datos Abiertos → buscar *"Transferencias de fondos y
asignaciones financieras"* (organización MEF, recurso `2024-Transferencias.csv`) y *"RENAMU 2025"* (recurso *"Data
completa del Registro Nacional de Municipalidades (RENAMU) 2025"*).

Los datos **no se versionan** por su tamaño: el notebook los descarga automáticamente.

## 3. Unidad de análisis

**Una municipalidad (un distrito) en el ejercicio 2024.** 1,891 filas: 196 provinciales y 1,695 distritales.

| Tabla | Qué representa cada fila |
|---|---|
| MEF original | una transferencia: unidad ejecutora × mes × rubro × concepto |
| RENAMU original | una municipalidad (provincial, distrital o de centro poblado) |
| Base integrada y matriz analítica | una municipalidad (distrito), 2024 |

## 4. Integración

- **Clave:** UBIGEO de 6 dígitos (texto, con ceros a la izquierda).
- **Cardinalidad:** MEF detalle → distrito es N:1 (se agrega por suma); MEF agregado ↔ RENAMU es 1:1, exigido con `validate="one_to_one"`.
- **Unión:** `LEFT JOIN` con RENAMU como tabla principal. Resultado: 1,891 emparejados, 0 sin pareja (100%).

## 5. Principales decisiones de calidad

| # | Decisión | Justificación |
|---|---|---|
| 1 | Excluir entidades no municipales y mancomunidades | No corresponden a un distrito |
| 2 | Leer los códigos geográficos como texto | Evita perder el cero inicial (`01` → `1`) |
| 3 | Excluir centros poblados y duplicados de UBIGEO (control preventivo) | Garantizan una fila por municipalidad |
| 4 | Conservar montos negativos (control preventivo) | Serían rectificaciones reales de la DGTP |
| 5 | Marcar atípicos con IQR sobre log10 del monto, sin eliminarlos | Los extremos son el objetivo del proyecto |

## 6. Gráficos (3)

| # | Pregunta que responde | Archivo |
|---|---|---|
| 1 | ¿Qué tan concentrado está el reparto? | `graficos/grafico_1_concentracion.png` |
| 2 | ¿De dónde viene el dinero? | `graficos/grafico_2_rubros.png` |
| 3 | ¿Quiénes están en el extremo superior? | `graficos/grafico_3_top15.png` |

## 7. Matriz analítica

`salidas/matriz_analitica_hito2.csv` — 1 fila = 1 municipalidad, 11 variables:
`ubigeo`, `distrito`, `departamento`, `ES_PROVINCIAL`, `MONTO_ACREDITADO`, `LOG_MONTO`, `DECIL_MONTO`,
`PCT_CANON`, `PCT_FONCOMUN`, `ACREDITACION_PCT`, `ATIPICO`.
El diccionario está en `salidas/diccionario_matriz_hito2.csv`.

## 8. Cómo ejecutar

### Google Colab (recomendado)
1. Abrir <https://colab.research.google.com>
2. `Archivo` → `Subir cuaderno` → `hito2_transferencias_municipales.ipynb`
3. `Entorno de ejecución` → `Ejecutar todas`

No requiere credenciales. Parámetros en la primera celda: `ANIO = 2024` y `GUARDAR_EN_DRIVE = False`
(`True` guarda las descargas en Drive para no repetirlas si Colab se desconecta).

### Local
```bash
git clone https://github.com/bbmedinad-lab/peru-municipal-transfers.git
cd peru-municipal-transfers
pip install pandas numpy matplotlib requests jupyter
jupyter notebook hito2_transferencias_municipales.ipynb
```

## 9. Estructura del repositorio

```
.
├── README.md
├── hito2_transferencias_municipales.ipynb   # notebook principal (fuentes → matriz analítica)
├── .gitignore
├── datos/                                   # se crea al ejecutar (no se versiona)
├── graficos/                                # 3 gráficos generados por el notebook
├── salidas/
│   ├── base_integrada_hito2.csv
│   ├── matriz_analitica_hito2.csv
│   └── diccionario_matriz_hito2.csv
└── hito1/                                   # versión entregada en el Hito 1
```

## 10. Siguiente paso

Incorporar población distrital (transferencias per cápita), estandarizar las variables y aplicar detección de
anomalías y segmentación de municipalidades.

## 11. Limitaciones

- El análisis es descriptivo: las diferencias son asociaciones, no causas.
- RENAMU 2025 y las transferencias 2024 son cortes contiguos, no idénticos.
- El monto transferido no mide la calidad del gasto ni de la gestión municipal.
- Parte de la desigualdad responde a reglas legales (canon, regalías): un Gini alto no equivale por sí solo a un reparto injusto.

## 12. Uso de IA

Se utilizó Claude (Anthropic) como apoyo en el desarrollo y depuración del código, la evaluación de alternativas
metodológicas y la redacción de la documentación. La validación de los datos, las decisiones analíticas y la
interpretación de los resultados son responsabilidad de los autores.

---
*Fuentes: Ministerio de Economía y Finanzas e Instituto Nacional de Estadística e Informática, a través de la
Plataforma Nacional de Datos Abiertos.*
