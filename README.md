# Adjusted BLUiR

Adjusted BLUiR is a research-oriented adaptation of **BLUiR (Bug Localization Using information Retrieval)** for file-level bug localization in Java projects. Given a collection of bug reports and a Java source-code snapshot, it represents each source file as a structured document and ranks files by their textual relevance to each report.

The project retains BLUiR's central idea—using source-code structure rather than treating code as undifferentiated text—while adapting the surrounding pipeline to a newer XML dataset format and a locally managed Lucene retrieval backend.

> This is an independent adaptation of a historical Java BLUiR codebase. It is not an official release from the authors of the BLUiR paper, nor is it intended to be a bit-for-bit reproduction of their experiments.

## Research background

BLUiR was introduced by Saha, Lease, Khurshid, and Perry in *Improving Bug Localization Using Structured Information Retrieval* (ASE 2013). Its key observation is that program constructs such as class names, method names, identifiers, and comments should be modeled separately during retrieval. A bug report acts as a query, and Java source files form the searchable document collection.

The original approach uses Eclipse JDT to recover code structure and Indri as its indexing and retrieval system. It also studies identifier splitting, stemming, stopword removal, and BM25-style term weighting. This repository preserves that architectural direction while updating the implementation boundary around the dataset and retrieval engine.

The XML format targeted here is compatible with the `Eclipse_Platform_UI.xml` dataset family. The provided IEEE reference describes the associated fine-grained bug-localization benchmark and ranking setting:

- Xin Ye, Razvan Bunescu, and Chang Liu, [*Mapping Bug Reports to Relevant Files: A Ranking Model, a Fine-Grained Benchmark, and Feature Evaluation*](https://ieeexplore.ieee.org/document/7270328), IEEE Transactions on Software Engineering, 2016.
- The particular Eclipse Platform UI XML artifact is also described by [tera-PROMISE](https://openscience.us/repo/defect/bug-reports/eclipseui.html).

The BLUiR publication documents the method and points to experimental artifacts, while building its retrieval layer on the open-source Indri toolkit. Since the provenance of this exact Java baseline cannot be established from the publication alone, this repository is presented as an adaptation rather than as the authors' official implementation.

## What was adapted

The adjustment work focuses on the parts of older research software that commonly limit reuse: data compatibility, external infrastructure, and experiment output.

| Area | BLUiR paper/baseline | This repository |
| --- | --- | --- |
| Bug-report input | Original BLUiR benchmark representation | Table/column XML representation used by the newer Eclipse dataset |
| Code representation | Structured fields extracted from the Java AST | Class, method, identifier, and comment fields extracted with Eclipse JDT |
| Retrieval backend | Indri | Apache Lucene |
| Text analysis | Stopword removal and Krovetz stemming | Lucene tokenizer, lowercase normalization, configured stopwords, and Krovetz stemming |
| Ranking | Indri's BM25-based TF-IDF formulation | Lucene BM25 with `k1 = 1.0` and `b = 0.3` |
| Query handling | Structured combinations of report and code fields | Summary and description searched across all four code fields |
| Results | Experiment-oriented ranked output | One TREC-formatted result file per bug report |

The implementation follows the logic of structured IR-based localization, but it should be understood as an engineering adaptation rather than an exact experimental replication. In particular, the active Lucene pipeline does not implement BLUiR's optional similar-bug feedback component, and the legacy evaluation stage is currently disconnected.

## Pipeline

For each run, the program performs four stages:

1. **Query extraction** — reads each bug's ID, summary, and description from the XML repository.
2. **Fact extraction** — recursively finds Java files, builds an AST with Eclipse JDT, and extracts structural text fields.
3. **Index construction** — tokenizes and indexes the generated TREC-style documents with Lucene.
4. **Retrieval** — searches the structured fields with BM25 and writes a ranked list for every bug report.

The indexed fields are configured in [`fields`](fields):

- `class`
- `method`
- `identifier`
- `comments`

Camel-case identifiers are split into constituent terms during fact extraction. The analyzer then lowercases terms, removes words listed in [`stopwords`](stopwords), and applies Krovetz stemming.

## Repository structure

```text
.
├── fields                         # Indexed source-code fields
├── stopwords                      # Stopword configuration
├── pom.xml                        # Maven configuration
└── src
    ├── bluir
    │   ├── core                   # Entry point and pipeline orchestration
    │   ├── entity                 # Bug-report model
    │   ├── extraction             # Query and source-code fact extraction
    │   ├── parser                 # Dataset XML parser
    │   ├── evaluation             # Legacy evaluation implementation
    │   └── utility                # File and text-processing utilities
    └── Custom_IR
        ├── IndexPhase             # Lucene document analysis and indexing
        └── QueryPhase             # Query loading, ranking, and result output
```

## Requirements

- Java Development Kit 11 or later (required by Lucene 9.x)
- Apache Maven 3.x
- A bug-report XML file in the supported schema
- A local checkout of the corresponding Java source code
- At least 2 GB of free space on the filesystem from which the program is launched

Run the program from the repository root. The current implementation resolves `fields`, `stopwords`, and `Settings.txt` relative to the working directory.

## Dataset format

The parser expects records organized as `<table>` elements beneath a `<database>` element. Each record must contain columns named `bug_id`, `summary`, `description`, and `files`.

```xml
<database>
    <table>
        <column name="bug_id">12345</column>
        <column name="summary">Short description of the failure</column>
        <column name="description">Detailed bug report text</column>
        <column name="files">src/example/Foo.java,src/example/Bar.java</column>
    </table>
</database>
```

`files` is the comma-separated ground-truth set of changed/fixed files. The active retrieval pipeline uses the bug ID, summary, and description; the file set is retained by the parsed bug-report model for evaluation-oriented workflows.

For a controlled experiment, the source directory should represent the project version appropriate for the selected reports—ideally the version immediately before each fix. This repository indexes one supplied source snapshot per run; it does not automate per-bug Git checkouts.

## Configuration

### 1. Set the run arguments

The current entry point deliberately defines its run configuration inside [`src/bluir/core/BLUiR.java`](src/bluir/core/BLUiR.java#L11-L19). Update the four values before running:

```java
args[0] = "-b";
args[1] = "/absolute/path/to/Eclipse_Platform_UI.xml";
args[2] = "-s";
args[3] = "/absolute/path/to/the/java/source/tree";
args[4] = "-w";
args[5] = "/absolute/path/to/experiment/output";
args[6] = "-n";
args[7] = "eclipse_ui_run";
```

| Argument | Meaning |
| --- | --- |
| `-b` | Path to the bug-report XML file |
| `-s` | Root directory recursively searched for Java source files |
| `-w` | Parent directory in which run artifacts are created |
| `-n` | Run name used to create `BLUiR_<name>` |
| `-a` | Legacy alpha parameter; accepted by the parser but not used by the active pipeline |

Although the entry point contains an argument parser, the values supplied on the command line are currently replaced by the block above. Therefore, edit this block rather than passing different values at launch.

### 2. Add the compatibility settings file

Create a local `Settings.txt` in the repository root:

```text
indripath=.
```

This setting is a compatibility requirement left by the original launcher. The active indexing and retrieval path uses Lucene and does not consume the Indri path. `Settings.txt` is intentionally ignored by Git so machine-specific configuration is not committed.

## Build and run

From the repository root:

```bash
mvn -Dproject.build.sourceEncoding=ISO-8859-1 clean compile
mvn -Dproject.build.sourceEncoding=ISO-8859-1 exec:java \
  -Dexec.mainClass="bluir.core.BLUiR"
```

The explicit source encoding accommodates comments inherited from the historical baseline without rewriting the Java sources.

Alternatively, import the repository as a Maven project in any Java IDE and run `bluir.core.BLUiR.main()` after updating the configuration block.

The Maven package produced by the current `pom.xml` is not a self-contained executable JAR; Maven's `exec:java` goal or an IDE run configuration supplies the dependency classpath.

## Generated artifacts

With `-w /tmp/experiments` and `-n eclipse_ui_run`, the pipeline creates:

```text
/tmp/experiments/
└── BLUiR_eclipse_ui_run/
    ├── FileIndex.txt              # Source path to document-ID mapping
    ├── docs/                      # TREC-style structured source documents
    ├── index/                     # Lucene index
    ├── parsed_docs.log            # Corpus parsing log
    ├── query                      # Extracted bug-report queries
    └── result/
        ├── 12345.txt              # Ranked files for bug 12345
        └── ...
```

Each result line follows the conventional TREC layout:

```text
12345 Q0 org.example.Foo.java 1 7.428100 CustomIR
```

The columns are query/bug ID, the constant `Q0`, source-file document ID, one-based rank, retrieval score, and run tag.

## Current research scope

- The program performs file-level localization for Java source code.
- A single source snapshot is indexed in each run.
- Retrieval results are produced for external analysis; the legacy `Evaluation` class is not invoked by the active pipeline.
- The result count is set to the number of indexed Java files, producing a complete ranking when Lucene returns all matches.
- Exact reproducibility of the published BLUiR values should not be expected because the retrieval engine, query construction, dataset, and experiment orchestration differ.

These boundaries make the repository most useful as a transparent implementation study and as a base for controlled experiments on structured IR, alternative analyzers, field weighting, or bug-localization datasets.

## References

If this repository supports academic work, cite the original BLUiR paper and the dataset paper as appropriate.

```bibtex
@inproceedings{Saha2013BLUiR,
  author    = {Ripon K. Saha and Matthew Lease and Sarfraz Khurshid and Dewayne E. Perry},
  title     = {Improving Bug Localization Using Structured Information Retrieval},
  booktitle = {2013 28th IEEE/ACM International Conference on Automated Software Engineering (ASE)},
  pages     = {345--355},
  year      = {2013},
  doi       = {10.1109/ASE.2013.6693093}
}

@article{Ye2016Mapping,
  author  = {Xin Ye and Razvan Bunescu and Chang Liu},
  title   = {Mapping Bug Reports to Relevant Files: A Ranking Model, a Fine-Grained Benchmark, and Feature Evaluation},
  journal = {IEEE Transactions on Software Engineering},
  volume  = {42},
  number  = {4},
  pages   = {379--402},
  year    = {2016},
  doi     = {10.1109/TSE.2015.2479232}
}
```

The tera-PROMISE page for `Eclipse_Platform_UI.xml` additionally requests citation of the dataset donor's ASE 2015 paper:

```bibtex
@inproceedings{Lam2015HyLoc,
  author    = {An Ngoc Lam and Anh Tuan Nguyen and Hoan Anh Nguyen and Tien N. Nguyen},
  title     = {Combining Deep Learning with Information Retrieval to Localize Buggy Files for Bug Reports},
  booktitle = {2015 30th IEEE/ACM International Conference on Automated Software Engineering (ASE)},
  pages     = {476--481},
  year      = {2015},
  doi       = {10.1109/ASE.2015.73}
}
```
