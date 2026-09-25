SWRLCLI
=======

SWRLCLI is a headless command-line tool for running [SWRL](https://www.w3.org/submissions/SWRL/) inference and SQWRL queries against OWL ontologies. It was developed by Damion Dooley at the [Centre for Infectious Disease Genomics and One Health (CIDGOH)](https://cidgoh.ca) at Simon Fraser University. The SWRLCLI code was generated with [Claude Code](https://claude.ai/code).

SWRLCLI is built directly on [SWRLAPI](https://github.com/protegeproject/swrlapi) and [OWLAPI](https://github.com/owlcs/owlapi) with no GUI dependency:

- **[OWLAPI](https://github.com/owlcs/owlapi)** handles all ontology I/O — loading OWL files in any supported format, resolving `owl:imports`, reading and writing axioms, and managing prefix declarations.
- **[SWRLAPI](https://github.com/protegeproject/swrlapi)** provides the SWRL rule engine, SQWRL query engine, and rule/query management layer. SWRLAPI is itself built on OWLAPI and uses Drools as its inference backend.

Build the fat JAR first (only needed once, or after code changes):

    mvn clean package

A wrapper script is provided for convenience:

    ./swrlcli [options] <ontology-file>

OWLAPI detects the ontology format from file content rather than extension. Commonly used extensions:

| Extension(s) | Format |
|---|---|
| `.owl`, `.rdf`, `.xml` | RDF/XML |
| `.ofn` | OWL Functional Syntax |
| `.owx` | OWL/XML |
| `.omn` | Manchester OWL Syntax |
| `.ttl`, `.turtle` | Turtle |
| `.n3` | Notation 3 |
| `.nt` | N-Triples |
| `.jsonld` | JSON-LD |
| `.obo` | OBO Format |

### Modes (exactly one required)

| Option | Description |
|---|---|
| `--infer` | Fire all SWRL rules; print inferred axioms as OWL Functional Syntax |
| `--rules <name[,name…]>` | Fire one or more named SWRL rules; print inferred axioms as OWL Functional Syntax. Accepts a comma-separated list (e.g. `--rules S2,S4,S7`) and/or may be repeated (e.g. `--rules S2 --rules S4`). Rules fire and display in the order specified. |
| `--rule-text <swrl>` | Run an inline SWRL rule expression; print inferred axioms as OWL Functional Syntax |
| &nbsp;&nbsp;`--rule-text-name <name>` | Name to assign to the inline rule (default: `cli-rule`). Only applies with `--rule-text`. |
| `--delete` | Delete the rules/queries named by `--rules` from the ontology instead of firing them. Requires `--rules` and `--save` to persist the change. |
| `--query <name>` | Run a named SQWRL query stored in the ontology |
| `--query-text <sqwrl>` | Run an inline SQWRL expression |
| &nbsp;&nbsp;`--query-name <name>` | Name to assign to the inline query result set (default: `cli-query`). Only applies with `--query-text`. |
| `--list-rules` | List all SWRL rules with active/inactive status. With `--format txt`: emits plain `.swrl` file syntax that can be redirected straight to a file, edited, and fed back in with `--file` (see developer iteration workflow below). With `--format markdown`: emits a `<pre>` block with variables coloured by entity type; add `--no-color` for spans-free output. |
| `--list-queries` | List all SQWRL queries stored in the ontology. Supports the same `--format txt` and `--format markdown` options as `--list-rules`. |

### Input options

| Option | Description |
|---|---|
| `--file <path>` | Load SWRL rules and/or SQWRL queries from a text file into the ontology before running. May be repeated to load from multiple files. Works alongside any mode. If a loaded rule or query shares a name with one already in the ontology, the existing one is replaced. |
| `--save` | After all `--file` inputs have been loaded, write the updated ontology back to the original file in the same format. Requires at least one `--file`. Because OWLAPI serialises the entire ontology, any comments or non-standard formatting in the original file will not be preserved. |

### Output options

| Option | Description |
|---|---|
| `--format tsv\|csv\|markdown\|txt` | Output format (default: `tsv`). `tsv` and `csv` apply to query results. `markdown` renders `--debug` evaluation reports as markdown tables, and renders `--list-rules`/`--list-queries` as a `<pre>` block in `.swrl` file syntax with variables coloured by entity type. `txt` renders `--list-rules`/`--list-queries` as plain-text `.swrl` file syntax (no HTML, no colour spans) — output can be piped directly to a file and used as `--file` input. |
| `--no-color` | Disable per-argument HTML colour spans in `--format markdown` output. Applies to both `--debug` evaluation tables and `--list-rules`/`--list-queries`; with listing modes, `--no-color` produces spans-free output that can be pasted directly into a `--file` input. |
| `--output-file <path>` | Write inferred axioms to this file in OWL Functional Syntax instead of stdout. Applies to `--infer`, `--rules`, and `--rule-text`. The file is always written even when no axioms were inferred (producing an empty ontology), so it can be unconditionally referenced as an `owl:imports` target. |
| `--output-iri <iri>` | Stamp the output ontology with this IRI (e.g. `http://example.org/inferred.ofn`). Required for the file to be usable as an `owl:imports` target. Only meaningful with `--output-file`. |
| `--config <path>` | Load a custom `swrlcli_config.yaml` from the given path instead of (or in addition to) the default locations. |
| `--ignore-imports` | Silently skip unresolvable `owl:imports` declarations |

### Debug options

| Option | Description |
|---|---|
| `--debug` | With `--rules`: instead of printing inferred axioms, print a body atom evaluation table showing each atom as PASS, FAIL, or SKIP with variable bindings, followed by the head (consequent) atoms and a satisfaction score. The table shows the single best-matching individual binding found by the path search — only one candidate row is reported per rule. With `--infer`: runs inference first, then prints the same evaluation table for every rule in the ontology. Output goes to stdout and can be redirected. |
| `--constraint <atom>` | With `--rules --debug`: seed a specific individual binding for evaluation. Format: `predicate(individual)` or `predicate(subject, object)`. The predicate and each individual argument may be given as a prefix-qualified CURIE (e.g. `obo:MyClass(recipe:r1.step1)`) **or** as a plain `rdfs:label` string (e.g. `'combining two materials'('add carrots to water')`). Label lookup is case-insensitive across the full imports closure. If an argument cannot be resolved to a declared individual, or a predicate/property in a body atom is not declared in the ontology, the evaluation table shows a **reference error** in red. Implies `--debug`. May be repeated to pin multiple atoms. When multiple `--rules` flags are used, constraints that don't match a given rule are silently skipped for that rule. With `--query` or `--query-text`: filter result rows to only those containing the specified individual IRI(s). |

### Examples

List all SWRL rules in an ontology:

    ./swrlcli --list-rules ontology.ofn

List all SWRL rules with coloured variables (markdown `<pre>` block):

    ./swrlcli --list-rules --format markdown ontology.ofn

List all SWRL rules as plain `.swrl` text, ready to paste into a `--file` input:

    ./swrlcli --list-rules --format markdown --no-color ontology.ofn

Export all rules to a `.swrl` file, edit them, then reload and run:

    ./swrlcli --list-rules --format txt ontology.ofn > my-rules.swrl
    # … edit my-rules.swrl in any text editor …
    ./swrlcli --file my-rules.swrl --infer ontology.ofn

To persist the edits back into the ontology file:

    ./swrlcli --file my-rules.swrl --save --list-rules ontology.ofn

Fire all rules and print inferred axioms:

    ./swrlcli --infer ontology.ofn

Write inferred axioms to a separate OWL file (suitable as an `owl:imports` target):

    ./swrlcli --infer --output-file inferred.ofn --output-iri http://example.org/inferred.ofn ontology.ofn

Fire a single named rule and capture inferences to a file:

    ./swrlcli --rules "MyRuleName" --output-file inferred.ofn --output-iri http://example.org/inferred.ofn ontology.ofn

Fire a single named rule:

    ./swrlcli --rules "MyRuleName" ontology.ofn

Fire multiple named rules in one inference pass:

    ./swrlcli --rules "RuleA","RuleB" ontology.ofn

Rule and query names are stored as `rdfs:label` annotations in the ontology, so they can be plain words or natural-language phrases — e.g. `"combine two materials"` or `"find large inputs"` — making them easy to reference by meaning rather than by opaque identifiers. Use `--list-rules` to see the exact names in your ontology.

Run an inline SWRL rule expression:

    ./swrlcli --rule-text "MyClass(?x) -> MyOtherClass(?x)" ontology.ofn

Run an inline rule with debug output and a custom name:

    ./swrlcli --rule-text "MyClass(?x) -> MyOtherClass(?x)" --rule-text-name "my-inline-rule" --debug --format markdown ontology.ofn

Debug a rule to see which body atoms are satisfied:

    ./swrlcli --rules "MyRuleName" --debug ontology.ofn

Debug all rules in one pass (runs inference first, then evaluates each rule):

    ./swrlcli --infer --debug --format markdown ontology.ofn > all-rules-report.md

Debug a rule with a specific individual bound to one of its body atoms:

    ./swrlcli --rules "MyRuleName" --constraint "MyClass(prefix:myIndividual)" ontology.ofn

Debug a rule and save the evaluation report as a markdown file:

    ./swrlcli --rules "MyRuleName" --debug --format markdown ontology.ofn > report.md

Run a named SQWRL query:

    ./swrlcli --query "myQuery" ontology.ofn

Run an inline SQWRL query and get CSV output:

    ./swrlcli --query-text "owl:Thing(?x) -> sqwrl:select(?x)" --format csv ontology.ofn

Run an inline SQWRL query with a result set name:

    ./swrlcli --query-text "owl:Thing(?x) -> sqwrl:select(?x)" --query-name "all-things" ontology.ofn

Filter query results to rows containing a specific individual:

    ./swrlcli --query "myQuery" --constraint "myProperty(prefix:myIndividual)" ontology.ofn

Load rules and queries from a file, then fire all rules:

    ./swrlcli --file my-rules.swrl --infer ontology.ofn

Load rules from a file and run a named rule with debug output:

    ./swrlcli --file my-rules.swrl --rules "MyRule" --debug --format markdown ontology.ofn

Load rules from a file and run a named query defined in that file:

    ./swrlcli --file my-rules.swrl --query "MyQuery" ontology.ofn

Load updated rules from a file and save them back into the ontology (replaces any same-named rules):

    ./swrlcli --file my-rules.swrl --save --list-rules ontology.ofn

Delete one or more named rules from the ontology and save:

    ./swrlcli --rules "RuleA","RuleB" --delete --save ontology.ofn

### SWRL file format

When using `--file`, each entry in the file is declared with a `rule` or `query` keyword followed by its name, then the SWRL/SQWRL expression on the next line(s).  The `rule`/`query` keyword is for readability — SWRLAPI automatically distinguishes queries from rules by the presence of `sqwrl:` built-ins in the head.

```
# Whole-line comments begin with #
# Blank lines are ignored

rule TransitivePartOf
# A # comment immediately after the rule/query header (before the first body atom)
# is stored as the rule's rdfs:comment description in the ontology.
# Multiple consecutive comment lines are joined with a space.
part of(?x, ?y) ^ part of(?y, ?z) -> part of(?x, ?z)

rule "Rule With Spaces In Name"
MyClass(?x) ^ hasProperty(?x, ?y) -> MyOtherClass(?x)

# Multi-line rule — body lines are joined with a space before parsing.
# Leading indentation is ignored.  ^ and -> may appear at the start of
# a continuation line.  Inline comments are also supported.
rule ComplexProcessRule
# Fires when a process input material exceeds 10 kg.
  "has specified input"(?p, ?m)   # quoted labels are resolved via rdfs:label
  ^ "has characteristic"(?m, ?c)
  ^ "mass in kilograms"(?c, ?mass)
  ^ swrlb:greaterThan(?mass, 10)  # plain prefix:local names work directly
  -> "large input process"(?p)

query AllMaterials
  Material(?x) -> sqwrl:select(?x)
```

Key formatting rules:

- Rule/query names containing spaces must be enclosed in double quotes.
- Multi-line expressions are joined with a single space before parsing; leading indentation is stripped from each line.
- `^` and `->` may appear at the start of a continuation line.
- A `#` anywhere on a body line starts an inline comment (everything from `#` to end of line is discarded). Exception: `#` inside an angle-bracket IRI (`<http://example.org/ont#fragment>`) is preserved.
- **`# rule Name` / `# query Name`** — a whole-line comment whose text begins with `rule ` or `query ` (case-insensitive) is treated as a directive with the `#` stripped. This means both `rule MyRule` and `# rule MyRule` are valid directive forms.
- **Description comments** — one or more `#` comment lines placed immediately after a `rule`/`query` header line (before the first body atom) are stored as the rule's `rdfs:comment` annotation in the ontology. Multiple consecutive lines are joined with a space. These descriptions are round-tripped by `--list-rules --format txt` and `--format markdown` and are visible in Protégé's rule editor.
- **Quoted predicate labels** — a double- or single-quoted string used as a predicate (e.g. `'has specified input'(...)` or `"has specified input"(...)`) is automatically resolved to its ontology IRI via `rdfs:label` lookup across the full imports closure. The resolved IRI is expressed as a prefixed name if a matching prefix is declared, or as `<full-iri>` otherwise. An unresolved label produces a warning and will cause a parse error. This resolution also applies to `--rule-text` and `--query-text` inline expressions. The `--list-rules --format markdown` output uses single quotes.
- Unquoted predicate names must be valid SWRLAPI identifiers: an IRI fragment name (e.g. `hasSpecifiedInput`) or a prefixed name (e.g. `ex:hasSpecifiedInput`).

### Colour configuration

When `--format markdown` is active, argument values in the debug table are wrapped in HTML class spans (e.g. `<span class="material">...</span>`) and a `<style>` block is prepended.  The mapping from predicates to entity types, and from entity types to CSS rules, can be customised via a YAML config file.

Config is loaded from these locations in order (later entries override earlier ones):

1. `~/.swrlcli/swrlcli_config.yaml` — global user config
2. `swrlcli_config.yaml` in the same directory as the ontology file — per-project config
3. The path given with `--config` — explicit override

Example `swrlcli_config.yaml`:

<pre>
entity_styles:
  <span style="color:blue">process</span>:              "span {color: blue;}"
  <span style="color:green">material</span>:             "span {color: green;}"
  <span style="color:tan">characteristic</span>:       "span {color: tan;}"
  <span style="color:saddlebrown">characteristic_value</span>: "span {color: saddlebrown;}"
  <span style="color:dimgray">information</span>:          "span {color: dimgray;}"
  <span style="color:red">error</span>:               "span {color: red;}"   # unbound variables — override to change highlight

predicate_styles:
  "has specified input":       [<span style="color:blue">process</span>, <span style="color:green">material</span>]
  "has specified output":      [<span style="color:blue">process</span>, <span style="color:green">material</span>]
  "has characteristic":        [<span style="color:green">material</span>, <span style="color:tan">characteristic</span>]
  "has characteristic value":  [<span style="color:tan">characteristic</span>, <span style="color:saddlebrown">characteristic_value</span>]
  "has quantity":              [<span style="color:saddlebrown">characteristic_value</span>, <span style="color:dimgray">information</span>]
  "mass in kilograms":         [<span style="color:saddlebrown">characteristic_value</span>]
  "combining two materials":   [<span style="color:blue">process</span>]
  "swrlb:add":                 [<span style="color:dimgray">information</span>]
  "temperature":               [<span style="color:tan">characteristic</span>]
  "characteristic value of":   [<span style="color:saddlebrown">characteristic_value</span>, <span style="color:tan">characteristic</span>]
</pre>

Entity style values follow the pattern `"element {css-properties}"`, where `element` is the HTML element to use (e.g. `span`) and the braces contain standard CSS.  If a predicate has more arguments than entries in its style list, the last entry is repeated.

The built-in `error` entity style (red) is applied automatically to any argument that is still unbound at display time — identifiable by a leading `?` — in both the match and variables columns.  Override it in the config to change the unbound-variable highlight color.

### Inverse property assertions and OWL reasoning

SWRLAPI/Drools operates on explicitly asserted axioms only — it does **not** materialise entailments from OWL semantics. A common consequence involves inverse object properties: if the ontology declares `hasPart owl:inverseOf isPartOf` and asserts `hasPart(A, B)`, the entailment `isPartOf(B, A)` is not automatically available in the Drools working memory. A SWRL rule whose body contains `isPartOf(?x, ?y)` will therefore not fire for that pair. The `--debug` evaluation table flags this case with an `INVERSE ONLY` note on the failing atom.

The correct fix is to run a full OWL DL reasoner first to materialise the entailed property assertions, then run SWRLCLI against the augmented ontology:

1. Open the ontology in [Protégé](https://protege.stanford.edu) and select **Reasoner → HermiT**, then run **Reasoner → Start reasoner** followed by **Reasoner → Export inferred axioms as ontology**.  Include *Object property assertions* in the export.  Save the result as a separate OWL file (e.g. `inferred.ofn`).
2. Add `owl:imports` of `inferred.ofn` to your working ontology, or load it alongside it.
3. Run SWRLCLI — the inverse assertions are now explicit and Drools will match them.

**Why not ELK?** ELK implements the OWL EL profile, which is optimised for fast TBox (class hierarchy) reasoning and does not support `owl:inverseOf` for materialising individual-level property assertions. [HermiT](http://www.hermit-reasoner.com) supports full OWL DL and correctly handles inverse property completion at the ABox level. Pellet/OpenLlet is another suitable alternative.

As a lightweight workaround when you control the rules, rewrite to use the asserted direction and swap the variables — replace `isPartOf(?x, ?y)` with `hasPart(?y, ?x)`.

### Import resolution

If a `catalog-v001.xml` file (as generated by Protégé) is present in the same directory as the ontology, the CLI applies it automatically to resolve `owl:imports` IRIs to local files. Only entries whose local file exists are used; no network fetches are attempted from the catalog.

### License

The software is licensed under the [BSD 2-clause License](https://opensource.org/licenses/BSD-2-Clause).
