# Midato Consent API
This document defines the API for access to Midato Consent Management System. 
(in-progress draft).

## API 
### Base
TBD

### Authorization
TBD

### Queries
#### Whether Active and Valid Consent Exists for a Patient
Check whether an active consent of a certain type (specified by `formId`) exists for a given patient (specified by the client organization's patient identifier in the form of organization's URL followed by the vertical slash followed by the patient identifier, e.g., `organizationURL|patientId`):

```
GET [base]/Consent/$status?patientIdentifier={organizationURL}|{patientId}&form={formId}
```

Response: 

 - HTTP `200` if the consent exists.
 - HTTP `404` if the consent does not exist.

#### Whether a Consent is Active and Valid by Consent ID
Check whether a consent (identified by `consentId`) is valid and active.
```
GET [base]/Consent/{consentId}/$valid
```
Response: 

 - HTTP `200` if the consent exists.
 - HTTP `404` if the consent does not exist.

#### Retrieve Consent as JSON object
```
GET [base]/Consent/{consentId}?_format=json
```

Response: FHIR Consent object.


#### Export Consent as PDF
```
GET [base]/Consent/{consentId}?_format=pdf
```

Response: PDF binary blob.

### Updates
#### Revoke a Consent
```
POST [base]/Consent/{consentId}/$revoke
```

#### Re-enact a Consent
```
POST [base]/Consent/{consentId}/$reenact
```

### Consent Object Sample
```json
{
  "resourceType": "Consent",
  "id": "{consentId}",
  "meta": {
    "lastUpdated": "2021-09-10T06:28:55.988+00:00",
  },
  "status": "active",
  "category": [ {
    "coding": [ {
      "system": "https://www.midatohealth.com/consent-types",
      "code": "{formNumber}",
      "display": "{formName}"
    } ]
  }],
  "patient": {
    "reference": "Patient/{patientId}",
    "display": "Doe, John"
  },
  "dateTime": "2021-09-08T07:49:44+00:00",
  "organization": [ {
    "display": "Sample E. Health"
  } ],
  "provision": {
    "period": {
      "start": "2021-09-10T04:52:11+00:00",
      "end": "2022-09-10T04:52:11+00:00"
    }
  }
}
```

### Consent Object with Contained Patient Sample
```json
{
  "resourceType": "Consent",
  "id": "{consentId}",
  "meta": {
    "lastUpdated": "2021-09-10T06:28:55.988+00:00",
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
          "given": [
            "Doe"
          ]
        }
      ],
      "gender": "male",
      "birthDate": "1986-05-17",
      "address": [
        {
          "use": "home",
          "line": [
            "1234 Main Street"
          ],
          "city": "Vancouver",
          "postalCode": "V2H1Y3",
          "country": "CAD"
        }
      ]
    },
  ],
  "status": "active",
  "category": [ {
    "coding": [ {
      "system": "https://www.midatohealth.com/consent-types",
      "code": "{formNumber}",
      "display": "{formName}"
    } ]
  }],
  "patient": {
    "reference": "#thePatient"
  },
  "dateTime": "2021-09-08T07:49:44+00:00",
  "organization": [ {
    "display": "Sample E. Health"
  } ],
  "provision": {
    "period": {
      "start": "2021-09-10T04:52:11+00:00",
      "end": "2022-09-10T04:52:11+00:00"
    }
  }
}
```