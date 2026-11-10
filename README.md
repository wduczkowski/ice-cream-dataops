# Ice Cream DataOps

This repository contains the Cognite Data Fusion (CDF) Toolkit configuration created for the Cognite Data Fusion Bootcamp. It provisions the data foundation, ingestion pipelines, data modeling resources, Cognite Functions, transformations, and workflow orchestration for the Ice Cream Factory use case.

The repository is configured for bootcamp project **15**:

- Test: `cdf-bootcamp-15-test`
- Production: `cdf-bootcamp-15-prod`
- CDF cluster: `westeurope-1`
- CDF Toolkit: `0.6.53`
- Python: `3.11`

## Repository structure

```text
.
|-- cdf.toml
|-- pyproject.toml
|-- ice-cream-dataops/
|   |-- config.test.yaml
|   |-- config.prod.yaml
|   `-- modules/bootcamp/
|       |-- data_foundation/
|       |-- ice_cream_api/
|       `-- use_cases/oee/
`-- .github/workflows/deploy.yaml
```

The modules provide:

- CDF groups and capabilities for developers, extractors, and the OEE pipeline.
- RAW staging resources, data sets, and data modeling spaces.
- REST hosted extractors for assets and time series.
- An extraction pipeline for Ice Cream API data points.
- Transformations for the asset hierarchy and time-series contextualization.
- Cognite Functions for data-point ingestion and OEE calculation.
- A scheduled workflow that orchestrates transformations and functions.

## Prerequisites

- Ubuntu on WSL 2, Linux, or another environment capable of running Python 3.11.
- Poetry.
- Access to the appropriate CDF project and Microsoft Entra ID applications/groups.
- The service-principal credentials and group source IDs assigned to the environment.

Install the project dependencies from the repository root:

```bash
poetry install
```

## Local environment

Create a `.env` file in the repository root. The file is ignored by Git and must never be committed. It should define the values required by the selected CDF environment:

```dotenv
CDF_PROJECT=cdf-bootcamp-15-test
CDF_CLUSTER=westeurope-1
IDP_TENANT_ID=<tenant-id>
IDP_TOKEN_URL=<token-url>
LOGIN_FLOW=client_credentials
IDP_CLIENT_ID=<admin-toolkit-client-id>
IDP_CLIENT_SECRET=<admin-toolkit-client-secret>
DATA_DEVELOPER_SOURCE_ID=<data-developer-group-object-id>
DATA_PIPELINE_OEE_CLIENT_ID=<oee-client-id>
DATA_PIPELINE_OEE_CLIENT_SECRET=<oee-client-secret>
DATA_PIPELINE_OEE_SOURCE_ID=<oee-group-object-id>
ICAPI_EXTRACTORS_CLIENT_ID=<extractor-client-id>
ICAPI_EXTRACTORS_CLIENT_SECRET=<extractor-client-secret>
ICAPI_EXTRACTORS_SOURCE_ID=<extractor-group-object-id>
```

Use environment-specific application credentials and group object IDs. Test credentials must not be reused for production.

## Build and validation

Verify authentication and Toolkit authorization:

```bash
poetry run cdf auth verify
```

Build the test configuration:

```bash
poetry run cdf build --env=test
```

Review a deployment without applying changes:

```bash
poetry run cdf deploy --dry-run --env=test
```

Deploy only after reviewing the dry-run output:

```bash
poetry run cdf deploy --env=test
```

Production deployment is handled by the GitHub Actions workflow and requires the `prod` GitHub Environment with production-specific variables and secrets.

## Schedules

The schedules follow the bootcamp requirements. The minute of the nightly project-specific jobs is set to the project number, `15`.

| Resource | Schedule | Purpose |
|---|---:|---|
| Asset hierarchy transformation | `15 0 * * *` | Runs every night at 00:15 |
| Data-point frontfill Function | `*/10 * * * *` | Pulls the latest hour every 10 minutes |
| Data-point backfill Function | `15 0 * * *` | Fills gaps for the last five days every night at 00:15 |
| OEE Function | `5-59/5 * * * *` | Recalculates OEE every five minutes, starting at minute 5 |
| Data workflow | `*/15 * * * *` | Runs the orchestration workflow every 15 minutes |
| Assets hosted extractor | `1d` | Refreshes source assets daily |
| Time-series hosted extractor | `1h` | Refreshes time-series definitions hourly |

CDF cron expressions use UTC.

## Workflow

The `wf_icapi_data_pipeline` workflow executes these tasks in order:

1. Create or update the asset hierarchy.
2. Contextualize time series with assets.
3. Extract Ice Cream API data points.
4. Calculate OEE time series.

The workflow uses the Ice Cream API extractor service principal for trigger authentication. Individual Functions and transformations use the credentials configured for their bounded module.

## CI/CD

The configuration has been validated locally on Ubuntu running under WSL against the `test` environment. The actual deployments to both `test` and `prod` are performed through GitHub Actions; production deployment requires manual selection and approval.

The GitHub Actions workflow in `.github/workflows/deploy.yaml`:

- deploys to `test` automatically after relevant changes are pushed to `main`;
- supports manual selection of `test` or `prod` through `workflow_dispatch`;
- uses GitHub Environment variables for IDs and Environment secrets for credentials;
- expects production deployments to be protected by a required reviewer.

The following GitHub Environment secrets are required:

- `IDP_CLIENT_SECRET`
- `DATA_PIPELINE_OEE_CLIENT_SECRET`
- `ICAPI_EXTRACTORS_CLIENT_SECRET`

The following GitHub Environment variables are required:

- `IDP_CLIENT_ID`
- `DATA_DEVELOPER_SOURCE_ID`
- `DATA_PIPELINE_OEE_CLIENT_ID`
- `DATA_PIPELINE_OEE_SOURCE_ID`
- `ICAPI_EXTRACTORS_CLIENT_ID`
- `ICAPI_EXTRACTORS_SOURCE_ID`

## Security

- Never commit `.env`, client secrets, or generated credentials.
- Keep test and production credentials separate.
- Use the least-privileged service principal assigned to each module.
- Always inspect the Toolkit dry run before deploying configuration changes.
