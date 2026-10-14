# knowledge-hub-n8n

Repositorio de los 5 flujos de n8n que construyen la **memoria semántica self-hosted** del Knowledge Hub, el episodio 2 de la serie self-hosted con Coolify. Estos flujos conectan tu wiki **Outline** con una base de datos **PostgreSQL + pgvector** y un servidor de **embeddings TEI** propio: leen los documentos de Outline, los trocean, calculan sus vectores, los guardan en pgvector y exponen una API RAG de búsqueda semántica (más un flujo de salud y otro de reindexado automático). Todo corre en tu infraestructura, sin enviar un solo token a un proveedor cloud.

Todo el trabajo pesado de cómputo (embeddings) lo hace un servidor **TEI** (Text Embeddings Inference) self-hosted que usa el modelo `intfloat/multilingual-e5-small`. n8n solo orquesta: pide documentos, trocea, llama al servidor de embeddings y habla con Postgres.

## Arquitectura

```
                         ┌──────────────────────────┐
                         │   Outline (wiki)         │
                         └────────────┬─────────────┘
                                      │
                       ┌──────────────┴──────────────┐
                    (01) indexar                 (04) indexar repo
                  POST /rag-index            POST /rag-index-repo
                       │                              │
                       ▼                              ▼
          ┌───────────────────────────────────────────────┐
          │         PostgreSQL + pgvector                  │
          │  knowledge_documents  ·  knowledge_chunks      │
          │  embedding vector(384) · índice HNSW           │
          └───────────────────────────────────────────────┘
                       ▲                              │
                    (02) búsqueda                  (03) salud
                  POST /rag-search              GET /rag-health
                       │                              │
                       └──────────────┬───────────────┘
                                      │
                    ┌─────────────────┴──────────────────┐
                    │  TEI  intfloat/multilingual-e5-small│
                    │  embeddings:80  (compartido 01/02/04)│
                    └─────────────────────────────────────┘

              ┌──────────────────────┐
          (05)  │  schedule cada hora  │──► dispara (01) vía POST /rag-index
              └──────────────────────┘
```

El servidor de embeddings (TEI) es **compartido**: los flujos 01, 02 y 04 lo usan. El 01 embebe los fragmentos al indexar Outline (`passage:`), el 02 embebe la pregunta del usuario (`query:`), y el 04 embebe los fragmentos del repositorio (`passage:`). El 03 nunca embebe: solo cuenta filas. El 05 no habla con la base de datos ni con TEI: solo hace una llamada HTTP al 01.

## Flujos, nodo por nodo

Los ficheros viven en `workflows/`. Cada flujo empieza con un nodo `Configuración` (excepto el 03) donde se editan las constantes del entorno. El 03 es el único que no necesita constantes porque solo consulta Postgres.

### 01 - Indexar Outline (`workflows/01-index-outline.json`)

**Objetivo:** leer todos los documentos de Outline, trocearlos, embebirlos y guardarlos en pgvector. Es **incremental**: si un documento no cambió, no se vuelve a trocear ni a embeber.

Orden de nodos:

1. **Indexar ahora** (`manualTrigger`) — disparador manual para lanzar el indexado desde el editor de n8n.
2. **Webhook indexar** (`webhook`, `POST /webhook/rag-index`) — el otro disparador, la ruta HTTP que usará el flujo 05 (y cualquier cliente externo). Protegido con `authentication: headerAuth` (credencial con la cabecera `X-RAG-API-Key`).
3. **Configuración** (`set`) — define tres constantes: `outline_url` (tu dominio de Outline, p. ej. `https://docs.example.com`), `embeddings_url` (p. ej. `http://embeddings:80/embed`) y `embedding_model` (`intfloat/multilingual-e5-small`). El resto de nodos las leen con `$('Configuración').first().json.<campo>`. Ambos disparadores entran aquí.
4. **Listar documentos** (`httpRequest`) — `POST {outline_url}/api/documents.list` con `limit: 100`, `offset: 0`. Es la API de Outline (misma credencial que da acceso con token). Devuelve la lista paginada; Outline incluye un `nextPath` para pedir la siguiente página. Tiene `retryOnFail` con 3 intentos.
5. **Normalizar lista** (`code`) — convierte la respuesta paginada en **un item por documento** (objeto `{ json: { id } }`), aceptando varias formas de respuesta (array, `data`, `documents`, `data.documents`), y elimina duplicados por `id` con un `Set`. Es un good example del patrón que verás en todos los Code nodes: usa `$input.all()` y devuelve un array.
6. **Leer documento** (`httpRequest`) — `POST {outline_url}/api/documents.info` con el `id` de cada item. Devuelve el cuerpo Markdown completo del documento.
7. **Normalizar documento** (`code`) — limpia el texto (normaliza saltos de línea, colapsa espacios y líneas en blanco), calcula la URL canónica (`outline_url` + `doc.url`) y construye el item final: `source_type: 'outline'`, `external_id` (el id de Outline), `title`, `collection`, `content` (título + texto, para que el título entre en el embedding), `canonical_url` y `metadata` (collection, parent, revision, updatedAt…).
8. **Guardar documento** (`postgres`, `executeQuery`) — el **upsert incremental**. Inserta o actualiza en `knowledge_documents` por la clave `(source_type, external_id)`. El `content_hash` es un `md5()` del cuerpo: si el hash no cambió, el `DO UPDATE ... WHERE content_hash IS DISTINCT FROM ...` **no actualiza nada**. Devuelve `document_id` y un booleano `changed` que dice si el contenido es distinto del que ya había (o si es nuevo).
9. **¿Documento cambiado?** (`if`) — rama `true` solo si `changed` es verdadero. Si el documento no cambió, la ejecución del item termina aquí: no se trocea ni se embebe.
10. **Trocear documento** (`code`) — **chunking**. Parte el texto en fragmentos de ~1.400 caracteres con un **solape** de 220 caracteres entre fragmentos consecutivos. Antes de cortar, retrocede hasta el último límite natural (párrafo `\n\n`, salto de línea, fin de frase `. ` o espacio) si está a más de 450 caracteres del inicio del fragmento. Cada fragmento guarda `chunk_index`, un `token_count` estimado (~1,3 × palabras) y metadata con el índice.
11. **Generar embeddings** (`httpRequest`) — `POST` a `embeddings_url` con `inputs = 'passage: ' + contenido`. El prefijo **`passage:` es obligatorio** en los modelos E5 (ver más abajo). Una llamada por fragmento.
12. **Normalizar embedding** (`code`) — valida que el vector tiene exactamente **384 dimensiones** (si no, lanza error) y lo asocia a su fragmento de origen (`$('Trocear documento').all()[i]`) para no perder la relación chunk ↔ vector.
13. **Guardar fragmentos** (`postgres`, `executeQuery`) — inserta o actualiza cada fragmento en `knowledge_chunks` por la clave `(document_id, chunk_index)`, con el vector serializado como `'[0.1,0.2,…]'` casteado a `vector`. El `ON CONFLICT ... DO UPDATE` hace que reindexar un chunk lo sobrescriba en lugar de duplicarlo.
14. **Preparar limpieza** (`code`) — calcula, por documento, el mayor `chunk_index` insertado (`keep_from_chunk_index = max + 1`). Sirve para detectar la cola sobrante.
15. **Borrar fragmentos sobrantes** (`postgres`, `executeQuery`) — `DELETE ... WHERE document_id = $1 AND chunk_index >= $2`. Si editaste un documento y ahora es **más corto** que antes, sus últimos fragmentos quedarían huérfanos; este nodo los elimina.
16. **Notas indexador** (`stickyNote`) — nota visual, no ejecutable. Documenta el flujo dentro del lienzo.

### 02 - Buscar (RAG) (`workflows/02-rag-search.json`)

**Objetivo:** recibir una pregunta, embeberla y devolver los fragmentos más cercanos de pgvector, con su documento de origen. Es la API RAG que consume cualquiera (un chat, un MCP, un script).

Orden de nodos:

1. **Webhook búsqueda** (`webhook`, `POST /webhook/rag-search`) — `responseMode: responseNode` (la respuesta la construyen los nodos `respondToWebhook`, no el webhook). Autenticado con `headerAuth`. Cuerpo esperado: `{ query, limit?, source_type? }`.
2. **Configuración** (`set`) — las mismas tres constantes que el 01. Aquí solo se usa `embeddings_url`; `outline_url` y `embedding_model` quedan por consistencia.
3. **Validar petición** (`code`) — normaliza el cuerpo: exige `query` no vacía; acota `limit` entre 1 y 20 (por defecto 5); valida `source_type` contra `outline | repo | all` (por defecto `all`). Devuelve `{ valid: true/false, error? }`.
4. **¿Petición válida?** (`if`) — rama `true` → `Embeddings de consulta`; rama `false` → `Responder no válida`.
5. **Embeddings de consulta** (`httpRequest`) — `POST` a `embeddings_url` con `inputs = 'query: ' + <pregunta>`. El prefijo **`query:` es obligatorio** en los modelos E5 (distinto del `passage:` de la indexación). `onError: continueErrorOutput` para que un fallo salte a `Responder error` en lugar de romper el webhook.
6. **Normalizar embedding** (`code`) — valida las 384 dimensiones del vector de la consulta y lo deja como `query_embedding`. Igual que en el 01, pero aquí solo hay un vector.
7. **Búsqueda semántica** (`postgres`, `executeQuery`) — la consulta vectorial:

   ```sql
   SELECT d.id, d.source_type, d.title, d.canonical_url, d.collection,
          c.chunk_index, c.content,
          ROUND((1 - (c.embedding <=> $1::vector))::numeric, 6) AS score
   FROM knowledge_chunks c
   JOIN knowledge_documents d ON d.id = c.document_id
   WHERE ($2::text = 'all' OR d.source_type = $2::text)
   ORDER BY c.embedding <=> $1::vector
   LIMIT LEAST(GREATEST($3::int, 1), 20)
   ```

   El operador **`<=>`** es la **distancia coseno** de pgvector: 0 = idéntico, 2 = opuesto. `ORDER BY embedding <=> $1` ordena de más a menos parecido (el índice HNSW acelera justamente esta ordenación). El `score` que devolvemos es `1 - distancia`, es decir una similitud entre 0 y 1 (1 = idéntico). El filtro `source_type` permite buscar solo en Outline (`outline`), solo en el repositorio (`repo`) o en todo (`all`).
8. **Responder resultados** (`respondToWebhook`, 200) — devuelve los fragmentos con su score y su documento de origen.
9. **Responder no válida** (`respondToWebhook`, 400) — `{ error: 'invalid_request' }` si la petición está mal formada.
10. **Responder error** (`respondToWebhook`, 502) — `{ error: 'search_unavailable' }` si falla el servidor de embeddings o la base de datos.
11. **Notas búsqueda** (`stickyNote`) — nota visual.

### 03 - Salud del índice (`workflows/03-rag-health.json`)

**Objetivo:** endpoint barato para saber cuánto hay indexado sin exponer contenido ni vectores.

Orden de nodos:

1. **Webhook salud** (`webhook`, `GET /webhook/rag-health`) — `responseMode: responseNode`, autenticado con `headerAuth`.
2. **Leer salud** (`postgres`, `executeQuery`) — una consulta con subselects que cuenta documentos y fragmentos por origen (`outline` / `repo`), totales, cuántos fragmentos tienen `embedding_model` y la última actualización (`MAX(updated_at)`) de cada origen. **No devuelve ni `content` ni `embedding`**, solo contadores y fechas.
3. **Responder salud** (`respondToWebhook`, 200) — devuelve los contadores.
4. **Responder error** (`respondToWebhook`, 503) — `{ error: 'health_unavailable' }` si Postgres no responde.
5. **Notas salud** (`stickyNote`) — nota visual.

### 04 - Indexar repositorio (`workflows/04-index-repository.json`)

**Objetivo:** indexar ficheros Markdown de un repositorio que **ya llegan preparados** en el cuerpo del webhook (los envía el cliente, p. ej. un MCP con `knowledge_index_repository`). A diferencia del 01, aquí no hay que listar ni leer: el contenido viene dado.

Orden de nodos:

1. **Webhook repositorio** (`webhook`, `POST /webhook/rag-index-repo`) — recibe un lote de documentos. Autenticado con `headerAuth`.
2. **Configuración** (`set`) — mismas constantes que el 01.
3. **Normalizar documentos** (`code`) — valida el lote: extrae el array de documentos, **exige `replace_all: true`** en el cuerpo (si no, lanza error) y exige que haya documentos. Filtra por `source_type === 'repo'`, descarta ids vacíos, títulos vacíos y contenido vacío o mayor de 200.000 caracteres, y deduplica por `external_id`. Cada documento queda con `source_type: 'repo'`, `external_id`, `title` (recortado a 200), `collection` (por defecto `repositorio`), `content` limpio, `canonical_url` y `metadata`.
4. **Guardar documento** (`postgres`, `executeQuery`) — **el mismo upsert incremental** que el 01 (mismo `md5(content)` y mismo `changed`). Por eso el flujo 01 y el 04 comparten esquema y lógica de cambio.
5. **¿Documento cambiado?** (`if`) — solo procesa los documentos cuyo `content_hash` cambió.
6. **Trocear documento** (`code`) — idéntico al 01: fragmentos de ~1.400 con solape de 220, corte en límite natural.
7. **Generar embeddings** (`httpRequest`) — `POST` a `embeddings_url` con `passage: ` (igual que el 01).
8. **Normalizar embedding** (`code`) — valida 384 dimensiones y asocia al fragmento.
9. **Guardar fragmentos** (`postgres`, `executeQuery`) — upsert en `knowledge_chunks` por `(document_id, chunk_index)`.
10. **Preparar limpieza** (`code`) — calcula el `keep_from_chunk_index` por documento (igual que el 01).
11. **Borrar fragmentos sobrantes** (`postgres`, `executeQuery`) — borra la cola de fragmentos obsoletos del documento que se acortó.

    A partir de aquí, la rama de **limpieza del repositorio** (que sale del nodo 3 al mismo tiempo que la rama de indexado):

12. **Preparar limpieza repo** (`code`) — reúne **todos** los `external_id` del lote actual y los une con `\n` en `external_ids_text`. La idea: este lote es el estado **completo** del repositorio (`replace_all: true`).
13. **Borrar fragmentos repo** (`postgres`, `executeQuery`) — borra los fragmentos de documentos `repo` cuyo `external_id` **no está en el lote actual** (`NOT (external_id = ANY(string_to_array($1, E'\n')))`). Es decir: ficheros que ya no existen en el repo.
14. **Borrar documentos repo** (`postgres`, `executeQuery`) — borra también la fila del documento huérfano en `knowledge_documents` (misma condición). Así el índice refleja **exactamente** el repositorio, no una acumulación histórica.
15. **Notas repositorio** (`stickyNote`) — nota visual.

### 05 - Reindexado automático (`workflows/05-auto-reindex.json`)

**Objetivo:** relanzar el indexado de Outline cada hora sin intervención humana (ver la sección *Reindexado automático*).

Orden de nodos:

1. **Cada hora** (`scheduleTrigger`) — dispara cada 1 hora (`hoursInterval: 1`).
2. **Configuración** (`set`) — aquí hay **una constante extra**: `n8n_url` (p. ej. `https://n8n.example.com`), además de `outline_url`, `embeddings_url` y `embedding_model`. `n8n_url` es la base para llamar a su propio webhook.
3. **Disparar indexado** (`httpRequest`) — `POST {n8n_url}/webhook/rag-index` con `authentication: genericCredentialType` / `genericAuthType: httpHeaderAuth` (la credencial `X-RAG-API-Key` del 01). Así el 05 reutiliza exactamente el mismo camino que un cliente externo. `retryOnFail` con 3 intentos.
4. **Notas auto-reindexado** (`stickyNote`) — nota visual que explica la estrategia.

## Modelo de embeddings

Usamos **`intfloat/multilingual-e5-small`**, servido por **TEI** (Text Embeddings Inference) en el repo `coolify-embeddings`. Motivos:

- **Multilingüe.** Está entrenado para alinear decenas de idiomas; el contenido de tu wiki y tus repos puede mezclar español, inglés y más sin perder calidad. Un modelo solo-inglés degradaría las búsquedas en español.
- **384 dimensiones.** Vectores pequeños: la tabla `vector(384)` y el índice HNSW ocupan y comparan mucho menos que un modelo de 768 o 1536 dims. Para un índice de conocimiento personal, 384 dims sobran.
- **~120 MB de modelo.** Cabe y corre bien en una CPU modesta del VPS, sin GPU. La inferencia es rápida porque el modelo es *small*.

**Coste de indexar 1.000 documentos.** Tomando documentos de tamaño habitual (unos 4.000 caracteres) salen ~4 fragmentos por documento tras el solape, es decir ~4.000 chunks. A ~1.400 caracteres por chunk, cada uno son ~350 tokens, así que ~1,4 M tokens en total. Con TEI self-hosted el **coste por token es cero**: lo único que pagas es la CPU de tu VPS, que en un modelo *small* y un lote de este tamaño son unos pocos minutos. Suponiendo ~0,02 $/hora de VPS, indexar 1.000 documentos cuesta céntimos de fracción.

**Alternativa cloud (honesta).** Si no quieres mantener TEI, `text-embedding-3-small` de OpenAI está sobre **0,02 $ por millón de tokens**, lo que para esos ~1,4 M tokens son del orden de **1-2 céntimos por 1.000 documentos**. El precio no es el problema; el problema es que devuelve **1536 dimensiones**, así que cambiarías de esquema (`vector(384)` → `vector(1536)`) y el índice HNSW habría que reconstruirlo. Además enviarías tu wiki y tus repos a un tercero, que es justo lo que este proyecto evita. La arquitectura está pensada para el modelo local; el proveedor cloud es solo una salida de emergencia documentada.

> Nota sobre los **prefijos E5**: los modelos E5 están entrenados con prefijos asimétricos. Los documentos se embeben con `passage: ` y las consultas con `query: `. Si los olvidas, la calidad de la búsqueda cae en picado porque el modelo produce vectores fuera de la distribución que espera. Por eso el 01 y el 04 usan `passage:` y el 02 usa `query:`.

## Reindexado automático (flujo 05)

**Estrategia elegida: polling horario.**

El motivo es que **Outline Open Source no envía webhooks cuando un documento se crea, cambia o se borra**. No hay un evento al que suscribirse (eso es característica de la versión Enterprise). Sin una señal de cambio, las opciones son dos: reindexar todo cada vez (caro e innecesario) o preguntar periódicamente. Elegimos preguntar.

Cada hora, el flujo 05 llama a `POST /webhook/rag-index` del flujo 01. El indexado del 01 es **incremental por `content_hash`**: por cada documento calcula el `md5` del contenido y lo compara con el que ya está en la base. Si el hash es igual, el upsert no modifica la fila y el nodo `¿Documento cambiado?` corta la ejecución para ese documento. Consecuencia: **solo se trocea y se embebe lo que realmente cambió**. Una pasada completa sobre un wiki que no ha cambiado no genera ni una llamada a TEI; una pasada sobre un wiki con un documento editado procesa ese documento y nada más.

El intervalo de una hora es un compromiso: suficientemente fresco para que las búsquedas reflejen los cambios del día, suficientemente espaciado para que el coste sea despreciable. Si necesitas frescura inmediata, puedes llamar al webhook del 01 a mano (o desde un hook de tu editor) en lugar de esperar a la siguiente hora; el 05 solo se encarga de que nada quede sin indexar por olvido.

## Puesta en marcha

Requisitos: tener ya desplegados (con Coolify) Outline, PostgreSQL + pgvector, el servidor TEI de embeddings y n8n 2.10.2.

1. **Importar los flujos.** En n8n: *Workflows → Import from File* y sube los 5 JSON de `workflows/`. Activará los webhooks cuando actives cada flujo (los `POST`/`GET` de la sección siguiente). Los flujos se importan inactivos; actívalos cuando estén configurados.
2. **Crear las credenciales** (n8n *Credentials → New*):
   - **PostgreSQL**: host `postgres`, puerto `5432`, tu base de datos, usuario y contraseña. Es la credencial que usan los nodos `postgres` de los flujos 01, 02, 03 y 04.
   - **Header Auth "Outline API"**: cabecera `Authorization`, valor `Bearer <token>`. Es el token de la API de Outline. La usan los nodos HTTP que llaman a `/api/documents.*`.
   - **Header Auth "RAG API Key"**: cabecera `X-RAG-API-Key`, valor `<clave>`. La usan los webhooks (para autenticar a quien llama) y el nodo `Disparar indexado` del 05 (para autenticarse contra el 01).
3. **Editar el nodo `Configuración` de cada flujo.** Es el paso que más se olvida: las constantes están en el lienzo, no en credenciales. En los flujos 01, 02 y 04 pon `outline_url` (tu dominio de Outline), `embeddings_url` (`http://embeddings:80/embed`) y `embedding_model` (`intfloat/multilingual-e5-small`). En el 05 añade `n8n_url` (`https://n8n.example.com`), la URL pública de tu propia instancia de n8n.
4. **Asignar las credenciales a cada nodo.** Al importar, n8n deja los nodos HTTP y Postgres sin credencial. Abre cada nodo HTTP (`Listar documentos`, `Leer documento`, `Generar embeddings`, etc.) y selecciona la credencial `httpHeaderAuth` correspondiente; abre cada nodo `postgres` y selecciona la de Postgres. El 05 usa en `Disparar indexado` la credencial `X-RAG-API-Key`.
5. **Comprobar que el esquema existe.** La base de datos debe tener las tablas `knowledge_documents` y `knowledge_chunks` con `embedding vector(384)` y el índice HNSW (lo crea el repo `coolify-pgvector`, imagen `pgvector 0.8.7` sobre PostgreSQL 17).
6. **Primer indexado.** Ejecuta el flujo 01 con *Execute Workflow* (disparador manual `Indexar ahora`) y luego consulta `GET /webhook/rag-health` para ver los contadores.
7. **Activar los flujos.** Deja activos el 01 (para que su webhook responda), el 02, el 03 y el 05. El 04 lo puedes dejar activo también si vas a indexar repositorios.

## Red entre recursos en Coolify

Los tres recursos implicados (PostgreSQL+pgvector, TEI de embeddings y n8n) tienen que verse entre sí **por nombre de servicio dentro de la misma red Docker**. En Coolify esto se hace marcando **Connect to Predefined Network** en cada recurso (la red predefinida del stack donde vive n8n), de forma que Docker los resuelve por su nombre de servicio en lugar de por IP.

Por eso las URLs de las constantes usan el **nombre del servicio**, no una IP ni un dominio:

- Postgres: host `postgres`, puerto `5432` (en la credencial de PostgreSQL).
- Embeddings: `http://embeddings:80/embed` (en `embeddings_url`). El puerto 80 es el puerto **interno** del contenedor TEI, no un puerto publicado al exterior.
- n8n llama a Outline por su dominio público (`outline_url`), porque Outline se expone con Traefik.

La clave: **el nombre del servicio es el DNS**. `embeddings:80` funciona porque n8n y el contenedor de embeddings están en la misma red. Si no marcas *Connect to Predefined Network*, cada recurso queda en su propia red aislada, `embeddings` no resuelve y los nodos HTTP fallan con `ENOTFOUND`. No uses `localhost` ni `127.0.0.1`: cada contenedor es su propio `localhost`, no el del vecino.

## Webhooks

| Método | Ruta | Flujo | Descripción |
|--------|------|-------|-------------|
| `POST` | `/webhook/rag-index` | 01 | Lanza el indexado incremental de Outline. |
| `POST` | `/webhook/rag-search` | 02 | Búsqueda semántica. Cuerpo: `{ query, limit?, source_type? }`. |
| `GET`  | `/webhook/rag-health` | 03 | Contadores del índice. |
| `POST` | `/webhook/rag-index-repo` | 04 | Indexa un lote de Markdown de repositorio (requiere `replace_all: true`). |

Todos exigen la cabecera **`X-RAG-API-Key: <clave>`** (credencial *Header Auth*). Ejemplo:

```bash
curl -X POST https://n8n.example.com/webhook/rag-search \
  -H 'X-RAG-API-Key: <clave>' \
  -H 'Content-Type: application/json' \
  -d '{"query": "cómo despliego con Coolify", "limit": 5, "source_type": "all"}'
```

## Trampas reales

Estas son las que costó encontrar. Si te saltas una, el flujo falla de una forma que no apunta al problema.

**(a) Los Code nodes de n8n corren "una vez para todos los items" por defecto.** El modo de ejecución de un nodo Code en n8n 2.x es **Run Once for All Items**: tu código recibe todos los items de golpe y **debe devolver un array**. El error clásico es escribir un `return { json: ... }` con un solo objeto. Con eso **colapsas todo a un item**: si venían 50 chunks, ahora hay 1, y el resto del flujo trabaja sobre datos incompletos sin dar ningún error. La regla: usa `$input.all()` para leer, recorre el array y devuelve un array de `{ json: ... }`. Todos los Code nodes de estos flujos lo hacen así.

**(b) Los nodos Postgres necesitan `options.queryReplacement` + `queryBatching: "independently"` para pasar los `$1..$n`.** Si escribes la query con marcadores de posición (`WHERE id = $1`) pero no configuras el reemplazo, Postgres no recibe valores y la query falla o devuelve nada. Tienes que poner en `options`:

```json
"queryBatching": "independently",
"queryReplacement": "={{ [$json.a, $json.b, ...] }}"
```

El array del `queryReplacement` se mapea posición a posición con `$1, $2, $3…`. Con `queryBatching: "independently"` cada item se ejecuta como su propia query (un INSERT por fragmento), que es lo que queremos aquí; con el batching por defecto, los items se agrupan y los `$1..$n` no cuadran.

**(c) Los nodos HTTP necesitan `authentication: "genericCredentialType"` + `genericAuthType: "httpHeaderAuth"`, o no envían la cabecera.** Es fácil configurar la credencial de cabecera, asignarla… y olvidar estos dos campos en el nodo. El resultado es un **401** silencioso: n8n no manda la cabecera `Authorization`/`X-RAG-API-Key` porque el nodo no está configurado para usar autenticación genérica. Los dos campos van juntos: `genericCredentialType` habilita la autenticación por credencial, y `genericAuthType: "httpHeaderAuth"` dice que el tipo es cabecera HTTP. Sin ambos, la cabecera no sale y Outline (o tu propio webhook) rechaza la llamada.

## Estructura del repositorio

```
knowledge-hub-n8n/
├── LICENSE
├── README.md
├── .gitignore
└── workflows/
    ├── 01-index-outline.json      # Outline → chunking → embeddings → pgvector
    ├── 02-rag-search.json         # pregunta → embedding → búsqueda coseno
    ├── 03-rag-health.json         # contadores del índice
    ├── 04-index-repository.json   # Markdown de repo → embeddings → pgvector
    └── 05-auto-reindex.json       # schedule horario → dispara el 01
```

Los flujos **no incluyen credenciales**: los JSON no contienen tokens, contraseñas ni claves. Al importarlos en tu n8n, los nodos quedan sin credencial asignada y las creas tú (Postgres, *Header Auth* "Outline API" y *Header Auth* "RAG API Key"). Esto es intencionado: cada instalación tiene sus propios secretos.

## Recursos de la serie

- `coolify-pgvector` — PostgreSQL 17 con pgvector, tablas `knowledge_documents` y `knowledge_chunks`, `vector(384)` e índice HNSW.
- `coolify-embeddings` — servidor TEI con `intfloat/multilingual-e5-small` (384 dims, prefijos `passage:`/`query:`).
- `knowledge-hub-n8n` — este repo: los flujos que orquestan todo.

## Licencia

MIT © Eduardo Navajas
