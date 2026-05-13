# TMR Virtual Hospital Integration — API Documentation

**Hanmak Technologies Ltd.**
**Version:** 1.0.0
**Base URL:** `https://medicentre.hanmak.co.ke`

---

## Overview

The TMR Virtual Hospital API exposes a controlled, read-only subset of MedicentreV3's clinical and administrative data to the TMR Virtual Hospital Application. All endpoints are prefixed with `/api/external/` and require OAuth2 Bearer token authentication except the token issuance endpoint itself.

All request and response bodies use `application/json` unless otherwise stated. The token endpoint uses `application/x-www-form-urlencoded`.

---

## Base URL

```
https://medicentre.hanmak.co.ke/api/external
```

---

## Authentication

The API uses the OAuth2 Client Credentials flow. All data endpoints require a valid Bearer token obtained from the token endpoint.

### Token Lifetime

Access tokens are valid for **3600 seconds (1 hour)**. The TMR application must request a new token before expiry.

### Authorization Header

All data endpoints require:

```
Authorization: Bearer {access_token}
```

---

## Scopes

Access is controlled at the endpoint level using OAuth2 scopes. The TMR application must request the relevant scope when obtaining a token.

| Scope | Grants Access To |
|---|---|
| `tmr:patients:read` | Patient biodata and allergy records |
| `tmr:history:read` | Patient clinical history |
| `tmr:visits:read` | OPD visit records and visit lists |
| `tmr:clinical:read` | Laboratory results, radiology, prescriptions |
| `tmr:ipd:read` | Inpatient admissions and ward rounds |
| `tmr:discharge:read` | Discharge details and discharge drugs |

Multiple scopes may be requested in a single token by space-separating them in the `scope` field.

---

## Standard Response Envelope

All data endpoints return a consistent response wrapper:

```json
{
  "success": true,
  "message": "string",
  "correlationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": { }
}
```

| Field | Type | Description |
|---|---|---|
| `success` | `boolean` | `true` if the request was processed successfully |
| `message` | `string` | Human-readable description of the result |
| `correlationId` | `string` | Unique identifier for this response, for log correlation |
| `generatedAtUtc` | `datetime` | UTC timestamp of response generation |
| `errorCode` | `string\|null` | Machine-readable error code on failure; `null` on success |
| `data` | `object\|null` | The response payload; `null` on failure |

## Paginated Response

List endpoints return data wrapped in a paged result:

```json
{
  "items": [],
  "totalCount": 0,
  "page": 1,
  "pageSize": 20,
  "totalPages": 0,
  "hasNextPage": false,
  "hasPreviousPage": false
}
```

---

## Error Codes

| Code | Description |
|---|---|
| `TMR_AUTH_FAILED` | Authentication failure — invalid or missing token |
| `TMR_INVALID_TOKEN` | Token could not be parsed or has an invalid signature |
| `TMR_TOKEN_EXPIRED` | Token has expired |
| `TMR_INSUFFICIENT_SCOPE` | Token does not carry the required scope for this endpoint |
| `TMR_NOT_FOUND` | The requested resource was not found or is outside the permitted branch |
| `TMR_VALIDATION_FAILED` | Request body failed validation |
| `TMR_INTERNAL_ERROR` | Unexpected server error |

---

## Patient Identifier

Requests that target a specific patient accept a flexible identifier object. Supply whichever identifiers the TMR system holds for the patient. Resolution follows this precedence order — the first populated field wins:

```json
{
  "patientId": "string",       // 1. MedicentreV3 internal system ID (highest precedence)
  "patientNumber": "string",   // 2. OPD / registration number
  "idNumber": "string",        // 3. National ID card number
  "passportNumber": "string",  // 4. Passport number
  "phoneNumber": "string",     // 5. Primary phone number
  "emailAddress": "string"     // 6. Email address (lowest precedence)
}
```

At least one field must be populated.

---

## Endpoints

---

### Authentication

---

#### POST /auth/token

Obtain an OAuth2 access token using the Client Credentials grant. The issued token encodes the client's permitted scopes, branch, and instance identity for use on all subsequent data requests.

**Content-Type:** `application/x-www-form-urlencoded`

**Request Parameters**

| Parameter | Required | Description |
|---|---|---|
| `grant_type` | ✅ | Must be `client_credentials` |
| `client_id` | ✅ | The API Key issued via the MedicentreV3 Mini Apps panel |
| `client_secret` | ✅ | The client secret generated for this API Key |
| `scope` | ✅ | Space-separated list of requested scopes |

**Example Request**

```
POST /api/external/auth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=JPbP7iRrILrSW8uxHG9hXHmq...
&client_secret=ERopejGfWF6txAzCtmVFT6u9Pz7...
&scope=tmr:patients:read tmr:visits:read tmr:clinical:read
```

**Example Response — 200 OK**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "tmr:patients:read tmr:visits:read tmr:clinical:read"
}
```

**Error Responses**

| Status | Error | Cause |
|---|---|---|
| `401` | `invalid_client` | Unrecognised `client_id` or incorrect `client_secret` |
| `400` | `invalid_scope` | Requested scope is not permitted for this client |

---

### Patient Information

---

#### POST /patients/biodata

Retrieve full biographical data for a patient. The patient is resolved using the supplied identifier. At least one identifier field must be provided.

**Scope required:** `tmr:patients:read`

**Request Body**

```json
{
  "identifier": {
    "patientId": "string",
    "patientNumber": "string",
    "idNumber": "string",
    "passportNumber": "string",
    "phoneNumber": "string",
    "emailAddress": "string"
  }
}
```

**Example Request**

```json
{
  "identifier": {
    "patientNumber": "P-2024-0001"
  }
}
```

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "Patient record retrieved.",
  "correlationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "patientId": "SYS-001",
    "patientNumber": "P-2024-0001",
    "firstName": "John",
    "lastName": "Kamau",
    "dateOfBirth": "1985-03-14",
    "age": 39,
    "sex": "Male",
    "phoneNumber": "254712345678",
    "secondPhoneNumber": "254722000001",
    "emailAddress": "john.kamau@email.com",
    "address": "Westlands, Nairobi",
    "idNumber": "30123456",
    "passportNumber": null,
    "allergies": [
      {
        "allergen": "Penicillin",
        "reaction": "Rash and swelling"
      }
    ]
  }
}
```

---

#### POST /patients/history

Retrieve the clinical encounter history for a patient. Results are paginated.

**Scope required:** `tmr:history:read`

**Request Body**

```json
{
  "patientId": "string",
  "fromDate": "2025-01-01T00:00:00Z",
  "toDate": "2026-05-13T00:00:00Z",
  "encounterType": "Outpatient",
  "pagination": {
    "page": 1,
    "pageSize": 20
  }
}
```

| Field | Required | Description |
|---|---|---|
| `patientId` | ✅ | The MedicentreV3 internal patient ID |
| `fromDate` | — | Filter encounters from this UTC date |
| `toDate` | — | Filter encounters to this UTC date |
| `encounterType` | — | Optional filter: `Outpatient`, `Inpatient` |
| `pagination.page` | — | Page number, default `1` |
| `pagination.pageSize` | — | Results per page, default `20`, max `100` |

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "2 history entries retrieved.",
  "correlationId": "a3f9c1d2-e8b0-47ab-8c21-d3f4e5b6c7d8",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "items": [
      {
        "entryId": "HIST-001",
        "patientId": "SYS-001",
        "encounterDateUtc": "2026-04-12T08:00:00Z",
        "encounterType": "Outpatient",
        "attendingClinician": "Dr. Otieno",
        "department": "General OPD",
        "primaryDiagnosis": "Upper Respiratory Tract Infection",
        "chiefComplaint": "Cough and fever for 3 days",
        "branchName": "Main Branch"
      }
    ],
    "totalCount": 2,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

---

### Visit Data

---

#### POST /visits

Retrieve a paginated list of outpatient visits. Results can be filtered by patient identifier, date range, visit type, or status.

**Scope required:** `tmr:visits:read`

**Request Body**

```json
{
  "patient": {
    "patientId": "string",
    "patientNumber": "string",
    "idNumber": "string",
    "passportNumber": "string",
    "phoneNumber": "string",
    "emailAddress": "string"
  },
  "fromDate": "2026-01-01T00:00:00Z",
  "toDate": "2026-05-13T00:00:00Z",
  "visitType": "OPD",
  "status": "Closed",
  "pagination": {
    "page": 1,
    "pageSize": 20
  }
}
```

All fields are optional. An empty body returns all visits within the client's permitted branch.

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "2 visits retrieved.",
  "correlationId": "b7e2d4f1-c9a8-53ab-9d12-e4f5a6b7c8d9",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "items": [
      {
        "visitId": "V-001",
        "visitNumber": "OPD-2024-0891",
        "patientId": "SYS-001",
        "visitDate": "2026-05-07T08:00:00Z",
        "visitType": "OPD",
        "status": "Closed",
        "department": "General OPD",
        "attendingClinician": "Dr. Otieno",
        "branchName": "Main Branch",
        "paymentMode": "Insurance",
        "insuranceScheme": "AAR"
      }
    ],
    "totalCount": 2,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

---

#### GET /visits/{visitId}/opd

Retrieve the complete OPD visit detail composite for a specific visit. Returns patient biodata, visit header, triage vitals, clinical notes, diagnoses, laboratory results, radiology examinations, medicines, and procedures in a single response.

**Scope required:** `tmr:clinical:read`

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `visitId` | `string` | The unique visit identifier |

**Example Request**

```
GET /api/external/visits/V-001/opd
Authorization: Bearer {access_token}
```

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "OPD visit detail retrieved.",
  "correlationId": "c1f8a6b4-d3e7-92ab-1c23-f4a5b6c7d8e9",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "patientBiodata": {
      "patientId": "SYS-001",
      "patientNumber": "P-2024-0001",
      "firstName": "John",
      "lastName": "Kamau",
      "age": 39,
      "sex": "Male",
      "allergies": [
        { "allergen": "Penicillin", "reaction": "Rash and swelling" }
      ]
    },
    "visitSummary": {
      "visitId": "V-001",
      "visitNumber": "OPD-2024-0891",
      "visitDate": "2026-05-07T08:00:00Z",
      "visitType": "OPD",
      "status": "Closed",
      "department": "General OPD",
      "attendingClinician": "Dr. Otieno",
      "branchName": "Main Branch",
      "paymentMode": "Insurance"
    },
    "clinicalNotes": {
      "triage": {
        "bloodPressure": "130/85 mmHg",
        "temperature": "38.2 °C",
        "pulseRate": "92 bpm",
        "respiratoryRate": "18 breaths/min",
        "oxygenSaturation": "97%",
        "weight": "74 kg",
        "height": "175 cm",
        "bmi": "24.2"
      },
      "doctorNotes": "Patient presents with productive cough and fever for 3 days...",
      "recordedAtUtc": "2026-05-07T09:15:00Z",
      "recordedBy": "Dr. Otieno"
    },
    "diagnoses": [
      {
        "diagnosisCode": "J06.9",
        "diagnosisName": "Acute Upper Respiratory Infection",
        "diagnosisType": "Primary"
      }
    ],
    "labTests": [
      {
        "labTestId": "LAB-001",
        "testName": "Full Blood Count",
        "testCategory": "Haematology",
        "status": "Resulted",
        "resultValue": "WBC: 11.2 x10³/μL",
        "referenceRange": "4.0–11.0 x10³/μL",
        "isAbnormal": true,
        "orderedAtUtc": "2026-05-07T08:30:00Z",
        "resultedAtUtc": "2026-05-07T10:45:00Z",
        "orderingClinician": "Dr. Otieno"
      }
    ],
    "radiologyExaminations": [],
    "medicines": [
      {
        "prescriptionId": "RX-001",
        "drugName": "Amoxicillin 500mg",
        "dosage": "500mg",
        "frequency": "Three times daily",
        "duration": "7 days",
        "route": "Oral",
        "quantity": "21 capsules",
        "status": "Dispensed",
        "prescribedAtUtc": "2026-05-07T09:20:00Z",
        "prescribingClinician": "Dr. Otieno"
      }
    ],
    "procedures": []
  }
}
```

---

### Clinical Data

---

#### POST /clinical/lab-results

Retrieve laboratory test results for a patient or a specific visit. Results are paginated.

**Scope required:** `tmr:clinical:read`

**Request Body**

```json
{
  "visitId": "string",
  "patientId": "string",
  "fromDate": "2026-01-01T00:00:00Z",
  "toDate": "2026-05-13T00:00:00Z",
  "pagination": {
    "page": 1,
    "pageSize": 20
  }
}
```

Either `visitId` or `patientId` must be provided.

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "1 lab result(s) retrieved.",
  "correlationId": "d2e3f4a5-b6c7-89ab-2d34-e5f6a7b8c9d0",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "items": [
      {
        "labTestId": "LAB-001",
        "visitId": "V-001",
        "patientId": "SYS-001",
        "testName": "Full Blood Count",
        "testCategory": "Haematology",
        "status": "Resulted",
        "resultValue": "WBC: 11.2 x10³/μL",
        "referenceRange": "4.0–11.0 x10³/μL",
        "isAbnormal": true,
        "orderedAtUtc": "2026-05-07T08:30:00Z",
        "resultedAtUtc": "2026-05-07T10:45:00Z",
        "orderingClinician": "Dr. Otieno"
      }
    ],
    "totalCount": 1,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

---

#### POST /clinical/radiology

Retrieve radiology examination records for a patient or visit.

**Scope required:** `tmr:clinical:read`

**Request Body** — Same structure as `/clinical/lab-results`

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "1 radiology examination(s) retrieved.",
  "correlationId": "e3f4a5b6-c7d8-90ab-3e45-f6a7b8c9d0e1",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "items": [
      {
        "examinationId": "RAD-001",
        "visitId": "V-001",
        "patientId": "SYS-001",
        "examinationType": "Chest X-Ray",
        "status": "Reported",
        "findings": "No active pulmonary lesion identified.",
        "requestedAtUtc": "2026-05-07T08:30:00Z",
        "reportedAtUtc": "2026-05-07T11:00:00Z",
        "requestingClinician": "Dr. Otieno",
        "reportingRadiologist": "Dr. Mwenda"
      }
    ],
    "totalCount": 1,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

---

#### POST /clinical/prescriptions

Retrieve prescription and dispensing records for a patient or visit.

**Scope required:** `tmr:clinical:read`

**Request Body** — Same structure as `/clinical/lab-results`

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "2 prescription(s) retrieved.",
  "correlationId": "f4a5b6c7-d8e9-01ab-4f56-a7b8c9d0e1f2",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "items": [
      {
        "prescriptionId": "RX-001",
        "visitId": "V-001",
        "patientId": "SYS-001",
        "drugName": "Amoxicillin 500mg",
        "dosage": "500mg",
        "frequency": "Three times daily",
        "duration": "7 days",
        "route": "Oral",
        "quantity": "21 capsules",
        "status": "Dispensed",
        "prescribedAtUtc": "2026-05-07T09:20:00Z",
        "prescribingClinician": "Dr. Otieno"
      }
    ],
    "totalCount": 2,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

---

### IPD (Inpatient) Data

---

#### POST /ipd/visits

Retrieve a paginated list of inpatient admissions. Results can be filtered by patient, date range, or status.

**Scope required:** `tmr:ipd:read`

**Request Body**

```json
{
  "patient": {
    "patientId": "string",
    "patientNumber": "string",
    "idNumber": "string",
    "passportNumber": "string",
    "phoneNumber": "string",
    "emailAddress": "string"
  },
  "fromDate": "2026-01-01T00:00:00Z",
  "toDate": "2026-05-13T00:00:00Z",
  "status": "Discharged",
  "pagination": {
    "page": 1,
    "pageSize": 20
  }
}
```

All fields are optional.

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "1 IPD admission(s) retrieved.",
  "correlationId": "a1b2c3d4-e5f6-78ab-9a0b-c1d2e3f4a5b6",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "items": [
      {
        "ipdVisitId": "IPD-001",
        "visitNumber": "IPD-2024-0042",
        "patientId": "SYS-001",
        "patientName": "John Kamau",
        "patientNumber": "P-2024-0001",
        "visitDate": "2026-05-02T08:00:00Z",
        "dischargeDateUtc": "2026-05-06T10:00:00Z",
        "ward": "General Ward",
        "bedNumber": "G-12",
        "admittingClinician": "Dr. Mwangi",
        "admissionDiagnosis": "Typhoid Fever",
        "status": "Discharged",
        "paymentMode": "Cash",
        "branchName": "Main Branch"
      }
    ],
    "totalCount": 1,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

---

#### GET /ipd/visits/{ipdVisitId}

Retrieve the header record of a specific inpatient admission.

**Scope required:** `tmr:ipd:read`

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `ipdVisitId` | `string` | The unique IPD visit identifier |

**Example Request**

```
GET /api/external/ipd/visits/IPD-001
Authorization: Bearer {access_token}
```

**Example Response — 200 OK** — Same structure as a single item from `POST /ipd/visits`.

---

#### POST /ipd/visits/{ipdVisitId}/ward-rounds

Retrieve the ward round records for an inpatient admission. Each ward round includes a full clinical composite — triage observations, doctor notes, diagnoses, lab results, radiology, and medicines as at the time of that round.

**Scope required:** `tmr:clinical:read`

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `ipdVisitId` | `string` | The unique IPD visit identifier |

**Request Body**

```json
{
  "page": 1,
  "pageSize": 20
}
```

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "1 ward round(s) retrieved.",
  "correlationId": "b2c3d4e5-f6a7-89bc-0b1c-d2e3f4a5b6c7",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "items": [
      {
        "wardRoundId": "WR-001",
        "ipdVisitId": "IPD-001",
        "patientId": "SYS-001",
        "wardRoundDateUtc": "2026-05-03T08:00:00Z",
        "patientBiodata": {
          "patientId": "SYS-001",
          "firstName": "John",
          "lastName": "Kamau",
          "age": 39,
          "sex": "Male"
        },
        "clinicalNotes": {
          "triage": {
            "bloodPressure": "118/76 mmHg",
            "temperature": "39.1 °C",
            "pulseRate": "98 bpm",
            "oxygenSaturation": "96%"
          },
          "doctorNotes": "Day 1 review. Patient febrile, responding to IV ceftriaxone...",
          "recordedAtUtc": "2026-05-03T08:30:00Z",
          "recordedBy": "Dr. Mwangi"
        },
        "diagnoses": [
          {
            "diagnosisCode": "A01.0",
            "diagnosisName": "Typhoid Fever",
            "diagnosisType": "Primary"
          }
        ],
        "labTests": [
          {
            "testName": "Widal Test",
            "testCategory": "Serology",
            "status": "Resulted",
            "resultValue": "S. Typhi O: 1/160",
            "isAbnormal": true
          }
        ],
        "medicines": [
          {
            "drugName": "Ceftriaxone 1g IV",
            "dosage": "1g",
            "frequency": "Once daily",
            "duration": "5 days",
            "route": "IV",
            "status": "Administered"
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
  }
}
```

---

#### GET /ipd/visits/{ipdVisitId}/discharge

Retrieve the discharge summary for an inpatient admission, including discharge notes, discharge drugs, and the recommended return date.

**Scope required:** `tmr:discharge:read`

**Path Parameters**

| Parameter | Type | Description |
|---|---|---|
| `ipdVisitId` | `string` | The unique IPD visit identifier |

**Example Request**

```
GET /api/external/ipd/visits/IPD-001/discharge
Authorization: Bearer {access_token}
```

**Example Response — 200 OK**

```json
{
  "success": true,
  "message": "Discharge details retrieved.",
  "correlationId": "c3d4e5f6-a7b8-90cd-1c2d-e3f4a5b6c7d8",
  "generatedAtUtc": "2026-05-13T09:00:00Z",
  "errorCode": null,
  "data": {
    "dischargeId": "DC-001",
    "ipdVisitId": "IPD-001",
    "patientId": "SYS-001",
    "dischargeDateUtc": "2026-05-06T10:00:00Z",
    "dischargeDiagnosis": "Typhoid Fever — Resolved",
    "dischargeCondition": "Recovered",
    "dischargeNotes": "Patient afebrile for 48 hours. Tolerating oral intake well...",
    "returnDate": "2026-05-19",
    "dischargingClinician": "Dr. Mwangi",
    "lengthOfStayDays": 4,
    "dischargeDrugs": [
      {
        "drugName": "Ciprofloxacin 500mg",
        "dosage": "500mg",
        "frequency": "Twice daily",
        "duration": "7 days",
        "route": "Oral",
        "quantity": "14 tablets",
        "instructions": "Take with food. Complete full course."
      },
      {
        "drugName": "Oral Rehydration Salts",
        "dosage": "1 sachet",
        "frequency": "After each loose stool",
        "route": "Oral",
        "instructions": "Dissolve in 1 litre of clean water."
      }
    ]
  }
}
```

---

## Quick Reference

| Method | Endpoint | Scope | Description |
|---|---|---|---|
| `POST` | `/auth/token` | — | Obtain access token |
| `POST` | `/patients/biodata` | `tmr:patients:read` | Patient biographical data |
| `POST` | `/patients/history` | `tmr:history:read` | Patient encounter history |
| `POST` | `/visits` | `tmr:visits:read` | List OPD visits |
| `GET` | `/visits/{visitId}/opd` | `tmr:clinical:read` | Full OPD visit composite |
| `POST` | `/clinical/lab-results` | `tmr:clinical:read` | Laboratory results |
| `POST` | `/clinical/radiology` | `tmr:clinical:read` | Radiology examinations |
| `POST` | `/clinical/prescriptions` | `tmr:clinical:read` | Prescriptions |
| `POST` | `/ipd/visits` | `tmr:ipd:read` | List IPD admissions |
| `GET` | `/ipd/visits/{ipdVisitId}` | `tmr:ipd:read` | Single IPD admission |
| `POST` | `/ipd/visits/{ipdVisitId}/ward-rounds` | `tmr:clinical:read` | Ward rounds |
| `GET` | `/ipd/visits/{ipdVisitId}/discharge` | `tmr:discharge:read` | Discharge summary |

---

*Hanmak Technologies Ltd. — TMR Integration API v1.0.0*
