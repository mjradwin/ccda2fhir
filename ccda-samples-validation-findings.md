# HL7 validator findings on the `ccda-samples` corpus (session notes)

Working notes so a later session can resume without re-deriving anything. Not part of the project:
kept on the fork's `notes/ccda-samples-corpus` branch (`mjradwin/ccda2fhir`), never in a PR. Written 2026-11-05.

> **Status (updated 2026-11-05, session 2): fix #2 is sent as PR
> [alvinhenrick/ccda2fhir#80](https://github.com/alvinhenrick/ccda2fhir/pull/80)** (from the fork
> `mjradwin/ccda2fhir`, branch `fix/display-table`; local remote `fork`). Fix #1 is
> [#81](https://github.com/alvinhenrick/ccda2fhir/pull/81) (`fix/encounter-type-null-coding`) and fix #3 is
> [#82](https://github.com/alvinhenrick/ccda2fhir/pull/82) (`fix/organization-home-use`). The other 4
> branches (#4, #5, #6, #7) are pushed to the fork too, but **no PRs are open for them yet**. **All 7 fixes have now been implemented and verified locally**, each on its own
> local branch off `main` (`26d759d`), with unit tests, clippy clean, and a before/after HL7 validator run.
> Results, corrections to section 4 and a PR grouping recommendation are in
> [section 10](#10-fix-experiments-session-2-2026-11-05). Read that first. Sections 4 and 6 are the
> original (session 1) analysis; where section 10 disagrees, section 10 is right.
> The user wants to review before anything outward-facing happens ("don't file any bug reports without
> checking with me first"): show final issue/PR text and get an explicit go-ahead before `gh issue create`,
> `git push` or `gh pr create`.

## 1. What was done

1. Installed Rust with rustup (`stable-aarch64-apple-darwin`, rustc 1.99.1; the project needs >= 1.87).
   In a new shell run `. "$HOME/.cargo/env"` (or open a new terminal) to get `cargo` on PATH.
2. `cargo build --release` in this repo (about 23 s). The binary is `target/release/ccda2fhir`
   (the README says `ccda2fhir-server`, which is the crate name; the binary is `ccda2fhir`).
   Subcommands: `convert`, `serve`, `healthcheck`.
3. Cloned `https://github.com/jddamore/ccda-samples` (shallow) to `~/dev/ccda-samples`, a sibling of this
   repo. 382 `.xml` files, from many EHR vendors (folder per vendor, some names contain spaces).
4. Converted a random 10 % sample (38 files), then all 382. **All 382 converted with exit 0, valid JSON, empty
   stderr.** Output: 14,748 resources.
5. Ran the official HL7 validator on all 382 bundles (FHIR R4 + US Core 6.1.0).
6. Compared the errors with `scripts/corpus-known-errors.txt` using the project's own
   `scripts/corpus_gate.py`, then traced the structural errors to source XML and converter code.

### Versions and provenance (for reproducibility)

| Thing | Value |
| --- | --- |
| This repo | `alvinhenrick/ccda2fhir`, branch `main`, commit `26d759d678a7985f478320c555fcd94a5104d1e6` (v0.1.1, "chore: release v0.1.1 (#79)") |
| Samples repo | `jddamore/ccda-samples`, commit `96a8e7a54e7e1eeb6b200ad26740ca6ff55dfb46` |
| Validator | `validator_cli.jar` 7.0.3, sha256 `57aa43fb40cec5de41def81c927725fd85908eeb71509b7f9313e04d9b8ec79b` (downloaded from the hapifhir/org.hl7.fhir.core GitHub release) |
| Validation settings | `-version 4.0.1 -ig hl7.fhir.us.core#6.1.0`, run as `-server` on port 8090, bundles POSTed to `/validateResource`, 4 in parallel. Same as `scripts/validate.sh`. |
| Terminology | The validator contacted `tx.fhir.org` (a few timeouts, see 5.4). No `--loinc-table` was passed to the converter. |
| `gh` | Logged in as `mjradwin`. Repo issue tracker is currently empty (`gh issue list -R alvinhenrick/ccda2fhir --state all` printed nothing). |

The repo has no `.github/ISSUE_TEMPLATE` (only `workflows/`). `CONTRIBUTING.md` asks for: never commit real
patient data (fixtures must be synthetic or from public HL7/ONC sets, source noted); never put document values
in logs or report messages; mapping changes cite the source (IG row, C-CDA template id, US Core element);
lossy mappings emit a `Report` issue with a stable code; PR titles use conventional-commit types
(`fix:`, `feat:` ...). Decisions go in `docs/decisions/` (ADRs).

## 2. Headline results

* 382 / 382 converted. 380 / 382 validated. **Two bundles got an empty response from the validator** and were
  not checked (cause unknown, nothing in the server log):
  * `Allscripts TouchWorks/LaurenGordon_170.315_b2_ciri__r21_sample2_rn_export.xml` (bundle `041`)
  * `Henry Schein/201703131227CDA_TurnerS_19700801.xml` (bundle `165`)
* Of the 380 validated: 377 have at least one error, 3 have none.
* Issue counts: 7,535 errors, 7,159 warnings, 14,640 information.
* Errors by kind: display-name mismatch 4,206 (359 files); "unable to find a profile match" 2,111 (123 files);
  other 765 (228 files); unknown code 453 (184 files).
* `corpus_gate.py` against the known-errors file: 1,118 distinct (normalised) error kinds, **1,082 not in the
  known list**, 36 known. The gate exits 1. That is expected for a different corpus than HL7's own examples
  (the known list is about 12 patterns: CPT displays, a few misused codes in HL7's examples, the 3141-9 and
  59408-5 vital-sign codes).
* Most of the 1,082 are **not converter bugs**: wrong display text in the source documents, invalid NPIs,
  period end before start, validator or terminology-server limits. About 20 kinds are structural and traced
  to the converter (section 4).

Resource mix across all 382 bundles: Observation 3,237; Condition 1,414; MedicationRequest 1,196;
Practitioner 1,100; PractitionerRole 1,020; Provenance 925; Organization 709; Device 589.

## 3. Project context that matters when writing issues

* **Display policy, ADR 0027** (`docs/decisions/0027-display-names.md`): if a code is in the built-in
  `ccda_terminology::official_display` table and the document's display differs, the official display is used,
  the document's wording is kept as `text`, and the report gets `coding.display.replaced`. If the code is not
  in the table, the document's display is kept unchanged ("we never guess a display"). The table "grows from
  validator findings". SNOMED, RxNorm and CPT are deliberately not bundled; LOINC displays come from an
  optional `--loinc-table` (`LONG_COMMON_NAME`). The table is `official_display()` at
  `crates/ccda-terminology/src/lib.rs:308`.
* **ADR 0066** (`0066-corpus-findings.md`): the first HL7-examples corpus run; documents the NPI,
  language, race/ethnicity `UNK`/`ASKU` (v3 NullFlavor system) and UCUM-normalisation decisions.
* **Known-errors file** `scripts/corpus-known-errors.txt`, gate `scripts/corpus_gate.py` (normalises
  indexes, uuids, line numbers; matches regexes from the known file; the CI corpus job fails on any other
  error).
* The converter's rule on codes: the known-errors file says "we do not add a code the document does not
  carry" (BodyWeightCode comment). This matters for the BMI-unit question (5.1).
* Goal already follows the "do not claim a profile you cannot satisfy" rule (ADR 0014: a goal with no
  description "does not claim `us-core-goal`"). Vitals do not follow it (issue 4).

## 4. Candidate issues (the "7")

Priority order. File/line references are from commit `26d759d`.

| # | Title (short) | Evidence size | Confidence in root cause | Session 2: verified fix? |
| --- | --- | --- | --- | --- |
| 1 | Encounter `type` gets `"coding": null` | 36 files, 41 resources | High (mechanism understood, not yet reproduced with a fix) | **Yes**: 80 → 0 errors, 36/36 files |
| 2 | Display-table gaps cascade into us-core-patient failures | 123 files, 2,111 errors | Medium-high (see caveat in 4.2) | **Yes, but the table alone was not enough** (10.2): Patient cascade cleared in 70/72 files; corpus 7,535 → 4,686 errors |
| 3 | Organization address/telecom `use: "home"` | 11 + 10 files | Medium (source element not fully pinned, see 4.3) | **Yes**: 23 → 0, 21/21 files; source pinned |
| 4 | Vitals claim a US Core profile but have no `effective[x]` | ~~roughly 30~~ **19 files, 61 errors** | High for the cause; count approximate | **Yes, but neither proposed option worked alone** (10.4) |
| 5 | Missing `deviceIdentifier` on implantable Devices | 18 Device resources | High | **Yes**: 18 → 0 |
| 6 | Goal has no `description` although the narrative has the text | 29 files, 42 Goals | High for the one traced case | **Yes**: 42 → 0; text fallback alone fixes only 19 (10.6); also found a parser bug |
| 7 | Encounter `identifier` with no system/value | 19 files (incl. 2 Practitioner) | High | **Yes**: 32 → 0, with a narrower design than proposed (10.7) |

### 4.1 Encounter `type[].coding = null` (bug, invalid JSON structure)

* **Symptom:** validator: `The property coding must be a JSON Array, not a Null (at Bundle.entry[].resource.type[])`
  and `... must be an Array, not a Null`. 36 files; 41 Encounter resources; it is the only `null` anywhere in
  all 382 bundles.
* **Example:** `Agastha/195352.xml` (bundle `015`). Source:
  ```xml
  <code nullFlavor="UNK">
    <originalText>New Patient<reference value="#Encounter1"/></originalText>
  </code>
  ```
  Output: `"type":[{"coding":null,"text":"New Patient"}]`.
* **Cause:** `crates/ccda-map/src/encounter.rs:215`: `if let Some(codings) = ty["coding"].as_array_mut()`.
  `serde_json::Value`'s `IndexMut<&str>` **inserts `null` for a missing key**. `map_codeable_concept`
  (`coding.rs:82`) correctly omits `coding` when there are no codings, then this line adds it back as `null`.
  The existing guard `if codings.is_empty() { remove("coding") }` is never reached because the value is
  `null`, not an array.
* **Proposed fix:** `if let Some(codings) = ty.get_mut("coding").and_then(Value::as_array_mut) { ... }`
  (a non-inserting lookup). Add a regression test: an Encounter whose `code` is a nullFlavor with only
  `originalText` must produce `type[0] == {"text": ...}` with no `coding` key.
* **Other sites checked:** `health_concern.rs:127` and `sdoh.rs:56` use the same
  `["category"].as_array_mut()` pattern but assign to `category` immediately in the `None` branch, so they
  are harmless. Searched with
  `grep -rn '\]\.as_array_mut()\|\]\.as_object_mut()\|\]\.as_str_mut' crates/*/src`.

### 4.2 Display-table gaps cascade into `us-core-patient` failures (highest value)

* **Symptom:** 72 bundles have `Unable to find a profile match for urn:uuid:... among choices:
  .../us-core-patient` on `Condition.subject`, `AllergyIntolerance.patient`, `CarePlan.subject`,
  `Observation.subject`, `Encounter.subject`, and so on (123 files in total counting other profile-match
  kinds). Every one of the 72 also has errors located on the Patient resource itself, i.e. the Patient fails
  its profile and every reference to it then fails too.
* **Patient-level errors behind it** (count = number of those 72 bundles):
  35 `The code provided (urn:oid:2.16.840.1.113883.6.238#2108-9) was not found in the value set 'Detailed
  Race'`; 32 `Wrong Display Name 'White European' ... Valid display is 'European'`; 14 `Wrong Display Name
  'Asked but no answer' for ...v3-NullFlavor#ASKU. Valid display is 'asked but unknown'`; 11 marital status
  `S` 'Single' vs 'Never Married'; 8 and 7 `The Coding provided (v3-NullFlavor#ASKU) was not found in the value
  set 'OMB Ethnicity/Race Categories'`; 7 `RWR` 'Receive Written' vs 'Received written'; 5 `UNK` 'UNKNOWN BY
  PATIENT' vs 'unknown'; 4 ethnicity `2186-5` 'Non Hispanic or Latino' vs 'Not Hispanic or Latino'.
* **Key observation:** in the ASKU/UNK cases the *code* is right and the ombCategory value-set error message
  is chained from the wrong display ("(error message = Wrong Display Name 'Asked but no answer' ...)"), so
  it looks like fixing the display alone fixes them. Examples: bundle `016` (ethnicity ASKU, race UNK) and
  bundle `141` (race and ethnicity ASKU).
* **Code/display pairs where the validator names a single valid display** (non-SNOMED/RxNorm/CPT/NUCC;
  count = bundles). These are candidate rows for `official_display`:
  * `http://terminology.hl7.org/CodeSystem/v3-ParticipationFunction`: `PCP` -> "primary care physician" (97),
    `ATTPHYS` -> "attending physician" (60), `ADMPHYS` -> "admitting physician" (59)
  * `http://terminology.hl7.org/CodeSystem/v3-NullFlavor`: `ASKU` -> "asked but unknown" (14),
    `UNK` -> "unknown" (5)
  * `http://terminology.hl7.org/CodeSystem/v3-MaritalStatus`: `S` -> "Never Married" (11)
  * `http://terminology.hl7.org/CodeSystem/v3-LanguageAbilityMode`: `RWR` -> "Received written" (7)
  * `urn:oid:2.16.840.1.113883.6.238` (CDC race/ethnicity): `2108-9` -> "European" (35),
    `2186-5` -> "Not Hispanic or Latino" (4)
  * `http://hl7.org/fhir/sid/cvx`: `88` -> "influenza, unspecified formulation" (50),
    `106` -> "DTaP, 5 pertussis antigens" (42),
    `166` -> "influenza, intradermal, quadrivalent, preservative free" (23),
    `150` -> "Influenza, split virus, quadrivalent, PF" (4)
  * `https://www.cdc.gov/nhsn/cdaportal/terminology/codesystem/hsloc.html`: `1160-1` -> "Urgent Care Center" (22)
  * ICD-10-CM `I10` -> "Essential (primary) hypertension" (8), `R50.9` -> "Fever, unspecified" (7), and
    a few more (`T86.11`, `E03.8`, `Z00.00`)
  * NCI Thesaurus: `C48623` -> "NOT APPLICABLE" (14), `C38192` -> "AURICULAR (OTIC)" (5)
  * NDC package codes (long, product-specific strings) are probably not worth bundling.
* **CAVEAT (not verified):** I did not re-run the validator after hypothetically fixing the display, so
  "fixing the table clears the Patient failures" is inferred from the chained error message, not proven. The
  `2178-2` Detailed-ethnicity error (2 bundles) is a validator limitation ("modifier extension
  vsacOpModifier"), not a converter issue. The race `2108-9` "not found in value set Detailed Race" error also
  carries the Wrong-Display text, so it probably clears with the display fix, but that should be confirmed.
* **LOINC (not part of this issue):** LOINC section-code display mismatches ('Medical Equipment' for 46264-8,
  'Assessments'/'ASSESSMENTS' for 51848-0, 'Discharge Diagnosis' for 78375-3, ...) are very common (hundreds
  of files). I ran without `--loinc-table`, which is how the project expects LOINC displays to be supplied,
  so these are not reported. If the project's intent is that those section codes are built in (ADR 0027 says
  the table "starts with the LOINC section codes"), it may be worth checking whether 46264-8, 51848-0 and
  78375-3 are missing from `loinc_display()` (the earlier grep for them in `crates` failed on a shell glob
  error; re-run `grep -rn '51848-0\|46264-8\|78375-3' crates/`).
* **Proposed fix:** add the rows above to `official_display()` (checked against the validator's own message,
  per ADR 0027), with a unit test per system. Optionally generate the v3 rows from the HL7 terminology
  package instead of hand-entering them.

### 4.3 Organization `address.use` / `telecom.use` = `home`

* **Symptom:** `Constraint failed: org-2: 'An address of an organization can never be of use 'home''`
  (11 files) and `org-3: 'The telecom of an organization can never be of use 'home''` (10 files).
* **Example:** bundle `038` = `Allscripts TouchWorks/Allscripts TW Alice rn.xml`. The Organization has
  `"address":[{"state":"WA","use":"home"}]`. The source has several `<addr use="H">` elements (lines 32, 64,
  77, 95 ...).
* **Cause (partly inferred):** `crates/ccda-map/src/patient.rs:351` and `:405` map `H` and `HP` to
  `"home"` in a shared address/telecom mapper that is also used for Organizations.
* **CAVEAT:** I did not trace which source element (custodian, author organization, ...) in the example
  produced the Organization address, nor confirm that the Organization mapping calls the same
  `patient.rs` helpers. Confirm before filing.
* **Proposed fix:** when the owner is an Organization, omit `use` (or map `H`/`HP` to `work`), per the R4
  invariants org-2/org-3. Report it with a stable code if the project wants lossy changes reported.

### 4.4 Vitals assert a US Core profile but lack `effective[x]`

* **Symptom:** `Observation.effective[x]: minimum required = 1, but only found 0` against
  `bodyweight`, `bodyheight`, `us-core-body-weight`, `us-core-body-height`, `bp`, `us-core-blood-pressure`,
  `vitalspanel`, `heartrate`, `resprate`, `oxygensat`, `bodytemp`, `bmi`, `us-core-pulse-oximetry`.
  Per-profile counts are 8, 8, 8, 5, 4, 4, 11, 1 ... and overlap, so "roughly 30 files" is approximate
  (see the recompute command in section 8).
* **Example:** bundle `137` = `Edaris Forerun/bates-rn-fixed.xml`. Source line 466:
  `<effectiveTime nullFlavor="UNK" />` on a weight observation (LOINC 29463-7). Output has no `effective[x]`
  but `meta.profile = [".../us-core-body-weight"]`.
* **Cause:** the converter sets the US Core vital-sign profile unconditionally, but the profile requires
  `effective[x]` (1..1).
* **Proposed fix:** either emit `_effectiveDateTime` with a `data-absent-reason` extension (an element with an
  extension satisfies min=1; the converter already does this for `Encounter.class`), or do not claim the
  profile when `effective[x]` is absent, as ADR 0014 does for Goal.
* **Related, lower priority:** blood-pressure panels (LOINC 55284-4) with no components (11 files, e.g.
  bundles `357`, `358`). In those the source seems odd (the readings appear in a reference range:
  `"referenceRange":[{"text":"90/50^140/90"}]`), so I would not file it.

### 4.5 Missing `Device.udiCarrier.deviceIdentifier`

* **Symptom:** `Device.udiCarrier.deviceIdentifier: minimum required = 1, but only found 0 (from
  us-core-implantable-device|6.1.0)`. 18 Device resources.
* **Cause:** `device_identifier()` at `crates/ccda-map/src/device.rs:211` only accepts an HRF that starts
  with `(01)`. Two real-world forms are not recognised:
  * a **bare 14-digit UDI-DI** (10 resources), e.g. `00643169007222`
    (example: bundle `130` = `Carefluence/Larson_Rebecca_0_Inpatient.xml`, source line 1776:
    `<id root="2.16.840.1.113883.3.3719" extension="00643169007222"/>`, FDA UDI OID `2.16.840.1.113883.3.3719`)
  * an **un-parenthesised GS1 element string** (8 resources), e.g.
    `01006431690072221716012821BLC200461H` (AI `01` + 14-digit GTIN, then `17`, `21`, ...)
* **Proposed fix:** treat `^\d{14}$` as the DI, and accept a leading `01` + 14 digits when the string is not
  parenthesised. Optionally extend `parse_gs1` (lot, expiry, serial) to the non-parenthesised form.
  Existing tests are at `device.rs:319-337` (`non_gs1_udi_keeps_the_carrier_only` currently asserts the
  `+H123PARTNO...` HIBCC form has no DI, which should stay true).
* ADR 0042 documents the current UDI behaviour ("Non-GS1 UDIs keep the carrier only").

### 4.6 Goal has no `description` (narrative text unused)

* **Symptom:** `Goal.description: minimum required = 1, but only found 0` (29 files).
* **Example:** bundle `006` = `Advanced Technologies Group/SLI_CCD_b6AliceNewman_ATG_ATGEHR_10162017.xml`,
  source lines ~2016-2025:
  ```xml
  <observation classCode="OBS" moodCode="GOL">
    <templateId root="2.16.840.1.113883.10.20.22.4.121" />
    <id nullFlavor="UNK" />
    <code nullFlavor="NI" />
    <text representation="TXT"><reference value="#goal1" /></text>
  ```
  and the section narrative has `<paragraph ID="goal1">a. Get rid of intermittent fever ... b. Need to gain
  more energy ...</paragraph>`.
* **Cause:** `goal.rs` (`description()` at line 124) uses only the observation `code` or `value`; the
  converter already resolves `originalText` references, but not the entry's own `text/reference`. It reports
  `goal.description.missing` (a warning) and emits the Goal anyway; the base R4 Goal needs `description`.
* **Proposed fix:** fall back to the resolved `text/reference` as `description.text`; if still nothing, emit
  `description` with a `data-absent-reason` extension so the resource is valid.
* **Not verified:** that every one of the 29 files has a resolvable `text/reference`.

### 4.7 Encounter `identifier` entries with no `system` / `value`

* **Symptom:** `Encounter.identifier.system: minimum required = 1` (17 files) and `...value` (9 files)
  against `us-core-encounter|6.1.0`.
* **Example:** bundle `114` = `Atos Pulse/bates_patienthealthrecord_08032017.xml`, source lines 432-433:
  `<id nullFlavor="UNK"/>` then `<id root="1.3.6.1.4.1.21367.2010.1.2.300.2.6" extension="170314B2AMB" .../>`.
  Output `identifier` is
  `[{"extension":[{"url":".../data-absent-reason","valueCode":"unknown"}]}, {"system":"urn:oid:...","value":"170314B2AMB",...}]`.
* **Cause:** a null `<id>` becomes an identifier that holds only a data-absent-reason extension. US Core
  requires `system` and `value` on every Encounter identifier.
* **Proposed fix:** omit identifiers that carry no data (a data-absent-reason identifier adds nothing next to
  a real one); if it is the only id, leave `identifier` out.
* **Not checked:** whether the same pattern (DAR-only identifier) also appears on other resource types
  (Coverage, Goal and Practitioner identifiers showed the same shape in the output; Practitioner shows
  `identifier.system/value` errors in 2 files). Worth one search before filing, so the issue can say "all
  resource types" if true: look for `{"extension":[{"url":".../data-absent-reason"...}]}` as an `identifier`
  element across all bundles.

## 5. Not filed, and why

### 5.1 Design questions (could be a discussion, not a bug)

* **BMI observation without a unit** (23 files): `valueQuantity` is `{"value":28.2}`; the `bmi` profile needs
  `unit`, `system` and `code`. Example: bundle `048` (`Amrita/Admission_Inpatient_3156206_CCD_201709180920.xml`,
  source line 718: `<value xsi:type="PQ" value="28.2"/>`, LOINC 39156-5). The project's stated rule is not to
  add a code the document does not carry, so inferring `kg/m2` is a policy choice. Same shape: pulse-oximetry
  concentration component (2 files) and oxygen saturation (1 file).
* **Coverage without `payor`** (8 files): the source has no payer; R4 Coverage requires `payor`. A
  `data-absent-reason` extension (or `display`) would satisfy it. Example: bundle `025` (the Coverage has only a
  DAR-only `identifier` of type `MB`).
* **Upper-case or non-canonical UCUM unit codes** (about 12 files in total): `KG`, `CEL`, `cel`, `DEG`,
  `[DEGF]`, `CM`, `MM[HG]`, `/MIN`, `Cel`, `mL/dL`. UCUM is case-sensitive; the validator rejects
  `The System URI could not be determined for the code 'KG'` and `Value is 'MM[HG]' but is fixed to 'mm[Hg]'`.
  ADR 0066 already normalises some spellings (`mcg`, `10+3/ul`, `mg/actuat`), so this would extend it with a
  small case-insensitive table for the vital-sign units.

### 5.2 Source-data problems (not converter bugs)

* NPI errors: `us-core-16` / `us-core-17` ("NPI must be 10 digits", "check digit must be valid"), 60 files
  for Organization/Practitioner. ADR 0066 already puts invalid NPIs under `urn:oid:2.16.840.1.113883.4.6`
  instead of the NPI system; these are probably identifiers that still land under the NPI system. Not
  investigated further, so not ruled out as a converter issue.
* Period end before start (`per-1`, 24 files across MedicationRequest timing bounds, ServiceRequest,
  Encounter, Composition event). Two traced: bundle `153` (`end` 2015 vs `start` 2017) and bundle `237`
  (`MedHost Enterprise/CCD_2222455_37176_506.xml` source `low=20161202110000+0000`,
  `high=20161202060000+0000`). Both are wrong in the source.
* `OIDs must be valid (2.201)` (3 files), wrong or retired SNOMED/RxNorm/CVX displays, `Unknown code`
  (CPT 99201 and similar, a licensing/edition limit already covered in the known-errors file for CPT).
* Codes the validator calls inactive ("no longer considered a correct display ... status = inactive").

### 5.3 Likely not worth filing

* BP panel with no components (see 4.4 "Related").

### 5.4 Validator and terminology-server limits

* `Cannot process resource http://cts.nlm.nih.gov/fhir/ValueSet/2.16.840.1.114222.4.11.877|20190518 ...
  vsacOpModifier` (Detailed ethnicity `2178-2`, 2 bundles).
* tx.fhir.org timeouts: `Operation exceeded time limit of 20 seconds` (4 files) and
  `java.net.SocketTimeoutException` (1 file). A re-run might clear them.

## 6. Draft issue text

Each draft is intentionally small. Before filing, confirm the "Not verified" items in section 4, and check
that no snippet contains real patient data (the samples look like the standard ONC test patients such as
"Alice Newman" and "Jeremy Bates", but I did **not** read the `ccda-samples` README to confirm that). The
snippets below use only structural XML, not names or ids.

Common footer to add to all issues: "Found by running the converter (v0.1.1, `26d759d`) over
`jddamore/ccda-samples` @ `96a8e7a` and validating with HL7 validator 7.0.3 against R4 + US Core 6.1.0
(`scripts/validate.sh` approach). None of these appear in `scripts/corpus-known-errors.txt`."

**Issue 1: `fix: Encounter.type gets "coding": null when the code has only originalText`**
> A C-CDA Encounter whose `<code nullFlavor="UNK">` carries only `<originalText>` produces
> `"type":[{"coding":null,"text":"..."}]`, which is not valid FHIR JSON (validator: "The property coding must
> be a JSON Array, not a Null"). 36 of 382 sample documents.
>
> Cause: `crates/ccda-map/src/encounter.rs:215` uses `ty["coding"].as_array_mut()`; serde_json's `IndexMut`
> inserts `null` for a missing key, after `map_codeable_concept` had correctly left `coding` out.
>
> Suggested fix: `ty.get_mut("coding").and_then(Value::as_array_mut)`, plus a test for a nullFlavor code with
> `originalText`.

**Issue 2: `feat: add displays for common v3, CDC race/ethnicity and CVX codes (Patient fails us-core-patient)`**
> The display is kept as written when a code is not in `official_display` (ADR 0027). For the codes below the
> validator rejects the document's wording; on the Patient this makes `us-core-patient` fail, and every resource
> that references the Patient then reports "Unable to find a profile match". 72 of 382 documents hit this
> cascade. [table of code -> display from 4.2, with counts]
>
> Suggested fix: add the rows to `official_display` (ADR 0027 says the table grows from validator findings),
> one test per system.

**Issue 3: `fix: Organization address/telecom must not have use "home"`** (after confirming 4.3)
> [symptom, example, cause at `patient.rs:351/405`, suggested fix]

**Issue 4: `fix: vitals claim a US Core profile but have no effective[x]`**
> [symptom, example with `<effectiveTime nullFlavor="UNK"/>`, two fix options]

**Issue 5: `fix: accept bare and un-parenthesised GS1 UDI forms for Device.udiCarrier.deviceIdentifier`**
> [two forms and counts, `device.rs:211`, suggested fix]

**Issue 6: `fix: Goal.description from the entry's text/reference`**
> [symptom, source snippet with `<code nullFlavor="NI"/>` and `<text><reference value="#goal1"/>`, suggested
> fix, fallback to data-absent-reason]

**Issue 7: `fix: do not emit Encounter.identifier entries that carry only a data-absent-reason`**
> [symptom, source snippet, suggested fix]

Proposed title prefix follows `CONTRIBUTING.md` (`fix:`/`feat:`). The author may prefer to retitle.

## 7. How to resume

1. `cd ~/dev/ccda2fhir && . "$HOME/.cargo/env"`; read this file.
2. If the user says "file all 7": re-read the draft bodies, fill in the bracketed parts from section 4, run the
   "not verified" checks you can do cheaply (4.3 source element; 4.7 DAR-only identifiers on other types; 4.6
   the 29 files), show the final text, get a yes, then
   `gh issue create -R alvinhenrick/ccda2fhir --title ... --body-file ...` for each. Append the attribution
   line from the session reminder to bodies if one is given. Do not close or comment on anything else.
3. Alternatively, offer to verify each fix locally first: branch, apply the fix, `cargo test --workspace`,
   reconvert the affected files and re-run the validator on just those bundles. That gives each issue a
   "verified" proposed fix, or a PR the author can take. Do not open PRs without asking.
4. If the scratchpad is gone (it lives under `/private/tmp/...` and may be cleaned), regenerate it with the
   commands in section 8; all numbers above came from these steps.

## 8. Reproduction recipe

Original working directory (a session scratchpad, possibly deleted):
`/private/tmp/claude-502/-Users-mradwin-dev-ccda2fhir/cd9c69b6-4c24-449d-a034-f81aec33a5a1/scratchpad/`
with `all/NNN.json` (converted bundles), `all/files.txt` (file list; **bundle N = line N** of that file),
`outcomes/NNN.json` and `.txt` (validator output), `gate/` and `gate_summary.md` (corpus_gate output),
`validator/validator_cli.jar`, `validate_all.sh`.

Use any scratch directory `$S`:

```bash
# 0. toolchain and converter
. "$HOME/.cargo/env"
cd ~/dev/ccda2fhir && cargo build --release

# 1. convert everything (names may contain spaces; list order = bundle number)
cd ~/dev/ccda-samples
mkdir -p $S/all
find . -name '*.xml' -not -path './.git/*' | sort > $S/all/files.txt
i=0
while IFS= read -r f; do
  i=$((i+1)); n=$(printf '%03d' $i)
  ~/dev/ccda2fhir/target/release/ccda2fhir convert "$f" > $S/all/$n.json 2> $S/all/$n.err
done < $S/all/files.txt

# 2. validator jar (204,631,233 bytes; sha256 in section 1)
mkdir -p $S/validator
curl -fsSL -o $S/validator/validator_cli.jar \
  https://github.com/hapifhir/org.hl7.fhir.core/releases/download/7.0.3/validator_cli.jar

# 3. validate: start the service, POST each bundle, convert the outcome to text
java -jar $S/validator/validator_cli.jar -server 8090 -version 4.0.1 -ig hl7.fhir.us.core#6.1.0 &
#    wait until `curl -s http://localhost:8090/` answers, then for each NNN (4 at a time):
curl -sS -m 600 -X POST -H 'Content-Type: application/fhir+json' \
  --data-binary @$S/all/NNN.json http://localhost:8090/validateResource -o $S/outcomes/NNN.json
python3 ~/dev/ccda2fhir/scripts/oo_to_text.py $S/outcomes/NNN.json > $S/outcomes/NNN.txt
#    (the full loop took about 6 minutes for 382 bundles)

# 4. compare with the project's known errors (gate wants *.validation.txt names)
mkdir -p $S/gate
for f in $S/outcomes/*.txt; do cp $f $S/gate/$(basename $f .txt).validation.txt; done
python3 ~/dev/ccda2fhir/scripts/corpus_gate.py $S/gate \
  ~/dev/ccda2fhir/scripts/corpus-known-errors.txt $S/gate_summary.md   # exits 1 = unknown errors exist
```

Notes:
* `scripts/validate.sh` was **not** used as is: it wants file names without spaces, writes its cache to
  `.cache/` (not gitignored in this checkout; `.gitignore` does list `/.cache/`, so it is actually ignored
  after all), and converts the fixtures itself. The recipe above reproduces its validator settings.
* Docker is not installed on this machine, so the README's `docker compose --profile validate` rig was not
  used.
* Handy queries used during the analysis (Python over the `outcomes/*.json` OperationOutcomes):
  group by `issue[].details.text` after normalising uuids/indexes/quoted strings; categorise by prefix
  (`Wrong Display Name`, `Unknown code`, `Unable to find a profile match`, everything else); find the Patient-located
  errors for bundles with a profile-match failure via `issue[].expression[0]`.
* To recompute the issue-4 file count precisely: collect the set of bundle numbers whose outcomes contain an
  error text matching `Observation.effective\[x\]: minimum required = 1`.
* Source documents live in `~/dev/ccda-samples/<vendor>/<file>.xml`; bundle-to-source mapping examples used
  above: 006 ATG Alice Newman, 015 Agastha 195352.xml, 025 (Coverage example), 038 Allscripts TouchWorks
  `Allscripts TW Alice rn.xml`, 048 Amrita Admission_Inpatient, 114 Atos Pulse bates_patienthealthrecord,
  130 Carefluence Larson_Rebecca, 137 Edaris Forerun bates-rn-fixed.xml, 237 MedHost Enterprise
  CCD_2222455_37176_506.xml.

## 9. Other context from the session

* The user is new to Rust; keep explanations plain. They chose "Option 2" (native build, not Docker).
* No existing validator jar was found on this machine, so a fresh 7.0.3 jar was downloaded into the
  scratchpad (not into the repo). Note `~/.fhir/packages` may also hold an older US Core (4.1.0); always
  pass `-ig hl7.fhir.us.core#6.1.0` explicitly.
* The user's requirement for any outward-facing action: **confirm before filing issues**, and include a
  proposed fix hint in each.

## 10. Fix experiments (session 2, 2026-11-05)

Every fix was implemented, unit-tested and checked with the HL7 validator before and after, on just the
affected bundles. All 7 were then merged together and the whole corpus was run again. Nothing is pushed.

### 10.0 Summary

| # | Local branch (off `main` 26d759d) | Errors on affected bundles, before → after | New errors | Unit tests added |
| --- | --- | --- | --- | --- |
| 1 | `fix/encounter-type-null-coding` | 80 → 0 (36/36 files cleared) | none | 1 (fails without the fix, passes with it; checked) |
| 2 | `fix/display-table` | Patient profile-match 1,951 → 6; whole corpus 7,535 → 4,686 | 1 tx.fhir.org timeout (noise) | 2 |
| 3 | `fix/organization-home-use` | org-2/org-3 23 → 0 (21/21); plus 39 Organization profile-match cascades removed | none | 1 |
| 4 | `fix/vitals-effective-required` | 61 → 0 (19/19) | none | 2 |
| 5 | `fix/device-udi-bare-di` (+ ADR 0042 update) | 18 → 0 (18/18) | none | 1 |
| 6 | `fix/goal-description-from-text` (+ ADR 0014 update) | 42 → 0 (29/29) | +8 Patient profile-match (interaction with #2, see 10.6) | 3 |
| 7 | `fix/identifier-absent-in-us-core` | 32 → 0 (19/19) | none | 2 |

**All 7 together** (merged into a throwaway branch; they merge without conflicts): whole corpus
**7,535 → 4,376 errors (−42 %)**, and nothing new except one tx.fhir.org timeout. Also checked on every
branch, and on all 7 merged: `cargo fmt`, `cargo clippy --workspace --all-targets` (no warnings),
`cargo test --workspace` (all pass; no golden files changed). The two CI validator jobs, replicated
locally: **project fixtures 0 failures** on `main` and with all fixes; **HL7 C-CDA-Examples corpus (19
docs): same set of error kinds** on `main` and with all fixes. Locally `corpus_gate.py` exits 1 on both,
only because of tx.fhir.org timeouts, so CI should behave the same as it does today.

### 10.1 Encounter `coding: null`: confirmed exactly as analysed

The one-line fix from 4.1 works. The regression test fails on `main` with
`left: [{"coding": Null, "text": "New Patient"}]` and passes with the fix. Ready to be a PR as is.

### 10.2 Display table: the original proposal would have missed the biggest part

**Correction to 4.2.** Adding rows to `official_display()` does **not** fix race/ethnicity, which is
most of the Patient cascade. The race/ethnicity extension (`category_extension()` in
`crates/ccda-map/src/patient.rs`) never goes through `official_display`, and:

* The wrong `ASKU` display **"Asked but no answer"** is not from the documents. It is a **constant in
  the converter** (`RACE_OMB` / `ETHNICITY_OMB` in `patient.rs`, used when the source is
  `<raceCode nullFlavor="ASKU"/>` with no displayName). The validator wants `asked but unknown`; `UNK`
  should be `unknown`. So this is a real converter bug, not a table gap.
* For OMB categories, the document's display (`Caucasian White`, `Non Hispanic or Latino`,
  `NOT HISPANIC/LATINO`, ...) was copied into `valueCoding.display`. Fix: the coding gets the official
  display (from the OMB table), and the document's wording goes to the extension's `text` sub-extension,
  which exists for that purpose.
* Detailed codes (`2108-9` "White European") now use `official_display` for the CDC race system.

The other rows (v3 ParticipationFunction / MaritalStatus / LanguageAbilityMode / NullFlavor / ActReason /
RoleCode, CVX, ICD-10-CM, NUCC, NCI, HSLOC, CDC race) were taken from the validator's own "Valid
display is ..." messages over the whole corpus. Only codes the validator named are included; SNOMED,
RxNorm, LOINC, CPT and NDC are left out, as ADR 0027 requires.

* The caveat in 4.2 is now settled: the Patient profile-match cascade went away in **70 of 72** bundles.
  The 2 left (bundles 298, 299) are the validator's VSAC `vsacOpModifier` limit on detailed ethnicity
  `2178-2`, not a converter problem.
* Side effect: the PCP/ATTPHYS/ADMPHYS, NUCC and CVX rows also remove about 600 Wrong Display errors on
  other resource types.
* A possible maintainer objection: ICD-10-CM and CVX rows are per-code and the list never ends. If they
  prefer, the PR could be cut down to the v3 + CDC race rows and the `patient.rs` change, which is where
  nearly all of the value is.

### 10.3 Organization `home`: source pinned, fix works

The source element is `author/assignedAuthor/representedOrganization/addr use="H"` (and `telecom`) in
`Allscripts TouchWorks/Allscripts TW Alice rn.xml`. The vendor really does mark a hospital's address as
home. `register_organization()` (`crates/ccda-map/src/author.rs:50`) is the only Organization builder,
and it calls the shared `patient::address` / `patient::telecom`, so the caveat in 4.3 is resolved. The fix
removes `use: "home"` only in `register_organization` (not in the shared helpers, so Patient and
RelatedPerson are unchanged) and reports `organization.home-use.dropped` (warning). Bonus: 39 "Unable to
find a profile match ... us-core-organization" errors went away as well.

### 10.4 Vitals `effective[x]`: neither proposed option works by itself (key learning)

Tried four variants on the 19 affected bundles:

| Variant | Result |
| --- | --- |
| A. Do not claim a US Core vitals profile without a time (the ADR 0014 approach) | 61 → 43, **0 files cleared**. The validator applies the **base R4 vital-signs profiles (`bodyweight`, `bp`, `vitalspanel`, ...) by LOINC code, whatever `meta.profile` says**, so they still need `effective[x]`. |
| B. `_effectiveDateTime` with a data-absent-reason extension | 61 → 61: every min-cardinality error becomes a **`vs-1`** error ("if effective is dateTime and has a value then ... precise to the day"). Validator 7.0.3 treats an extension-only `dateTime` as breaking vs-1 (arguably a validator bug). |
| C. `effectivePeriod: {"extension": [DAR]}` | 61 → 18. The 18 left are vs-1 again, only on Observations that also claim a specific US Core vitals profile (weight, height, BP, pulse-ox). |
| **D. C + A** (DAR Period **and** no US Core vitals claim without a real time) | **61 → 0, 19/19 files, no new errors.** This is what is on the branch. |

Implementation: one post-pass at the end of `map_vitals()` (report code `vitals.effective.missing`),
plus a `timed` check at the top of `vital_signs_profile()` in `profile.rs` (it used to be checked only
for the generic `us-core-vital-signs` profile, not the specific ones). For the issue/PR text: say
plainly that the `_effectiveDateTime` form looks like the obvious choice but trips vs-1 in the
validator.

### 10.5 Device UDI: works; both forms are the same ONC test device

All 18 Devices have the same GTIN `00643169007222`: 10 have it bare, 8 as
`01006431690072221716012821BLC200461H` (= `(01)00643169007222(17)160128(21)BLC200461H`). The fix accepts
both, and the unbracketed element string is also parsed for dates, lot and serial (fixed-length AIs
11/17; variable 10/21 run to a GS `\x1d` or the end of the string). The HIBCC test (`+H123...` → no DI)
still passes. Open point for the maintainer: these strings are kept in `carrierHRF` as before, though
strictly they are not HRF. A bare 14-digit DI under the FDA UDI root is not checked against the GS1
check digit; that could be added.

### 10.6 Goal description: needs three layers, and a parser bug turned up

The 42 Goals with no description fall into three groups:

1. **Entry `text/reference` resolves** (the case traced in 4.6): `Statement.text` is already resolved
   from the narrative; `description()` just never used it. A fallback to it fixes **19 of 42**.
2. **`<code nullFlavor="UNK"><originalText>Get rid of ...<reference value="#goal1"/></originalText>`
   where `#goal1` does not exist in the narrative** (Freedom Medical). **This is a separate, general
   parser bug** in `parse_code()` (`crates/ccda-model/src/types.rs:170`): when `originalText` has a
   `<reference>`, the inline text is thrown away, so a dangling reference loses the text altogether. It
   affects every coded element, not only goals. Fixed in its own commit (resolved reference first, then
   the inline text). The whole corpus and the HL7 corpus show no new errors from it.
3. **No wording anywhere** (e.g. Henry Schein placeholder goals with `nullFlavor="NA"` on everything,
   Meditech `<code nullFlavor="UNK"/>`): `description` now carries a data-absent-reason (`Goal.description`
   is 1..1 in base R4), and the `us-core-goal` gate in `profile.rs` now needs `description.text` or
   `.coding`, so ADR 0014's "does not claim us-core-goal" still holds. ADR 0014 updated.

End result: 21 Goals got real text, 21 got a data-absent-reason. **Interaction:** once these Goals claim
`us-core-goal`, the validator checks `Goal.subject` against `us-core-patient`. In 8 bundles the Patient
already fails (fix #2's problem), so 8 new profile-match errors show up on this branch alone. With #2
merged they disappear (confirmed in the all-7 run). Say this in the PR, or merge #2 first.

Note the 4.6 claim "the converter already resolves `originalText` references, but not the entry's own
`text/reference`" was half wrong: the entry's text *was* resolved, it just was not used.

### 10.7 Identifiers: DAR-only identifiers are everywhere; only fix them where US Core needs it

**Correction to 4.7.** Identifiers that are only a data-absent-reason (`{"extension":[DAR]}`) are
everywhere: about 950 in the corpus, on 16 resource types (Observation 284, Condition 154,
MedicationRequest 123, Procedure 119, ...). This is the IG's "id with nullFlavor" rule and base FHIR
accepts it. They only fail where **US Core makes `identifier.system` and `.value` 1..1**, which in this
corpus means **Encounter (17 files) and Practitioner (2 files)**. So "omit them everywhere" would be a
broad, debatable change. The fix is limited to those two:

* a DAR-only identifier next to a real one is dropped (it adds nothing);
* if all identifiers are DAR-only, they become one `{"_system": DAR, "_value": DAR}`. This is **the shape
  the converter already uses** for a Practitioner with no `<id>` at all (`author.rs`,
  `practitioner.identifier.absent`), so it is consistent with existing code;
* 10 of the 32 errors were a different case found only by trying: ids whose root is not an OID or UUID
  (`root="MMD"`, `root="ENC05FDC..."`), which map to `{"value": "73"}` with no system (already reported as
  `identifier.root.invalid`). For these two resource types, `_system` now gets a data-absent-reason.

Helper `with_system_and_value()` in `crates/ccda-map/src/identifier.rs`, used in `encounter.rs` and in
`register_practitioner`. Possible follow-up: other US Core profiles with the same rule (e.g.
us-core-organization identifier slices) did not show up in this corpus.

### 10.8 Other corrections to session 1's notes

* The 2 bundles with "empty response from the validator" (041, 165) were transient: on retry both
  validated in milliseconds with ordinary errors (per-1 in the source, a LOINC display).
* Corpus note: CI's HL7 corpus job gets 19 documents out of 446 files (the rest are fragments). That
  matches ADR 0066.
* Section 4.2 LOINC question: `46264-8`, `51848-0`, `78375-3` are **not** in `loinc_display()` (grep
  only hits test fixtures and goldens). The validator wants e.g. 'Evaluation Note' for 51848-0 (bundle
  165). Not changed; it fits the "LOINC via `--loinc-table`" policy, but the maintainer may want the
  common section codes built in.

### 10.9 Recommendation: how to split into PRs (for least maintainer effort)

Keep them separate, since they touch different files and merge cleanly in any order (checked), but send
them in this order and frame them as follows:

1. **Ready to send as PRs now (clear bugs, small, no policy question):**
   * #1 `coding: null`: 8 lines; the easiest possible review.
   * #2 race/ethnicity + display rows: the biggest win (−38 % of all errors). Send this one early, since
     #6's numbers look better on top of it.
   * #3 Organization `home`: 54 lines, mostly the test.
   * #5 UDI forms: with the ADR 0042 update.
   * The `originalText` parser fix (first commit on the #6 branch) could also be **its own tiny PR**: it is
     a general bug and easy to review alone. Splitting it out is a `git cherry-pick 9c44f9b` onto a new
     branch off `main`.
2. **Send with a short design note, or as an issue first ("here is a verified fix, want a PR?"),
   because each picks a data-absent-reason/profile policy the maintainer may want to own (they write ADRs
   for this):**
   * #4 vitals: the "why not the obvious `_effectiveDateTime`" explanation (10.4) is the valuable part.
   * #6 goal text/DAR (ADR 0014 updated on the branch).
   * #7 identifiers (no ADR; the PR should point to the existing `practitioner.identifier.absent`
     precedent).

If the maintainer would rather have fewer PRs: #4, #6 and #7 bundle naturally as "fill required elements
with data-absent-reason instead of emitting invalid resources", and #1, #3 and #5 as "small mapping
fixes from the ccda-samples corpus". Bundling #2 with anything is not recommended (it is the largest
diff, and table rows invite line-by-line review).

Before opening any PR: rebase on the then-current `main`, rerun `cargo test --workspace`, and use the PR
body to cite the corpus numbers above. CONTRIBUTING asks mapping changes to cite their source (IG row,
template id, US Core element); each commit message names the FHIR constraint (org-2/3, vs-1, US Core
cardinality).

### 10.10 How the verification was done (reusable)

Session 2 scratchpad: `/private/tmp/claude-502/-Users-mradwin-dev-ccda2fhir/f97ea886-de76-4e75-ab51-0ea85f53d90b/scratchpad/`
(may be cleaned). It reused session 1's scratchpad (`.../cd9c69b6-.../scratchpad/`) for the baseline:
`all/files.txt` (bundle number = line number), `outcomes/NNN.json` (baseline validator output), and
`validator/validator_cli.jar`.

* Validator: `java -jar validator_cli.jar -server 8090 -version 4.0.1 -ig hl7.fhir.us.core#6.1.0`, left
  running in the background.
* `verify.sh <label> <NNN ...|all>`: `cargo build --release`, copies the binary to `<label>/` (so you can
  switch branches as soon as the build is done), converts the listed bundles, POSTs them 4 at a time,
  and writes `<label>/NNN.json` and `NNN.outcome.json`.
* `compare.py <label> [regex]`: normalises error texts (uuids, `[n]` indexes, chained "(error message
  = ...)") and prints the error total before → after, the count matching the regex, files fully
  cleared, the top errors removed, and **every error that is new (a possible regression)**.
* `pick.py <regex>`: bundle numbers whose baseline has an error matching the regex (used to choose
  the affected bundles for each fix).
* `suite.sh <binary> <dir> <out>`: the same thing for a folder of XML (used for the project fixtures
  and the HL7 C-CDA-Examples corpus, then `scripts/corpus_gate.py`).
* zsh gotcha: `$L` holding a space-separated list is not word-split; use `${=L}`.

## 11. Related corpora (checked 2026-11-05)

* **jddamore/ccda-samples has 401 documents, not 382.** 19 files use an upper-case `.XML` extension and
  the case-sensitive `find -name '*.xml'` in section 8 skipped them. Use `-iname`. Those 19 have **not been
  converted or validated** yet.
* **onc-healthit/2015-ccda-certification-samples** (last commit 2019-04-22) is the upstream
  (jddamore's README cites its old name `siteadmin/2015-C-CDA-Certification-Samples` and says it took what
  was there in January 2018, "copied, unzipped, renamed, formatted and organized", with duplicates of the
  same vendor + patient + document type removed). It is organised by testing lab (DrummondGroup, ICSA
  Labs, InfoGard_UL, SLICompliance), and most documents sit inside zips, some of them nested 2-3 deep.
  Unpacked: 1,213 XML files (10 unparseable), **651 distinct documents** by `ClinicalDocument/id` +
  `effectiveTime`.
  * **All 401 jddamore documents are in it** (397 by id and time; the other 4 are HealthGrid files that
    ONC stores as `.txt`). 395 match even after whitespace normalisation, so jddamore's "formatting"
    barely changed content.
  * **About 333 distinct ONC documents are not in jddamore**, nearly all from DrummondGroup (filed under
    "ETT Version x.y.z" folders), dated 2016-2017. They look like mostly the same vendors: the
    duplicates jddamore removed and resubmissions across validator versions. Only a few look like new
    vendors (rough name heuristic). As a test set they add volume and repeated documents more than new
    vendor variety.
* **HL7/C-CDA-Examples** is already in the project's CI (the 19 full documents; ADR 0066). The converter is
  clean on it apart from known errors.

## 12. The 19 `.XML` files and the 333 ONC-only documents (2026-11-05)

Both sets were converted with `main` and with all 7 fixes merged, and validated with the same settings.
All conversions exited 0.

| Category | 19 `.XML` (main → fixes) | 333 ONC-only (main → fixes) |
| --- | --- | --- |
| Total errors | 305 → 186 | 5,756 → 3,699 (−36 %) |
| 1 `coding: null` | 2 → 0 | 68 → 0 (30 files) |
| 2 Patient cascade (us-core-patient profile match) | 82 → 0 | 1,151 → 31 (the 31 are the VSAC detailed-ethnicity validator limit) |
| 2 displays in the table's systems | 37 → 0 | 649 → 27 |
| 3 org home | 0 | 16 → 0 (13 files) |
| 4 vitals effective | 10 → 10 (vs-1, see 12.1) | 152 → 30 (vs-1, see 12.1) |
| 5 UDI | 0 | 8 → 0 |
| 6 goal description | 4 → 0 | 28 → 0 (20 files) |
| 7 identifier system/value | 0 | 17 → 0 (10 files) |
| New errors from the fixes | none | none |

So the 7 fixes carry over to new documents; nothing was tuned to the 382.

### 12.1 Bug 4 has a sibling: start-only vital-sign times trip vs-1 (on `main` too)

A vital sign with `<effectiveTime><low value="20150622"/></effectiveTime>` maps to
`effectivePeriod: {"start": "2015-06-22"}`. Under a specific US Core vitals profile the validator reports
vs-1 at `.effective.ofType(Period)`, even though vs-1 only constrains a dateTime (the same validator quirk
as in 10.4). Counts: 15 errors in 4 of the 382 (bundles 142, 203, 204, 206); 10 in 2 of the 19
(MDIntellisys); 30 in 6 of the ONC-only. Session 1 missed it because its pattern for bug 4 matched only
the min-cardinality message. **Experiment (not committed):** in the `map_vitals` post-pass, turn a
start-only `effectivePeriod` into `effectiveDateTime`. On the 6 affected files vs-1 went 25 → 0 with no
other new errors. That is a candidate extra commit for `fix/vitals-effective-required`.

### 12.2 Candidate bug 8: Condition.encounter points at an Encounter that claims no profile

The header (encompassing) Encounter often has no code, so it has no `type`, and the converter
(correctly) does not claim `us-core-encounter`. US Core Condition requires `encounter` to reference a
`us-core-encounter`, so the **Condition** fails ("Unable to find a profile match ... us-core-encounter").
The Encounter itself shows no error. Counts: 30 errors in 27 of the 382; 8 in the 19; 36 in 25 of the
ONC-only. Not tried yet. Fix ideas: give the Encounter `type: [{"extension": [DAR]}]` so it can claim
the profile (US Core `type` is 1..*), or leave out `Condition.encounter` when the target cannot claim
`us-core-encounter` (loses data, so it should be reported).

### 12.3 Other leftovers, smaller

* More display rows for the table (validator-named): `v3-ObservationInterpretation#N` → "Normal"
  (14 errors), CVX 118 → "HPV, bivalent", ICD-10-CM Z68.27 → "Body mass index [BMI] 27.0-27.9, adult".
  They could go on PR #80.
* ICD-10-CM codes written without the dot (`T8611`, `E039`, `E663`; NextGen MediTouch, 2 files in the
  19, none elsewhere). This is a source problem; the converter could add the dot after the third
  character. Low value.
* Still unmapped, in all sets: BMI with no unit (5.1), non-canonical UCUM units, invalid NPIs
  (us-core-16/17), observation-lab profile-match cascades (54 in ONC-only; not traced yet), and
  Practitioner referenced where `us-core-relatedperson` is expected (8 in ONC-only; not traced).

### 12.4 Why session 1 missed 12.1 and 12.2 (process lesson)

1. The "Unable to find a profile match" errors were treated as one bucket, and the Patient cascade
   (about 1,950 of 2,111) explained most of it. The bucket was never split by target profile.
2. These errors sit on the *referring* resource. The broken target often shows no error at all, because
   it correctly claims no profile. A scan for errors located on the resource itself cannot see them.
3. The before/after check (`compare.py`) prints only removed and new errors. An error that is the
   same before and after never appears.

From now on: split profile-match errors by target profile and look up the target resource
(`analyze.py` in the session 2 scratchpad does this); after each fix, read the top *remaining* errors;
write category patterns against the constraint (any `effective[x]` / `vs-1` error), not one message.

Sources: the 333 ONC-only documents were staged as `onconly/oNNN.xml` with `onconly/files.tsv` (name →
original path inside the ONC repo or its zips) in the session 2 scratchpad.

### 12.5 Done after section 12 (2026-11-05)

* `fix/vitals-effective-required`: commit `bf6de97` adds the 12.1 fix (start-only vital time →
  `effectiveDateTime`, reported as `vitals.effective.start-only`). On the 12 files with the pattern
  (4 of the 382, 2 of the 19, 6 ONC-only) vitals errors went 55 → 0; the original 19 bug-4 bundles still
  go 61 → 0; fixtures have 0 failures; no new errors. Pushed to the fork, still no PR.
* PR #80 (`fix/display-table`): commit `5da9ca6` adds the 3 rows from 12.3 (ObservationInterpretation N,
  CVX 118, ICD-10-CM Z68.27). On the 8 ONC-only files that use them, their 17 errors went to 0; no new
  errors. Pushed, and the PR description was updated.
