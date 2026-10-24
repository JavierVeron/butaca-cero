# Plan: Blog de Series y Películas con posts automáticos (CrewAI) — versión gratuita

## Context
Blog en español donde los artículos (reseñas, curiosidades, listas) se generan solos con un equipo de agentes CrewAI. Cada post lleva imagen destacada + imagen en el cuerpo. Catálogo: series y películas de cualquier época, **excluyendo India y China** y **sin documentales**. Géneros: comedia, terror, ciencia ficción, acción, romántica. La web muestra posts, con buscador y filtro por género. Es un proyecto de prueba: **todo gratis salvo los créditos de Claude** (el usuario ya los tiene). Nombre del blog: **Butaca Cero**. Carpeta de trabajo: `C:\DESARROLLO\EDUCACIONIT\89878\MULTI_AGENTES`. Repositorio: https://github.com/JavierVeron/butaca-cero

## Estado actual (actualizado 2026-10-24)
**Fase actual: Fase 0 casi completa. Pausado; se retoma la semana siguiente.**

Hecho:
- [x] Plan definido (stack, 8 agentes, posts a demanda, límite de 250 palabras, Haiku para todo).
- [x] Nombre elegido: Butaca Cero; sin dominio propio (URL gratuita de GitHub Pages: `https://JavierVeron.github.io/butaca-cero/`).
- [x] Estructura de carpetas, `.gitignore`, `.env.example`, `README.md`, `generador/pyproject.toml`.
- [x] Repositorio público `butaca-cero` creado y primer commit subido a `main`.
- [x] `.env` local con `ANTHROPIC_API_KEY` y `TMDB_API_KEY` (verificada con una consulta real a TMDB; no se sube al repo).

Pendiente para cerrar la Fase 0:
- [ ] Crear la hoja de Google Sheets (pestaña `posts`, con las columnas definidas) y pasar su ID.
- [ ] Crear la service account de Google, descargar `credenciales-google.json` a `MULTI_AGENTES` y compartir la hoja con su email como Editor.
- [ ] Decidir la versión de Python: instalar 3.12 o 3.13 (CrewAI puede no soportar la 3.14 instalada; `requires-python` está en `>=3.11,<3.14`).
- [ ] Revocar y regenerar la clave de Anthropic (quedó expuesta en el chat) y fijar un límite mensual de gasto en la consola de Anthropic. Opcional: regenerar la de TMDB.

Próximo paso (Fase 1): cliente TMDB + `filters.py` (IN/CN, documental, mapeo de categorías) con tests, `validate_post` y `sheets.py`.
Decisión abierta: el género TMDB de acción (10759 en series) es amplio; propuesta: priorizar títulos de acción pura sobre mezclas con comedia o animación.

## Decisiones tomadas
- **LLM**: Claude (vía CrewAI/LiteLLM), **Haiku 5.5 (`claude-haiku-5-5`) para todos los agentes** (mínimo costo). El modelo de cada agente es configurable en `agents.yaml`; si la calidad del Redactor no alcanza, se sube solo ese agente.
- **Largo de los posts**: **máximo 250 palabras** por post (cuerpo). Validado en código, no solo en el prompt.
- **Datos e imágenes de películas**: TMDB API (gratis, no comercial, con atribución). Las imágenes se usan por URL oficial de TMDB.
- **"Base de datos"**: **Google Sheets**. Una fila por post.
- **Web**: sitio estático con **Astro**, publicado en **GitHub Pages** (gratis, HTTPS, subdominio `usuario.github.io/blog`).
- **Automatización**: **GitHub Actions** con cron (genera post → escribe en Sheet → reconstruye la web).
- **Dominio**: subdominio gratuito de GitHub Pages; dominio propio opcional más adelante.

## Google Sheets como base de datos
Hoja `posts` con columnas:
`id | fecha | tmdb_id | tmdb_ids | formato | tipo (pelicula/serie) | titulo_original | anio | pais | categoria | tags | titulo_post | slug | bajada | cuerpo_md | imagen_destacada_url | imagen_cuerpo_url | alt_destacada | alt_cuerpo | plataformas | estado (draft/published)`

- **Escritura (CrewAI → Sheet)**: Publicador usa `gspread` con una *service account* de Google (gratis). La hoja se comparte con el email de esa cuenta.
- **Lectura (web ← Sheet)**: en cada build, Astro descarga la hoja como CSV (publicada en la web o vía API) y genera una página por cada fila con `estado=published`. No hay servidor consultando la hoja en vivo; cuando hay un post nuevo, GitHub Actions reconstruye el sitio.
- **Anti-duplicados**: el Scout lee la columna `tmdb_id` y descarta lo ya cubierto.
- **Límite práctico**: celdas ≤ 50.000 caracteres (sobra para un post); Sheets aguanta miles de filas.
- Ventaja: el usuario puede ver/editar/despublicar posts desde Google Sheets sin código.

## Reglas de negocio (filtros duros, en código, no solo en prompts)
- Excluir `origin_country` ∈ {IN, CN}.
- Excluir género TMDB "Documentary" (id 99).
- Mapeo TMDB → categorías: Comedia(35), Terror(27), Ciencia ficción(878 / TV 10765), Acción(28 / TV 10759), Romántica(10749). Solo se aceptan títulos que caigan en ≥1.

## Agentes y roles (Crew)
| Agente | Rol | Herramientas |
|---|---|---|
| **Scout (Curador)** | Elige título según categoría y variedad de épocas; aplica filtros duros | TMDB discover, Sheet (tmdb_id usados) |
| **Investigador** | Ficha: sinopsis, reparto, año, plataformas, curiosidades, ratings | TMDB details/credits/watch-providers |
| **Redactor** | Escribe el post en español (título, bajada, cuerpo markdown con marcador de imagen, tags), **máx. 250 palabras** | LLM (Haiku) |
| **Editor / Fact-checker** | Revisa tono, hechos vs. ficha, longitud (≤250 palabras), spoilers; aprueba o corrige (máx. 2 vueltas) | LLM (Haiku) |
| **Curador de Imágenes** | Elige imagen destacada (backdrop) e imagen de cuerpo (póster/still distinto) + alt-text | TMDB images |
| **SEO / Taxonomía** | Slug, meta description, categoría, tags | LLM (Haiku) |
| **Publicador** | Agrega la fila en Google Sheets con `estado` | gspread |
| **Intérprete de pedidos** (solo modo a demanda) | Convierte un pedido en lenguaje natural en una ficha estructurada (`tipo`, `categoria`, `pais`, `cantidad`, `epoca`, `formato`); rechaza pedidos que violen los filtros duros | LLM (Haiku) |

Flujo automático: `Scout → Investigador → Redactor → Editor → Imágenes → SEO → Publicador`.
Flujo a demanda: `Intérprete → Scout (con filtros del pedido) → Investigador (×N títulos) → Redactor → Editor → Imágenes → SEO → Publicador`.
Salidas estructuradas con Pydantic (`output_pydantic`).

## Posts a demanda
Ejemplo: `python generate.py --pedido "Creame un post sobre 5 series de acción de USA"`
- El Intérprete produce `{tipo: serie, categoria: accion, pais: US, cantidad: 5, formato: lista}`.
- El Scout consulta TMDB con esos filtros (`with_origin_country`, género, tipo, rango de años) y elige los N títulos (por puntaje, popularidad o época según el pedido), descartando los que no pasen los filtros duros.
- El Investigador arma la ficha de cada título; el Redactor escribe un post en formato lista (un párrafo por título).
- Imagen destacada: backdrop de un título; imagen de cuerpo: otra distinta.
- Pedidos aceptados: tipo (película/serie), género, país, cantidad, época ("de los 90"), estilo (ranking, imperdibles, poco conocidas).
- Pedidos como "series de India/China" o "documentales" se rechazan con un mensaje claro.
- En el Sheet se agregan las columnas `formato` (individual/lista) y `tmdb_ids` (todos los títulos incluidos, para el anti-duplicados).
- Se puede lanzar desde la terminal o desde GitHub Actions (botón "Run workflow" con una casilla de texto para el pedido).
- Largo: el límite de 250 palabras aplica al post completo; en listas, el Redactor reparte las palabras entre los títulos (ej. 5 títulos ≈ 40–45 palabras cada uno).
- Costo: un post de lista usa más tokens (investiga N títulos); todos los agentes usan Haiku.

## Buscador y filtro por género (sin servidor)
En el build, Astro genera un `indice.json` liviano (título, tipo, año, categoría, slug, imagen) con todos los posts. El navegador lo descarga una vez y filtra localmente:
- Filtro por género: clic en "Terror" muestra solo posts con esa categoría; también páginas `/categoria/terror`.
- Buscador: coincidencia mientras se escribe (Fuse.js o Pagefind, tolerante a acentos); combinable con el filtro de género y de tipo (película/serie).

## Fases
**Fase 0 – Fundamentos y nombre** (🟡 en curso)
- [x] Carpeta `MULTI_AGENTES`, `PLAN.md`, repo en GitHub, `.env` base.
- [x] Clave de Anthropic y clave de TMDB en `.env` (TMDB probada).
- [ ] Credenciales de Google (service account + Sheets API) y ID de la hoja.
- [ ] Python 3.12/3.13 instalado.
- [ ] Clave de Anthropic regenerada + límite mensual de gasto.
- [x] **Nombre elegido: Butaca Cero.** Repositorio público `butaca-cero`; URL gratuita `https://JavierVeron.github.io/butaca-cero/` (sin comprar dominio por ahora; en Astro se configura `site` y `base: '/butaca-cero'`).

**Fase 1 – Datos y filtros**
- [ ] Cliente TMDB + `filters.py` (IN/CN, documental, mapeo de categorías) con tests.
- [ ] Validador de post (`validate_post`): cuerpo ≤ 250 palabras, 2 imágenes distintas, categoría válida; si falla, el Redactor reintenta (máx. 2).
- [ ] Hoja de Google creada con las columnas definidas; módulo `sheets.py` (leer/agregar).

**Fase 2 – Pipeline CrewAI (CLI local)**
- [ ] `agents.yaml`, `tasks.yaml`, `crew.py`, tools. Comando automático: `python generate.py --categoria terror --n 1`.
- [ ] Comando a demanda: `python generate.py --pedido "5 series de acción de USA"` (agente Intérprete de pedidos).
- [ ] Resultado: filas nuevas en Sheets.

**Fase 3 – Web (Astro)**
- [ ] Home, página de post (imagen destacada arriba + imagen en el cuerpo), `/categoria/{slug}`, buscador + filtros, atribución TMDB, sitemap/SEO básico. Datos leídos desde el Sheet en build.

**Fase 4 – Publicación gratuita**
- [ ] Deploy en GitHub Pages con GitHub Actions.

**Fase 5 – Automatización**
- [ ] Workflow con cron: genera N posts/día (rotando categorías) → escribe en Sheet → rebuild y deploy. Secrets en GitHub. Tope de posts diarios y de gasto.
- [ ] Segundo workflow manual (`workflow_dispatch`) con input de texto para generar un post a demanda.

**Fase 6 – Calidad y extras**
- [ ] Revisión manual (`estado=draft` por defecto al inicio), manejo de errores/reintentos, páginas legales mínimas; luego: "dónde ver", listas, links entre posts relacionados.

## Estructura propuesta
```
MULTI_AGENTES/
  PLAN.md
  generador/             # Python
    config.py, sheets.py
    tmdb/client.py, tmdb/filters.py
    crew/agents.yaml, tasks.yaml, crew.py, tools.py, schemas.py
    generate.py
    tests/
  web/                   # Astro
    src/pages/, src/components/, src/lib/sheet.ts
  .github/workflows/     # generar.yml, deploy.yml
```

## Riesgos / notas
- TMDB: uso no comercial y atribución obligatoria; si se monetiza hay que pedir licencia.
- Las imágenes se enlazan a TMDB (no se copian al repo).
- Sheets: respetar cuotas de la API (más que suficiente para pocos posts/día).
- Sin spoilers por defecto (el Editor los rechaza).
- Credenciales de Google y claves van en secrets/`.env`, nunca en el repo.

## Verificación
- Tests unitarios de filtros (IN/CN y documental rechazados).
- `generate.py --categoria comedia --n 1`: aparece fila en Sheet con 2 imágenes distintas, categoría correcta, país no IN/CN, sin tmdb_id repetido.
- `npm run build` en `web/`: se generan páginas de posts, `/categoria/*` e `indice.json`; `npm run dev` para probar búsqueda (con acentos) y filtro por género.
- Disparar el workflow manualmente en GitHub y comprobar que el sitio publicado muestra el post nuevo.
- Revisar manualmente 5 posts por categoría antes de activar el cron.
