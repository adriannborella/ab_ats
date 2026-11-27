# ab_ats — Resume-Matcher + Claude Code

Revisión de CVs "como un ATS y un recruiter" con [Resume-Matcher](https://github.com/srbhr/Resume-Matcher) 1.4.0 en
Docker, operado con **Claude Code**. Sin API keys.

- **Resume-Matcher** (http://localhost:3000) guarda CVs y ofertas, calcula score de keywords, hace el ATS parse-check,
  revisa la redacción y genera el PDF. Corre en modo **Coding agent**: no llama a ningún LLM.
- **Claude Code** hace la parte de IA (leer el CV, extraer keywords, adaptar el CV a cada oferta) usando `./rmx`.

Guía completa (instalación, cómo pedirle cosas a Claude, flujo, reglas, troubleshooting): **[GUIA.md](GUIA.md)**.

> Privacidad: el contenido del CV que procesa Claude Code pasa por Anthropic (igual que cualquier conversación con
> Claude). Score, ATS check y PDF son locales. Puerto publicado sólo en `127.0.0.1`.

## Inicio rápido

```bash
mkdir -p cv jobs/.work output
docker compose up -d
cp ~/MiCV.pdf cv/                 # tu CV
claude                            # Claude Code en esta carpeta
```
Y pedile: *"Cargá mi CV de cv/MiCV.pdf como master y hacé el ATS check"*.

## Estructura

```
docker-compose.yml     servicio resume-matcher (modo agent)
rmx                    atajo al CLI de la app dentro del contenedor
GUIA.md                guía de uso
CLAUDE.md              instrucciones que Claude Code lee automáticamente
cv/        -> /input/cv    (ro)  CV original PDF/DOCX
jobs/      -> /input/jobs  (ro)  ofertas .md/.txt; jobs/.work = JSON temporales
output/    -> /output      (rw)  PDFs exportados
```

`cv/`, `jobs/`, `output/` y `backups/` tienen datos personales: están en `.gitignore`, no se comparten.

## Comandos

```bash
docker compose up -d                  # levantar
docker compose down                   # bajar (el volumen con los datos queda)
docker compose ps
docker compose logs -f resume-matcher
./rmx --help                          # herramientas del CLI
./rmx list_resumes
```

## PDF en 2 páginas

Activá **Plantilla y formato → Opciones → Modo compacto** en el editor de cada CV (queda guardado por CV).
Sin eso, la plantilla puede mandar toda la sección Experience a la página 2.

## Backup, restore y borrado

```bash
# Backup (contiene CVs: guardalo seguro)
mkdir -p backups
docker run --rm --user 0 --entrypoint tar \
  -v ab_ats_rm-data:/data:ro -v "$PWD/backups":/backup \
  ghcr.io/srbhr/resume-matcher:1.4.0 czf /backup/rm-data-$(date +%F).tgz -C /data .

# Restore (sobre un volumen vacío)
docker run --rm --user 0 --entrypoint tar \
  -v ab_ats_rm-data:/data -v "$PWD/backups":/backup \
  ghcr.io/srbhr/resume-matcher:1.4.0 xzf /backup/rm-data-AAAA-MM-DD.tgz -C /data

# Borrar todos los datos (irreversible)
docker compose down
docker volume rm ab_ats_rm-data
```

Para borrar un solo CV: `curl -X DELETE http://localhost:3000/api/v1/resumes/<resume_id>` (o desde el dashboard).
