---
name: sw-rest-api
description: Reference for writing correct Silent Witness REST API integration code. Covers the full pipeline (case creation, file upload, report generation with crash analysis and biomechanics) using HTTP/curl. Use when writing, reviewing, or debugging REST API integration code.
---

# Silent Witness REST API Integration Reference

Use this skill when writing code that integrates with the Silent Witness REST API directly (no SDK). This covers endpoints, field names, request/response formats, and async handling.

**Documentation Portal:** [docs.silentwitness.ai](https://docs.silentwitness.ai)

**Relevant docs:**
- [API Reference Overview](https://docs.silentwitness.ai/api-reference/overview)
- [Create Case](https://docs.silentwitness.ai/api-reference/cases/create)
- [Analysis Types Guide](https://docs.silentwitness.ai/guides/analysis-types)
- [End-to-End Example](https://docs.silentwitness.ai/guides/end-to-end-example)
- [Authentication](https://docs.silentwitness.ai/authentication)
- [Error Handling](https://docs.silentwitness.ai/error-handling)
- [File Upload](https://docs.silentwitness.ai/api-reference/files/upload)
- [Objects Reference](https://docs.silentwitness.ai/api-reference/objects/overview)
- [Rate Limiting](https://docs.silentwitness.ai/rate-limiting)

## Environments

| Environment | API Base URL | Dashboard |
|-------------|-------------|-----------|
| **Production** | `https://api.silentwitness.ai/api` | [app.silentwitness.ai](https://app.silentwitness.ai) |
| **Staging** | `https://api.staging.silentwitness.ai/api` | [app.staging.silentwitness.ai](https://app.staging.silentwitness.ai) |

Use staging for development and testing. API keys are environment-specific — a staging key won't work in production and vice versa.

## Authentication

- **Auth header**: `X-API-Key: $API_KEY`
- **Rate limit**: 100 requests/min per API key

All responses follow:

```json
{
  "success": true,
  "data": { ... }
}
```

Error responses:

```json
{
  "success": false,
  "error": "Error description",
  "details": { "field": "...message" }
}
```

## Pipeline Overview

```
1. POST /api/organizations        → Create organization (optional, one-time)
2. POST /api/cases                → Create case with parties, vehicles, occupants, accident
3. POST /api/files/upload         → Upload damage photos (multipart, repeat per file)
4. PUT  /api/cases/:id            → Link file IDs to plaintiff vehicle
5. POST /api/reports              → Create report (starts async processing)
6. GET  /api/reports/:id          → Poll every 5s until status is "completed"
7. Download PDF/DOCX              → Use signed URLs from output (expire after 1 hour)
```

## Endpoints

### 1. Create Organization (optional)

```
POST /api/organizations
Content-Type: application/json
```

```json
{
  "name": "Smith & Associates Law Firm",
  "phone_number": "555-123-4567",
  "street_address": "123 Main Street, Suite 400",
  "city": "San Francisco",
  "state": "CA",
  "zip_code": "94102",
  "country": "United States"
}
```

**Required**: `name`, `phone_number`, `street_address`, `city`, `state`, `zip_code`, `country`

**Response** (201):

```json
{
  "success": true,
  "data": {
    "id": "org_abc123def456",
    "account_id": "acc_xyz789",
    "name": "Smith & Associates Law Firm"
  }
}
```

### 2. Create Case

```
POST /api/cases
Content-Type: application/json
```

```json
{
  "plaintiff_name": "John Smith",
  "defendant_name": "Bob Johnson",
  "attorney_name": "Jane Attorney, Esq.",
  "organization_id": "org_abc123def456",
  "side": "plaintiff",
  "analysis_type": "accident_injury",

  "accident": {
    "description": "Rear-end collision at red light",
    "date": "2024-01-10",
    "time": "14:30",
    "location": "123 Main St at Oak Ave, Los Angeles, CA"
  },

  "vehicles": [
    {
      "role": "plaintiff",
      "vehicle_maker": "Toyota",
      "vehicle_model": "Camry",
      "vehicle_year": "2020",
      "vehicle_vin": "4T1B11HK5LU123456",
      "vehicle_type": "sedan"
    },
    {
      "role": "defendant",
      "vehicle_maker": "Ford",
      "vehicle_model": "F-150",
      "vehicle_year": "2019",
      "vehicle_type": "truck"
    }
  ],

  "occupants": [
    {
      "name": "John Smith",
      "age": 45,
      "gender": "male",
      "height_inches": 70,
      "weight_lbs": 180,
      "position": "driver",
      "seatbelt_worn": true,
      "airbag_deployed": "yes",
      "alleged_injuries": ["cervical_spine", "lumbar_spine"],
      "injury_severity": "moderate",
      "pre_existing_conditions": "None"
    }
  ]
}
```

**Required**: `plaintiff_name`, `defendant_name`, `attorney_name`

**Optional**: `name` (auto-generated as "LastName v. LastName"), `side` (`plaintiff`|`defense`, default `plaintiff`), `analysis_type`, `organization_id`, `accident`, `vehicles`, `occupants`

**Analysis type values:**

| Value | Description | Requires Occupants? |
|-------|-------------|:-------------------:|
| `accident_only` | Crash analysis only (default) | No |
| `accident_injury` | Crash + biomechanics analysis | Yes |
| `delta_v_only` | Delta-V calculation only | No |
| `biomechanics_only` | Biomechanics analysis only | Yes |

**Response** (201):

```json
{
  "success": true,
  "data": {
    "case": {
      "id": "case_xyz789abc123",
      "name": "Smith v. Johnson",
      "plaintiff_name": "John Smith"
    },
    "invoice": {
      "id": "inv_xyz789",
      "amount": 299.00,
      "status": "pending"
    }
  }
}
```

### 3. Upload Files

```
POST /api/files/upload
Content-Type: multipart/form-data
```

**Form fields:**
- `file` (file, required) — binary file data
- `caseId` (string, required) — case ID
- `fileCategory` (string, required) — see values below
- `vehicleIndex` (int, optional) — 0 for plaintiff, 1 for defendant

```bash
curl -X POST "https://api.silentwitness.ai/api/files/upload" \
  -H "X-API-Key: $API_KEY" \
  -F "file=@/path/to/front_damage.jpg" \
  -F "caseId=case_xyz789abc123" \
  -F "fileCategory=crash_analysis_plaintiff" \
  -F "vehicleIndex=0"
```

**Response** (201):

```json
{
  "success": true,
  "data": {
    "fileId": "file_img001",
    "fileName": "front_damage.jpg",
    "status": "ready"
  }
}
```

**File category values:**
- `crash_analysis_plaintiff` — plaintiff vehicle photos/EDR/docs
- `crash_analysis_defendant` — defendant vehicle photos/EDR/docs

**Constraints:**
- Max file size: 50MB
- Supported images: JPG, JPEG, PNG, GIF, HEIC, HEIF, WebP
- Supported docs: PDF, DOC, DOCX, CSV, CDR
- Upload 4-8 photos per vehicle for best results

### 4. Link Files to Vehicle

```
PUT /api/cases/:id
Content-Type: application/json
```

```json
{
  "vehicles": [
    {
      "role": "plaintiff",
      "image_file_ids": ["file_img001", "file_img002", "file_img003"]
    }
  ]
}
```

### 5. Create Report

```
POST /api/reports
Content-Type: application/json
```

```json
{
  "case_id": "case_xyz789abc123",
  "type": "technical_report",
  "options": {
    "include_biomechanics": true,
    "use_demo_data": false
  }
}
```

**Required**: `case_id`, `type`

**Report type values:** `technical_report`

**Options:**
- `include_biomechanics` (bool, default false) — include biomechanics analysis for occupants
- `use_demo_data` (bool, default false) — use synthetic crash params for testing (bypasses ML)

**Response** (200):

```json
{
  "success": true,
  "data": {
    "id": "rpt_abc789xyz",
    "case_id": "case_xyz789abc123",
    "type": "technical_report",
    "status": "pending",
    "progress": {
      "message": "Starting analysis..."
    }
  }
}
```

### 6. Poll Report Status

```
GET /api/reports/:id
```

Poll every 5 seconds. Max timeout: 15 minutes.

**During processing:**

```json
{
  "success": true,
  "data": {
    "id": "rpt_abc789xyz",
    "status": "processing",
    "progress": {
      "current_step": "delta_v_calculation",
      "steps_completed": [],
      "message": "Calculating delta-v from damage photos..."
    }
  }
}
```

**When completed:**

```json
{
  "success": true,
  "data": {
    "id": "rpt_abc789xyz",
    "status": "completed",
    "progress": {
      "current_step": "report_generation",
      "steps_completed": ["delta_v_calculation", "biomechanics_analysis", "report_generation"],
      "message": "Report generation complete"
    },
    "output": {
      "pdf_url": "https://storage.silentwitness.ai/reports/rpt_abc789xyz.pdf",
      "docx_url": "https://storage.silentwitness.ai/reports/rpt_abc789xyz.docx"
    }
  }
}
```

**Status values:** `pending` → `processing` → `completed` | `failed` | `cancelled`

**Progress steps:** `delta_v_calculation` → `biomechanics_analysis` → `report_generation`

### 7. Download Report

```bash
curl -o report.pdf "$PDF_URL"
curl -o report.docx "$DOCX_URL"
```

Signed URLs expire after 1 hour. Call `GET /api/reports/:id` again for fresh URLs.

### Cancel Report

```
DELETE /api/reports/:id
```

Only `pending` or `processing` reports can be cancelled.

### List Reports

```
GET /api/reports?case_id=case_abc123
```

### List Files

```
GET /api/files?caseId=case_abc123
```

### Delete File

```
DELETE /api/files/:fileId
```

### List Organizations

```
GET /api/organizations?page=1&limit=20
```

### Get Case

```
GET /api/cases/:id
```

Returns full case with nested vehicles (including `crash_parameters`), occupants, accident, and report.

### List Cases

```
GET /api/cases?page=1&limit=20&organization_id=org_xyz&status=active
```

## Pagination

List endpoints support:
- `page` (int, default 1)
- `limit` (int, default 20, max 100)

Response includes:

```json
{
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

## Field Reference

### Vehicle Fields

**Input** (create/update): `vehicle_maker`, `vehicle_model`, `vehicle_year`, `vehicle_vin`, `vehicle_type`

**Output** (GET response): `make`, `model`, `year`, `vin`, `type`

**Vehicle type values:** `sedan`, `suv`, `truck`, `van`, `coupe`, `compact`, `motorcycle`

### Occupant Fields

| Field | Required | Type | Description |
|-------|:--------:|------|-------------|
| `age` | Yes | int | 1-120 |
| `gender` | Yes | string | `male`, `female`, `other` |
| `position` | Yes | string | See values below |
| `alleged_injuries` | Yes | string[] | At least 1. See values below |
| `name` | No | string | Occupant name |
| `height_inches` | No | int | Height in inches |
| `weight_lbs` | No | int | Weight in pounds |
| `injury_severity` | No | string | `minor`, `moderate`, `serious`, `severe`, `critical` |
| `pre_existing_conditions` | No | string | Free text |
| `seatbelt_worn` | No | bool | Default: true |
| `airbag_deployed` | No | string | `yes`, `no`, `partial`, `unknown` |

**Position values:** `driver`, `front_passenger`, `rear_left`, `rear_center`, `rear_right`

**Injury types:** `head_brain`, `cervical_spine`, `thoracic_spine`, `lumbar_spine`, `shoulder`, `hip`, `knee`, `foot_ankle`

### Crash Parameters (on GET case response)

Returned in `vehicles[].crash_parameters` after analysis completes:

| Field | Type | Description |
|-------|------|-------------|
| `delta_v_min` | float | Delta-V at peak acceleration (mph) |
| `delta_v_max` | float | Final maximum delta-V (mph) |
| `delta_v_method` | string | `ml`, `edr`, `vector`, `hybrid` |
| `pdof_degrees` | float | Principal Direction of Force (0-360°) |
| `crash_pulse_min_ms` | float | Time at peak acceleration (ms) |
| `crash_pulse_max_ms` | float | Time at final maximum (ms) |
| `calculated_at` | string | ISO 8601 timestamp |

## HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Created |
| 400 | Bad request (missing/invalid fields) |
| 401 | Unauthorized (bad or missing API key) |
| 403 | Forbidden (valid key, no permission) |
| 404 | Not found |
| 422 | Validation error |
| 429 | Rate limited (100 req/min) |
| 500-504 | Server error |

## Common Mistakes to Avoid

1. **Don't use `Content-Type: application/json` for file uploads** — uploads are `multipart/form-data`
2. **Don't forget to link files to vehicles** — upload gives you file IDs, then `PUT /api/cases/:id` links them
3. **Don't poll without a timeout** — cap at 15 minutes
4. **Don't cache report URLs forever** — signed URLs expire after 1 hour
5. **Don't skip the `fileCategory` field** — uploads without it won't be associated correctly
6. **Vehicle field names differ between input and output** — `vehicle_maker` on create, `make` on GET
7. **Occupant `alleged_injuries` needs at least 1 entry** — empty array will fail validation
8. **`use_demo_data` is for testing only** — never use in production
9. **`accident_injury` and `biomechanics_only` require occupants** — creating a case with these types but no occupants will skip biomechanics analysis
