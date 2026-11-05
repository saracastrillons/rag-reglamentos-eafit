# Asistente RAG sobre los reglamentos de pregrado de EAFIT

Proyecto final de **SI3003 Inteligencia Artificial**, Universidad EAFIT, en la **Opción A: sistema RAG**.

[![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/saracastrillons/rag-reglamentos-eafit/blob/main/rag_reglamentos_eafit.ipynb)

## Integrantes

- _Nombre 1_
- _Nombre 2_
- _Nombre 3_
- _Nombre 4_

## Problema

Los estudiantes de EAFIT tienen dudas frecuentes sobre normas académicas: cancelación de materias, promedios, materias perdidas, supletorios, reingresos y devoluciones de dinero. La información está en reglamentos extensos en PDF, difíciles de consultar rápido, y muchas veces terminan preguntando en pasillos o redes y reciben respuestas imprecisas.

El prototipo es un asistente que responde esas preguntas **solo** a partir de los reglamentos oficiales. Junto con cada respuesta muestra el documento, el artículo y la página de los fragmentos en que se apoya, y dice cuándo la información no está en los documentos.

**Por fuera del alcance:** casos personales que requieren datos del estudiante, trámites en línea, interpretación jurídica o decisiones oficiales, normas que no estén en los documentos cargados (calendario académico, valores de matrícula, parqueaderos) y la actualización automática de los reglamentos.

## Opción elegida y por qué

Elegimos **RAG** porque el conocimiento ya existe en documentos escritos y la tarea principal es encontrar y explicar lo que dicen, no hacer cálculos. RAG hace que el modelo responda con base en el texto oficial y no en su conocimiento general, que puede estar desactualizado o corresponder a otra universidad. Además, mostrar la fuente permite verificar cada respuesta contra el reglamento.

## Documentos

| Documento | Archivo en el repo | Páginas | Fuente oficial |
|---|---|---|---|
| Reglamento Académico de los Programas de Pregrado (con modificaciones hasta el acta 019 del 9 de diciembre de 2025) | [`docs/reglamento-academico-pregrado.pdf`](docs/reglamento-academico-pregrado.pdf) | 79 | [eafit.edu.co · reglamento académico de pregrado](https://www.eafit.edu.co/institucional/reglamentos/reglamento-academico-de-los-programas-de-pregrado) |
| Reglamento Económico de la Relación Académica del Estudiante con la Universidad, versión VI (noviembre de 2020) | [`docs/reglamento-economico.pdf`](docs/reglamento-economico.pdf) | 10 | [EAFIT · reglamento económico](https://universidadeafit.widen.net/s/wggqcsdnhf/reglamento_economico) |

Los dos son públicos y se descargaron el 5 de noviembre de 2026 desde la [página de reglamentos de EAFIT](https://www.eafit.edu.co/institucional/reglamentos/reglamentos).

## Herramientas

| Etapa | Herramienta |
|---|---|
| Extracción de texto por página | PyMuPDF |
| Fragmentación | Corte por artículo + `RecursiveCharacterTextSplitter` de LangChain (1000 caracteres, solapamiento de 150) |
| Embeddings | NVIDIA NIM `nvidia/nemotron-3-embed-1b` |
| Índice vectorial | Qdrant en memoria, distancia coseno |
| LLM | NVIDIA NIM `nvidia/nemotron-3-super-120b-a12b` |
| Orquestación | LangChain |
| Interfaz (opcional) | Gradio |

Partimos del notebook del curso `03_rag_workshop_papers_nvidia_nim_colab.ipynb` y le hicimos estos cambios:

1. **Fuente con artículo y página.** MarkItDown pierde los números de página, así que leemos con PyMuPDF página por página. El texto se corta primero por artículo y cada fragmento guarda en sus metadatos el documento, el número y nombre del artículo y la página.
2. **Fragmentos con sentido.** Un artículo trata un solo tema. Si es largo, se corta preferiblemente antes de un "Parágrafo". Cada fragmento empieza con un encabezado `[Documento · Artículo N: título]`.
3. **Contexto insuficiente en dos niveles.** Si ningún fragmento supera el umbral de similitud, el sistema responde sin llamar al LLM. Si alguno lo supera, el prompt obliga al modelo a responder una frase fija cuando los fragmentos no alcanzan, y las pruebas detectan esa frase.
4. **Respuesta con fuentes.** Cada respuesta muestra los fragmentos usados con documento, artículo, página, similitud y un extracto.

## Cómo ejecutarlo

### Opción 1: Google Colab (recomendada)

1. Abre el notebook con el botón **Abrir en Colab** de arriba.
2. Crea una clave gratuita en https://build.nvidia.com/.
3. En Colab abre el panel 🔑 **Secrets**, crea un secreto llamado `NVIDIA_API_KEY` con la clave y activa el acceso para el notebook.
4. Ejecuta *Entorno de ejecución → Ejecutar todas*. El notebook descarga los PDF del repositorio si no están.

### Opción 2: local

```bash
git clone https://github.com/saracastrillons/rag-reglamentos-eafit.git
cd rag-reglamentos-eafit
python -m venv .venv
.venv\Scripts\activate          # Windows  (en macOS/Linux: source .venv/bin/activate)
pip install -r requirements.txt
copy .env.example .env          # en macOS/Linux: cp .env.example .env
```

Pega tu clave en `.env` (`NVIDIA_API_KEY=...`) y abre el notebook con `jupyter notebook rag_reglamentos_eafit.ipynb`.

> **Seguridad:** la clave nunca va en el código ni en el repositorio. El notebook la lee desde los secretos de Colab, desde `.env` o la pide de forma oculta. El archivo `.env` está en `.gitignore`.

## Estructura

```
rag-reglamentos-eafit/
├── rag_reglamentos_eafit.ipynb   # notebook principal (Partes 0 a 7)
├── docs/                         # los dos reglamentos en PDF
├── resultados/                   # se genera al correr las pruebas (CSV y Markdown)
├── requirements.txt
├── .env.example
└── README.md
```

## Casos de prueba

Las respuestas esperadas salen de leer directamente el reglamento. El notebook (Parte 5) ejecuta los cinco casos, muestra las fuentes y guarda la tabla en `resultados/`. Además calcula dos comprobaciones objetivas: si recuperó el artículo esperado y si el sistema se abstuvo.

| # | Tipo | Pregunta | Resultado esperado | Fuente esperada |
|---|---|---|---|---|
| 1 | Habitual | ¿Qué porcentaje de los créditos matriculados puedo cancelar en un semestre y qué condiciones hay? | Hasta el 50 % de los créditos. La asignatura no debe haber sido evaluada en más del 75 %, la cancelación se hace dentro de la fecha del calendario y no se pueden cancelar prácticas ni asignaturas con matrícula condicional (salvo fuerza mayor). | Académico, Art. 50 |
| 2 | Habitual | ¿Qué pasa si pierdo la misma materia por tercera vez? | Cuarta matrícula obligatoria en el semestre siguiente, con matrícula condicional y máximo 12 créditos. No se puede cancelar en la 3.ª, 4.ª ni 5.ª matrícula. Si la pierde en la cuarta, queda retirado por rendimiento. | Académico, Art. 51 |
| 3 | Habitual | ¿Con qué promedio me retiran de la universidad por bajo rendimiento académico? | Primera matrícula, promedio semestral menor que 2.5. Segunda o tercera, semestral y acumulado menor que 2.8. Cuarta en adelante, menor que 3.0. | Académico, Art. 52 |
| 4 | Cruza los dos documentos | Si cancelo una materia durante la segunda semana de clases, ¿me devuelven el dinero? | Se puede cancelar (Art. 50), y el Art. 50 remite al Reglamento Económico. Devolución del 90 % antes de iniciar la 2.ª semana, del 75 % antes de terminarla y nada desde la 3.ª. Los de primer semestre no pueden retirar materias. | Académico, Art. 50 + Económico, Art. 17 |
| 5 | Límite (información ausente) | ¿Cuánto cuesta el parqueadero de la universidad para estudiantes? | Debe decir que la información no está en los reglamentos, sin inventar una tarifa. | Ninguna |

### Resultados obtenidos

> **Pendiente de completar.** Ejecutar la Parte 5 del notebook y pegar aquí la respuesta obtenida de cada caso, desde `resultados/resultados_pruebas.md`, con su valoración manual.

| # | Respuesta obtenida (resumen) | ¿Recuperó el artículo esperado? | ¿Se abstuvo? | Valoración |
|---|---|---|---|---|
| 1 | _completar_ | _completar_ | _completar_ | _completar_ |
| 2 | _completar_ | _completar_ | _completar_ | _completar_ |
| 3 | _completar_ | _completar_ | _completar_ | _completar_ |
| 4 | _completar_ | _completar_ | _completar_ | _completar_ |
| 5 | _completar_ | _completar_ | _completar_ | _completar_ |

**Criterios de valoración:**
- **Relevancia:** ¿la respuesta atiende la pregunta?
- **Respaldo:** ¿cada afirmación aparece en los fragmentos mostrados?
- **Citas:** ¿el artículo y la página citados son correctos?
- **Abstención:** ¿se abstuvo solo cuando debía?

## Análisis de resultados

> Completar con lo observado al ejecutar. Abajo están las limitaciones que se anticipan por el diseño, para confirmar o descartar con las pruebas.

**Limitación principal (a confirmar en la Parte 6): la similitud no distingue "habla del tema" de "contiene la respuesta".** Ante "¿Cuánto vale la matrícula de Ingeniería de Sistemas?", el Reglamento Económico tiene muchos artículos sobre matrícula con similitud alta, pero ninguno dice cuánto cuesta. El umbral no filtra esos fragmentos, así que la abstención depende de que el LLM respete la instrucción del prompt. Si no la respeta, puede responder con información relacionada (componentes o recargos de la matrícula) como si fuera la respuesta.

**Otras limitaciones:**
- **Referencias internas sin resolver.** El reglamento académico remite a otros artículos por su nombre ("el artículo de este reglamento denominado inasistencia a las evaluaciones"). El sistema solo ve el fragmento recuperado y no sigue la referencia.
- **Preguntas que dependen de la situación personal.** Por ejemplo, "¿con qué promedio me retiran?" depende de en qué matrícula va el estudiante. El sistema puede dar la regla general, pero no aplicarla a un caso concreto.
- **Vigencia.** Los reglamentos cambian por actas del Consejo Académico. El índice refleja la versión descargada y no se actualiza solo.

**Posibles mejoras:**
1. **Búsqueda híbrida (vectorial + BM25)**, como se vio en clase, para preguntas con términos exactos ("Artículo 52", "matrícula condicional", "UMES").
2. **Seguir las referencias internas:** cuando un fragmento menciona "el artículo denominado X", recuperar también ese artículo.
3. **Re-ranking** de los fragmentos recuperados antes de pasarlos al LLM.

## Aviso

Este asistente orienta, pero no reemplaza a la Oficina de Admisiones y Registro ni constituye una interpretación oficial de los reglamentos.
