# VasQ

**Software release: VasQ v1.0.0**

**VasQ** is an AI chat system for exploring a human brain vascular single-nucleus RNA-seq atlas and generating biomedical hypotheses about blood-brain barrier biology, regional vascular specialization, and drug delivery.

![VasQ workflow](VasQ_workflow.png)

## 1. Use VasQ online

**[Open VasQ](https://vasq-production-65c6.up.railway.app/vasq/)**

The hosted web interface is **free to use and requires no login**. Open the link in your browser and enter a research question. You do not need to install software, download the dataset, create a Railway account, or supply an OpenAI API key. The hosted service uses the operator's API credentials and infrastructure.

Example questions:

- How do PLVAP, AQP1, and SLCO2A1 rank as markers for fenestrated capillary cells?
- Which vascular cell types and which brain regions are enriched for small-vessel-disease susceptibility genes?
- What drugs failed because they cannot cross human BBB due to no route into receptor-mediated transcytosis?

VasQ is intended for research exploration and hypothesis generation, not clinical decision-making.

## What VasQ provides

- Natural-language expression queries with filters for brain region, region layer, cell class, cell type, and sex.
- Matrix-derived mean log-normalized expression, percentage of expressing cells, and cell counts; groups with fewer than 10 cells are excluded.
- Interactive expression plots, a paginated results table, and CSV downloads. For multi-gene queries, eligible results remain in the table and up to three gene profiles are plotted.
- Scientific evidence retrieval with source links, including drug and small-molecule evidence when the query specifically asks about them.

The atlas covers 41 brain regions, approximately 443,000 nuclei, and approximately 15,000 highly variable genes. Expression results are calculated directly from the matrix. The complete matrix, individual-cell records, and full expression table are not sent to the language model; a compact set of aggregate results is provided for interpretation.

## Choose how to run VasQ

| Option | Railway account | OpenAI API key | Dataset download |
| --- | --- | --- | --- |
| Hosted website above | Not needed | Not needed | Not needed |
| Your own computer | Not needed | Your own | Required |
| Your own Railway deployment | Your own | Your own | Required on your service |

Free access applies to the hosted website. If you run your own copy, you are responsible for its API usage and any hosting costs.

## 2. Run VasQ locally on Linux

### System requirements

The local instructions below describe a Linux setup using Docker Engine and the Docker Compose plugin. The application runs inside a Linux-based Python container. A Railway account is not required for local installation. Actual tested environment versions are not yet recorded; see the testing notes below.

| Component | Requirement or configured version |
| --- | --- |
| Container runtime | Docker Engine with the Docker Compose plugin on Linux. The commands below use `docker compose`. |
| Python inside the container | Python 3.10, configured by `vasq/Dockerfile`. |
| Browser | A browser with JavaScript enabled for the web interface. |
| Network | Required for OpenAI API calls, web retrieval, initial image/package downloads, and browser assets loaded from external CDNs. |
| API access | Your own OpenAI API key and access to compatible models for your local instance. |
| Data | The three matching expression files described below. |
| Hardware | The application uses CPU-based matrix analysis and remotely hosted OpenAI models. No local GPU or other specialized hardware is required by the provided workflow. |
| Memory and disk | Sufficient RAM to load the selected matrix and metadata, and disk space for the data and Docker images. Minimum requirements have not been benchmarked for this release. |

Git is needed only if you clone the repository rather than download a ZIP. Python and the server dependencies are installed inside the container; a separate host Python installation is not needed for the Docker setup.

### Software dependencies

The following server dependencies are declared in [`vasq/requirements.txt`](vasq/requirements.txt) in the configuration update accompanying this README:

| Package | Declared version |
| --- | --- |
| Django | 4.2.15 |
| django-ninja | 1.3.0 |
| OpenAI Python SDK | 2.53.0 |
| HTTPX | 0.27.0 |
| pandas | 1.5.3 |
| NumPy | 1.26.4 |
| SciPy | 1.13.1 |
| Requests | 2.32.3 |
| tiktoken | >=0.12.0 |
| WhiteNoise | Not pinned |

The local Dockerfile installs the following system packages through `apt-get`: `gcc`, `gfortran`, `g++`, `libblas-dev`, `libhdf5-dev`, `liblapack-dev`, `libopenblas-dev`, and `python3-dev`. Their exact versions and the Python patch versions are not pinned.

The web interface loads Plotly.js (`plotly-latest.min.js`, not pinned), marked 4.0.0, html2pdf.js 0.10.1, JSZip 3.10.1, Font Awesome 5.15.4, and Satoshi fonts from external CDNs. See [`index.html`](vasq/vasq/templates/vasq/index.html) for the requested asset URLs. No Node.js build step is included in the provided setup.

### Tested environments and installation time

Python 3.10 describes the local Docker configuration, not a verified test environment. Exact tested Linux distribution/version, Docker/Compose versions, browser versions, and a frozen list of installed package versions have not yet been recorded in the supplied release materials.

A typical installation time on a standard desktop computer has not yet been measured. The initial build downloads a Python image and installs dependencies; total setup time also depends on network speed and dataset download size. Record the elapsed build/setup time together with the machine specification before reporting a benchmark.

### Download the code

```bash
git clone --branch railway-deploy https://github.com/Knight-Initiative-for-Brain-Resilience/VasQ-assistant.git
cd VasQ-assistant
```

If you downloaded a ZIP instead, extract it and open a terminal in the extracted project folder. Run the remaining commands from that folder.

### Add the expression data

Download the expression data from the [VasQ Figshare project](https://figshare.com/projects/Atlas_of_Regional_Molecular_Specialization_in_the_Human_Brain_Vasculature/273930) and place the following files in `vasq/data/`:

| File | Contents |
| --- | --- |
| `VasQ_adata_X_sparse.npz` | Sparse CSR log-normalized expression matrix; rows are nuclei and columns are genes. |
| `VasQ_cell_meta_table.csv` | Per-nucleus region, region layer, cell class, cell type, age at death, and sex. |
| `VasQ_gene_names.csv` | Gene symbols in the same order as the matrix columns. |

All three expression data files are distributed through Figshare and must be downloaded into `vasq/data/` before running VasQ locally. If the Figshare files are not accessible, contact the authors for the data or a private access link before continuing. Local expression analysis requires all three files.

Docker mounts `vasq/data/` at `/data` inside the application container. Downloaded data are excluded from Git by default.

### Configure your own OpenAI API key

Copy the configuration template:

```bash
cp .env-shared.example .env-shared
```

Open `.env-shared` in a text editor and replace the placeholder for `OPENAI_API_KEY` with **your own key**. The template also contains the model and reasoning settings:

```dotenv
OPENAI_API_KEY=replace_with_your_own_openai_api_key
OPENAI_MODEL=gpt-5.6-sol
OPENAI_HELPER_MODEL=gpt-5.6-luna
OPENAI_WEB_MODEL=gpt-5.6-sol
OPENAI_MAIN_REASONING_EFFORT=high
OPENAI_HELPER_REASONING_EFFORT=none
OPENAI_WEB_REASONING_EFFORT=high
```

These are the model settings documented for the hosted VasQ workflow; model access depends on your API account. If you select alternative models, verify support for the Chat Completions features, structured output, and Responses API web search used by VasQ, and adjust reasoning settings as needed.

Keep `.env-shared` private. It is excluded from Git and Docker image builds; Docker Compose loads it as runtime environment variables.

### Start the web interface

```bash
cd vasq
docker compose build
docker compose run --rm web python manage.py migrate --noinput
docker compose up
```

Open **[VasQ on your computer](http://127.0.0.1:8000/vasq/)** in your browser. This address refers to your own computer and works after the application starts.

Stop the application with `Ctrl+C`, then remove its containers with:

```bash
docker compose down
```

The local setup runs a single VasQ service. Git LFS and a separate knowledge-graph service are not required.

## Demo using the Figshare dataset

The [Figshare dataset](https://figshare.com/projects/Atlas_of_Regional_Molecular_Specialization_in_the_Human_Brain_Vasculature/273930) is the data source for this demo. Use the three matching files listed above; no separate reduced demo subset is distributed here. For review, ensure that the supplied download link or private access link allows the reviewers to retrieve the files.

### Run the demo

1. Follow the local installation instructions above and open `http://127.0.0.1:8000/vasq/`, or open the hosted VasQ website to use the dataset already deployed by the maintainers.
2. Submit each of the following example questions separately:

   **Example 1:** How do PLVAP, AQP1, and SLCO2A1 rank as markers for fenestrated capillary cells?

   **Example 2:** Which vascular cell types and which brain regions are enriched for small-vessel-disease susceptibility genes?

   **Example 3:** What drugs failed because they cannot cross human BBB due to no route into receptor-mediated transcytosis?

3. Inspect the response and, where returned, the expression plots and results table. Download the expression table as CSV for expression queries.
4. Download the single example-output PDF from the [same Figshare project](https://figshare.com/projects/Atlas_of_Regional_Molecular_Specialization_in_the_Human_Brain_Vasculature/273930) to view an actual VasQ response. It illustrates one example only; reference outputs are not provided for all three questions.

### Expected output and reference examples

| Example | Expected output |
| --- | --- |
| 1. PLVAP, AQP1, and SLCO2A1 as fenestrated capillary markers | A comparative interpretation of the three genes, supported by expression tables and plots for eligible groups. Tables report mean log-normalized expression, expressing cells (%), and cell counts, and can be downloaded as CSV. Groups with fewer than 10 cells are excluded. |
| 2. Small-vessel-disease susceptibility genes | A source-linked discussion of susceptibility genes and their expression patterns across relevant vascular cell types and brain regions, with expression tables and plots where applicable. Expression patterns should be distinguished from a formal statistical enrichment test. |
| 3. Drug failure, BBB penetration, and receptor-mediated transcytosis | A source-linked assessment of drug-delivery evidence, distinguishing documented BBB penetration limitations from other reasons for failure and identifying whether a proposed transcytosis mechanism is supported. An expression plot or CSV is not required for this evidence-focused question. |

**Reference output:** One PDF containing an actual VasQ example output is available to download and view in the [VasQ Figshare project](https://figshare.com/projects/Atlas_of_Regional_Molecular_Specialization_in_the_Human_Brain_Vasculature/273930), alongside the dataset. This PDF illustrates one example response and any tables or plots included in that run. It does not provide reference outputs for all three questions listed above.

Expression values depend on the dataset release and selected groups. Literature results and generated wording can vary with sources, models, and query date; the example PDF documents one reference run rather than requiring identical generated text on every run. The example PDF is separate from the downloadable CSV tables generated by the web interface.

### Demo runtime

Demo runtime on a standard desktop computer has not yet been benchmarked. The first expression query loads the matrix into memory and can take longer than subsequent queries. End-to-end time also includes API latency and, for literature questions, web retrieval. Measure the first query and subsequent queries separately and report the computer specification, dataset release, and model settings with the timings.

## Using your own expression data

VasQ is configured for the supplied human brain vascular atlas. Replacing the expression files is possible, but use with a different dataset requires matching input formats and validation of the atlas-specific labels and prompts.

1. Prepare a log-normalized expression matrix with **cells/nuclei as rows and genes as columns**. The application does not normalize raw counts during loading.
2. Save the matrix as `VasQ_adata_X_sparse.npz`. The loader accepts CSR arrays named `data`, `indices`, `indptr`, and `shape`, or a dense array named `X`. CSR is recommended for sparse data.
3. Create `VasQ_gene_names.csv` with a header and gene names in its first column, in exactly the same order as the matrix columns. The loader converts these names to uppercase.
4. Create `VasQ_cell_meta_table.csv` with one row per matrix row, in exactly the same order. The loader uses row positions for alignment; matching cell IDs alone does not reorder the data.
5. Include the metadata columns below. Replace the three files together in `vasq/data/` for local Docker use, or in `/data` on Railway, and restart the application to clear the in-memory dataset cache.
6. Run a query for a gene present in the replacement matrix and compare its table with an independently calculated expression summary before relying on the results.

| Required metadata column | Alternative column name accepted by the loader |
| --- | --- |
| `brain_region` | `region_name` |
| `region_layer` | Same name |
| `cell_class` | `Cell_class` |
| `cell_type` | `Cell_type` |
| `age_at_death` | `ageatdeath` |
| `sex` | Same name |

The number of metadata rows must equal the number of matrix rows, and the number of gene-name entries must equal the number of matrix columns. Use trusted data files, retain the same normalization convention when comparing datasets, and validate region and cell-type resolution for any new vocabulary. The minimum group size of 10 cells still applies.

## 3. Optional: Deploy on Railway

To host your own VasQ instance, deploy the repository on Railway using `vasq/Dockerfile.railway` and configure your OpenAI API key and application settings.

Store the three expression data files in a persistent volume mounted at `/data`; the data files do not need to be uploaded to GitHub.

An independent deployment requires your own Railway account and OpenAI API key, with associated API and hosting costs. 
## Acknowledgements

VasQ was informed by [scChat](https://github.com/li-group/scChat) and [KG-RAG](https://github.com/BaranziniLab/KG_RAG). We acknowledge their contributions to conversational single-cell analysis and biomedical retrieval.

## License

The source code is licensed under the **Apache License 2.0**; see [LICENSE](LICENSE). Refer to the Figshare dataset release for its separate data-use terms.
