# CL + Uberon workbench

A template for developing related terms in the Cell Ontology (CL) and Uberon together,
with a reasoner checking the result as you go. It gives you:

- **Branch checkouts** of CL and Uberon (or any ODK-style ontology repo) as git
  worktrees, declared in one config file and kept in sync with GitHub.
- **Merged knowledgebases**: CL with its Uberon import slice replaced by the full
  `uberon-edit` closure of a chosen Uberon branch, and the reverse. So a new Uberon
  term and the CL class defined from it can be reasoned over together before either
  PR is merged.
- **Owlery** (ELK) serving every KB at one URL, one process per KB, so a rebuilt KB
  reloads without restarting the others. `wb watch` rebuilds and reloads when a
  checkout or patch changes.
- **`wb q` / `wb compare` / `wb unsat`**: DL queries written with CURIEs and
  `'labels'`, before/after comparison between KBs, and an unsatisfiable-class check.
- **`oq`** (onto-query): OLS4 / Ubergraph lookups against *published* ontologies, for
  comparing with what's on your branches.

## Quick start

Requirements: Java 11+, [uv](https://docs.astral.sh/uv/), git, about 16 GB RAM.
Or open the repo in a GitHub Codespace / dev container (`.devcontainer/`), which sets
everything up.

```sh
# "Use this template" on GitHub, then clone your copy, or clone this repo directly
uv sync                 # Python env with wb and oq
uv run wb setup         # ROBOT + Owlery into .tools/
uv run wb up            # clone CL + Uberon (master), build KBs, serve on :8090
uv run wb q cl-master__uberon-master "'GABAergic neuron' and (RO:0002100 some 'cerebellum')" --direct
```

The first `wb sync` makes blob-less partial clones (file contents are fetched on
demand), so it takes a few minutes, not the time of a full clone.

## Configuration

`workbench.yaml` is the only thing to edit. It is committed, so a team working from
one copy of the template shares the same set-up. `workbench.local.yaml` (git-ignored;
see `workbench.local.yaml.example`) is deep-merged on top for machine-specific
settings.

### Sources and checkouts

```yaml
sources:
  cl:
    repo: https://github.com/obophenotype/cell-ontology.git   # or your fork
    edit: src/ontology/cl-edit.owl
    catalog: src/ontology/catalog-v001.xml
    checkouts:
      - master
      - my-new-neuron-types          # a branch
      - pr: 3788                     # someone's PR, to reason over it with your Uberon branch
    sync: ff
```

Each checkout is a worktree at `sources/<source>/<name>/`. Edit `cl-edit.owl` there
(Protégé, a text editor, an agent), commit and push as usual: these are ordinary git
checkouts.

Already have clones? Point at them in `workbench.local.yaml` instead:

```yaml
sources:
  cl:
    path: ~/GitHub/cell-ontology
    checkouts: auto        # every worktree of that clone, named by branch
```

### Sync policy

| `sync` | `wb sync` does |
|---|---|
| `ff` (default for managed sources) | fetch, then fast-forward each clean checkout to its upstream (PR checkouts to the PR head). Checkouts with uncommitted changes or local commits are left alone and reported. |
| `fetch` | fetch only; `wb status` shows ahead/behind |
| `none` (default for `path:` sources) | nothing; you manage the checkout |

`wb sync --prune` removes clean managed worktrees that you've taken out of the config.

### KBs

A KB is a merge of **parts**: a source's edit-file import closure, a local `file:`
(see `patches/`), or a `url:`. `{source}` in the name expands over that source's
checkouts (all of them, unless `matrix` restricts it):

```yaml
kbs:
  - name: cl-{cl}__uberon-{uberon}
    parts:
      - {source: cl, drop_import_subjects: [UBERON]}
      - {source: uberon, drop_import_subjects: [CL]}
    matrix: {cl: "*", uberon: [master, my-uberon-branch]}
    heap: 10G
```

`drop_import_subjects` handles the overlap. CL imports a slice of Uberon and Uberon
imports a slice of CL, and each slice is from the last release, not your branch. For
each file the edit file imports from `imports/`, the build drops every axiom whose
*subject* is a term with one of the given prefixes. Uberon's own axioms then come only
from `uberon-edit`, and CL's only from `cl-edit`. Axioms that only *mention* those
terms (e.g. a GO class `occurs in` some Uberon term, a CL GCI) are kept.

`wb kbs` shows what the config expands to and whether each KB is current.

### Serving

```yaml
serve:
  port: 8090            # http://localhost:8090/kbs/<kb>/{subclasses,superclasses,equivalent,instances}
  backend_port_base: 9100
  reasoner: elk
  heap: 6G
  kbs: "*"              # which KBs `wb serve` starts by default; globs allowed
```

## Commands

| | |
|---|---|
| `wb sync [SOURCE…] [--prune]` | create/update checkouts per config and `sync` policy |
| `wb status` | checkouts (branch, commit, ahead/behind, uncommitted) and KBs (built, served, needs reload) |
| `wb build [KB…] [--force]` | build KBs (globs ok). Per-checkout closures are cached and keyed by commit plus uncommitted changes, so only what changed is rebuilt |
| `wb serve [KB…]` / `wb restart [KB…]` / `wb stop [KB…]` | manage Owlery backends and the proxy |
| `wb up` | sync + build + serve |
| `wb watch [KB…] [--interval 20] [--sync-every 600]` | rebuild and reload served KBs whose checkouts or patches change; optionally pull remote changes too |
| `wb q KB EXPR [--super\|--equiv\|--inst] [--direct]` | DL query (Manchester syntax with CURIEs, `'labels'` or `<IRIs>`) |
| `wb compare KB_A KB_B EXPR …` | results that differ between two KBs, e.g. a branch vs master, or patched vs not |
| `wb unsat KB` | unsatisfiable classes |
| `oq …` | OLS4 / Ubergraph lookups (`uv run oq --help`) |

Results print as `CURIE<TAB>label`. When a label is ambiguous, the OBO term is used if
there is exactly one; otherwise use the CURIE.

## A typical loop

1. Add your Uberon branch and CL branch as checkouts, then run `wb sync`.
2. `wb serve 'cl-*__uberon-*'`, then `wb watch` in another terminal.
3. Try the idea as a patch first (`patches/`, a `file:` part), and use `wb compare`
   to see what it changes.
4. Move the axioms into `uberon-edit.obo` / `cl-edit.owl` on the branches and commit.
   `watch` rebuilds the KB, and `wb compare cl-master__uberon-master
   cl-mybranch__uberon-mybranch EXPR` shows the effect of both PRs together.
5. Run `wb unsat` before opening PRs: the merged KB also catches cross-ontology
   clashes (e.g. taxon constraints in Uberon against a CL class's soma location).

## Layout

```
workbench.yaml            shared config (commit it)
workbench.local.yaml      your overrides (ignored)
patches/                  experimental OWL axioms (file: parts)
src/workbench/            the wb CLI
sources/<src>/.repo       partial bare clone        (ignored)
sources/<src>/<checkout>  worktrees                 (ignored)
build/parts/              cached per-checkout closures (ignored)
build/kbs/<kb>.ofn        KBs; <kb>.json records the commits they were built from
run/                      Owlery configs, logs, PIDs  (ignored)
```

All reasoner inputs are OWL functional syntax written by OWLAPI. Nothing goes through
an OWL → RDF → OWL round trip.

See `docs/design.md` for the reasons behind these choices and the alternatives.
