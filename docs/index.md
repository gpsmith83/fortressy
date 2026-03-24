# Fortressy docs index

This folder is the **system of record** for product/engineering knowledge.

## Product

- Requirements (MVP + fast-follow): `docs/requirements.md`
- MVP milestone plan: `docs/mvp-plan.md`

## Development

- Local dev + CI-equivalent commands: `docs/dev.md`

## References

- Architectural analysis + feature backlog: `research.md`

## How changes should be verified

- All gameplay rules should have at least one of:
  - a GUT unit test (preferred for pure logic)
  - an integration test scene
  - a manual verification note in the relevant plan

## Conventions

- Keep docs short and link outward (progressive disclosure).
- If a rule matters, make it mechanically checkable (tests or scripts).
