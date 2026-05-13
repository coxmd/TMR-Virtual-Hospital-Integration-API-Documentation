# TMR Virtual Hospital Integration — API Documentation

**Hanmak Technologies Ltd.**
**Version:** 1.0.0
**Base URL:** `https://medicentre.hanmak.co.ke`

---

## Overview

The TMR Virtual Hospital API exposes a controlled, read-only subset of MedicentreV3's clinical and administrative data to the TMR Virtual Hospital Application. All endpoints are prefixed with `/api/external/` and require OAuth2 Bearer token authentication except the token issuance endpoint.

---

## Base URL

```
https://medicentre.hanmak.co.ke/api/external
```

---

## Property Name Casing

**Request bodies** use **PascalCase** — the C# view model property names exactly.
**Response bodies** use **camelCase** — ASP.NET Core default JSON serialisation.

| ✅ Request (PascalCase) | ✅ Response (camelCase) |
|---|---|
| `"ClientId"` | `"clientId"` |
| `"PatientNumber"` | `"patientNumber"` |
| `"VisitId"` | `"visitId"` |
| `"Pagination"` | `"pagination"` |
| `"FromDate"` | `"fromDate"` |

---

## Authentication

**Token Lifetime:** 3600 seconds (1 hour)

**Authorization Header (all data endpoints):**
```
Authorization: Bearer {access_token}
```

---

## Scopes

| Scope | Guards Access To |
|---|---|
| `tmr:patients:read` | Patient biodata and allergy records |
| `tmr:history:read` | Patient clinical history |
| `tmr:visits:read` | OPD visit records and visit lists |
| `tmr:clinical:read` | Laboratory results, radiology, prescriptions |
| `tmr:ipd:read` | Inpatient admissions |
| `tmr:discharge:read` | Discharge details and discharge drugs |

---

## Standard Response Envelope

All endpoints return the same camelCase wrapper:

```json
{
  "success": true,
  "message": "string",
  "errorCode": null,
  "data": {},
  "generatedAtUtc": "2026-05-13T10:40:44.8702392Z",
  "correlationId": "9de5cd88b7384aa0a02bfa935037a00d"
}
```

> **Note:** `correlationId` is a 32-character hex string without hyphens.

---

## Paginated Response

List endpoints return data inside a paged result nested under `data`:

```json
{
  "items": [],
  "totalCount": 0,
  "page": 1,
  "pageSize": 20,
  "totalPages": 1,
  "hasNextPage": false,
  "hasPreviousPage": false
}
```

---

## Patient Identifier

All requests that resolve a patient accept a `PatientIdentifierViewModel`. Resolution order (first populated field wins):

```json
{
  "PatientId": "string",       // 1. Internal system ID (highest)
  "PatientNumber": "string",   // 2. OPD / registration number
  "IDNumber": "string",        // 3. National ID number
  "PassportNumber": "string",  // 4. Passport number
  "PhoneNumber": "string",     // 5. Primary phone
  "EmailAddress": "string"     // 6. Email address (lowest)
}
```

---

## Error Codes

| Code | Description |
|---|---|
| `TMR_AUTH_FAILED` | Invalid or missing token |
| `TMR_INVALID_TOKEN` | Token signature invalid |
| `TMR_TOKEN_EXPIRED` | Token has expired |
| `TMR_INSUFFICIENT_SCOPE` | Token does not carry the required scope |
| `TMR_NOT_FOUND` | Resource not found or outside permitted branch |
| `TMR_VALIDATION_FAILED` | Request body failed validation |
| `TMR_INTERNAL_ERROR` | Unexpected server error |

---

## Endpoints

---

### Authentication

---

#### POST /auth/token

Obtain an OAuth2 access token using the Client Credentials grant.

**Content-Type:** `application/x-www-form-urlencoded`

**Request Body (PascalCase form keys):**

| Key | Value |
|---|---|
| `GrantType` | `client_credentials` |
| `ClientId` | API Key from the Mini Apps Configuration panel |
| `ClientSecret` | Secret generated for this API Key |
| `Scope` | Space-separated list of requested scopes |

**Example Request:**

```
POST /api/external/auth/token
Content-Type: application/x-www-form-urlencoded

GrantType=client_credentials
&ClientId=LMl%2BjL5SnEd%2B...
&ClientSecret=qyc78YoRiZDB...
&Scope=tmr:patients:read tmr:visits:read tmr:clinical:read
```

**Example Response — 200 OK:**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "tmr:patients:read tmr:visits:read tmr:clinical:read"
}
```

**Error Responses:**

| Status | Cause |
|---|---|
| `401` | Unrecognised `ClientId` or incorrect `ClientSecret` |
| `400` | Requested scope is not permitted for this client |

---

### Patient Information

---

#### POST /patients/biodata

Retrieve full biographical data for a patient.

**Scope required:** `tmr:patients:read`
**Content-Type:** `application/json`

**Request Body:**

```json
{
  "Identifier": {
    "PatientId": "string",
    "PatientNumber": "string",
    "IDNumber": "string",
    "PassportNumber": "string",
    "PhoneNumber": "string",
    "EmailAddress": "string"
  }
}
```

**Example — lookup by OPD number:**

```json
{
  "Identifier": {
    "PatientNumber": "DH1281119"
  }
}
```

**Example Response — 200 OK** *(verified against live staging data)*

```json
{
  "success": true,
  "message": "Patient file found.",
  "errorCode": null,
  "data": {
    "patientId": "145",
    "patientNumber": "DH1281119",
    "firstName": "John Mwembei",
    "lastName": "Dummy",
    "dateOfBirth": "1960-12-31T21:00:00",
    "age": 66,
    "sex": "Male",
    "phoneNumber": "07XXXXXXXX",
    "secondPhoneNumber": "",
    "emailAddress": "",
    "address": "",
    "idNumber": null,
    "passportNumber": null,
    "allergies": [
      {
        "allergen": "Unknown",
        "reaction": ""
      }
    ]
  },
  "generatedAtUtc": "2026-05-13T10:40:44.8702392Z",
  "correlationId": "9de5cd88b7384aa0a02bfa935037a00d"
}
```

**Field Notes:**
- `message` is `"Patient file found."` on success — not `"Successful."`
- `secondPhoneNumber`, `emailAddress`, `address` may be empty strings
- `idNumber`, `passportNumber` may be `null` if not recorded at registration
- `allergies` is always an array; `reaction` may be an empty string

---

#### POST /patients/history

Retrieve the clinical encounter history for a patient. Results are paginated.

**Scope required:** `tmr:history:read`
**Content-Type:** `application/json`

**Request Body:**

```json
{
  "Identifier": {
    "PatientId": "string",
    "PatientNumber": "string",
    "IDNumber": "string",
    "PassportNumber": "string",
    "PhoneNumber": "string",
    "EmailAddress": "string"
  },
  "FromDate": "2025-01-01T00:00:00Z",
  "ToDate": "2026-05-13T00:00:00Z",
  "EncounterType": "string",
  "Pagination": {
    "Page": 1,
    "PageSize": 20
  }
}
```

**Example Request:**

```json
{
  "Identifier": {
    "PatientNumber": "DH1281119"
  },
  "Pagination": {
    "Page": 1,
    "PageSize": 10
  }
}
```

**Example Response — 200 OK:**

```json
{
  "success": true,
  "message": "Successful.",
  "errorCode": null,
  "data": {
    "items": [
      {
        "entryId": "61106",
        "patientId": "145",
        "encounterDateUtc": "2022-09-12T06:48:51Z",
        "encounterType": "Outpatient",
        "attendingClinician": "Gekonge Wycliffe",
        "department": "Out Patient Department",
        "primaryDiagnosis": "Plasmodium falciparum malaria, unspecified",
        "chiefComplaint": null,
        "branchName": "Main Branch"
      }
    ],
    "totalCount": 1,
    "page": 1,
    "pageSize": 10,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  },
  "generatedAtUtc": "2026-05-13T09:00:00.0000000Z",
  "correlationId": "a3f9c1d2e8b047ab8c21d3f4e5b6c7d8"
}
```

---

### Visit Data

---

#### POST /visits

Retrieve a paginated list of outpatient visits.

**Scope required:** `tmr:visits:read`
**Content-Type:** `application/json`

**Request Body:**

```json
{
  "Patient": {
    "PatientId": "string",
    "PatientNumber": "string",
    "IDNumber": "string",
    "PassportNumber": "string",
    "PhoneNumber": "string",
    "EmailAddress": "string"
  },
  "FromDate": "2026-01-01T00:00:00Z",
  "ToDate": "2026-05-13T00:00:00Z",
  "VisitType": "string",
  "Status": "string",
  "Pagination": {
    "Page": 1,
    "PageSize": 20
  }
}
```

All fields are optional. An empty body `{}` returns all visits within the client's permitted branch.

**Example — filter by patient:**

```json
{
  "Patient": {
    "PatientNumber": "DH1281119"
  },
  "Pagination": { "Page": 1, "PageSize": 20 }
}
```

**Example — no filter:**

```json
{
  "Pagination": { "Page": 1, "PageSize": 20 }
}
```

**Example Response — 200 OK:**

```json
{
  "success": true,
  "message": "Successful.",
  "errorCode": null,
  "data": {
    "items": [
      {
        "visitId": "61106",
        "visitNumber": "61106",
        "patientId": "145",
        "visitDate": "2022-09-12T06:48:51",
        "visitType": "Outpatient",
        "status": "Closed",
        "department": "Out Patient Department",
        "attendingClinician": "Gekonge Wycliffe",
        "branchName": "Main Branch",
        "paymentMode": "Cash Payers",
        "insuranceScheme": null
      }
    ],
    "totalCount": 1,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  },
  "generatedAtUtc": "2026-05-13T09:00:00.0000000Z",
  "correlationId": "b7e2d4f1c9a853ab9d12e4f5a6b7c8d9"
}
```

**Field Notes:**
- `visitType` is `"Outpatient"` — not `"OPD"`
- `insuranceScheme` is `null` for cash payers

---

#### GET /visits/{visitId}/opd

Retrieve the complete OPD visit detail composite.

**Scope required:** `tmr:clinical:read`

**Path Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `visitId` | `string` | The unique visit identifier |

**Example Request:**

```
GET /api/external/visits/61106/opd
Authorization: Bearer {access_token}
```

**Example Response — 200 OK** *(verified against live staging data)*

```json
{
  "success": true,
  "message": "Successful.",
  "errorCode": null,
  "data": {
    "patientBiodata": {
      "patientId": "145",
      "patientNumber": "DH1281119",
      "firstName": "John Mwembei",
      "lastName": "Dummy",
      "dateOfBirth": "1960-12-31T21:00:00",
      "age": 66,
      "sex": "Male",
      "phoneNumber": "07XXXXXXXX",
      "secondPhoneNumber": "",
      "emailAddress": "",
      "address": "",
      "idNumber": null,
      "passportNumber": null,
      "allergies": [
        { "allergen": "Unknown", "reaction": "" }
      ]
    },
    "visitSummary": {
      "visitId": "61106",
      "visitNumber": "61106",
      "patientId": "145",
      "visitDate": "2022-09-12T06:48:51",
      "visitType": "Outpatient",
      "status": "Closed",
      "department": "Out Patient Department",
      "attendingClinician": "Gekonge Wycliffe",
      "branchName": "Main Branch",
      "paymentMode": "Cash Payers",
      "insuranceScheme": null
    },
    "clinicalNotes": {
      "triage": {
        "bloodPressure": "120/80 mmHg",
        "temperature": "38.0 C",
        "pulseRate": null,
        "respiratoryRate": null,
        "oxygenSaturation": null,
        "weight": "60 Kg",
        "height": "0.7 m",
        "bmi": "122.45 Kg/M2",
        "additionalNotes": null
      },
      "doctorNotes": null,
      "recordedAtUtc": "2022-09-12T06:48:51Z",
      "recordedBy": null
    },
    "diagnoses": [
      {
        "diagnosisCode": "B50.9",
        "diagnosisName": "Plasmodium falciparum malaria, unspecified",
        "diagnosisType": null,
        "notes": null
      }
    ],
    "labTests": [
      {
        "labTestId": "56425",
        "visitId": "61106",
        "patientId": "145",
        "testName": "BS for Malaria - Falciparum",
        "testCategory": null,
        "orderedAtUtc": "2022-09-12T06:55:41Z",
        "resultedAtUtc": null,
        "status": "Pending",
        "resultValue": "",
        "resultUnit": " ",
        "referenceRange": "Seen",
        "isAbnormal": false,
        "orderingClinician": "Wycliffe"
      },
      {
        "labTestId": "56426",
        "visitId": "61106",
        "patientId": "145",
        "testName": "Blood Sugar - Random blood sugar",
        "testCategory": null,
        "orderedAtUtc": "2022-09-12T06:55:41Z",
        "resultedAtUtc": null,
        "status": "Pending",
        "resultValue": "11",
        "resultUnit": "mmol/l",
        "referenceRange": "High",
        "isAbnormal": true,
        "orderingClinician": "Wycliffe"
      }
    ],
    "radiologyExaminations": [],
    "medicines": [
      {
        "prescriptionId": "179057",
        "visitId": "61106",
        "patientId": "145",
        "drugName": "PARACETAMOL 500MG",
        "dosage": "Tablet",
        "frequency": "3",
        "duration": "3 Days",
        "route": "Tablet",
        "quantity": "2",
        "status": "Billed",
        "prescribedAtUtc": "2022-09-12T10:12:05Z",
        "prescribingClinician": "g.wycliffe"
      },
      {
        "prescriptionId": "179058",
        "visitId": "61106",
        "patientId": "145",
        "drugName": "DICLOFENAC 50MG",
        "dosage": "Tablet",
        "frequency": "3",
        "duration": "5 Days",
        "route": "Tablet",
        "quantity": "1",
        "status": "Dispensed",
        "prescribedAtUtc": "2022-09-12T10:12:05Z",
        "prescribingClinician": "g.wycliffe"
      },
      {
        "prescriptionId": "179059",
        "visitId": "61106",
        "patientId": "145",
        "drugName": "ARTEMETHER INJECTION 80MG INJECTION",
        "dosage": "ml",
        "frequency": "2",
        "duration": "5 Days",
        "route": "Injection",
        "quantity": "5",
        "status": "Dispensed",
        "prescribedAtUtc": "2022-09-12T10:12:05Z",
        "prescribingClinician": "g.wycliffe"
      }
    ],
    "procedures": [
      {
        "procedureName": "Apendix Surgery",
        "notes": "Procedure: na\nSurgeon: Gekonge\nAssistant Surgeon: Wycliffe\nAnaesthetist: Wycliffe\nScrub Nurse: Nyamwaya\nDoctor: Gekonge\nAnaesthesia Type: Abdomen and part of chest\nIncision: upper abdomen\nBiopsy Specimen: blood\nComplications: none\nEstimated Blood Loss: 0.8\nTheatre Count: Correct\nStatus: Completed",
        "performedAtUtc": "2022-09-12T09:56:20Z",
        "performedBy": "Gekonge"
      }
    ]
  },
  "generatedAtUtc": "2026-05-13T11:10:34.6428598Z",
  "correlationId": "4965f0bf8256469399bf3787646459c5"
}
```

**Field Notes:**
- `triage.pulseRate`, `respiratoryRate`, `oxygenSaturation` may be `null`
- `clinicalNotes.doctorNotes` may be `null`
- `clinicalNotes.recordedBy` may be `null`
- `diagnoses[].diagnosisType` and `notes` may be `null`
- `labTests[].testCategory` and `resultedAtUtc` may be `null`
- `labTests[].resultValue` may be an empty string when result is pending
- `medicines[].dosage` is the dosage form (e.g. `"Tablet"`, `"ml"`) — not the strength
- `medicines[].frequency` and `quantity` are string numbers (e.g. `"3"`, `"2"`)
- `radiologyExaminations` and `procedures` may be empty arrays

---

### Clinical Data

---

#### POST /clinical/lab-results

Retrieve laboratory test results for a patient or visit.

**Scope required:** `tmr:clinical:read`
**Content-Type:** `application/json`

**Request Body:**

```json
{
  "Patient": {
    "PatientId": "string",
    "PatientNumber": "string",
    "IDNumber": "string",
    "PassportNumber": "string",
    "PhoneNumber": "string",
    "EmailAddress": "string"
  },
  "VisitId": "string",
  "FromDate": "2026-01-01T00:00:00Z",
  "ToDate": "2026-05-13T00:00:00Z",
  "Pagination": {
    "Page": 1,
    "PageSize": 20
  }
}
```

Either `VisitId` or a populated `Patient` identifier must be provided.

**Example — by visit:**

```json
{
  "VisitId": "61106",
  "Pagination": { "Page": 1, "PageSize": 20 }
}
```

**Example — by patient:**

```json
{
  "Patient": {
    "PatientNumber": "DH1281119"
  },
  "Pagination": { "Page": 1, "PageSize": 20 }
}
```

**Example Response — 200 OK** *(item structure verified from live OPD composite)*

```json
{
  "success": true,
  "message": "Successful.",
  "errorCode": null,
  "data": {
    "items": [
      {
        "labTestId": "56425",
        "visitId": "61106",
        "patientId": "145",
        "testName": "BS for Malaria - Falciparum",
        "testCategory": null,
        "orderedAtUtc": "2022-09-12T06:55:41Z",
        "resultedAtUtc": null,
        "status": "Pending",
        "resultValue": "",
        "resultUnit": " ",
        "referenceRange": "Seen",
        "isAbnormal": false,
        "orderingClinician": "Wycliffe"
      },
      {
        "labTestId": "56426",
        "visitId": "61106",
        "patientId": "145",
        "testName": "Blood Sugar - Random blood sugar",
        "testCategory": null,
        "orderedAtUtc": "2022-09-12T06:55:41Z",
        "resultedAtUtc": null,
        "status": "Pending",
        "resultValue": "11",
        "resultUnit": "mmol/l",
        "referenceRange": "High",
        "isAbnormal": true,
        "orderingClinician": "Wycliffe"
      }
    ],
    "totalCount": 2,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  },
  "generatedAtUtc": "2026-05-13T09:00:00.0000000Z",
  "correlationId": "d2e3f4a5b6c789ab2d34e5f6a7b8c9d0"
}
```

**Field Notes:**
- `testCategory` may be `null`
- `resultedAtUtc` is `null` when result is still pending
- `resultValue` may be an empty string for pending tests
- `isAbnormal` is always a boolean

---

#### POST /clinical/radiology

Retrieve radiology examination records for a patient or visit.

**Scope required:** `tmr:clinical:read`
**Content-Type:** `application/json`

**Request Body** — same structure as `/clinical/lab-results`

**Example Request:**

```json
{
  "VisitId": "61106",
  "Pagination": { "Page": 1, "PageSize": 20 }
}
```

**Example Response — 200 OK:**

```json
{
  "success": true,
  "message": "Successful.",
  "errorCode": null,
  "data": {
    "items": [
      {
        "examinationId": "RAD-001",
        "visitId": "61106",
        "patientId": "145",
        "examinationType": "Chest X-Ray",
        "status": "Reported",
        "findings": "No active pulmonary lesion identified.",
        "requestedAtUtc": "2022-09-12T08:30:00Z",
        "reportedAtUtc": "2022-09-12T11:00:00Z",
        "requestingClinician": "Gekonge Wycliffe",
        "reportingRadiologist": "Dr. Mwenda"
      }
    ],
    "totalCount": 1,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  },
  "generatedAtUtc": "2026-05-13T09:00:00.0000000Z",
  "correlationId": "e3f4a5b6c7d890ab3e45f6a7b8c9d0e1"
}
```

---

#### POST /clinical/prescriptions

Retrieve prescription and dispensing records for a patient or visit.

**Scope required:** `tmr:clinical:read`
**Content-Type:** `application/json`

**Request Body** — same structure as `/clinical/lab-results`

**Example Request:**

```json
{
  "VisitId": "61106",
  "Pagination": { "Page": 1, "PageSize": 20 }
}
```

**Example Response — 200 OK** *(item structure verified from live OPD composite)*

```json
{
  "success": true,
  "message": "Successful.",
  "errorCode": null,
  "data": {
    "items": [
      {
        "prescriptionId": "179057",
        "visitId": "61106",
        "patientId": "145",
        "drugName": "PARACETAMOL 500MG",
        "dosage": "Tablet",
        "frequency": "3",
        "duration": "3 Days",
        "route": "Tablet",
        "quantity": "2",
        "status": "Billed",
        "prescribedAtUtc": "2022-09-12T10:12:05Z",
        "prescribingClinician": "g.wycliffe"
      },
      {
        "prescriptionId": "179058",
        "visitId": "61106",
        "patientId": "145",
        "drugName": "DICLOFENAC 50MG",
        "dosage": "Tablet",
        "frequency": "3",
        "duration": "5 Days",
        "route": "Tablet",
        "quantity": "1",
        "status": "Dispensed",
        "prescribedAtUtc": "2022-09-12T10:12:05Z",
        "prescribingClinician": "g.wycliffe"
      },
      {
        "prescriptionId": "179059",
        "visitId": "61106",
        "patientId": "145",
        "drugName": "ARTEMETHER INJECTION 80MG INJECTION",
        "dosage": "ml",
        "frequency": "2",
        "duration": "5 Days",
        "route": "Injection",
        "quantity": "5",
        "status": "Dispensed",
        "prescribedAtUtc": "2022-09-12T10:12:05Z",
        "prescribingClinician": "g.wycliffe"
      }
    ],
    "totalCount": 3,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  },
  "generatedAtUtc": "2026-05-13T09:00:00.0000000Z",
  "correlationId": "f4a5b6c7d8e901ab4f56a7b8c9d0e1f2"
}
```

**Field Notes:**
- `dosage` is the dosage form (`"Tablet"`, `"ml"`, `"Injection"`) — not the strength
- `frequency` and `quantity` are string numbers (`"3"`, `"2"`, `"5"`)
- `status` values observed: `"Billed"`, `"Dispensed"`

---

### IPD (Inpatient) Data

---

#### POST /ipd/visits

Retrieve a paginated list of inpatient admissions.

**Scope required:** `tmr:ipd:read`
**Content-Type:** `application/json`

**Request Body:**

```json
{
  "Patient": {
    "PatientId": "string",
    "PatientNumber": "string",
    "IDNumber": "string",
    "PassportNumber": "string",
    "PhoneNumber": "string",
    "EmailAddress": "string"
  },
  "FromDate": "2026-01-01T00:00:00Z",
  "ToDate": "2026-05-13T00:00:00Z",
  "Status": "string",
  "Pagination": {
    "Page": 1,
    "PageSize": 20
  }
}
```

All fields are optional.

**Example — all admissions:**

```json
{
  "Pagination": { "Page": 1, "PageSize": 20 }
}
```

**Example — active admissions only:**

```json
{
  "Status": "Active",
  "Pagination": { "Page": 1, "PageSize": 20 }
}
```

**Example Response — 200 OK** *(verified against live staging data)*

```json
{
  "success": true,
  "message": "Successful.",
  "errorCode": null,
  "data": {
    "items": [
      {
        "ipdVisitId": "62627",
        "visitNumber": "62627",
        "patientId": "587",
        "patientName": "Margret Nyangasi Malowa Dummy",
        "patientNumber": "DH5131219",
        "visitDate": "2026-02-24T12:48:42",
        "visitType": "Inpatient",
        "dischargeDateUtc": null,
        "ward": "Out Patient Department",
        "bedNumber": null,
        "admittingClinician": "Lugongo Abel",
        "admissionDiagnosis": "",
        "status": "Active",
        "paymentMode": "Cash Payers",
        "insuranceScheme": null,
        "branchName": "Main Branch"
      }
    ],
    "totalCount": 1,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  },
  "generatedAtUtc": "2026-05-13T19:27:32.9656335Z",
  "correlationId": "71d1cf7fa5984fe591ab7abbf6afb40c"
}
```

**Field Notes:**
- `visitType` is `"Inpatient"` — not `"IPD"`
- `dischargeDateUtc` is `null` for active admissions
- `bedNumber` may be `null` if not yet assigned
- `admissionDiagnosis` may be an empty string
- `status` values: `"Active"` for current admissions, `"Discharged"` for completed
- `insuranceScheme` is `null` for cash payers

---

#### GET /ipd/visits/{ipdVisitId}

Retrieve the record of a specific inpatient admission.

**Scope required:** `tmr:ipd:read`

**Path Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `ipdVisitId` | `string` | The unique IPD visit identifier |

**Example Request:**

```
GET /api/external/ipd/visits/62627
Authorization: Bearer {access_token}
```

**Example Response — 200 OK** — Same shape as a single item from `POST /ipd/visits`.

---

#### POST /ipd/visits/{ipdVisitId}/ward-rounds

Retrieve ward round records for an inpatient admission.

**Scope required:** `tmr:clinical:read`

**Path Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `ipdVisitId` | `string` | The unique IPD visit identifier |

**Request Body:**

```json
{
  "Page": 1,
  "PageSize": 20
}
```

**Example Response — 200 OK:**

```json
{
  "success": true,
  "message": "Successful.",
  "errorCode": null,
  "data": {
    "items": [
      {
        "wardRoundId": "WR-001",
        "ipdVisitId": "62627",
        "patientId": "587",
        "wardRoundDateUtc": "2026-02-25T08:00:00Z",
        "clinicalNotes": {
          "triage": {
            "bloodPressure": "118/76 mmHg",
            "temperature": "39.1 C",
            "pulseRate": null,
            "respiratoryRate": null,
            "oxygenSaturation": null,
            "weight": null,
            "height": null,
            "bmi": null,
            "additionalNotes": null
          },
          "doctorNotes": null,
          "recordedAtUtc": "2026-02-25T08:30:00Z",
          "recordedBy": "Lugongo Abel"
        },
        "diagnoses": [
          {
            "diagnosisCode": "A01.0",
            "diagnosisName": "Typhoid Fever",
            "diagnosisType": null,
            "notes": null
          }
        ],
        "labTests": [],
        "medicines": [
          {
            "prescriptionId": "180001",
            "visitId": "62627",
            "patientId": "587",
            "drugName": "CEFTRIAXONE 1G IV",
            "dosage": "ml",
            "frequency": "1",
            "duration": "5 Days",
            "route": "Injection",
            "quantity": "5",
            "status": "Administered",
            "prescribedAtUtc": "2026-02-25T09:00:00Z",
            "prescribingClinician": "Lugongo Abel"
          }
        ],
        "radiologyExaminations": [],
        "procedures": []
      }
    ],
    "totalCount": 1,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  },
  "generatedAtUtc": "2026-05-13T09:00:00.0000000Z",
  "correlationId": "b2c3d4e5f6a789bc0b1cd2e3f4a5b6c7"
}
```

---

#### GET /ipd/visits/{ipdVisitId}/discharge

Retrieve the discharge summary for an inpatient admission.

**Scope required:** `tmr:discharge:read`

**Path Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `ipdVisitId` | `string` | The unique IPD visit identifier |

**Example Request:**

```
GET /api/external/ipd/visits/62627/discharge
Authorization: Bearer {access_token}
```

**Example Response — 200 OK:**

```json
{
  "success": true,
  "message": "Successful.",
  "errorCode": null,
  "data": {
    "dischargeId": "DC-001",
    "ipdVisitId": "62627",
    "patientId": "587",
    "dischargeDateUtc": "2026-03-01T10:00:00Z",
    "dischargeDiagnosis": "Typhoid Fever — Resolved",
    "dischargeCondition": "Recovered",
    "dischargeNotes": "Patient afebrile for 48 hours. Tolerating oral intake well.",
    "returnDate": "2026-03-15",
    "dischargingClinician": "Lugongo Abel",
    "lengthOfStayDays": 5,
    "dischargeDrugs": [
      {
        "prescriptionId": "180010",
        "drugName": "CIPROFLOXACIN 500MG",
        "dosage": "Tablet",
        "frequency": "2",
        "duration": "7 Days",
        "route": "Tablet",
        "quantity": "14",
        "status": "Dispensed",
        "instructions": "Take with food. Complete full course."
      }
    ]
  },
  "generatedAtUtc": "2026-05-13T09:00:00.0000000Z",
  "correlationId": "c3d4e5f6a7b890cd1c2de3f4a5b6c7d8"
}
```

---

## Quick Reference

| Method | Endpoint | Scope | Key Request Fields |
|---|---|---|---|
| `POST` | `/auth/token` | — | `GrantType`, `ClientId`, `ClientSecret`, `Scope` |
| `POST` | `/patients/biodata` | `tmr:patients:read` | `Identifier.PatientNumber` etc. |
| `POST` | `/patients/history` | `tmr:history:read` | `Identifier`, `Pagination` |
| `POST` | `/visits` | `tmr:visits:read` | `Patient`, `Pagination` |
| `GET` | `/visits/{visitId}/opd` | `tmr:clinical:read` | — |
| `POST` | `/clinical/lab-results` | `tmr:clinical:read` | `VisitId` or `Patient`, `Pagination` |
| `POST` | `/clinical/radiology` | `tmr:clinical:read` | `VisitId` or `Patient`, `Pagination` |
| `POST` | `/clinical/prescriptions` | `tmr:clinical:read` | `VisitId` or `Patient`, `Pagination` |
| `POST` | `/ipd/visits` | `tmr:ipd:read` | `Patient`, `Pagination` |
| `GET` | `/ipd/visits/{ipdVisitId}` | `tmr:ipd:read` | — |
| `POST` | `/ipd/visits/{ipdVisitId}/ward-rounds` | `tmr:clinical:read` | `Page`, `PageSize` |
| `GET` | `/ipd/visits/{ipdVisitId}/discharge` | `tmr:discharge:read` | — |

*Hanmak Technologies Ltd. — TMR Integration API v1.0.0*
