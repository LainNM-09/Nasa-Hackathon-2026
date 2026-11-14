# Fixtures recuperados del levantamiento original de PSI

Fuente: salida original de Colab compartida por el usuario y conservada en la biblioteca como `Pasted markdown(1).md`, 9 noviembre 2026.

## Contenido
- `filter_options_response.json`: **respuesta JSON completa** de GET /repo/filter-options, reconstruida fielmente desde la salida textual (sólo se eliminaron escapes Markdown de presentación).
- `repo_search_first_hit_PSI-119.json`: **un registro real íntegro** dentro de `hits.hits` de GET /repo/search?source=psi&type=investigation&from=0&size=5. No es toda la respuesta de búsqueda; ésta quedó truncada en el archivo conservado.
- `files_v2_request_PSI-142.json`: **cuerpo de solicitud real** usado para POST /files/v2; no es una respuesta.
- `repo_search_original_truncated.txt`: salida pegada de búsqueda conservada, incluyendo su corte, para revisar procedencia.

## No recuperado literalmente
- Respuesta JSON íntegra de GET /repo/search.
- Respuesta JSON de GET /repo/investigations/{accession}.
- Respuesta JSON íntegra de POST /files/v2. La conversación anterior constata que sí fue pegada en Colab para PSI-142, pero su cuerpo literal no está accesible en los archivos recuperados.

No usar un fragmento como fixture de respuesta completa. Los endpoints provienen de pruebas, pero pueden cambiar.
