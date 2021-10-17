# Midato Consent API

This document defines the API for access to Midato Consent Management System. The API is broadly based on and compliant with [HL7 FHIR](https://www.hl7.org/fhir) with custom operations defined to facilitate consent management by providing short-hands that fit common use-cases.
(This document is an in-progress draft).

## Base

TBD

## Authorization

TBD

## Basic Concepts

### Patient Identification
Midato Health Consent API enables clients to identify patients using their own local identifier, so that clients do not have to store and track Midato's native IDs for each patient on their end. This makes the integration of clients' local workflows with the Consent API more straightforward.

Following FHIR data structures, when the patient identifier is given as a full JSON object (such as parameters in the body of a `POST` call), it should be provided as the following:
```json
{
  "system": "{organizationURL}",
  "value": "{patientId}"
}
```

When provided as a path parameter value, a fully-qualified client-side patient identifier is constructed by client organization's URL, followed by the vertical slash (`|`), followed by the patient identifier; i.e., `{organizationURL}|{patientId}`. 

For example, a patient identified by the ID `123456` at a hypothetical organization with the URL `https://sample-ehealth.com` will be identified as the following in Midato Consent API calls:

As full JSON object:
```json
{
  "system": "https://sample-ehealth.com/",
  "value": "123456"
}
```
As a path parameter value:
```
https://sample-ehealth.com|123456
```


## Queries

### Check Consent Status for a Patient

Check the status of the consent of a certain type (specified by `formId`) for a given patient, identified by providing a [fully qualified client-side identifier](#patient-identification) as a path parameter value:

```
GET [base]/Consent/$status?patientIdentifier={organizationURL}|{patientId}&category={formId}
```

The response returns the status of the _latest_ record for the given type of consent for the specified patient. So, if there are multiple records corresponding to this patient and this type of consent, the status of the most recent consent record is returned. If no such consent exists the response will be an `HTTP 404`.


The response is a FHIR Parameter resource similar to the following:

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
GET [base]/Consent/{consentId}/$status
```

The response is similar to the [above](#check-consent-status-for-a-patient).

### Retrieve Consent as JSON object

Retrieve the full Consent object.

```
GET [base]/Consent/{consentId}
```

The response is a [FHIR Consent object](#fhir-consent-object-samples) if the consent exists, or `HTTP 404` if it does not exist.

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

The response is a human-readable PDF binary blob if the consent exists, or `HTTP 404` if it does not exist.

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

## Consent Management Transactions

### Initiate Consent Capture for an Existing Patient
This call initiates capturing a consent for a patient. The consent type is specified by the `formId` and the patient is specified by a [fully qualified client-side identifier](#patient-identification) as a JSON object parameter value.

```
POST [base]/Consent/$capture
```
The body of the request is a FHIR Parameter object similar to the following:
```json
{
  "resourceType" : "Parameters",
  "parameter" : [
    {
      "name" : "patientIdentifier",
      "valueIdentifier" : {
        "system": "{organizationURL}",
        "value": "{patientId}"
      }
    },
    {
      "name" : "consentType",
      "valueString" : "{formId}"
    }
  ]
}
```

The response returns a [FHIR Consent objects](#fhir-consent-object-samples) in `draft` status. The client may store the `consentId` from this object in order to check the status of the consent or retrieve it later.

The response returns an `HTTP 400` if the patient is not recognizable or the form identifier is unknown. 

### Initiate Consent Capture for a New Patient
Similar to the above, this call also initiates capturing a consent but for a patient who is not currently known and is introduced by the request. In other words, this call creates a new patient and immediately initiates capturing a consent for this patient.

The patient should be provided as a [FHIR Patient object](#fhir-patient-object-samples) and must include: 
- the `identifier` attribute including the client-side patient identifier.
- the `name` attribute including the patient's full name. 
- the `telecom` attributes with at least one communication method (phone number or email address) to be used by the consent capturing process. If more than one `telecom` option is provided, the `rank` attribute should be provided to specify the order of preference.
 
The consent type is specified by the `formId`.

```
POST [base]/Consent/$capture
```
The body of the request is a FHIR Parameter object similar to the following:
```json
{
  "resourceType" : "Parameters",
  "parameter" : [
    {
      "name" : "patient",
      "valuePatient" : {
        "resourceType": "Patient",
        "identifier": [
          {
            "system": "https://sample-ehealth.com/",
            "value": "123456"
          }
        ],
        "name": [
          {
            "family": "Doe",
            "given": ["John"]
          }
        ],
        "gender": "male",
        "birthDate": "1986-05-17",
        "telecom": [
          {
            "system": "phone",
            "value": "(03) 5555 6473",
            "use": "mobile",
            "rank": 1
          },
          {
            "system": "email",
            "value": "john@doe.org",
            "rank": 2
          }
        ]
      }
  },
  {
    "name" : "consentType",
    "valueString" : "{formId}"
  }]
}
```

The response returns a [FHIR Consent objects](#fhir-consent-object-samples) in `draft` status. The client may store the `consentId` from this object in order to check the status of the consent or retrieve it later.

If the patient with the client-side fully-qualified identifier already exists, this call will update the communication details and initiates the consent capture.

The response returns an `HTTP 400` if the form identifier is unknown. 

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
          "family": "Doe",
          "given": ["John"]
        }
      ],
      "gender": "male",
      "birthDate": "1986-05-17",
      "telecom": [
          {
            "system": "phone",
            "value": "(03) 5555 6473",
            "use": "mobile"
          },
          {
            "system": "email",
            "value": "john@doe.org"
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

### FHIR Patient Object Samples
```json
{
  "resourceType": "Patient",
  "identifier": [
    {
      "system": "https://sample-ehealth.com/",
      "value": "123456"
    }
  ],
  "name": [
    {
      "family": "Doe",
      "given": ["John"]
    }
  ],
  "gender": "male",
  "birthDate": "1986-05-17",
  "telecom": [
    {
      "system": "phone",
      "value": "(03) 5555 6473",
      "use": "mobile",
      "rank": 1
    },
    {
      "system": "email",
      "value": "john@doe.org",
      "rank": 2
    }
  ]
}
```
