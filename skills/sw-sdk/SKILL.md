---
name: sw-sdk
description: Reference for writing correct Silent Witness SDK integration code. Covers the full pipeline (case creation, file upload, report generation with crash analysis and biomechanics) using Go and TypeScript SDKs. Use when writing, reviewing, or debugging SW SDK integration code.
---

# Silent Witness SDK Integration Reference

Use this skill when writing code that integrates with the Silent Witness API using the Go or TypeScript SDK. This covers the correct patterns, method signatures, field names, and async handling.

**Documentation Portal:** [docs.silentwitness.ai](https://docs.silentwitness.ai)

**Relevant docs:**
- [SDK Overview](https://docs.silentwitness.ai/sdks/overview)
- [SDK Quickstart](https://docs.silentwitness.ai/sdks/quickstart)
- [Analyzing a Case](https://docs.silentwitness.ai/guides/analyze-case)
- [Analysis Types Guide](https://docs.silentwitness.ai/guides/analysis-types)
- [SDK Reference](https://docs.silentwitness.ai/sdk-reference/organizations/overview)
- [Generating a Report](https://docs.silentwitness.ai/guides/generate-report)
- [File Upload](https://docs.silentwitness.ai/api-reference/files/upload)
- [Objects Reference](https://docs.silentwitness.ai/api-reference/objects/overview)
- [Rate Limiting](https://docs.silentwitness.ai/rate-limiting)

## Environments

| Environment | API Base URL | Dashboard |
|-------------|-------------|-----------|
| **Production** | `https://api.silentwitness.ai` | [app.silentwitness.ai](https://app.silentwitness.ai) |
| **Staging** | `https://api.staging.silentwitness.ai` | [app.staging.silentwitness.ai](https://app.staging.silentwitness.ai) |

Use staging for development and testing. API keys are environment-specific — a staging key won't work in production and vice versa.

## SDK Packages

- **Go**: `github.com/silentwitness/sw-go-sdk` (import as `silentwitness`)
- **TypeScript**: `@silentwitness/sdk` (class: `SilentWitnessClient`)

## Client Initialization

### Go

```go
import silentwitness "github.com/silentwitness/sw-go-sdk"

client := silentwitness.NewClient(silentwitness.Config{
    APIKey:  os.Getenv("SW_API_KEY"),
    BaseURL: os.Getenv("SW_BASE_URL"), // https://api.staging.silentwitness.ai for staging
})
defer client.Close()
```

### TypeScript

```typescript
import { SilentWitnessClient } from "@silentwitness/sdk";

const client = new SilentWitnessClient({
    apiKey: process.env.SW_API_KEY,
    baseUrl: process.env.SW_BASE_URL, // https://api.staging.silentwitness.ai for staging
});
```

## Pipeline Overview

The full integration pipeline has 7 steps:

| Step | Method | Sync/Async | Notes |
|:----:|--------|:----------:|-------|
| 1 | Initialize client | Setup | API key from env var `SW_API_KEY` |
| 2 | `CreateOrganization` | Sync | Optional, one-time per law firm |
| 3 | `CreateCase` | Sync | Required fields: plaintiffName, defendantName, attorneyName |
| 4 | `UploadFile` | Sync | One call per file, needs `purpose` field |
| 5 | `CreateReport` | Async | Starts crash analysis; add occupants for biomechanics |
| 6 | `GetReport` (poll) | Async | Poll every 5s until `completed` or `failed` |
| 7 | Download PDF | Sync | Use `reportUrl` from completed report |

**Key insight**: There is NO separate crash analysis or biomechanics API. `CreateReport` triggers crash analysis automatically from uploaded photos. Including `occupants` in the same request triggers integrated biomechanics analysis.

## Method Signatures & Field Names

### Organizations

```go
// Go
resp, err := client.CreateOrganization(ctx, &silentwitness.CreateOrganizationParams{
    Name: "Law Firm Name",  // required
    // Optional: PhoneNumber, StreetAddress, City, State, ZipCode, Country
})
// Returns: resp.Organization.Id, resp.Organization.Name
```

```typescript
// TypeScript
const resp = await client.createOrganization({
    name: "Law Firm Name",  // required
});
// Returns: resp.organization.id, resp.organization.name
```

### Cases

Required fields: `plaintiffName`, `defendantName`, `attorneyName`.

```go
// Go
resp, err := client.CreateCase(ctx, &silentwitness.CaseParams{
    PlaintiffName:  silentwitness.String("Jane Smith"),
    DefendantName:  silentwitness.String("Bob Johnson"),
    AttorneyName:   silentwitness.String("Jane Doe, Esq."),
    OrganizationId: silentwitness.String(orgID),  // optional
    // Optional: Name (auto-generated as "LastName v. LastName"), Side ("plaintiff"/"defense")
})
// Returns: resp.ID, resp.Name
```

```typescript
// TypeScript
const resp = await client.createCase({
    plaintiffName: "Jane Smith",
    defendantName: "Bob Johnson",
    attorneyName: "Jane Doe, Esq.",
    organizationId: orgId,  // optional
});
// Returns: resp.id, resp.name
```

### File Upload

One call per file. Required fields: `caseId`, `filename`, `content`, `purpose`.

**Purpose values:**
- `crash_analysis_plaintiff` — Plaintiff vehicle files (photos, EDR, docs)
- `crash_analysis_defendant` — Defendant vehicle files

```go
// Go
resp, err := client.UploadFile(ctx, &silentwitness.UploadFileParams{
    CaseID:   caseID,
    Filename: "front-damage.jpg",
    Content:  photoBytes,  // []byte
    Purpose:  silentwitness.FilePurposeCrashPlaintiff,
})
// Returns: resp.FileID
```

```typescript
// TypeScript
const resp = await client.uploadFile({
    caseId: caseId,
    filename: "front-damage.jpg",
    content: photoData,  // Buffer
    purpose: "crash_analysis_plaintiff",
});
// Returns: resp.fileId
```

**Constraints:**
- Max file size: 50MB
- Supported image formats: JPEG, PNG, WebP
- Supported doc formats: PDF, DOC, DOCX, CSV
- Upload 4-8 photos per vehicle for best analysis results

### Create Report

This is the main analysis trigger. Required: `caseId`, `type`, `plaintiff.imageFileIds` (or `edrFileId`).

```go
// Go — crash analysis only
resp, err := client.CreateReport(ctx, &silentwitness.CreateReportParams{
    CaseID: caseID,
    Type:   silentwitness.ReportTypeTechnicalReport,
    Plaintiff: &silentwitness.VehicleDataParams{
        ImageFileIds: []string{fileID1, fileID2},
        VehicleMaker: silentwitness.String("Toyota"),
        VehicleModel: silentwitness.String("Camry"),
        VehicleYear:  silentwitness.String("2020"),  // 4 digits
        VehicleType:  silentwitness.String("sedan"),
    },
    Defendant: &silentwitness.VehicleDataParams{  // optional, for two-vehicle crashes
        VehicleMaker: silentwitness.String("Ford"),
        VehicleModel: silentwitness.String("F-150"),
        VehicleYear:  silentwitness.String("2019"),
        VehicleType:  silentwitness.String("truck"),
    },
    AccidentDescription: silentwitness.String("Rear-end collision at red light"),
    AccidentDate:        silentwitness.String("2024-03-15"),  // YYYY-MM-DD
    AccidentTime:        silentwitness.String("14:30"),       // HH:MM
    AccidentLocation:    silentwitness.String("Main St & Oak Ave, LA, CA"),
})
// Returns: resp.ReportID
```

```typescript
// TypeScript — crash analysis only
const resp = await client.createReport({
    caseId,
    type: "technical_report",
    plaintiff: {
        imageFileIds: [fileId1, fileId2],
        vehicleMaker: "Toyota",
        vehicleModel: "Camry",
        vehicleYear: "2020",
        vehicleType: "sedan",
    },
    defendant: {  // optional
        vehicleMaker: "Ford",
        vehicleModel: "F-150",
        vehicleYear: "2019",
        vehicleType: "truck",
    },
    accidentDescription: "Rear-end collision at red light",
    accidentDate: "2024-03-15",
    accidentTime: "14:30",
    accidentLocation: "Main St & Oak Ave, LA, CA",
});
// Returns: resp.reportId
```

**Vehicle type values:** `sedan`, `suv`, `truck`, `van`, `coupe`, `compact`, `motorcycle`

### Adding Biomechanics (Occupants)

Add `occupants` to `CreateReport` to include biomechanics analysis. Seatbelt/airbag data is **per-occupant**, NOT per-vehicle.

```go
// Go — add to CreateReportParams
Occupants: []silentwitness.OccupantDataParams{
    {
        Name:         silentwitness.String("John Smith"),
        Age:          silentwitness.Int32(45),           // required, 1-120
        Gender:       silentwitness.String("male"),      // required: "male"/"female"/"other"
        HeightInches: silentwitness.Int32(70),            // optional
        WeightLbs:    silentwitness.Int32(180),           // optional
        Position:     silentwitness.String("driver"),     // required
        AllegedInjuries: []string{                        // required, at least 1
            "cervical_spine",
            "lumbar_spine",
        },
        InjurySeverity:        silentwitness.String("moderate"),  // optional
        PreExistingConditions: silentwitness.String("Prior L4-L5 herniation"),
        SeatbeltWorn:          silentwitness.Bool(true),          // default: true
        AirbagDeployed:        silentwitness.String("no"),        // "yes"/"no"/"partial"/"unknown"
    },
},
```

```typescript
// TypeScript — add to createReport params
occupants: [{
    name: "John Smith",
    age: 45,                // required
    gender: "male",         // required
    heightInches: 70,
    weightLbs: 180,
    position: "driver",     // required
    allegedInjuries: [      // required, at least 1
        "cervical_spine",
        "lumbar_spine",
    ],
    injurySeverity: "moderate",
    preExistingConditions: "Prior L4-L5 herniation",
    seatbeltWorn: true,
    airbagDeployed: "no",
}],
```

**Position values:** `driver`, `front_passenger`, `rear_left`, `rear_center`, `rear_right`

**Injury types:** `head_brain`, `cervical_spine`, `thoracic_spine`, `lumbar_spine`, `shoulder`, `hip`, `knee`, `foot_ankle`

**Severity values:** `minor`, `moderate`, `serious`, `severe`, `critical`

**Max occupants per request:** 10

### Poll for Report Completion

Poll `GetReport` every 5 seconds. Status values: `pending` → `processing` → `completed` | `failed`.

```go
// Go
for {
    report, err := client.GetReport(ctx, reportID)
    if err != nil {
        log.Fatalf("Status check failed: %v", err)
    }

    switch report.Status {
    case silentwitness.ReportStatusCompleted:
        fmt.Printf("PDF: %s\n", *report.ReportURL)

        // Access computed crash results
        if report.CrashResults != nil {
            r := report.CrashResults
            if r.FinalMaxDeltaVMph != nil {
                fmt.Printf("Delta-V: %.2f mph\n", *r.FinalMaxDeltaVMph)
            }
            if r.PDOFDegrees != nil {
                fmt.Printf("PDOF: %.0f° (%s)\n", *r.PDOFDegrees, *r.PDOFDirection)
            }
        }
        return
    case silentwitness.ReportStatusFailed:
        if report.ErrorMessage != nil {
            log.Fatalf("Failed: %s", *report.ErrorMessage)
        }
        log.Fatal("Report failed")
    }

    time.Sleep(5 * time.Second)
}
```

```typescript
// TypeScript
while (true) {
    const report = await client.getReport(reportId);

    if (report.status === "completed") {
        console.log(`PDF: ${report.reportUrl}`);

        // Access computed crash results
        if (report.crashResults) {
            console.log(`Delta-V: ${report.crashResults.finalMaxDeltaVMph} mph`);
            console.log(`PDOF: ${report.crashResults.pdofDegrees}° (${report.crashResults.pdofDirection})`);
        }
        break;
    }
    if (report.status === "failed") {
        throw new Error(`Failed: ${report.errorMessage}`);
    }

    await new Promise(resolve => setTimeout(resolve, 5000));
}
```

### Crash Results Fields (on completed report)

| Go Field | TS Field | Type | Description |
|----------|----------|------|-------------|
| `DeltaVAtPeakMph` | `deltaVAtPeakMph` | *float | Delta-V at peak acceleration (mph) |
| `FinalMaxDeltaVMph` | `finalMaxDeltaVMph` | *float | Final maximum Delta-V (mph) |
| `TimeAtPeakMs` | `timeAtPeakMs` | *float | Time at peak acceleration (ms) |
| `TimeAtFinalMaxMs` | `timeAtFinalMaxMs` | *float | Time at final maximum (ms) |
| `PeakAccelerationGs` | `peakAccelerationGs` | *float | Peak acceleration (g's) |
| `PDOFDegrees` | `pdofDegrees` | *float | Principal Direction of Force (0-360°) |
| `PDOFDirection` | `pdofDirection` | *string | Human-readable (e.g., "Rear", "Front") |
| `ProcessingSuccessful` | `processingSuccessful` | bool | Whether crash analysis succeeded |

**Important:** In Go, crash result fields are pointers — always nil-check before accessing.

### Download Report

`reportUrl` is a signed URL. Download directly. URLs expire after 1 hour — call `GetReport` again for a fresh URL.

## Helper Function: StartAnalysis

For simpler integrations, `StartAnalysis` abstracts the entire pipeline into one call. It handles case creation, file upload, analysis, and polling automatically.

```go
// Go
poller, err := silentwitness.StartAnalysis(ctx, &silentwitness.AnalyzeCaseRequest{
    AnalysisType:   silentwitness.AnalysisTypeTechnicalReport,
    OrganizationID: silentwitness.String("org_abc123"),
    CaseName:       silentwitness.String("Smith v. Jones"),
    Plaintiff: &silentwitness.VehicleAnalysisData{
        Images:      [][]byte{frontPhoto, rearPhoto},  // raw bytes, NOT file IDs
        VehicleMake: silentwitness.String("Toyota"),
    },
})
result, err := poller.Wait(ctx)
```

```typescript
// TypeScript
import { startAnalysis } from "@silentwitness/sdk";

const poller = startAnalysis(client, {
    organizationId: "org_abc123",
    caseName: "Smith v. Jones",
    plaintiff: {
        images: [new Uint8Array(frontPhoto), new Uint8Array(rearPhoto)],
        vehicleMake: "Toyota",
    },
});
const result = await poller.wait();
```

**Key difference from low-level SDK:**
- `StartAnalysis` takes raw image **bytes** (not file IDs) — it uploads for you
- Returns a poller with `.Wait()` / `.wait()` that handles polling internally
- No need to call CreateCase, UploadFile, CreateReport separately

## Common Mistakes to Avoid

1. **Don't create separate crash/biomechanics API calls** — both are triggered by `CreateReport`
2. **Don't put seatbelt/airbag on the vehicle** — they're per-occupant fields
3. **Don't pass raw image bytes to CreateReport** — it takes file IDs from `UploadFile`. Only `StartAnalysis` takes raw bytes.
4. **Don't forget to nil-check Go pointer fields** — crash results are `*float64`, `*string`
5. **Don't hardcode API keys** — always use `SW_API_KEY` env var
6. **Don't poll without a timeout** — cap at 15 minutes with exponential backoff
7. **Don't cache report URLs forever** — signed URLs expire after 1 hour

## Error Codes

| Code | Meaning | Common Fix |
|------|---------|------------|
| `UNAUTHENTICATED` | Bad or missing API key | Check `SW_API_KEY` |
| `INVALID_ARGUMENT` | Bad params (file too large, missing field, bad format) | Check constraints above |
| `NOT_FOUND` | Case or file ID doesn't exist | Verify IDs from previous steps |
| `FAILED_PRECONDITION` | Files not ready | Wait briefly after upload |
| `RESOURCE_EXHAUSTED` | Rate limited | Add delays between calls |
