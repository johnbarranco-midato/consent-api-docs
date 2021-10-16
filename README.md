# Midato Consent API

This document defines the API for access to Midato Consent Management System. The API is broadly based on and compliant with [HL7 FHIR](https://www.hl7.org/fhir) with custom operations defined to facilitate consent management by providing short-hands that fit common use-cases.
(This document is an in-progress draft).

## Base

TBD

## Authorization

TBD

## Queries

### Check Consent Status for a Patient

Check the status of the consent of a certain type (specified by `formId`) for a given patient (specified by the client organization's patient identifier in the form of organization's URL followed by the vertical slash followed by the patient identifier, e.g., `organizationURL|patientId`):

```
GET [base]/Consent/$status?patientIdentifier={organizationURL}|{patientId}&category={formId}
```

The response returns the status of the _latest_ record for the given type of consent for the specified patient. So, if there are multiple records corresponding to this patient and this type of consent, the status of the most recent consent record is returned. If no such consent exists the response will be an `HTTP 404`.


Response Examples:

```json
{
  "resourceType" : "Parameters",
  "parameter" : [
    {
      "name" : "status",
      "valueString" : "{draft|rejected|active|inactive|expired}"
    }
  ]
}
```

The meaning of the status values are as the following:
|status|meaning|
|-|-|
|draft|there is a draft consent that is pending and is not yet accepted by the patient.|
|rejected|the patient has rejected a proposed draft consent.|
|active|there is an active and valid consent.|
|inactive|the consent has been revoked.|
|expired|the consent has expired.|

### Check Consent status by Consent ID

Check the status of a consent (identified by `consentId`). 

```
GET [base]/Consent/{consentId}/$valid
```

The response is similar to the [above](#check-consent-status-for-a-patient).

### Retrieve Consent as JSON object

Retrieve the full Consent object.

```
GET [base]/Consent/{consentId}
```

The response is a [FHIR Consent object](#fhir-consent-object-samples) if it exists and `HTTP 404` is it does not exist.

### Retrieve All Consent for a Patient
Retrieve all Consent objects corresponding to a patient. The request can optionally specify a consent type (by providing a `formId`):

```
GET [base]/Consent?patientIdentifier={organizationURL}|{patientId}&category={formId}
```

The response is a [FHIR Bundle object](#fhir-bundle-object-samples) containing zero or more [FHIR Consent objects](#fhir-consent-object-samples).

### Export Consent as PDF

```
GET [base]/Consent/{consentId}?_format=pdf
```

Response: PDF binary blob.

## Updates

### Revoke a Consent
Revokes a consent specified by `consentId`.
```
POST [base]/Consent/{consentId}/$revoke
```

The response is: 
- `HTTP 200` if there is an active consent identified by `consentId`. The status of the consent  is set to `inactive`.
- `HTTP 400` if there is a consent identified by `consentId` but its status is not `active`.
- `HTTP 404` if there is no consent identified by `consentId`.

### Re-enact a Consent
Re-enacts a previously revoked consent specified by `consentId`.
```
POST [base]/Consent/{consentId}/$reenact
```

The response is: 
- `HTTP 200` if there is an inactive consent identified by `consentId`. The status of the consent  is set to `active`.
- `HTTP 400` if there is a consent identified by `consentId` but its status is not `inactive`.
- `HTTP 404` if there is no consent identified by `consentId`.

## Data Objects
### FHIR Consent Object Samples

```json
{
  "resourceType": "Consent",
  "id": "{consentId}",
  "meta": {
    "lastUpdated": "2021-09-10T06:28:55.988+00:00"
  },
  "status": "active",
  "category": [
    {
      "coding": [
        {
          "system": "https://www.midatohealth.com/consent-types",
          "code": "{formNumber}",
          "display": "{formName}"
        }
      ]
    }
  ],
  "patient": {
    "reference": "Patient/{patientId}",
    "display": "Doe, John"
  },
  "dateTime": "2021-09-08T07:49:44+00:00",
  "organization": [
    {
      "display": "Sample E. Health"
    }
  ],
  "provision": {
    "period": {
      "start": "2021-09-10T04:52:11+00:00",
      "end": "2022-09-10T04:52:11+00:00"
    }
  }
}
```

FHIR Consent Object Sample with Contained Patient:

```json
{
  "resourceType": "Consent",
  "id": "{consentId}",
  "meta": {
    "lastUpdated": "2021-09-10T06:28:55.988+00:00"
  },
  "contained": [
    {
      "resourceType": "Patient",
      "id": "thePatient",
      "identifier": [
        {
          "system": "https://sample-ehealth.com/",
          "value": "123456"
        }
      ],
      "name": [
        {
          "use": "official",
          "family": "John",
          "given": ["Doe"]
        }
      ],
      "gender": "male",
      "birthDate": "1986-05-17",
      "address": [
        {
          "use": "home",
          "line": ["1234 Main Street"],
          "city": "Vancouver",
          "postalCode": "V2H1Y3",
          "country": "CAD"
        }
      ]
    }
  ],
  "status": "active",
  "category": [
    {
      "coding": [
        {
          "system": "https://www.midatohealth.com/consent-types",
          "code": "{formNumber}",
          "display": "{formName}"
        }
      ]
    }
  ],
  "patient": {
    "reference": "#thePatient"
  },
  "dateTime": "2021-09-08T07:49:44+00:00",
  "organization": [
    {
      "display": "Sample E. Health"
    }
  ],
  "provision": {
    "period": {
      "start": "2021-09-10T04:52:11+00:00",
      "end": "2022-09-10T04:52:11+00:00"
    }
  }
}
```

### FHIR Bundle Object Samples
Bundle with a single Consent object:
```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 1,
  "entry": [
    {
      "fullUrl": "{TBD Midato Health API base URL}/Consent/{consentId}",
      "resource": {
        "resourceType": "Consent",
        //...
      }
    }
  ]
}
```

Empty bundle:
```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 0
}
```