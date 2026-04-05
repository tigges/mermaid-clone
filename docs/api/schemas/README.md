# Import Wizard API Schemas (v1)

This directory contains starter JSON Schemas that align with the 6-step wizard flow:

1. `session.schema.json`
2. `page-map.schema.json`
3. `cpm.schema.json`
4. `preview-graph.schema.json`
5. `artifact-manifest.schema.json`
6. `settings.schema.json`
7. `visual-qa-report.schema.json`

## Notes

- These schemas are intentionally target-independent.
- Use the canonical process model (`cpm.schema.json`) as the single source of truth.
- Target compilers (Mermaid, BPMN, ProcessMap-V1, etc.) should consume CPM and produce artifacts with `lossReport`.
- The OpenAPI contract in `../openapi/import-wizard-api.v1.yaml` references equivalent structures in component schemas.
