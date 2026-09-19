# Azure Purchase Order Processor

## Overview

This repository contains an Azure integration project for processing purchase-order files with Azure Logic Apps, Azure Blob Storage, HTTP integration, and an Azure Resource Manager (ARM) deployment template.

It demonstrates practical Azure integration patterns, workflow orchestration, file handling, API integration, and infrastructure-as-code.

## Business / Integration Scenario

The workflow models a common purchase-order integration pattern:

1. Purchase-order files are placed in an inbound Blob Storage location.
2. The Logic App discovers the files on a scheduled run.
3. Each source file is copied to an archive location before processing.
4. The original file content is sent to a downstream HTTP endpoint.
5. The downstream response is saved as a JSON file for subsequent processing or consumption.
6. The original input file is deleted only after the output file has been created successfully.

This provides a simple source-to-target integration pattern while preserving an archive copy of the original input.

---

# Architecture

The solution is composed of the following Azure services and artifacts:

| Component | Role |
|---|---|
| Azure Logic Apps (Stateful) | Orchestrates the purchase-order processing workflow. |
| Azure Blob Storage | Stores incoming purchase-order files, archived files, and generated output JSON. |
| Azure Blob Storage API Connection | Provides the Logic App actions used to list, read, copy, create, and delete blobs. |
| HTTP action | Sends purchase-order content to a downstream API/service. |
| Azure Resource Manager (ARM) template | Defines the Workflow App / Function App resource and related web configuration. |
| ARM parameters file | Supplies the Logic App name and App Service Plan resource ID used by the template. |

The workflow references an Azure Blob connection named `azureblob` and uses an App Service Plan supplied through the ARM deployment parameters.

---

# Repository Structure

```text
Purchase-Order-Processor/
├── README.md
├── load-order.json
├── parameters.json
└── template.json
```

### `load-order.json`

The Logic App workflow definition. It contains the trigger, Blob Storage operations, loop, HTTP integration, output-file creation, and source-file deletion logic.

### `template.json`

The ARM deployment template. It defines the Azure `Microsoft.Web/sites` Workflow App / Function App resource, its system-assigned managed identity, publishing credential policies, and web configuration.

### `parameters.json`

The ARM deployment parameter file. It supplies the Logic App name and the App Service Plan resource ID required by `template.json`.

---

# Workflow Details

## 1. Scheduled trigger

The Logic App uses a **Recurrence** trigger configured with:

```json
{
  "interval": 1,
  "frequency": "Week"
}
```

The supplied workflow therefore executes once per week.

> Change the schedule to match the processing frequency required by the target environment.

## 2. Discover inbound files

The `Lists_blobs_(V2)` action uses the `azureblob` connection to list files from:

```text
/purchaseorderxml
```

The returned collection is passed to a `For_each` loop.

## 3. Iterate through each purchase-order file

For each file returned by Blob Storage, the workflow performs the following processing sequence.

### Step 3.1 – Archive the source file

The `Copy_blob_(V2)` action creates an archive copy using:

```text
archive/<UTC timestamp>_<original filename>
```

Overwrite is enabled for the copy operation.

### Step 3.2 – Read the source file

`Get_blob_content_(V2)` reads the content of the original source blob after the archive operation succeeds.

### Step 3.3 – Send the purchase order to an HTTP endpoint

The `HTTP` action performs a `POST` request and sends the blob content as the request body.

The repository contains the following deployment placeholder:

```text
https://YOUR-API-HOST.example.com/purchaseorder
```

Replace it with the appropriate test or production endpoint before deployment.

The HTTP action uses chunked content transfer.

### Step 3.4 – Create the OMS output file

After the HTTP action succeeds, `Create_blob_(V2)` writes the HTTP response into:

```text
/omspo
```

The output file name follows this pattern:

```text
oms_<UTC timestamp>.json
```

### Step 3.5 – Delete the original source file

`Delete_blob_(V2)` removes the original input file only after creation of the output JSON succeeds.

This dependency is important because a failed downstream operation does not immediately remove the source document.

---

# Purchase-Order Payload

The workflow includes a representative purchase-order response structure containing fields such as:

- Purchase-order number and date
- Buyer and supplier information
- Ship-to and bill-to information
- Line items
- Item codes and descriptions
- Quantities and units of measure
- Unit prices and line amounts
- Delivery dates
- Tax type and tax rate
- Tax amount and grand total
- Payment terms
- Delivery terms
- Order status
- Processing remarks

The repository uses generic example values for the portfolio version.

---

# ARM Deployment Template

## `template.json`

The ARM template defines a `Microsoft.Web/sites` resource with:

```text
kind: functionapp,workflowapp
```

The resource uses a **system-assigned managed identity** and references an App Service Plan through a deployment parameter.

The supplied resource configuration is targeted at **Central India** and uses the Logic App name:

```text
la-order-process
```

The template also contains web configuration and publishing credential policies, including HTTPS-only access, TLS 1.2 settings, and disabled basic publishing credentials.

Review the environment-specific settings before production deployment.

---

# ARM Parameters

## `parameters.json`

The parameter file contains:

```text
sites_la_order_process_name
serverfarms_ASP_RGPOProcessor_a091_externalid
```

The Logic App name is:

```text
la-order-process
```

The App Service Plan value is represented by:

```text
REPLACE_WITH_APP_SERVICE_PLAN_RESOURCE_ID
```

Replace this value with the resource ID of the App Service Plan that should host the Workflow App.

---

# Resource Rename

The source export used the Logic App naming convention containing:

```text
la_order_process_02
la-order-process-02
```

The repository version uses:

```text
la_order_process
la-order-process
```

The parameter name and references were updated consistently in the ARM template and parameter file.

---

# Deployment Prerequisites

Before using these files in Azure, prepare:

1. An Azure subscription.
2. An Azure resource group.
3. An App Service Plan suitable for the Workflow App.
4. An Azure Storage account with the required folders.
5. An Azure Blob Storage API connection referenced by the Logic App as `azureblob`.
6. A downstream HTTP API endpoint.
7. Appropriate permissions for the Logic App and dependent resources.

Expected logical storage paths are:

```text
/purchaseorderxml
/archive
/omspo
```

---

# Deployment Considerations

## App Service Plan

Replace:

```text
REPLACE_WITH_APP_SERVICE_PLAN_RESOURCE_ID
```

with the correct App Service Plan resource ID.

## Blob Storage connection

Ensure the required Blob Storage connection exists and is referenced using:

```text
azureblob
```

## Downstream API

Replace:

```text
https://YOUR-API-HOST.example.com/purchaseorder
```

with the appropriate downstream endpoint and configure its authentication requirements.

## Authentication and authorization

Configure authentication, authorization, network access, identity permissions, and API security according to the target Azure environment.

---

# Operational Considerations

## Archive-before-processing

The workflow archives the input before downstream processing. Therefore an archive copy can exist even if a later processing step fails.

## Source-file deletion

The source file is deleted only after the output JSON file is created successfully.

## Potential duplicate processing

When a later operation fails, the source file can remain in the inbound location. A retry can therefore result in the same purchase order being submitted again.

For production workloads, consider an idempotency strategy based on a unique order identifier or another business key.

## File naming

Archive files follow:

```text
<UTC timestamp>_<original filename>
```

Output files follow:

```text
oms_<UTC timestamp>.json
```

Ensure the naming strategy supports the required uniqueness and traceability.

## Scheduling

The current workflow is configured for weekly execution. Change this when the required business SLA calls for a different frequency.

---

# Security

Do not commit secrets or credentials to source control.

Avoid placing the following directly in workflow or ARM JSON:

- Azure access keys
- Connection strings
- Client secrets
- API keys
- SAS tokens
- Bearer tokens
- Passwords
- Private certificates
- Personal contact information

Use Azure Key Vault, managed identities, secure application settings, or protected CI/CD variables for sensitive configuration.

---

# Troubleshooting Guide

## No files are being processed

Check:

- The Logic App trigger schedule.
- The presence of files in `/purchaseorderxml`.
- The `azureblob` API connection.
- Storage permissions.

## HTTP processing fails

Check:

- The configured API endpoint.
- API authentication.
- Network connectivity.
- The request format expected by the downstream service.
- Whether the HTTP action configuration matches the intended runtime behavior.

## Output file is not created

Check the HTTP action first because `Create_blob_(V2)` depends on successful HTTP completion.

## Source file is not deleted

The delete action is dependent on successful creation of the `/omspo` output file. A failure before that point can intentionally leave the source available for investigation or retry.

## ARM deployment fails

Check:

- The App Service Plan resource ID.
- The resource group and Azure region.
- Resource naming requirements.
- Deployment identity permissions.
- Compatibility of exported settings with the target environment.

---

# GitHub Repository Metadata

### Repository name

```text
Purchase-Order-Processor
```

### Repository description

```text
Azure Logic App workflow for automated purchase-order processing using Blob Storage, HTTP integration, archiving, and ARM-based deployment.
```

### Suggested topics

```text
azure
azure-logic-apps
logic-apps
azure-blob-storage
arm-template
azure-integration
workflow-automation
purchase-order
integration-patterns
infrastructure-as-code
```

---

# Git Workflow

```bash
git add README.md load-order.json parameters.json template.json
git commit -m "Add purchase order processor workflow"
git push origin main
```

---

# Portfolio Value

This project demonstrates practical cloud integration concepts including:

- Scheduled workflow orchestration
- Blob-based file ingestion
- File archival before processing
- HTTP-based system integration
- Writing downstream responses back to cloud storage
- Controlled source-file cleanup
- Azure Resource Manager infrastructure definition
- Parameterized deployment configuration
- Managed identity configuration
- Enterprise-style integration workflow design

---

## Author

**Vishal Sharma**

GitHub Profile: https://github.com/vpng2019
