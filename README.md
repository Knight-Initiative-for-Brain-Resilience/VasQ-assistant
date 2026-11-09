# VasQ

**Software release: VasQ v1.0.0**

**VasQ** is an AI chat system for exploring a human brain vascular single-nucleus RNA-seq atlas and generating biomedical hypotheses about blood-brain barrier biology, regional vascular specialization, and drug delivery.

![VasQ workflow](VasQ_workflow.png)

## 1. Use VasQ online

**[Open VasQ](https://vasq-production-65c6.up.railway.app/vasq/)**

The hosted web interface is **free to use and requires no login**. Open the link in your browser and enter a research question. You do not need to install software, download the dataset, create a Railway account, or supply an OpenAI API key.

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

## 2. Run VasQ locally on macOS

### System requirements

The following instructions use Docker Desktop on macOS. VasQ runs inside a Linux-based Python container.

| Component | Requirement or configured version |
| --- | --- |
| Container runtime | Docker Desktop for Mac with Docker Compose. Start Docker Desktop before running the commands below. |
| Python inside the container | Python 3.11 (`python:3.11-slim`), configured by `vasq/Dockerfile`. |
| Browser | A browser with JavaScript enabled for the web interface. |
| Network | Required for OpenAI API calls, web retrieval, initial image/package downloads, and browser assets loaded from external CDNs. |
| API access | Your own OpenAI API key and access to compatible models for your local instance. |
| Data | The three matching expression files described below. |
| Hardware | The application uses CPU-based matrix analysis and remotely hosted OpenAI models. No local GPU or other specialized hardware is required by the provided workflow. |
| Memory and disk | Sufficient RAM to load the selected matrix and metadata, and disk space for the data and Docker images. Minimum requirements have not been benchmarked for this release. |

Git is needed only if you clone the repository rather than download a ZIP. Python and the server dependencies are installed inside the container; a separate host Python installation is not needed for the Docker setup.

### Software dependencies

Python dependencies and version requirements are listed in [`vasq/requirements.txt`](vasq/requirements.txt) and installed automatically during the Docker build:

| Package | Version requirement |
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

System dependencies are installed automatically during the Docker build; see [`vasq/Dockerfile`](vasq/Dockerfile). The web interface also loads JavaScript libraries, stylesheets, and fonts from external CDNs, so internet access is required.

### Tested environment and build time

Local installation, startup, and an example query were successfully tested on an Apple silicon Mac using Docker Desktop on November 9, 2026.

| Component | Test configuration |
| --- | --- |
| Host platform | macOS on Apple silicon |
| Docker Desktop | 4.99.0 |
| Docker Engine / CLI | 29.9.3, build `5457199` (reported by `docker --version`) |
| Docker Compose | v5.7.0 |
| Container Python | Python 3.11, using `python:3.11-slim` |
| Django | 4.2.15, confirmed in the startup log |
| OpenAI Python SDK | 2.53.0, confirmed in the runtime log |

The recorded `docker compose build` wall-clock time was **62.19 seconds (approximately 1 minute)**, including the base-image download and dependency installation. This is the image-build time only; it excludes Docker Desktop installation, code and dataset downloads, API configuration, database initialization, static-file collection, and application startup. Build times vary with hardware, network speed, and Docker caching.

Database migration and static-file collection completed successfully, and the application served the local web interface and its CSS, JavaScript, and image assets.

### Download the code

```bash
git clone --branch railway-deploy https://github.com/Knight-Initiative-for-Brain-Resilience/VasQ-assistant.git
cd VasQ-assistant
```

Run the following setup steps from this repository root. If you downloaded a ZIP, open a terminal in the extracted repository folder instead.

### Add the expression data

Download the expression data from the [VasQ Figshare project](https://figshare.com/projects/Atlas_of_Regional_Molecular_Specialization_in_the_Human_Brain_Vasculature/273930) and place the following files in `vasq/data/`:

| File | Contents |
| --- | --- |
| `VasQ_adata_X_sparse.npz` | Sparse CSR log-normalized expression matrix; rows are nuclei and columns are genes. |
| `VasQ_cell_meta_table.csv` | Per-nucleus region, region layer, cell class, cell type, age at death, and sex. |
| `VasQ_gene_names.csv` | Gene symbols in the same order as the matrix columns. |

Docker Compose mounts this directory at `/data` inside the application container in read-only mode.

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
Use models available to your OpenAI API account that support VasQ’s required features. If you change models, adjust the reasoning settings as needed.

Keep `.env-shared` private and excluded from Git and Docker images. Docker Compose loads its settings when the application starts.

### Start the web interface

```bash
cd vasq
docker compose build
docker compose run --rm web python manage.py migrate --noinput
docker compose run --rm web python manage.py collectstatic --noinput
docker compose up
```
After the application starts, open [VasQ on your computer](http://127.0.0.1:8000/vasq/) in your browser. This address refers to your Mac, with VasQ served on port 8000.

Stop the application by pressing `Ctrl+C` in the terminal.

## Demo using the Figshare dataset

The [Figshare dataset](https://figshare.com/projects/Atlas_of_Regional_Molecular_Specialization_in_the_Human_Brain_Vasculature/273930) is the data source for this demo. Use the three matching files listed above.

### Run the demo

1. Open your [local VasQ instance](http://127.0.0.1:8000/vasq/) after starting it, or use our [hosted VasQ website](https://vasq-production-65c6.up.railway.app/vasq/).
2. Submit each of the following example questions separately:

   **Example 1:** How do PLVAP, AQP1, and SLCO2A1 rank as markers for fenestrated capillary cells?

   **Example 2:** Which vascular cell types and which brain regions are enriched for small-vessel-disease susceptibility genes?

   **Example 3:** What drugs failed because they cannot cross human BBB due to no route into receptor-mediated transcytosis?

3. Inspect the response and, where returned, the expression plots and results table. Download the expression table as CSV for expression queries.

### Expected output

| Example | Expected output |
| --- | --- |
| 1. PLVAP, AQP1, and SLCO2A1 as fenestrated capillary markers | A comparative interpretation of the three genes, supported by expression tables and plots for eligible groups. Tables report mean log-normalized expression, expressing cells (%), and cell counts, and can be downloaded as CSV. Groups with fewer than 10 cells are excluded. |
| 2. Small-vessel-disease susceptibility genes | A source-linked discussion of susceptibility genes and their expression patterns across relevant vascular cell types and brain regions, with expression tables and plots where applicable. Expression patterns should be distinguished from a formal statistical enrichment test. |
| 3. Drug failure, BBB penetration, and receptor-mediated transcytosis | A source-linked assessment of drug-delivery evidence, distinguishing documented BBB penetration limitations from other reasons for failure and identifying whether a proposed transcytosis mechanism is supported. An expression plot or CSV is not required for this evidence-focused question. |

**Run Queue mode:** Enter up to 20 questions, one per line, and click **Run Queue**. VasQ processes each question independently and sequentially, without carrying over conversation history between questions. A separate PDF report is automatically downloaded after each question is processed. Each report includes the question, the generated response with source links where provided, and any plots and tables displayed for that question. Full expression results, when available, can be downloaded separately as CSV from the web interface. Allow multiple downloads in your browser if prompted.

### Demo runtime

Each question typically takes approximately 1–6 minutes to complete. Runtime varies with query complexity, dataset loading, API response times, and web retrieval. In Run Queue mode, questions are processed sequentially, so the total runtime increases with the number of questions.

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
