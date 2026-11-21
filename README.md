# Amazon Electronics Ingestion

A practical Azure ingestion runbook for loading Amazon Electronics reviews and product metadata into a raw data zone. It covers the format repair needed to turn Python-dictionary metadata into newline-delimited JSON.

**Azure Blob Storage · Azure ML Compute · AzCopy · Python**

## Data flow

```text
Reviews JSONL ───────────────────────────────→ raw/reviews_Electronics_5.json
Compressed product metadata → format repair → raw/meta_Electronics_fixed.json
```

## Prepare the data

Obtain the Amazon Electronics review and metadata files under the dataset's original terms. The datasets and cloud resources are not included in this repository.

The older metadata dump uses Python dictionary literals rather than strict JSON. Convert that version with Python's literal parser; do not use `eval`.

```python
import ast
import gzip
import json

with gzip.open("meta_Electronics.json.gz", "rt", encoding="utf-8") as source:
    with open("meta_Electronics_fixed.json", "w", encoding="utf-8") as target:
        for line in source:
            if line.strip():
                target.write(json.dumps(ast.literal_eval(line), ensure_ascii=False) + "\n")
```

Use `json.loads` instead for metadata files that already contain valid JSON.

## Upload to Azure

Create the destination container first. Replace `YOUR_ACCOUNT` with your own storage account and use a short-lived SAS with only the permissions needed for the upload.

```sh
export AZURE_STORAGE_ACCOUNT=YOUR_ACCOUNT
read -s AZURE_STORAGE_SAS_TOKEN
export AZURE_STORAGE_SAS_TOKEN
azcopy copy ./reviews_Electronics_5.json "https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/azureml/raw/reviews_Electronics_5.json?${AZURE_STORAGE_SAS_TOKEN}"
azcopy copy ./meta_Electronics_fixed.json "https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/azureml/raw/meta_Electronics_fixed.json?${AZURE_STORAGE_SAS_TOKEN}"
az storage blob list --account-name "$AZURE_STORAGE_ACCOUNT" --container-name azureml --prefix raw/ --output table
unset AZURE_STORAGE_SAS_TOKEN
```

`azureml` is the container; `raw/` is a virtual blob prefix. Confirm file counts and sizes after upload. Keep SAS tokens out of source files and screenshots.

## What this repository demonstrates

- Handling semi-structured source formats before cloud ingestion.
- Distinguishing storage containers from blob prefixes.
- Using AzCopy and checking the resulting blob inventory.
- Preparing consistent inputs for the [Amazon Reviews Data Pipeline](https://github.com/MohZak-AI/amazon-reviews-data-pipeline).

This repository is a documented workflow, not a packaged application or an automated infrastructure deployment.
