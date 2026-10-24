# Butaca Cero – Blog de Series y Películas (posts automáticos con CrewAI)

Ver [PLAN.md](PLAN.md) para el plan completo.

## Estructura
- `generador/` – Python: agentes CrewAI, cliente TMDB, Google Sheets.
- `web/` – Astro: el sitio estático.
- `.github/workflows/` – automatización (GitHub Actions).

## Claves necesarias (Fase 0)
1. **Anthropic**: API key en https://console.anthropic.com (poner un límite mensual de gasto).
2. **TMDB**: cuenta gratuita y API key en https://www.themoviedb.org/settings/api
3. **Google Sheets**: service account (Google Cloud Console), activar Sheets API, descargar el JSON como `credenciales-google.json` y compartir la hoja con el email de la service account.
4. **GitHub**: repositorio público.

Copiar `.env.example` a `.env` y completar.
