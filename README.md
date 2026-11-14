# Flame in Freefall

Diseño: [`flame_in_freefall_diseno_fase1.md`](flame_in_freefall_diseno_fase1.md).

## M1 · Cliente PSI e inventario

Inventario de **solo metadatos** de las investigaciones de Combustion Science de NASA PSI
(API `geode-py`). No descarga archivos.

```bash
uv sync
uv run fif-inventory                      # inventario completo (~3 min, 49 llamadas)
uv run fif-inventory --only PSI-159       # snapshot parcial (no mueve CURRENT)
uv run pytest                             # pruebas offline
uv run pytest -m live                     # prueba contra la API real (PSI-159)
```

### Salida (`data/psi/`, fuera de git)

```
snapshots/<run_id>/{investigations,publications,resources,versions,runs}.parquet
CURRENT            # run_id del último snapshot completo
inventory.duckdb   # vistas
```

| Vista DuckDB | Contenido |
| --- | --- |
| `investigations` | Ficha por investigación (snapshot CURRENT). `project_type` es el valor PSI tal cual. `source_json` = ficha completa sin `remote_url` ni datos de contacto. |
| `publications` | Publicaciones de cada ficha (título, DOI, autores). Entrada para M3. |
| `resources` | Un registro de archivo PSI por fila (`file_id` = `id` de PSI). `resource_id` = hash estable de `(inv_id, file_id, file_version)`. |
| `versions` / `versions_history` | Por investigación y ejecución: `version`, `totalFileSize`, conteos del árbol y diff (añadidos/retirados/cambiados por `file_id`) contra el CURRENT previo. |
| `runs` | Bitácora de ejecuciones (alcance, conteos, llamadas, avisos). |

### Marcas de calidad (`resources.quality_flags`) — marcan, no resuelven

| Marca | Significado |
| --- | --- |
| `path_version_conflict` | La misma ruta tiene ≥2 registros distintos (otro `id`/versión). Se conservan todos. Hoy: 54 rutas de PSI-20. |
| `repeated_in_tree` | El mismo registro (nodo idéntico) aparece n veces en el árbol; se guarda una vez con `tree_occurrences = n`. Hoy: PSI-106, 21, 22, 68, 69, 99. |
| `missing_category`, `path_outside_investigation`, `unparsed_updated`, `status_not_active` | Defensivas; sin casos hoy. |

Si un mismo `id` aparece con metadatos distintos, el inventario se detiene con `PsiSchemaError`.

### Observado el 9 nov 2026 (snapshot `20261110T011635Z`)

- 24 investigaciones: 19 `Microgravity Investigation`, 5 `Ground Investigation` (campo `projectType`).
- 66,602 nodos de archivo en los árboles = 60,900 registros únicos, 60,831 rutas únicas.
- `totalFileSize` de PSI coincide con la suma de nodos (incluye repeticiones): ~12.3 TiB.
- 22 archivos *Experimental table* (coinciden con `experimentalTable` de cada ficha); PSI-10 y PSI-159 tienen 2, PSI-60/62/115/117 ninguno.
- 352 publicaciones, 284 con DOI.

### Notas de la API (endpoints observados, sin contrato)

- `POST /files/v2` con `folder: ""` devuelve el árbol completo anidado en `files`; los archivos llevan
  `directory: false`, las carpetas `displayName`. La categoría viene en `sortingCategory`/`subcategory`.
- `/download` no se implementa en M1 (es de M2).
- No se persisten `remote_url` (posible URL firmada) ni email/teléfono de contactos.

Fixtures de prueba en `tests/fixtures/`: respuestas reales del 9 nov 2026, saneadas y recortadas.

## M2 · Ensayos (tablas experimentales)

```bash
uv run fif-trials download   # 22 tablas (~211 KB), streaming, tope de bytes, sha256
uv run fif-trials draft      # borradores YAML en mappings/experimental_tables/ (no sobrescribe)
uv run fif-trials build      # filas sin pérdida → perfiles → ensayos → enlaces
```

`/download` recibe el **nombre del archivo** (no la ruta) y la versión del registro;
devuelve una URL firmada que nunca se registra ni se guarda.

**Arquitectura de mapeo:** reglas reutilizables en código (lectura, parsers de
valores, registro de unidades, patrones de encabezado, normalizadores de ID) +
un YAML declarativo y versionado por tabla que solo enlaza columnas con reglas.
Una tabla nueva = un YAML (borrador automático); una unidad nueva = una entrada
en `trials/units.py`. Revisión humana: ver `mappings/experimental_tables/REVIEW.md`.

| Vista DuckDB | Contenido |
| --- | --- |
| `downloads` | Manifiesto de descargas (sha256, ruta local, snapshot de inventario). |
| `table_rows` | Cada registro CSV (encabezado, datos, vacíos, notas) con sus celdas crudas. |
| `table_profiles` | Perfil por columna: llenado, números/aprox./rangos/listas/texto, unidades en celdas. |
| `trials` | Entidad Ensayo para todas las granularidades (`row_grain`: trial, test_day, session, condition_matrix). Valores como envolvente `_min`/`_max` en unidades canónicas; procedencia por campo en `field_provenance_json`; columnas originales siempre. |
| `trial_resource_links` | Ensayo → recursos del inventario; `match_type` = `exact_literal` o `exact_after_normalization` (+ `normalization`). |
| `tables` | Una fila por tabla: mapeo, versión, estado de revisión, conteos. |

Reglas de datos (decididas el 11 nov 2026):
- `gravity_g` siempre `null` hasta tener evidencia por ensayo; `project_type` queda como contexto.
- `o2_location` = `unknown` salvo texto explícito o revisión documentada.
- Solo se convierte con unidad explícita (encabezado, celda o revisor con fuente); el valor y la unidad originales se conservan. Una celda cuya unidad contradice la declarada, o un resultado fuera del dominio matemático (fracción > 1, presión < 0), no se convierte.
- `conversion_verified` (factor probado) ≠ `interpretation_verified` (significado físico, solo humano).
- IDs: quitar espacios es la única normalización; los IDs sin coincidencia exacta quedan sin enlazar; IDs de < 5 caracteres no se buscan en rutas.

Estado 11 nov 2026: 22 tablas, 2,892 registros CSV, 1,997 observaciones
(1,875 `trial`, 112 `test_day`, 2 `session`, 8 `condition_matrix`), 894 ensayos
enlazados (6,092 enlaces), 44 sin enlace. PSI-159 Normal: 179 ensayos, 175 enlazados.

## M3 · Corpus de texto

```bash
uv run fif-corpus download                         # nivel 1: Science Documents, Reports, Presentations (≤50 MB)
uv run fif-corpus --include-engineering download   # nivel 2: Engineering Documents
uv run fif-corpus build                            # DataCite, OpenAlex, Semantic Scholar, NTRS → documentos, páginas, oraciones
```

Extensible sin tocar el pipeline: fuentes = adaptador + entrada en `config/sources.yaml`;
formatos = entrada en `corpus/formats.py`; géneros y permisos = reglas + YAML de
revisión (`config/doc_genre_overrides.yaml`, `config/permission_overrides.yaml`).
Incremental: respuestas HTTP en caché, descargas por `resource_id`, PDFs abiertos
por URL, texto por sha256 (solo si `allowed`). Segunda ejecución: 16 s, 0 llamadas.

| Vista | Contenido |
| --- | --- |
| `documents` | Un documento por sha256 (o por abstract / texto abierto de una obra); `evidence_level`, `processing_status`, `permission` (+ regla y base verificable), `notices`, `doc_genre`, `content_nature` (planned/observed/descriptive/mixed), `text_quality`. |
| `document_resources` | Documento ↔ recursos e investigaciones de PSI (deduplicación sin perder relaciones). |
| `doc_pages`, `doc_sentences` | Texto original por página; oraciones con `page_no`, `char_start/char_end` (`page.text[start:end] == text`), sección y tipo de sección. |
| `works`, `work_investigations`, `access_options` | Publicaciones (DOI o título), nivel de evidencia y ubicaciones de acceso con decisión de permiso. |
| `link_candidates` | `same_work_candidate` y `title_link_candidate` (NTRS): candidatos, nunca fusiones. |
| `investigation_rights` | Licencia por investigación: ficha PSI + DataCite. |
| `corpus_coverage` | Por investigación: licencia; documentos con texto, con texto sospechoso o escaso, bloqueados (y con aviso de terceros), no descargados, con fallo de descarga, no procesables, con género incierto, observados/planificados; obras por nivel de evidencia y textos abiertos fallidos; candidatos de enlace pendientes. Un documento compartido cuenta en cada investigación (las sumas no son totales únicos). |

Estado 12 nov 2026: 137 documentos PSI de nivel 1 (de 193 recursos; 110 con texto,
9 bloqueados, 8 con 404 en origen, 6 `.ppt`, 4 sin capa de texto) + 122 de nivel 2 sin
descargar; 299 obras (103 con abstract, 3 con texto completo, 193 solo título);
3,501 páginas y 40,981 oraciones. Pendientes: `docs/corpus_review.md`.

## M4 · Hallazgos

Cuatro etapas desacopladas y reproducibles; ninguna escribe el estado de otra.

```bash
uv sync --extra nli                         # solo si se clasifica en local (lento en CPU)
uv run fif-findings export-classify         # oraciones sin puntaje → bundle para Kaggle (solo texto CC0/abierto)
uv run fif-findings import-scores <dir>     # puntajes NLI calculados en Kaggle → caché local
uv run fif-findings select                  # prefiltro + clasificador → unidades (ventana 2+2, sección)
uv run fif-findings refsheet                # hoja de anotación del conjunto de referencia
uv run fif-findings export [--limit 20]     # unidades → bundle de extracción (piloto con --limit)
uv run fif-findings extract-local           # abstracts de editores con Ollama (nunca salen de la laptop)
uv run fif-findings import <run_dir>        # corrida terminada (Kaggle o local) → runs/ (inmutable)
uv run fif-findings verify                  # verificación determinista por campo
uv run fif-findings review                  # revisión científica desde reviews/findings/*.yaml
uv run fif-findings eval                    # recall/exactitud global y por investigación
```

Kaggle: `kaggle/fif_kaggle.py` (autocontenido; `job.json` elige pasos: env, classify, extract) y
`kaggle/fif_compat_check.py` (GPU, internet, vLLM + Qwen3-8B, sin datos del proyecto).

| Vista | Contenido |
| --- | --- |
| `sentence_selection` | Toda oración no-encabezado con `prefilter_status` (eligible, deferred_planned, excluded_*), ruta (kaggle/local) y puntaje. Nada se descarta en silencio. |
| `sentence_labels` | Puntajes de todas las etiquetas para toda oración clasificada (pasen o no). |
| `units` | Objetivo + contexto; `ctx_map_json` mapea cada carácter a `sent_idx`, página y posición. |
| `finding_candidates` | Salida del LLM tal cual, por corrida (`parse_status`: ok, empty, invalid_json, schema_error). |
| `finding_checks` | Una fila por comprobación y campo: pass, dropped, fail, unchecked (interpretativo → revisión humana). |
| `evidence` | Citas localizadas: documento, página, `char_start/char_end`, `match_type`. |
| `machine_findings` / `findings` | Resultado de la verificación (`source_verified`, `numeric_verified`, `extraction_verified`) + revisión científica separada; `display_status` nunca presenta lo automático como confirmado. |
| `findings_for_m5` | `machine_verified` no rechazados científicamente. |
| `findings_coverage` | Por investigación: oraciones elegibles/diferidas/excluidas, unidades, hallazgos por estado y nivel de evidencia. |

Reglas: las citas deben ser un tramo contiguo de la fuente (exacto o exacto tras normalizar espacios,
registrado) que toque la oración objetivo; los números y unidades deben aparecer en su cita; las
conversiones usan el registro de M2 (solo con unidad explícita, originales conservados); un campo sin
respaldo se descarta con motivo, nunca se repara ni se imputa.

Configuración validada en Kaggle (detalles y resultados en [docs/m4_pilot_eval.md](docs/m4_pilot_eval.md)):
NLI fijado al commit `8e7e5af…` en fp16; Qwen3-8B con vLLM 0.33.0 en un venv aislado (TP=2 en 2×T4);
gramática JSON con sangría fija (`--json-whitespace indent2`); prompt **p-4**. Cada versión del prompt se
archiva en `src/flame_in_freefall/findings/prompts/`; cambiar su texto exige una versión nueva (lo comprueba un test).

**Limitaciones actuales** (piloto real 1, 20 unidades): 19 `machine_verified`, de los cuales ~7 tienen tipo
erróneo según una evaluación preliminar del asistente (no es revisión científica). El modelo casi no usa
`limitation` y marca como `result` limitaciones, métodos y afirmaciones de literatura citada. Los campos
interpretativos suelen estar mal asignados (la verificación los marca `unchecked`). El NLI deja pasar el 62 %
de las oraciones elegibles y su umbral está sin calibrar. El lote completo está pendiente de mejorar la calidad.
`machine_verified` nunca equivale a validación científica.

## M6 · Escenarios, cobertura y huecos

```bash
uv run fif-scenarios validate    # esquema, fuentes y estado de scenarios/*.yaml y tolerances.yaml
uv run fif-scenarios build       # cobertura desde ensayos M2 + capa de hallazgos M4 + diagnóstico → vistas DuckDB
```

- **Escenarios** (`scenarios/`): Artemis (ancla), tránsito/ISS y Marte. Parámetros **provisionales y
  configurables**; cada valor indica su fuente (`challenge_statement`, `physical_reference`, `team_assumption`) y
  `verified`. Un escenario solo puede marcarse `verified` cuando todas sus fuentes lo están.
- **Tolerancias** (`scenarios/tolerances.yaml`, versión `t-1`, **exploratorias**): O₂ ±3 pp, presión ±15 %, flujo ±5 cm/s.
- **Cobertura experimental** (`scenario_trial_coverage`, `scenario_coverage`, `scenario_dimension_summary`):
  `in_range` (valor o rango completo dentro) / `overlaps` (un rango del ensayo, p. ej. una matriz de condiciones,
  solo toca el rango del escenario: no es cobertura) / `near` / `outside` / **`unknown`** (no registrado; nunca
  cuenta como fuera ni se imputa). Todos los
  resultados llevan `verification_level = draft_mapping_unreviewed` mientras los mapeos de M2 sigan en borrador.
- **Gravedad:** `gravity_g` permanece nulo. El `project_type` de PSI se guarda como `gravity_context` de la
  investigación, no como condición verificada del ensayo, y no implica equivalencia física con el escenario.
- **Huecos** (`scenario_gaps`), por familia: `system` (adquisición, procesamiento, extracción),
  `source_structure` (sin ensayos tabulados, con la causa comprobada), `not_assessable` (ningún ensayo registra
  la condición) y `scientific_possible` (posible hueco de conocimiento, solo dentro del corpus procesado, provisional
  y con la escasez marcada `unknown` mientras haya huecos de sistema abiertos).
- **Capa de hallazgos** (`scenario_findings_evidence`) y **huecos declarados por los autores** (`declared_gaps`):
  siempre con su `display_status`; un `future_work` no es un hueco científico confirmado.

## M5 · Normalización, vecinos y relaciones (provisional)

```bash
uv sync --extra ml               # transformers + torch (CPU) para embeddings
uv run fif-relations validate    # relations/config.yaml y vocab/concepts.yaml
uv run fif-relations build       # conceptos, embeddings, pares candidatos, relaciones, singularidad → vistas
```

- **Una corrida activa por unidad** (la más reciente que pasó a verificación); las anteriores quedan como
  historial (`m5_findings.active = false`, `superseded_by_run`) y nunca cuentan como evidencia independiente.
- **Vocabulario** (`vocab/concepts.yaml`, v-0, **borrador**): todos los conceptos `proposed`. Emparejamiento
  determinista; lo no emparejado queda `unmapped`. Las sugerencias por similitud (`vocab_proposals`) nunca se
  aprueban solas.
- **Embeddings** `intfloat/multilingual-e5-small` @ `614241f…` en CPU, prefijo `query: `, media + L2. Solo
  recuperan pares candidatos (τ y k exploratorios en `relations/config.yaml`). **Limitación conocida:** con los
  datos actuales τ = 0.80 no discrimina (pares no relacionados ≈ 0.80); los `related_topic` son candidatos
  exploratorios, nunca relaciones científicas.
- **Relaciones** (`relations`, reglas `rr-1`): exigen la misma variable y el mismo efecto normalizados, direcciones
  declaradas y condiciones comparadas por código. `consistent` requiere todas las condiciones esenciales conocidas
  y comparables; `extends_candidate` requiere además una declaración explícita de ampliación; con condiciones
  desconocidas la relación queda `unresolved_*`; `direction = none` solo con declaración explícita de ausencia de
  efecto. La similitud sola da `related_topic` (navegación).
- **Independencia:** pares de la misma fuente se excluyen; de la misma obra (aunque sean documentos distintos), con
  vínculo de obra pendiente o del mismo documento → no independientes.
- **Estado:** toda relación con hallazgos no revisados es `provisional`; `confirmed`/`rejected` solo desde
  `reviews/relations/*.yaml`. Una `contradiction_candidate` pasa a M6 como `contradiction_possible`, sujeta a revisión.

## M7 · API, panel y demostración

```bash
uv sync --extra ml --extra app
uv run fif-app check        # consistencia entre módulos (16 reglas; exit 1 si alguna falla)
uv run fif-app api          # API de solo lectura (http://127.0.0.1:8000/docs)
uv run fif-app panel        # panel Streamlit ES/EN (lee solo la API)
uv run fif-app snapshot     # congela las vistas en data/demo/demo.duckdb
uv run fif-app demo         # check + API + panel sobre la instantánea (ver docs/demo.md)
```

- Contrato JSON: `{"data": ..., "meta": {"versions": ..., "limitations": [...]}}`. Toda respuesta lleva las
  limitaciones activas: mapeos borrador, escenarios provisionales, tolerancias exploratorias, gravedad no
  registrada, texto sin procesar, hallazgos sin revisión, similitud no discriminativa y relaciones provisionales.
- Secciones del panel: Cobertura, Mapa de huecos (cobertura por condición, corte O₂ × presión, familias de huecos
  separadas, huecos declarados por los autores), Explorador, Estudio + contraste, Novedades y Consistencia.
- Insignias visibles: «verificado automáticamente · sin revisión científica», «provisional», «mapeo borrador»,
  «supuesto», «exploratorio».
