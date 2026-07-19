# Issue tracker: Local Markdown

Issues and specs for this repo live as Markdown files in `.scratch/`.

## Conventions

- One feature per directory: `.scratch/<feature-slug>/`
- The spec is `.scratch/<feature-slug>/spec.md`
- Implementation issues are stored separately at `.scratch/<feature-slug>/issues/<NN>-<slug>.md`, numbered from `01`
- Triage state is recorded as a `Status:` line near the top
- Comments are appended under `## Comments`

## Publishing and fetching

When publishing, create the appropriate file under `.scratch/<feature-slug>/`.
When fetching, read the referenced path or issue number.

## Wayfinding

- Map: `.scratch/<effort>/map.md`
- Child ticket: `.scratch/<effort>/issues/NN-<slug>.md`
- Ticket fields: `Type:`, `Status:`, and optionally `Blocked by:`
- Claim by setting `Status: claimed`
- Resolve by adding `## Answer`, setting `Status: resolved`, and updating the map
