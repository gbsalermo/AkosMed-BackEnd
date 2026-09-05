# AkosMed Documentation

Read in this order:

```text
1. ../CONTINUITY.md
2. MASTER_DOSSIER.md
3. ROADMAP.md
4. MVP_SCOPE.md
5. DOMAIN_MODEL.md
6. API_CONTRACT.md
7. ENGINEERING_RULES.md
```

## Canonical responsibilities

| File | Purpose |
|---|---|
| `../CONTINUITY.md` | Current checkpoint, active branch/stage and exact next task |
| `MASTER_DOSSIER.md` | Consolidated architecture/product handoff |
| `ROADMAP.md` | Canonical execution order |
| `MVP_SCOPE.md` | What the backend MVP must and must not do |
| `DOMAIN_MODEL.md` | Canonical entity/relationship map |
| `API_CONTRACT.md` | Endpoint map and HTTP conventions |
| `ENGINEERING_RULES.md` | Technical guardrails and Definition of Done |

## Source-of-truth precedence

When something conflicts:

```text
1. tested code on main, once implementation exists
2. live Swagger/OpenAPI for HTTP contracts, once controllers exist
3. CONTINUITY.md for current stage/checkpoint
4. ROADMAP.md for execution order
5. canonical technical document for the topic
6. historical discussion/old commits
```

Do not maintain duplicated copies of the same rule across many documents. If a technical decision changes, update the canonical file and only adjust references/checkpoints elsewhere.