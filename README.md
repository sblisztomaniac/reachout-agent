# Reachout Agent - AI-Native Outreach Copilot

An AI-powered outreach assistant for freelancers and solo founders that automates business discovery, research, and personalized message generation.

## Project Status

**Phase**: Foundation (Ralph autonomous development)
**Backend**: Not yet implemented
**Docs**: Complete and ZeroDB-aligned

## Documentation

- [PRD](docs/prd.md) - Product requirements and architecture
- [Data Model](docs/datamodel.md) - ZeroDB table and embedding schemas
- [Backlog](docs/backlog.md) - Epic-based user stories
- [Sprint Plan](docs/sprintplan.md) - 1-day MVP build plan

## Development Approach

This project uses [Ralph](https://github.com/snark-tank/ralph) autonomous development:
- Ralph autonomously implements backend following the PRD
- Each iteration is tracked in `progress.txt`
- Story completion tracked in `prd.json`

## Ralph Files

- `prd.json` - User stories in Ralph format (generated from docs/)
- `progress.txt` - Iteration learnings and context
- `AGENTS.md` - Project-specific agent instructions
- `ralph.sh` - Autonomous execution loop (in ralph/ subdirectory)

## Architecture

**Backend**: FastAPI + AI-Native Studio APIs
**Data Layer**: ZeroDB (tables + embeddings)
**Agents**: Discovery, Enrichment, Qualification, Drafting, Learning
**External Tools**: Web search MCPs, Firecrawl scraper

See [PRD](docs/prd.md) for full architecture.

## Getting Started

Once Ralph completes foundation:
```bash
cd backend
pip install -r requirements.txt
python -m uvicorn main:app --reload
```

Frontend lives in separate repo.
