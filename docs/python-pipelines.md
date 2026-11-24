# Loculus: what the Python components process

> Snapshot from 2026-11-23, based on the READMEs and code in the repository.
> Local file, ignored through `.git/info/exclude`. Where each pipeline stores its data: see [`data-map.md`](data-map.md).

Python is ~25% of the code and appears in **9 components**. Almost all of them are **workers around the Kotlin backend**: they pull data, process it and push results back through the REST API. None of them touches the backend's tables directly, except `ena-submission`, which owns its own schema.

| Component | Lines | Kind | When it runs | What it processes |
|---|---|---|---|---|
| `preprocessing/nextclade` | 10,784 | worker (Deployment per organism × pipeline version) | continuously polls the backend | Nextclade alignment, QC and metadata normalization of submissions |
| `ena-submission` | 10,067 | Deployment + CronJob (01:00 and 13:00) | continuous | deposits released sequences to ENA/INSDC (project, sample, reads, assembly) |
| `ingest` | 4,390 | CronJob (Snakemake) | scheduled per organism | imports sequences from NCBI Virus/GenBank into Loculus |
| `cli` | 4,706 | user CLI | on demand | login, submit and download from the command line (in development) |
| `raw-reads-processing` | 2,039 | FastAPI service | called by preprocessing | validates read files (FASTQ/BAM) and rejects those containing human DNA |
| `loculus-silo` | 1,912 | worker | every 30 s | downloads released data, transforms it and builds the SILO search index |
| `taxonomy` | 991 | FastAPI service + build job | continuous | NCBI taxonomy lookups (host validation, lineages) |
| `preprocessing/dummy` | 310 | test worker | dev and CI | fake preprocessing that accepts or rejects without real analysis |
| `kubernetes/config-processor` | ~200 | init container | at every pod start | injects secrets and fetches external references into config files |

---

## 1. Overview: the lifecycle of a sequence

```mermaid
flowchart LR
    subgraph Sources
        U[Submitter<br/>website / cli]
        NCBI[(NCBI Virus<br/>GenBank)]
    end

    subgraph Loculus
        BE[[Kotlin backend<br/>+ Postgres]]
        PP[preprocessing<br/>nextclade 🐍]
        RR[raw-reads-processing 🐍]
        TX[taxonomy-service 🐍]
        SI[loculus-silo<br/>importer 🐍]
        SILO[(SILO + LAPIS<br/>search index)]
        WEB[Website]
    end

    subgraph INSDC
        ENA[(ENA / EBI)]
    end

    ING[ingest 🐍<br/>Snakemake] -->|submit / revise / revoke| BE
    NCBI --> ING
    U -->|submit| BE
    BE -->|unprocessed data| PP
    PP -->|validate reads| RR
    PP -->|validate host| TX
    PP -->|processed data + errors/warnings| BE
    BE -->|approved → released| SI
    SI --> SILO
    SILO --> WEB
    BE --> WEB
    BE -->|released| ENAS[ena-submission 🐍]
    ENAS -->|project, sample, reads, assembly| ENA
    ENAS -->|INSDC accessions<br/>external metadata| BE
    ENAS -.->|already-deposited accessions<br/>so they are not re-ingested| ING
```

---

## 2. Preprocessing (Nextclade): from raw submission to validated data

The central component: every submitted sequence goes through it before human review. There is one instance per organism and pipeline version (`_preprocessingFromValues.tpl`).

```mermaid
sequenceDiagram
    autonumber
    participant BE as Backend
    participant PP as prepro (Python)
    participant NC as Nextclade CLI
    participant RR as raw-reads-processing
    participant TX as taxonomy-service

    loop every few seconds
        PP->>BE: POST /extract-unprocessed-data (batch of N, leased)
        BE-->>PP: NDJSON with metadata + raw sequences
    end
    PP->>PP: write FASTA to a temporary directory
    PP->>NC: nextclade run (alignment, QC, clade, mutations)<br/>+ nextclade sort (segment / subtype detection)
    NC-->>PP: TSV/JSON with alignment, insertions and QC
    opt raw reads attached
        PP->>RR: POST /process-files (presigned S3 URLs)
        RR-->>PP: format errors or human-DNA errors
    end
    opt host field present
        PP->>TX: GET /taxa?scientific_name=...
        TX-->>PP: tax_id and common name
    end
    PP->>PP: processing_functions (dates, authors, regex,<br/>options, lineage, display name, ENA accession)
    PP->>BE: POST /submit-processed-data<br/>(normalized data + errors[] + warnings[])
    Note over BE: errors block the entry<br/>warnings-only entries go to submitter review
```

**Processing functions** (`processing_functions.py`, 1,652 lines): `check_date`, `parse_date_into_range`, `parse_and_assert_past_date`, `parse_timestamp`, `concatenate`, `check_authors`, `extract_regex`, `check_regex`, `process_options`, `is_above_threshold`, `is_variant`, `assign_custom_lineage`, `build_display_name`, `resolve_host_taxon_id`, `scientific_name_from_id`, `common_name_from_id`, `check_ena_accession`. In `values.yaml`, each metadata field maps to one of these functions plus its arguments.

---

## 3. Ingest: import from NCBI without duplicates

Runs as a Kubernetes CronJob driving a Snakemake DAG. The trickiest part is **segmented viruses**: NCBI stores each segment separately, while a Loculus entry groups all segments of a sample.

```mermaid
flowchart TD
    A[NCBI datasets CLI<br/>download by taxon ID] --> B[format_ncbi_metadata<br/>rename and transform fields<br/>authors, country → country/state/division]
    B --> C{segmented<br/>organism?}
    C -- no --> E
    C -- yes --> D1[nextclade sort / alignment<br/>assign segment]
    D1 --> D2[merge_group_overrides<br/>+ heuristic_group_segments<br/>group by isolate]
    D2 --> E[calculate_sequence_hashes<br/>md5 of metadata + sequence]
    E --> F[get_loculus_depositions<br/>+ filter_out_depositions<br/>drop what Loculus itself deposited to ENA]
    F --> G[compare_hashes<br/>vs. hashes already in the backend]
    G --> H{triage}
    H -->|submit| I[POST /submit]
    H -->|revise| J[POST /revise]
    H -->|no_change| K[skip]
    H -->|blocked| L[skip and log]
    H -->|revoke| M[regroup_and_revoke<br/>manual trigger only]
    I & J --> N[autoapprove<br/>approve ingested entries]
```

Known bottleneck: `prepare_files.py` uses list membership for `to_submit`/`to_revise`/`to_revoke`, which is O(n²) (#7443, task 01).

---

## 4. ENA submission: deposit to INSDC

Once a sequence is released in Loculus it can be deposited to ENA (EBI), which mirrors to GenBank and DDBJ. The state of each (accession, version) lives in its own tables (`ena_deposition_schema`).

```mermaid
flowchart LR
    CJ[CronJob 01:00/13:00<br/>get-ena-submission-list] -->|/get-released-data| BE[(Backend)]
    CJ -->|list + Slack notification| H[human review]
    H -->|PR to repo<br/>pathoplexus/ena-submission| GH[(GitHub)]
    GH --> T[trigger_submission_to_ena<br/>insert into submission_table]
    T --> P[create_project<br/>one per group × organism]
    P --> S[create_sample<br/>BioSample]
    S --> R{raw reads?}
    R -- yes --> RW[create_raw_reads<br/>ERR/SRA]
    R -- no --> AS
    RW --> AS[create_assembly<br/>GCA via webin-cli]
    AS --> UP[upload_external_metadata_to_loculus<br/>INSDC accessions → backend]
    UP --> BE
    P & S & RW & AS -.->|failure| SL[notifications → Slack]
```

```mermaid
stateDiagram-v2
    direction LR
    [*] --> READY_TO_SUBMIT
    READY_TO_SUBMIT --> SUBMITTED_PROJECT
    SUBMITTED_PROJECT --> SUBMITTED_SAMPLE
    SUBMITTED_SAMPLE --> SUBMITTED_RAW_READS: with reads
    SUBMITTED_SAMPLE --> SUBMITTING_ASSEMBLY: no reads
    SUBMITTED_RAW_READS --> SUBMITTING_ASSEMBLY
    SUBMITTING_ASSEMBLY --> SUBMITTED_ALL: GCA received
    SUBMITTED_ALL --> SENT_TO_LOCULUS
    SUBMITTED_ALL --> HAS_ERRORS_EXT_METADATA_UPLOAD
    SENT_TO_LOCULUS --> [*]
```

---

## 5. Raw reads: validation and human-data protection

A small FastAPI service with one endpoint (`POST /process-files`). It protects **privacy**: reads from clinical samples can contain the patient's DNA.

```mermaid
flowchart TD
    PP[preprocessing] -->|files: presigned URLs<br/>+ accessionVersion| API[POST /process-files]
    API --> DL[download files from S3/MinIO]
    DL --> V[file_format_validation<br/>readtools.jar: valid FASTQ/BAM?]
    V --> DC[deacon server<br/>k-mer index of the human genome]
    DC --> Q{human read proportion<br/>above limit, or human bp above max?}
    Q -- yes --> E1[error: remove host reads]
    Q -- no --> OK[ok]
    V -- invalid --> E2[format error]
    E1 & E2 & OK --> PP
```

---

## 6. SILO importer: make the data searchable

```mermaid
flowchart LR
    L[loop every 30 s] --> G[GET /get-released-data<br/>with ETag]
    G --> C{304 not modified<br/>and hard refresh<br/>not due?}
    C -- yes --> L
    C -- no --> V[validate NDJSON<br/>count, pipelineVersion]
    V --> LD[fetch lineage definitions<br/>per pipeline version]
    LD --> TR[transform to<br/>SILO input format]
    TR --> SP[run SILO preprocessing<br/>C++ binary]
    SP --> IDX[(new index<br/>→ LAPIS serves queries)]
```

Full rebuild on every change; incremental import is in progress (#7733, PRs #7746 and #7747).

---

## 7. Taxonomy: NCBI taxonomy lookups

```mermaid
flowchart LR
    FTP[(NCBI FTP<br/>taxdump)] --> DLJ[ncbi_tax_download<br/>builds SQLite: tax_id,<br/>name, parent, depth]
    DLJ --> S3[(S3 loculus-public<br/>ncbi_taxonomy_DATE.sqlite)]
    S3 -->|init container| SVC[taxonomy_service FastAPI]
    SVC -->|GET /taxa?scientific_name| PP[preprocessing<br/>host validation]
    SVC -->|GET /taxa/:id| PP
    SVC -->|POST /silo-lineage| SILO[lineage tree<br/>for SILO]
```

---

## Takeaways

- **The Kotlin backend owns the state.** The Python components are stateless workers that talk to it through the API, so each one scales and can be replaced independently. `ena-submission` is the exception, with its own state-machine schema.
- **External binaries do the heavy lifting:** Nextclade (Rust), SILO (C++), deacon (Rust), readtools (Java) and NCBI `datasets`. Python orchestrates, transforms metadata and handles errors.
- **The risks from the other analyses:**
  - bus factor 1 on preprocessing, ENA and ingest;
  - Nextclade pinned at 3.24.0 (reverted on purpose in #7780);
  - Nextclade deliberately pinned in Dependabot's conda config (`ignore`, #7784);
  - `processing_functions.py` as a hotspot;
  - the `uv` CVE in `ena-submission`.
- **Four dependency managers coexist:** conda/micromamba (prepro, ingest, ENA, raw-reads, taxonomy), uv (cli), pip (dummy, config-processor) and pyproject. Standardizing them is an easy proposal.

## Dive deeper

- [Analyses: the generated charts, with takeaways](analyses.md)
- [Bottlenecks and optimization opportunities](bottlenecks.md)
- [Tasks: 18 concrete contributions](../tasks/README.md)
- [Data map: where data lives](data-map.md)
- [CVEs and advisories](cves.md)
- [Security review](security.md)
