# VasQ

**VasQ** is an AI chat system for exploring a human brain vascular single-nucleus RNA-seq atlas and generating biomedical hypotheses about blood-brain barrier biology, regional vascular specialization, and drug delivery.

## 1. Use VasQ online

**[Open VasQ](https://vasq-production-65c6.up.railway.app/vasq/)**

The hosted web interface is **free to use and requires no login**. Open the link in your browser and enter a research question. You do not need to install software, download the dataset, create a Railway account, or supply an OpenAI API key. The hosted service uses the operator's API credentials and infrastructure.

Example questions:

- Which vascular cell types express CLDN5?
- Using only the VasQ matrix, compare CLDN5 expression between Cortex and White Matter.
- What published evidence links CLDN5 to blood-brain barrier function?

VasQ is intended for research exploration and hypothesis generation, not clinical decision-making.

## What VasQ provides

- Natural-language expression queries with filters for brain region, region layer, cell class, cell type, and sex.
- Matrix-derived mean log-normalized expression, percentage of expressing cells, and cell counts; groups with fewer than 10 cells are excluded.
- Interactive expression plots, a paginated results table, and CSV downloads. For multi-gene queries, eligible results remain in the table and up to three gene profiles are plotted.
- Scientific evidence retrieval with source links, plus drug and small-molecule evidence when explicitly requested.

The atlas covers 41 brain regions, approximately 443,000 nuclei, and approximately 15,000 highly variable genes. Expression results are calculated directly from the matrix. The complete matrix, individual-cell records, and full expression table are not sent to the language model; a compact set of aggregate results is provided for interpretation.

## Choose how to run VasQ

| Option | Railway account | OpenAI API key | Dataset download |
| --- | --- | --- | --- |
| Hosted website above | Not needed | Not needed | Not needed |
| Your own computer | Not needed | Your own | Required |
| Your own Railway deployment | Your own | Your own | Required on your service |

Free access applies to the hosted website. If you run your own copy, you are responsible for its API usage and any hosting costs.

## 2. Run VasQ locally

### Requirements

- [Docker Desktop](https://docs.docker.com/desktop/) running on your computer, with Docker Compose available.
- Git, if cloning the repository instead of downloading a ZIP.
- Your own OpenAI API key and access to compatible models.
- The three expression data files listed below.

### Download the code

```bash
git clone --branch railway-deploy https://github.com/congliu-design/VasQ.git
cd VasQ
```

If you downloaded a ZIP instead, extract it and open a terminal in the extracted project folder. Run the remaining commands from that folder.

### Add the expression data

Download the expression data from the [VasQ Figshare project](https://figshare.com/projects/Atlas_of_Regional_Molecular_Specialization_in_the_Human_Brain_Vasculature/273930) and place the following files in `vasq/data/`:

| File | Contents |
| --- | --- |
| `VasQ_adata_X_sparse.npz` | Sparse CSR log-normalized expression matrix; rows are nuclei and columns are genes. |
| `VasQ_cell_meta_table.csv` | Per-nucleus region, region layer, cell class, cell type, age at death, and sex. |
| `VasQ_gene_names.csv` | Gene symbols in the same order as the matrix columns. |

The matrix and cell-metadata files are not included in this code package. The gene-name file is included; use the matching matrix and metadata from the same dataset release. If the Figshare files are not accessible, contact the authors for the data or a private access link before continuing. Local expression analysis requires all three files.

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

## 3. Deploy your own instance on Railway

This is optional. Use the hosted website if you only want to ask questions.

### Create and configure the service

1. Put the code in a GitHub repository you control and create a Railway service connected to it. Select the branch containing this version of VasQ.
2. Keep the Railway build root at the **repository root**, because the deployment Dockerfile copies files using `vasq/` paths.
3. Set `RAILWAY_DOCKERFILE_PATH=vasq/Dockerfile.railway` in the service variables. See [Railway's Dockerfile guide](https://docs.railway.com/builds/dockerfiles).
4. Add the OpenAI variables from `.env-shared.example` to Railway's service variables, using **your own API key**. Railway does not read your local `.env-shared` automatically.
5. Generate a long random secret and set `DJANGO_SECRET_KEY` to it. For example, run `python -c 'import secrets; print(secrets.token_urlsafe(50))'` on your computer. Set `DEBUG=False`.
6. Generate a public domain for the service. Set `ALLOWED_HOSTS` to its hostname, such as `your-service.up.railway.app`, and `CSRF_TRUSTED_ORIGINS` to its full HTTPS origin, such as `https://your-service.up.railway.app`. Multiple entries can be separated by commas.

### Provide the expression data

All three data files must exist under `/data` **inside the deployed service**. Running successfully on your computer does not upload those files to Railway.

For a GitHub-based deployment, attach a [Railway volume](https://docs.railway.com/volumes) at `/data` and populate it with the three files before running expression queries. Transfer or download the actual dataset files into that volume using an authorized data source. A Figshare project page is not a direct file-download URL.

Use the current [Railway CLI volume-file commands](https://docs.railway.com/cli/volume) to upload the files from your computer. After installing the CLI, run `railway login` and `railway link`, selecting your own project, environment, and VasQ service. From the repository root, run:

```bash
railway volume files upload ./vasq/data/VasQ_adata_X_sparse.npz /VasQ_adata_X_sparse.npz
railway volume files upload ./vasq/data/VasQ_cell_meta_table.csv /VasQ_cell_meta_table.csv
railway volume files upload ./vasq/data/VasQ_gene_names.csv /VasQ_gene_names.csv
```

Select the volume attached at `/data` if prompted. These remote paths are relative to the volume root, so `/VasQ_gene_names.csv` appears inside the container as `/data/VasQ_gene_names.csv`.

The volume covers any files copied to `/data` during the image build; an attached empty volume still needs to be populated.

### Start and verify

The Dockerfile starts Gunicorn using Railway's `PORT` environment variable. It uses one worker with threads because chat jobs and result tables use an in-process cache. Keep this configuration unless you also add shared job and cache storage.

After deployment, open `https://YOUR-RAILWAY-DOMAIN/vasq/`. Check that the page loads, then run an expression query and confirm that a table and plot are returned. A successful image build or page load alone does not confirm dataset access or API access.

Your deployment uses your Railway resources and the OpenAI credentials you configure. Visitors do not need their own credentials. No visitor login or per-user API-key entry is implemented in this version.

## How it works

![VasQ workflow](VasQ_workflow.png)

The active application is in `vasq/`, using Django, django-ninja, WhiteNoise, and Plotly.js. It classifies questions and resolves entities, computes atlas expression summaries when applicable, retrieves scientific or therapeutic evidence when requested, and synthesizes the results. OpenAI Chat Completions supports classification and synthesis; the Responses API supports web search.

The current workflow does not require SPOKE/KG-RAG or Google Vertex AI Search.

## Troubleshooting

| Problem | What to check |
| --- | --- |
| Local address does not open | Start Docker Desktop and `docker compose up`; check that port 8000 is available. |
| Missing data or no expression results | Confirm all three real data files are available at `/data` inside the running container and belong to the same release. |
| API authentication or model-access error | Check the key, account access, model names, and reasoning settings on the instance you are running. |
| Railway hostname or CSRF error | Set `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS` for your own domain, then redeploy. |

## Acknowledgements

VasQ was informed by [scChat](https://github.com/li-group/scChat) and [KG-RAG](https://github.com/BaranziniLab/KG_RAG). We acknowledge their contributions to conversational single-cell analysis and biomedical retrieval.

## License

The source-code license is provided in [LICENSE](LICENSE). Refer to the dataset release for its data-use terms.
