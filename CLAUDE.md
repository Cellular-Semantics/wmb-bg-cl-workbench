# CL + Uberon workbench

Reasoner-informed co-development of CL and Uberon terms. README.md has the user
guide; this file is for agents working in the repo.

## Where things are

- `workbench.yaml` (+ `workbench.local.yaml`) declares sources, checkouts, KBs and
  serving. Run `uv run wb kbs` and `uv run wb status` to see the current state; don't
  guess paths.
- Ontology edits happen in the checkouts that `wb status` lists (under
  `sources/<src>/<name>/`, or wherever `path:` points). Edit only the edit files
  (`src/ontology/cl-edit.owl`, `src/ontology/uberon-edit.obo`) and follow each repo's
  own CLAUDE.md / contributing rules. Never edit `imports/` or `components/` by hand.
- `patches/*.ofn` hold experimental axioms not yet in either repo (placeholder IDs
  `obo:EX_…`).

## Checking a change with the reasoner

1. Before editing: query the relevant KB (`uv run wb q KB EXPR`) and keep the output.
2. Edit, then `uv run wb build KB && uv run wb restart KB`. If `wb watch` is running,
   it does this for you; check `wb status` until the KB is served again.
3. Re-run the query, and `uv run wb compare` against a KB built from master.
4. `uv run wb unsat KB` must not report anything new.

Use the merged `cl-*__uberon-*` KBs for anything that crosses the two ontologies
(soma location, part_of chains, taxon constraints). The single-ontology KBs see only
the other ontology's last-release import slice.

`oq` (`uv run oq …`) queries the *published* ontologies (OLS4, Ubergraph). Use it for
the released state and for ontologies not in the workbench, not as a check of a
branch.

## Rules

- Commit or push in the ontology checkouts only when asked. `wb sync` never
  overwrites uncommitted work (ff-only, clean checkouts only).
- Don't round-trip reasoner inputs through RDF; keep `.ofn`.
- Uberon partonomy in CL's import can differ from uberon-edit. When they disagree,
  the merged KB is the one to trust for a branch.
