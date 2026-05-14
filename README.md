# resume-compiler

> Cloud Run Flask + Tectonic LaTeX service for HyreAgent.ai PDF generation.

## Purpose

Takes a JSON resume payload (sections, contact, work history) and returns a typeset PDF compiled with [Tectonic](https://tectonic-typesetting.github.io/). Used by the `platform/services/ai/` service when a user generates a resume or cover letter.

## Tech stack

| Layer | Choice |
| --- | --- |
| Runtime | Google Cloud Run (us-central1) |
| Language | Python 3.13 |
| Framework | Flask |
| PDF engine | [Tectonic](https://tectonic-typesetting.github.io/) (modern LaTeX) |
| Auth | JWT validation against Supabase using `SUPABASE_JWT_SECRET` (RELI-08 fail-closed) |
| Templates | Jinja2-rendered `.tex` files in `templates/` |
| Tests | pytest + a Tectonic-backed integration suite |
| Lint / format | Ruff + Black + mypy strict |

## Endpoints

All endpoints require a valid Supabase JWT in `Authorization: Bearer <token>` (no anon access — see RELI-05).

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/compile` | Compile a structured resume payload to PDF; returns the PDF binary |
| `POST` | `/generate` | Call Groq to author + compile a resume tailored to a JD |
| `POST` | `/generate-cover-letter` | Call Groq to author + compile a cover letter |
| `POST` | `/parse` | Extract structured sections from an uploaded PDF |
| `GET` | `/healthz` | Liveness probe (no auth) |

## Prerequisites

- Python 3.13 (locked via `.python-version`)
- [`uv`](https://docs.astral.sh/uv/) for dependency management (or `pip` + `venv`)
- Docker (for local container builds)
- `gcloud` CLI authenticated to the `resume-compiler-jobagent` GCP project
- Tectonic ≥ 0.15 (installed by the Dockerfile in production)

## Getting started

```bash
git clone -b dev git@github.com:HyreAgent-ai/resume-compiler.git
cd resume-compiler
uv sync
cp .env.example .env
# fill SUPABASE_JWT_SECRET (from Supabase dashboard → API → JWT Secret)
#      GROQ_API_KEY
uv run flask --app app run --port 8080
```

## Branching

- `main` → deployed to Cloud Run prod
- `dev` → deployed to a separate Cloud Run revision tagged `dev`
- Feature branches: `<username>_<issue#>_<short>`

## Deployment

```bash
gcloud run deploy resume-compiler --source . --region us-central1
```

Env vars must be set in the Cloud Run service config — see [docs/wiki/Resume-Compiler-Deploy](https://github.com/HyreAgent-ai/docs/wiki/Resume-Compiler-Deploy).

**`minScale` policy:** keep at `0` (cold-start tolerated for the ~10s typesetting flow). The previous `minScale: 1` cost $15-25/mo for marginal benefit; documented in [cost-and-scalability.md](https://github.com/HyreAgent-ai/docs/wiki/Cost-and-Scalability).

## Threat model

- **RCE via LaTeX shell-escape:** Tectonic runs with `--openin_any=p` (paranoid mode). Templates cannot `\input` arbitrary files.
- **Path traversal via template names:** templates are an allowlist, not a free-form parameter.
- **Auth fail-closed:** `require_auth` returns 500 if `SUPABASE_JWT_SECRET` is unset (RELI-08).
- **Mass assignment in resume payload:** Pydantic models with `extra='forbid'` reject unknown fields.

Full model: [docs/wiki/Resume-Compiler-Threat-Model](https://github.com/HyreAgent-ai/docs/wiki/Resume-Compiler-Threat-Model).

## Contributing

See [CONTRIBUTING.md](https://github.com/HyreAgent-ai/.github/blob/main/CONTRIBUTING.md). All template changes require a `pdfdiff` regression test in `tests/golden/`.

## License

[Apache-2.0](./LICENSE)

## Contact

- Maintainer: [@Siddardth7](https://github.com/Siddardth7)
