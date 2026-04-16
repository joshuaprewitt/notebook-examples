---
name: systemlink-test
description: >-
  Create Python-based device test applications that integrate with NI SystemLink.
  Use when the user asks to create a new test, build a test script, integrate a Python
  test with SystemLink, create a functional or parametric test, report test results
  to Test Monitor, handle work items in a test context, package a test as a nipkg,
  or deploy a test application to a SystemLink-managed system. Covers the full lifecycle:
  work item integration, product/spec resolution, test execution with step logging,
  result and file upload, asset/DUT tracking, nipkg packaging, and CI/CD deployment.
---

# SystemLink Python Test Application

Create Python-based functional/parametric device test applications that integrate
with NI SystemLink across the full test lifecycle.

## When to Use

- Creating a new Python test application for an electronic or mechanical device
- Integrating an existing test script with SystemLink (results, work items, assets)
- Setting up test result reporting with steps, measurements, limits, and inputs/outputs
- Packaging a test application as a `.nipkg` for feed deployment
- Creating work item templates for test scheduling

## Prerequisites

- Python 3.10+
- `nisystemlink-clients` package (sole communication layer — no direct HTTP calls)
- SystemLink server with Test Monitor, Asset Management, Work Order, and File services
- Valid API key with appropriate permissions

## Procedure

Follow these phases in order when creating a new test application.

### Phase 1: Project Setup

1. Create the project directory structure (see [project structure](./references/project-structure.md))
2. Create `requirements.txt` with `nisystemlink-clients` and any hardware driver packages
3. Create a configuration module that reads `HttpConfiguration` from environment variables or config file
4. **Never hard-code API keys** — use environment variables or SystemLink system credentials

### Phase 2: Initialization Module

Build the initialization logic that runs before any test steps execute.

1. **Prompt for work item ID** — accept via CLI argument, input prompt, or barcode scan
2. **Query the work item** — use the work item client to fetch the full record
3. **Resolve the product** — query Test Monitor Product API by part number from the work item:
   - If product exists: read product `properties` as test specs (limits, conditions)
   - If product does not exist: prompt operator for data sheet or manual spec entry, then create the product with specs stored as properties
4. **Resolve the DUT** — query the Asset API using the work item's DUT resource assignment; extract serial number, model, and asset properties
5. **Resolve the system** — confirm the assigned system matches the local host
6. **Read work item properties** — make them available as test parameters (profile, overrides)
7. **Validate** — abort with clear error if any required parameter is missing
8. **Display summary** — show resolved parameters for operator confirmation

### Phase 3: Test Execution

Build the test runner that creates results and steps.

1. **Create a test result** with status `RUNNING` before any steps execute. Include:
   - `programName`, `serialNumber`, `partNumber`, `operator`, `hostName`, `startedAt`
   - Property `workItemId` set to the originating work item ID
2. **Transition work item** to "In Progress"
3. **Execute steps sequentially**. For each step:
   - Log **inputs** as step properties with `input.` prefix (stimulus/config applied to DUT)
   - Log **outputs** as step properties with `output.` prefix (raw DUT response)
   - Record the **measurement** (primary value under evaluation)
   - Record **limits** from product specs: `lowLimit`, `highLimit`, `units`, `comparisonType`
   - Determine step **status** by comparing measurement against limits
   - Record `step.startedAt`, `step.duration`, `step.limitSource`
   - Continue after failures unless safety-critical
4. **Update the result** — set final status (`PASSED`/`FAILED`/`ERRORED`), `totalTimeInSeconds`
5. **Upload files** — all generated files (logs, waveforms, data) to File Service with metadata:
   - `resultId`, `workItemId`, `minionId`, workspace, file type, timestamp
   - Store file IDs in result property `fileIds`
6. **Transition work item** to "Completed" or "Failed"

### Phase 4: Result and Step Schema

Every test result MUST conform to the schema defined in the
[requirements reference](./references/requirements.md) Section 6.

**Required result properties:**

| Key | Value |
|---|---|
| `workItemId` | The SystemLink work item ID |
| `fileIds` | Comma-separated file IDs of uploaded artifacts |

**Required step properties:**

| Key Pattern | Description |
|---|---|
| `input.<name>` | Stimulus or configuration applied to the DUT |
| `output.<name>` | Raw response or reading from the DUT |
| `step.startedAt` | Step start timestamp (ISO 8601) |
| `step.duration` | Step execution time in seconds |
| `step.limitSource` | Origin of limits (e.g., `"product:P-BAT-001"`) |

**Step types:** `NumericLimit`, `StringValue`, `PassFail`

**Comparison types:** `GELE` (≥ low, ≤ high), `EQ`, `GT`, `LT`

### Phase 5: SDK Client Usage

All SystemLink communication MUST use `nisystemlink-clients`. No direct HTTP.

| Operation | Client Class |
|---|---|
| Test results, steps, products | `TestMonitorClient` |
| Work items | Work item / work order client |
| DUT and fixture assets | `AssetManagementClient` |
| Real-time tags | `TagClient` |
| File upload | `FileClient` |

Configure `HttpConfiguration` once at startup and pass to all clients.
Use the SDK's model classes (`TestResult`, `TestStep`, `Product`) — do not construct raw JSON.

### Phase 6: Packaging and Deployment

1. **Package as `.nipkg`** — include Python source, dependency manifest, config files, install/uninstall scripts
2. **Version** with semantic versioning in package metadata
3. **Upload** to a SystemLink feed via `slcli feed package upload`
4. **Deploy** to test systems via SystemLink software deployment / NI Package Manager

### Phase 7: Work Item Template

The CI/CD pipeline must provision a work item template for this test.

1. Create a `deploy/work-item-template.json` in the repo
2. Template type: `testplan`
3. Template name: match the test program name
4. Include:
   - Resource requirements (system, fixtures, DUTs)
   - Configurable properties for test parameters
   - Part number filter restricting to the target product
   - Description and summary
   - Workflow association for state transitions (NEW → SCHEDULED → IN_PROGRESS → COMPLETED/FAILED)
5. Provision via `slcli workitem template create` in the pipeline

### Phase 8: Error Handling

| Scenario | Behavior |
|---|---|
| Step out of limits | Record FAILED, continue remaining steps |
| Hardware timeout | Retry once, abort with ERRORED and diagnostics |
| Server unreachable | Queue result locally, retry on next cycle |
| Missing serial/part number | Abort, do not create partial result |
| API key expired | Log error, halt, alert operator |

Retry transient HTTP errors (429, 500, 502, 503) with exponential backoff, up to 3 attempts.

## Key Rules

1. **`nisystemlink-clients` only** — no direct HTTP calls to SystemLink APIs
2. **No hard-coded credentials** — use environment variables or config files
3. **workItemId on every result** — links results to originating work order
4. **Limits from product specs** — never hard-code limits; read from product properties
5. **All files linked** — uploaded files carry `resultId`, `workItemId`, `minionId`; result carries `fileIds`
6. **Continue after step failure** — do not abort on first failed step
7. **Headless compatible** — must run on Windows and Linux without GUI

## References

- [Full requirements document](./references/requirements.md)
- [Project structure](./references/project-structure.md)
