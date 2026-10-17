# Building the `.ttl`: step by step

This turns the v2 design pattern (`../presentation-1/course-reflection/02-document-acts.md`) into an OWL ontology you can reason over and query. Every ROBOT command below was dry-run on 2026-10-17 against CCO v2.2 (2026-08-13) with `~/tools/robot.jar` and Java 25.

**Where you work:** `student-work/luay9/final-project/ontology/`
**Tooling:** ROBOT (`java -jar ~/tools/robot.jar`), HermiT (bundled in ROBOT), and Protégé if you want to look at the result.
**Rule:** never import full CCO. Extract only the terms the pattern uses.

```text
final-project/ontology/
├── cco-terms.txt              # step 2 input (already written)
├── cco-module.ttl             # step 2 output: the slice of BFO+CCO we use
├── meeting-mandate.ttl        # steps 3–5: YOUR ontology (classes, axioms, IAO alignment)
├── test-individuals.ttl       # step 6: synthetic Program A data
├── meeting-mandate-full.ttl   # step 7 output: merged + reasoned
├── queries/cq1…cq5.rq         # step 9
└── results/                   # step 9–10 outputs (CSV, report)
```

Shortcut used below:

```bash
R="java -jar $HOME/tools/robot.jar"
CCO=/c/Users/luaya/UB_applied_Onto/Ontology-Tradecraft/projects/project-2/assignment/CommonCoreOntologiesMerged.ttl
cd /c/Users/luaya/UB_applied_Onto/Ontology-and-Intel-Analysis/student-work/luay9/final-project/ontology
```

---

## Step 1: Fix the competency questions first (30 min)

The CQs decide what goes in. Anything no CQ needs stays out. Start with these five:

| # | Competency question | What it tests |
|---|---|---|
| CQ1 | Which mandates has the chair had to issue more than once? | `precedes` between mandates: the recurrence signal |
| CQ2 | Who issued a given mandate, and in virtue of what role? | `has agent` + `realizes` Authority Role (felicity condition) |
| CQ3 | Which documents carry a given mandate? | minutes `has continuant part` mandate; the identifier links documents |
| CQ4 | Which meetings are decision meetings? | the v2 definition, checked by the **reasoner**, not asserted |
| CQ5 | What work does a mandate prescribe, and which objective does it serve? | `prescribes` Planned Act; link to Objective |

Write them into `03-competency-questions.md`, and say for each one which classes and relations answer it.

## Step 2: Extract the CCO module (5 min)

`cco-terms.txt` lists the 16 CCO classes and 12 relations the pattern uses. BOT extraction pulls in each term plus its superclass chain up to BFO.

```bash
$R extract --input $CCO --method BOT --term-file cco-terms.txt --individuals exclude --output cco-module.ttl
```

**Check:** about 95 classes and 30 object properties (`grep -c "owl:Class" cco-module.ttl`). If you add a term later, add its IRI to `cco-terms.txt` and re-run. Never hand-edit `cco-module.ttl`.

## Step 3: Write the ontology header and new classes (1–2 h)

Create `meeting-mandate.ttl`. Use one namespace of your own for new terms. Everything else is a CCO/BFO IRI.

```turtle
@prefix :     <https://github.com/luay9/meeting-mandate#> .
@prefix owl:  <http://www.w3.org/2002/07/owl#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix skos: <http://www.w3.org/2004/02/skos/core#> .
@prefix obo:  <http://purl.obolibrary.org/obo/> .
@prefix cco:  <https://www.commoncoreontologies.org/> .
@prefix dcterms: <http://purl.org/dc/terms/> .

<https://github.com/luay9/meeting-mandate> a owl:Ontology ;
    dcterms:title "Meeting & Mandate ontology"@en ;
    dcterms:description "Design pattern for executive meetings, the mandates they produce, and the documents that carry them."@en ;
    dcterms:license <https://creativecommons.org/licenses/by/4.0/> ;
    owl:versionInfo "0.1" .

# Declare every CCO/BFO relation you use. Put this block in EVERY .ttl file
# (including test-individuals.ttl). Without it, ROBOT parses each file on its own,
# reads the undeclared relations as plain annotations, and the reasoner never sees them.
obo:BFO_0000055 a owl:ObjectProperty .   # realizes
obo:BFO_0000057 a owl:ObjectProperty .   # has participant
obo:BFO_0000063 a owl:ObjectProperty .   # precedes
obo:BFO_0000101 a owl:ObjectProperty .   # is carrier of
obo:BFO_0000117 a owl:ObjectProperty .   # has occurrent part
obo:BFO_0000178 a owl:ObjectProperty .   # has continuant part
obo:BFO_0000196 a owl:ObjectProperty .   # bearer of
cco:ont00001833 a owl:ObjectProperty .   # has agent
cco:ont00001986 a owl:ObjectProperty .   # has output
cco:ont00001816 a owl:ObjectProperty .   # is output of
cco:ont00001942 a owl:ObjectProperty .   # prescribes
cco:ont00001916 a owl:ObjectProperty .   # designates
```

The new classes. Keep the list this short: everything else is reused.

| New class | Parent | Why it's new |
|---|---|---|
| `:DecisionMeeting` | `cco:Act of Meeting` | the v2 subtype; **defined**, so the reasoner classifies it |
| `:Mandate` | `cco:Process Regulation` | a process regulation that came out of a meeting |
| `:MeetingMinutes` | `cco:Report` | the minutes document type (downward population, IAO-Intel style) |
| `:OfficialLetter` | `cco:Document Content Entity` | letter type |
| `:MeetingPresentation` | `cco:Document Content Entity` | deck type |
| `:MandateIdentifier` | `cco:Non-Name Identifier` | an ID like "M06 v1.1" |

Every class gets `rdfs:label`, `skos:definition` (genus-differentia, from `01-definition.md`) and `skos:prefLabel` in `@en` and `@ar`.

## Step 4: Write the axioms (1–2 h)

Two kinds:
- **Defined classes (≡)** let the reasoner infer membership. That's how CQ4 is answered.
- **Necessary conditions (⊑)** constrain what counts as a member.

```turtle
# Decision meeting ≡ Act of Meeting and (has occurrent part some
#     (Act of Directive Communication and (has output some Mandate)))
:DecisionMeeting a owl:Class ;
    rdfs:label "decision meeting"@en ;
    skos:prefLabel "decision meeting"@en , "اجتماع قرار"@ar ;
    skos:definition "A meeting that has as part an act of directive communication, performed by a person bearing an authority role, whose output is a mandate."@en ;
    owl:equivalentClass [ a owl:Class ;
        owl:intersectionOf ( cco:ont00000144
            [ a owl:Restriction ; owl:onProperty obo:BFO_0000117 ;
              owl:someValuesFrom [ a owl:Class ; owl:intersectionOf ( cco:ont00000133
                  [ a owl:Restriction ; owl:onProperty cco:ont00001986 ; owl:someValuesFrom :Mandate ] ) ] ] ) ] .

# Mandate ⊑ Process Regulation and (is output of some Act of Directive Communication)
:Mandate a owl:Class ;
    rdfs:label "mandate"@en ;
    skos:prefLabel "mandate"@en , "توجيه"@ar ;
    skos:definition "A process regulation that is the output of an act of directive communication performed in a meeting."@en ;
    rdfs:subClassOf cco:ont00001324 ,
        [ a owl:Restriction ; owl:onProperty cco:ont00001816 ; owl:someValuesFrom cco:ont00000133 ] .

# Mandate identifier ⊑ Non-Name Identifier and (designates some Mandate)
:MandateIdentifier a owl:Class ;
    rdfs:subClassOf cco:ont00000649 ,
        [ a owl:Restriction ; owl:onProperty cco:ont00001916 ; owl:someValuesFrom :Mandate ] .
```

**Don't** write "minutes ⊑ has continuant part some Mandate". Not every set of minutes contains a mandate (the status briefing again). Assert it per individual instead.

## Step 5: IAO alignment, as annotations only (20 min)

No `owl:imports` of IAO. Use SKOS mappings from `03-file-engineering-iao-alignment.md`:

```turtle
:MeetingMinutes skos:exactMatch obo:IAO_0000088 .   # report
:Mandate        skos:closeMatch obo:IAO_0000033 .   # directive information entity
:OfficialLetter skos:closeMatch obo:IAO_0000310 .   # document
:MeetingPresentation skos:closeMatch obo:IAO_0000310 .
:MandateIdentifier   skos:closeMatch obo:IAO_0020000 .  # identifier
```

Because these are annotations, they never affect reasoning. That's the point: there's no second "document" class to clash with CCO.

## Step 6: Synthetic test individuals (1 h)

In `test-individuals.ttl` (start it with the same prefixes and relation-declaration block as step 3), create Program A, using the same data as the synthetic timeline:

- `chair_1` (Person) bears `authority_role_1`; `secretary_1` (Person)
- `meeting_1` … `meeting_4` (Act of Meeting); **do not** type any as DecisionMeeting, so the reasoner has to infer it
- `meeting_3` is a status briefing with no deciding part. It must **not** be classified as a decision meeting
- `deciding_m06_v1_0` and `deciding_m06_v1_1` (Act of Directive Communication), each `has output` a `:Mandate`, with `deciding_m06_v1_0 precedes deciding_m06_v1_1`
- `documenting_1` (Act of Official Documentation) → `minutes_file_1` (Information Bearing Entity) `is carrier of` `minutes_1` (`:MeetingMinutes`), which `has continuant part` `m06_v1_0`
- `id_m06_v1_0` (`:MandateIdentifier`) `designates` `m06_v1_0`, with an `rdfs:label "M06 v1.0"`
- the same for M01 and M05 (re-issued), and M02 (issued once)

Use names like "chair_1" only. No real people or programmes.

## Step 7: Merge and reason (5 min)

```bash
$R merge --input cco-module.ttl --input meeting-mandate.ttl --input test-individuals.ttl \
  reason --reasoner hermit --axiom-generators "SubClass ClassAssertion" --include-indirect true \
  --output meeting-mandate-full.ttl
```

**Checks:**
1. It finishes without an "inconsistent ontology" error.
2. `meeting_1`, `meeting_2` and `meeting_4` are now typed `:DecisionMeeting`. `meeting_3` (the briefing) is **not**. That's your v2 definition passing a machine test.
3. Each `:Mandate` individual is also inferred to be a `cco:Process Regulation` and a `bfo:generically dependent continuant`.

## Step 8: A deliberate failure test (15 min)

Show that the axioms actually bite. Temporarily add a bad individual, for example something typed both `:Mandate` and `cco:Person`. A generically dependent continuant can't be an object, and BFO makes those disjoint. Re-run step 7 and confirm HermiT reports it inconsistent. Then remove it, and record the result in the validation write-up.

## Step 9: Answer the competency questions with SPARQL (1–2 h)

One file per CQ in `queries/`. For example, CQ1 (re-issued mandates):

```sparql
PREFIX cco: <https://www.commoncoreontologies.org/>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
SELECT ?earlierID ?laterID WHERE {
  ?d1 cco:ont00001986 ?earlier .  ?d2 cco:ont00001986 ?later .
  ?d1 obo:BFO_0000063 ?d2 .
  ?i1 cco:ont00001916 ?earlier ; rdfs:label ?earlierID .
  ?i2 cco:ont00001916 ?later   ; rdfs:label ?laterID .
}
```

Run against the **reasoned** file, so CQ4 can use inferred types:

```bash
mkdir -p results
$R query --input meeting-mandate-full.ttl --query queries/cq1.rq results/cq1.csv
```

**Check:** CQ1 returns exactly M01, M05 and M06 (v1.0 → v1.1), and not M02.

## Step 10: Quality report (10 min)

```bash
$R report --input meeting-mandate.ttl --fail-on none --output results/report.tsv
```

Fix anything in **your** file: missing labels or definitions, duplicate labels. Run it on `meeting-mandate.ttl` only. CCO's own metadata gaps aren't yours to fix.

## Step 11: Write up and push

- Write the 7 packet documents listed in `README.md`. The validation section is steps 7–10, with the actual outputs.
- Add 2 slides to the deck for Presentation 2: **"What's implemented"** (the class table from step 3 + the merge/reason command) and **"How it was tested"** (meeting_3 not classified, the failure test, CQ1 results).
- Commit and push (see the main README for when).

---

**Estimated total:** 6–9 hours, spread over Week 9 (Oct 19–25).

**Known traps:**
- **Undeclared relations become annotations.** This was found in the dry run: `:DecisionMeeting` was not inferred until the relations were declared as `owl:ObjectProperty` in the same file (step 3). With the declarations, `meeting_1` was classified as a decision meeting, the briefing `meeting_3` correctly wasn't, and a Mandate-that-is-a-Person was reported inconsistent.
- `robot reason --output /dev/null` fails on Windows. Always write to a real file.
- Use CCO's opaque IRIs (`ont00000144`), not labels, in Turtle. Put the label in a comment.
- Don't type meetings as `:DecisionMeeting` by hand. If you assert it, the reasoner proves nothing.
