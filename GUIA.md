# Guía de uso — Resume-Matcher + Claude Code

Cómo levantar este setup desde cero y usar Claude Code para revisar y adaptar un CV a ofertas reales,
"como lo haría un ATS y un recruiter". Pensado para pasárselo a otro dev.

---

## 1. Cómo funciona

```
 vos ──pedido──▶ Claude Code ──./rmx <tool>──▶ Resume-Matcher (Docker, :3000)
                 (lee, escribe,                 guarda CVs/ofertas, valida que no se
                  extrae keywords)              inventó nada, score, ATS check, PDF
                                                        │
                                   UI http://localhost:3000 ◀── vos revisás y editás
```

- [Resume-Matcher](https://github.com/srbhr/Resume-Matcher) (`ghcr.io/srbhr/resume-matcher:1.4.0`) corre en modo
  **Coding agent** (`LLM_PROVIDER=agent`): **no llama a ningún LLM** y no necesita API keys.
- **Claude Code** hace la parte de IA usando el CLI de la app (`./rmx`, que ejecuta
  `python -m app.mcp_server.cli` dentro del contenedor). Todo lo que guarda aparece en la UI.
- Es la forma de usar tu plan de Claude que el propio Resume-Matcher recomienda: vos usás Claude Code; la app nunca
  llama a Claude.

**Privacidad**: el texto que Claude Code lee o escribe (tu CV, las ofertas) pasa por Anthropic, como cualquier
conversación con Claude, y cuenta para el uso de tu plan. El score, el ATS check y el PDF se calculan localmente.
Procesá sólo tu CV o el de alguien que lo haya aceptado.

## 2. Requisitos

- Docker Engine + Docker Compose v2 (probado con Docker 29 / Compose v5 en Linux).
- [Claude Code](https://docs.claude.com/en/docs/claude-code) instalado y logueado (`claude` en la terminal), con un
  plan de Claude o API key.

## 3. Puesta en marcha

Archivos del setup: `docker-compose.yml`, `rmx`, `README.md`, `GUIA.md`, `CLAUDE.md`, `.gitignore`.

```bash
cd ab_ats
mkdir -p cv jobs/.work output
chmod +x rmx
docker compose up -d
curl -s http://localhost:3000/api/v1/health     # {"status":"healthy"}
./rmx get_status                                # el CLI responde
```

Abrí http://localhost:3000. En **Settings** el provider debe figurar como **Coding agent** (viene así por env).

> **No compartas** `cv/`, `jobs/`, `output/`, `backups/` ni el volumen `ab_ats_rm-data`: tienen datos personales.

## 4. Usarlo con Claude Code

### 4.1 Arrancar
```bash
cd ab_ats
claude
```
Claude Code lee automáticamente `CLAUDE.md` (cómo usar `./rmx` y las reglas: no inventar datos, una adaptación por
oferta, exportar en 2 páginas).

**Opcional — menos confirmaciones**: por defecto Claude Code te pide permiso antes de cada comando. Si querés que corra
`./rmx` sin preguntar, decidilo vos y creá `.claude/settings.json` en esta carpeta (o usá `/permissions` dentro de Claude Code):
```json
{
  "permissions": {
    "allow": ["Bash(./rmx:*)", "Bash(docker compose ps:*)", "Bash(docker compose logs:*)"]
  }
}
```

### 4.2 Qué pedirle (en orden)

| Paso | Qué hacés vos | Qué le pedís a Claude |
|---|---|---|
| 1. ATS check del original | Copiás tu CV a `cv/` | *"Hacé el ATS check de cv/MiCV.pdf y decime qué falla"* |
| 2. CV maestro | — | *"Cargá cv/MiCV.pdf como master, copiando fielmente, y pasame el link para revisarlo"* |
| 3. Corregir datos | Le pasás tu CV viejo/LinkedIn si hace falta | *"Revisá que empresas y fechas coincidan con esto: …"* |
| 4. Mejorar redacción | — | *"Mejorá la redacción del master sin inventar números y mostrame qué cambiaste"* |
| 5. Oferta | Guardás la oferta en `jobs/empresa-rol.md` | *"Calculá el score de mi master contra jobs/empresa-rol.md y decime las keywords faltantes"* |
| 6. Versión adaptada | — | *"Adaptá mi CV a esa oferta, sin agregar nada que no tenga, y exportá el PDF"* |
| 7. Revisión recruiter | — | *"Hacé la revisión tipo recruiter del checklist de GUIA.md"* |
| 8. Datos que faltan | Respondés sus preguntas | *"Agregá este trabajo: …"* / *"Tengo estas métricas: …"* |

Los PDFs quedan en `output/`. Cada CV también se ve y edita en http://localhost:3000/resumes/<id>.

### 4.3 Buenas prácticas con Claude
- **Pasale datos reales**: métricas (tiempos, %, usuarios), nombres de empresas, fechas con mes. Sin eso, Claude no
  puede mejorar los bullets sin inventar, y no lo va a hacer.
- **Revisá cada cambio** que reporta antes de enviar el CV; pedile que te muestre el diff si querés.
- **Una oferta por archivo** en `jobs/`; para varias: *"Adaptá mi CV a todas las ofertas de jobs/"*.
- Si algo te parece una exageración (p. ej. una keyword que agregó), pedile que la saque: la versión adaptada se
  puede corregir sin rehacerla.

### 4.4 Otros agentes
Codex, Cursor o Copilot funcionan igual: tienen que poder correr `./rmx` en esta carpeta y seguir esta guía
(o `CLAUDE.md`, que podés copiar como `AGENTS.md`).

## 5. Qué hace Claude por debajo (referencia)

Útil para entender o depurar lo que hace el agente. Rutas de archivos **dentro del contenedor**: `/input/cv`,
`/input/jobs`, `/output`. JSON grandes por stdin desde `jobs/.work/`.

| Paso | Comandos |
|---|---|
| ATS check de un archivo | `./rmx parse_check_file file_path=/input/cv/MiCV.pdf` |
| Texto del CV | `./rmx extract_document_text file_path=/input/cv/MiCV.pdf` |
| Esquema del CV | `./rmx get_resume_schema` |
| Crear master | `./rmx create_master_resume - < jobs/.work/master.json` → `{"resume_data": {...}, "title": "..."}` |
| Leer / reemplazar CV | `./rmx get_resume resume_id=<id>` / `./rmx update_resume - < x.json` → `{"resume_id", "resume_data"}` |
| Revisión (páginas, issues, redacción) | `./rmx verify_resume resume_id=<id>` |
| Cargar oferta + keywords | `./rmx add_job - < x.json` → `{"job_description", "company", "role", "keywords": {required_skills, preferred_skills, experience_requirements, education_requirements, key_responsibilities, keywords}}` |
| Score CV↔oferta | `./rmx score_resume resume_id=<id> job_id=<job_id>` |
| Validar adaptación (dry-run) | `./rmx validate_tailored_resume - < x.json` |
| Guardar adaptación | `./rmx tailor_and_verify - < x.json` → `{"source_resume_id", "job_id", "resume_data", "title"}` |
| Recalcular score | `./rmx recalculate_ats_score resume_id=<id>` |
| Exportar PDF | `echo '{"resume_id":"<id>","output_path":"/output/X.pdf","print_settings":{"compactMode":true}}' \| ./rmx export_resume_pdf -` |
| Tracker de postulaciones | `./rmx list_applications` |

Esquema mínimo de `resume_data`:
```json
{"personalInfo": {"name": "", "title": "", "email": "", "phone": "", "location": "", "linkedin": "", "github": "", "website": ""},
 "summary": "",
 "workExperience": [{"id": 1, "title": "", "company": "", "location": "", "years": "Nov 2022 - Aug 2024",
                     "description": ["bullet"], "descriptionStyles": ["bullet"]}],
 "education": [{"id": 1, "institution": "", "degree": "", "years": "Nov 2016"}],
 "personalProjects": [],
 "additional": {"technicalSkills": ["Back-end: Django, ..."], "languages": [], "certificationsTraining": [], "awards": []}}
```

## 6. Reglas que hace cumplir la app (aprendidas)

- En una versión adaptada se mantienen **idénticos** al master: `personalInfo`, empleadores, títulos, fechas, cantidad
  de entradas y de bullets. `technicalSkills` sólo se puede **reordenar** (si no: `source preservation failed`).
  Para cambiar algo de eso, cambialo primero en el master.
- `update_resume` reemplaza **todo** el `resume_data`: siempre partir de `get_resume`.
- `tailor_and_verify` crea un CV nuevo y una tarjeta en el tracker cada vez: **una sola vez por oferta**.
- `descriptionStyles` debe tener el mismo largo que `description`.
- Mezclar formatos de fecha (`2015 - 2019` con `Mar 2022 - Present`) baja `date_consistency`: usar `Mon YYYY` en todo.
- Reducciones > 100% ("decreased 400%") son incorrectas: usar valores absolutos (15 → 4 min).
- `grounding_warnings` tipo "Review summary against the source resume" son avisos: verificar contra el master.

## 7. Checklist de revisión tipo recruiter

- [ ] Cargo y seniority arriba.
- [ ] Stack visible (summary/skills y dentro de la experiencia).
- [ ] Bullets con impacto: verbo + qué + resultado medible (sin inventar números).
- [ ] Fechas con el mismo formato, sin solapamientos sin explicar.
- [ ] Ubicación, remote/híbrido y zona horaria.
- [ ] 1–2 páginas.
- [ ] Empresas con nombre real (anónimas restan credibilidad, salvo NDA declarado).

## 8. Formato del PDF (2 páginas)

La plantilla `swiss-single` sin compactar puede mover toda la sección Experience a la página 2.
- **UI**: editor del CV → **Plantilla y formato** → **Opciones** → **Modo compacto** (se guarda por CV).
- **API**:
  ```bash
  curl -X PATCH -H 'Content-Type: application/json' http://localhost:3000/api/v1/resumes/<id>/template-settings -d \
  '{"template":"swiss-single","pageSize":"A4","margins":{"top":10,"bottom":10,"left":10,"right":10},
    "spacing":{"section":3,"item":2,"lineHeight":3},
    "fontSize":{"base":3,"headerScale":3,"headerFont":"serif","bodyFont":"sans-serif"},
    "compactMode":true,"showContactIcons":false,"accentColor":"blue"}'
  ```
- Para empresas de EE. UU. podés usar `"pageSize":"LETTER"` (verificar que siga en 2 páginas).

## 9. Troubleshooting

| Síntoma | Causa / solución |
|---|---|
| Claude pide permiso para cada `./rmx` | Es el comportamiento por defecto; ver §4.1 "Opcional — menos confirmaciones" |
| Botones de IA en la UI muestran un prompt para pegar | Es el modo agent: pedíselo a Claude Code en la terminal |
| Settings muestra otro provider | Se guardó en la UI y pisa al env: elegir **Coding agent** |
| `No such container: resume-matcher` | `docker compose up -d` |
| `render_failed` al exportar | El frontend no responde: `docker compose ps` / `docker compose logs -f resume-matcher` |
| PDF en 3 páginas con la página 1 casi vacía | Activar Modo compacto (§8) |
| Archivo no encontrado en el CLI | Las rutas son del contenedor: `/input/cv`, `/input/jobs`, `/output` |
| CV con `processing_status: failed` | Borrarlo: `curl -X DELETE http://localhost:3000/api/v1/resumes/<id>` |
