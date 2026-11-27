# ab_ats — instrucciones para Claude Code

Resume-Matcher 1.4.0 corre en Docker (contenedor `resume-matcher`, http://localhost:3000) en modo **agent**:
la app no llama a ningún LLM; **vos** hacés la parte de IA con `./rmx <tool>` y la app valida, puntúa,
hace el ATS check y genera el PDF. Flujo completo, formatos JSON y reglas: **GUIA.md** (leela antes de empezar).

## Cómo trabajar
- Respondé en el idioma del usuario. Antes de nada: `curl -s http://localhost:3000/api/v1/health`; si falla, `docker compose up -d`.
- `./rmx --help` lista herramientas; `./rmx <tool> --help` muestra el schema de argumentos.
- Rutas de archivos se leen **dentro del contenedor**: `/input/cv`, `/input/jobs`, `/output`.
- JSON grandes: escribilos en `jobs/.work/` y pasalos por stdin (`./rmx <tool> - < jobs/.work/x.json`).
- `update_resume` reemplaza todo el `resume_data`: partí siempre de `get_resume`.
- `tailor_and_verify` **una sola vez por oferta**; los ajustes posteriores con `update_resume` + `recalculate_ats_score` + `verify_resume`.
- Exportá con `print_settings: {"compactMode": true}` a `/output/...` y confirmá `page_count` ≤ 2 y `parse_check_file`.

## Reglas
- Nunca inventes empleadores, títulos, fechas, métricas, skills ni responsabilidades. Si falta un dato, preguntá.
- Copiá el CV original **fielmente** al crear el master; mejorá la redacción en un paso aparte y mostrá qué cambiaste.
- Las keywords faltantes que el CV no respalda se reportan como gaps, no se agregan.
- No borres CVs, ofertas ni volúmenes sin confirmación del usuario.
- Sólo procesá CVs del usuario o de personas que aceptaron que su CV pase por Anthropic.
