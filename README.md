SuperMarkets
=============

Short overview
--------------

SuperMarkets is a small Python project scaffold for building supermarket analytics and data processing tools. The repository will use:

- uv (pyproject.toml + uv.lock) for dependency and virtual environment management
- Ruff for linting and formatting
- pre-commit for developer hooks
- docker-compose to run the system locally (up/down)

This repository currently contains documentation and CSV sample data in the `docs/` directory.

Docs
----

Primary documentation lives in the `docs/` folder. Start with:

- docs/supermarket_home.md — home / quick start information
- docs/00-architecture-overview.md
- docs/01-core-flows.md
- docs/02-data-model-and-contracts.md
- docs/03-cloud-big-data-rationale.md
- docs/04-deployment-and-observability.md

Next steps
----------

This is an initial repository skeleton. Subsequent steps will add:

- pyproject.toml and uv.lock managed by uv
- Ruff configuration and pre-commit hooks
- Dockerfile(s) and docker-compose.yml to run the service
- Application packages and tests

License & Contributing
----------------------

See the `docs/` folder for project-level architecture and design notes. If you plan to contribute, follow the coding style enforced by Ruff and use the pre-commit hooks provided.

Contact
-------

Project workspace: docs/


