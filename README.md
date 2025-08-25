# course-downloader

## Scaffolding (2025-08-24 22:11:47.4)

This repository now includes lightweight, import-safe scaffolding for the course_recorder package.

Basic import check:

Activate your venv then run:
python3 -c "import course_recorder; print('ok')"

Run tests:

python3 -m pytest -q

Notes:

- The scaffolding is import-safe and avoids runtime Playwright/OBS calls.
- Playwright OS-level dependencies are not installed in this environment. See:
.kilocode/logs/task-2025-08-24 22:11:47.3 - OS-libs-install.md

Files added: (see .kilocode/logs/task-2025-08-24 22:11:47.4 - Create project scaffolding and core modules.md)