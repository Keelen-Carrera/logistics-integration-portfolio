# 🔄 Logistics Integration Portfolio

> **A collection of integration architecture patterns and data transformation examples inspired by real-world enterprise logistics system design.**
>
> All schemas, field names, endpoints, credentials, and data structures are **fully fictional and generic**. No proprietary platform schemas, internal APIs, or company-specific configurations are referenced. These projects exist to demonstrate architectural thinking, DataWeave transformation skills, and integration engineering patterns applied to common logistics and supply chain challenges.

---

## 👤 About

I'm a Software Integration Engineer with experience designing and building enterprise data pipelines across Transportation Management Systems (TMS), Freight ERP platforms, CRM tools, EDI trading partners, and warehouse systems. My work sits at the intersection of **systems integration**, **data transformation**, and **secure API design** — with a focus on logistics and supply chain operations.

**Core stack:** MuleSoft Anypoint Platform · DataWeave 2.0 · RESTful APIs · EDI X12 · SOQL · XML / JSON / CSV · CloudHub · GitHub Actions (CI/CD)

## Overview

This project simulates an enterprise logistics data integration platform designed to automate data exchange between transportation management systems.

It demonstrates real-world patterns such as:
- API-based system integration
- Data transformation pipelines
- Workflow automation
- Queue-based processing (if applicable)

---

## 📁 Project Index

| # | Project | Integration Pattern | Formats |
|---|---------|-------------------|---------|
| 1 | [TMS → Freight ERP: Account Sync](#1-tms--freight-erp-account-sync) | Scheduled poll → XML POST | JSON → XML |
| 2 | [TMS → Freight ERP: Load to Shipment](#2-tms--freight-erp-load-to-shipment) | Event-driven load creation | JSON → XML |
| 3 | [Shipment Status Update Flows](#3-shipment-status-update-flows) | Real-time event push (4 event types) | JSON → XML |
| 4 | [Freight ERP → CRM Sync](#4-freight-erp--crm-sync) | Scheduled org-to-company sync | XML → JSON |
| 5 | [EDI X12 214 Status Transmission](#5-edi-x12-214-status-transmission) | Freight status → EDI trading partner | XML → X12 |
| 6 | [SFTP File Intake → ERP Import](#6-sftp-file-intake--erp-import) | File listener → transform → API POST | JSON → XML |
| 7 | [Error Handling & Alerting Pattern](#7-error-handling--alerting-pattern) | Reusable error notification module | JSON |
| 8 | [CRM ↔ ERP Org Matching Utility](#8-crm--erp-org-matching-utility) | Bidirectional data reconciliation | CSV / JSON |
| 9 | [CI/CD: Mule App Deploy Pipeline](#9-cicd-mule-app-deploy-pipeline) | GitHub Actions → CloudHub deployment | YAML |
| 10 | [Integration Health Monitor](#10-integration-health-monitor) | Scheduled health checks + GitHub Issue alerts | YAML / JSON |

---

## 1. TMS → Freight ERP: Account Sync

### Overview

When a new carrier, customer, or vendor account is created in a Transportation Management System (TMS), it needs to be reflected in the freight ERP as an **Organization record**. This integration polls the TMS on a schedule, maps the account structure to a generic ERP XML schema, and POSTs it via the ERP's HTTP API. After a successful creation, the ERP's response is parsed and a reference key is written back to the TMS to link the two records bidirectionally.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   FLOW: account-to-org                          │
│                                                                 │
│  [Scheduler]                                                    │
│      │                                                          │
│      ▼                                                          │
│  [HTTP GET / Query] ──► Fetch new accounts since last_run       │
│      │                  (watermark stored in Object Store)      │
│      ▼                                                          │
│  [For Each Account]                                             │
│      │                                                          │
│      ├──► [Transform: DWL] ──► Map account fields → ERP XML    │
│      │                                                          │
│      ├──► [HTTP POST] ──────► ERP organization endpoint        │
│      │                                                          │
│      ├──► [Transform: DWL] ──► Parse ERP response              │
│      │                                                          │
│      └──► [HTTP PATCH] ────► Write ERP reference back to TMS   │
│                                                                 │
│  [Global Error Handler] ──► SMTP alert on failure              │
└─────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

- **Watermarking:** The query filters by `createdAt >= last_run_timestamp` using a value stored in the Mule Object Store, ensuring only net-new accounts are processed on each run.
- **Idempotency guard:** An `erpReferenceId` field check prevents re-processing accounts that already have an ERP reference.
- **Response writeback:** After a successful ERP POST, the generated Organization code is extracted from the response and written back to the TMS — linking the two systems bidirectionally.

### DataWeave: Account → ERP Organization XML

**Input (dummy TMS account data):**
```json
{
  "id": "ACC-001",
  "name": "Acme Freight Co.",
  "billingAddress": {
    "street": "123 Logistics Blvd",
    "city": "Houston",
    "state": "TX",
    "postalCode": "77001",
    "country": "US"
  },
  "phone": "555-000-1234",
  "accountType": "Carrier",
  "transportProfile": {
    "mcNumber": "MC-000000",
    "taxId": null
  },
  "contacts": [
    {
      "id": "CON-001",
      "name": "Jane Doe",
      "email": "jane.doe@example.com",
      "phone": "555-000-5678",
      "title": "Operations Manager"
    }
  ]
}
```

**DataWeave Transformation (`map-account-to-org.dwl`):**
```dataweave
%dw 2.0
output application/xml writeDeclaration=false
---
FreightOrganization: {
  SystemContext: {
    CompanyCode:  p('erp.company.code'),
    SubmittedBy:  p('erp.integration.user')
  },
  OrgHeader: {
    FullName:  payload.name,
    Phone:     payload.phone default null,
    IsActive:  "true",
    OrgType: (payload.accountType match {
      case "Carrier"  -> "CAR"
      case "Customer" -> "CUS"
      case "Vendor"   -> "VEN"
      else            -> "OTH"
    })
  },
  Address: {
    Street:   payload.billingAddress.street    default null,
    City:     payload.billingAddress.city      default null,
    State:    payload.billingAddress.state     default null,
    PostCode: payload.billingAddress.postalCode default null,
    Country:  payload.billingAddress.country   default null
  },
  (if (payload.transportProfile.mcNumber != null)
    { TransportProfile: { MCNumber: payload.transportProfile.mcNumber } }
  else {}),
  ContactCollection: {
    (payload.contacts map (contact) -> {
      Contact: {
        Name:  contact.name,
        Email: contact.email default null,
        Phone: contact.phone default null,
        Role:  contact.title default null
      }
    })
  }
}
```

**DataWeave: Parse ERP Response & Update TMS (`update-account-ref.dwl`):**
```dataweave
%dw 2.0
output application/json skipNullOn="everywhere"
import * from dw::core::Strings
// ERP processing log uses pipe-delimited entries — split and filter for errors
var logLines = (payload.processingLog substringBy $ == "|")
---
{
  "id":         vars.sourceAccountId,
  "erpOrgCode": payload.createdOrgCode default null,
  "syncStatus": if (isEmpty(payload.createdOrgCode)) "FAILED" else "SYNCED",
  "errors": (
    (logLines filter ((line) -> line contains("Error"))) default []
  ) map ((line) ->
    "Account [" ++ vars.sourceAccountName ++ "]: " ++ (line withMaxSize 500)
  )
}
```

---

## 2. TMS → Freight ERP: Load to Shipment

### Overview

When a load (a domestic freight movement with origin, destination, carrier, and cargo details) is finalized in the TMS, this integration creates a corresponding **Shipment record** in the freight ERP — including stop details, cargo line items, carrier financials, and port/location codes.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  FLOW: load-to-shipment                         │
│                                                                 │
│  [Scheduler]                                                    │
│      │                                                          │
│      ▼                                                          │
│  [HTTP GET / Query] ──► Fetch unprocessed loads                 │
│      │                  (status not in: Unassigned, Cancelled)  │
│      ▼                                                          │
│  [For Each Load]                                                │
│      │                                                          │
│      ├──► [Transform: DWL] ──► Map load → ERP Shipment XML     │
│      │       • First stop  → pickup address + location code    │
│      │       • Last stop   → delivery address + location code  │
│      │       • Line items  → cargo dimensions / weight         │
│      │       • Financials  → carrier and customer charge codes │
│      │                                                          │
│      ├──► [HTTP POST] ──────► ERP shipment endpoint            │
│      │                                                          │
│      ├──► [Transform: DWL] ──► Extract shipment + consol keys  │
│      │                                                          │
│      └──► [HTTP PATCH] ────► Write shipment # + consol # back  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### DataWeave: Load → ERP Shipment XML

**Input (dummy load data):**
```json
{
  "id": "LOAD-2024-00042",
  "status": "Dispatched",
  "firstStop": {
    "expectedDate": "2024-06-15",
    "shippingAppointmentTime": "08:00-12:00",
    "shippingHours": null,
    "location": {
      "name": "Acme Origin Warehouse",
      "street": "100 Origin Way",
      "city": "Dallas",
      "state": "TX",
      "country": "US",
      "postalCode": "75201"
    }
  },
  "lastStop": {
    "expectedDate": "2024-06-17",
    "receivingAppointmentTime": null,
    "receivingHours": "09:00-17:00",
    "location": {
      "name": "Destination DC",
      "city": "Atlanta",
      "state": "GA",
      "country": "US"
    }
  },
  "totalWeightLbs": 4500,
  "totalVolumeCft": 120,
  "carrier": { "name": "FastHaul Trucking", "code": "FHTRUCK" },
  "lineItems": [
    {
      "description": "Industrial Machinery Parts",
      "weightLbs": 4500,
      "handlingUnitCount": 6,
      "handlingUnitType": "PLT",
      "lengthIn": 48,
      "widthIn": 40,
      "heightIn": 36,
      "freightClass": "85"
    }
  ],
  "carrierCharges": [
    { "chargeType": "LineHaul",      "quantity": 1, "unitPrice": 1850.00 },
    { "chargeType": "FuelSurcharge", "quantity": 1, "unitPrice":  210.00 }
  ],
  "createdBy": { "employeeId": "EMP-042", "name": "Alex Rivera" }
}
```

**DataWeave (`map-load-to-shipment.dwl`):**
```dataweave
%dw 2.0
output application/xml writeDeclaration=false
var weightUnit = "LB"
var volUnit    = "CF"
---
FreightShipment: {
  ShipmentReference: payload.id,
  ShipmentType: "DOM",
  ShipmentDetails: {
    TotalWeight: payload.totalWeightLbs,
    WeightUnit:  weightUnit,
    TotalVolume: payload.totalVolumeCft,
    VolumeUnit:  volUnit
  },
  // Resolve estimated pickup time:
  // Priority 1 → appointment time  (take start of range if "HH:mm-HH:mm")
  // Priority 2 → hours window      (take start of range)
  // Priority 3 → default to 17:00  (end-of-business fallback)
  PickupWindow: {
    EstimatedPickup: do {
      var apptTime = payload.firstStop.shippingAppointmentTime default ""
      var hoursWin = payload.firstStop.shippingHours default ""
      var date     = payload.firstStop.expectedDate as String
      var resolvedTime =
        if (!isEmpty(apptTime))      (apptTime  splitBy "-")[0]
        else if (!isEmpty(hoursWin)) (hoursWin  splitBy "-")[0]
        else                         "17:00"
      ---
      date ++ "T" ++ resolvedTime ++ ":00"
    }
  },
  Stops: {
    (["Pickup", "Delivery"] map (stopType, idx) -> {
      Stop @(type: stopType): {
        LocationName: if (idx == 0) payload.firstStop.location.name
                      else          payload.lastStop.location.name,
        Street:  if (idx == 0) payload.firstStop.location.street default null
                 else          null,
        City:    if (idx == 0) payload.firstStop.location.city
                 else          payload.lastStop.location.city,
        State:   if (idx == 0) payload.firstStop.location.state
                 else          payload.lastStop.location.state,
        Country: if (idx == 0) payload.firstStop.location.country
                 else          payload.lastStop.location.country
      }
    })
  },
  CargoLines: {
    (payload.lineItems map (item) -> {
      CargoLine: {
        Description:   item.description,
        HandlingUnits: item.handlingUnitCount,
        UnitType:      item.handlingUnitType,
        WeightLbs:     item.weightLbs,
        LengthIn:      item.lengthIn,
        WidthIn:       item.widthIn,
        HeightIn:      item.heightIn,
        FreightClass:  item.freightClass
      }
    })
  },
  ChargeLines: {
    (payload.carrierCharges map (charge) -> {
      ChargeLine: {
        ChargeType: charge.chargeType,
        Quantity:   charge.quantity,
        UnitPrice:  charge.unitPrice
      }
    })
  },
  AssignedCarrier: {
    Name: payload.carrier.name,
    Code: payload.carrier.code
  },
  SubmittedBy: {
    EmployeeId: payload.createdBy.employeeId,
    Name:       payload.createdBy.name
  }
}
```

**DataWeave: Write Shipment + Consol Numbers Back (`update-load-ref.dwl`):**
```dataweave
%dw 2.0
output application/json skipNullOn="everywhere"
import * from dw::core::Strings
var logLines = (payload.processingLog substringBy $ == "|")
---
{
  "id":                vars.sourceLoadId,
  "erpShipmentNumber": payload.createdRecords.shipmentKey default null,
  "erpConsolNumber":   payload.createdRecords.consolKey   default null,
  "syncStatus": if (isEmpty(payload.createdRecords.shipmentKey)) "FAILED" else "SYNCED",
  "errors": (
    (logLines filter ((line) -> line contains("Error"))) default []
  ) map ((line) ->
    "Load [" ++ vars.sourceLoadName ++ "]: " ++ (line withMaxSize 500)
  )
}
```

---

## 3. Shipment Status Update Flows

### Overview

Four separate event flows push shipment milestone updates to the freight ERP as structured XML event payloads. Each event type represents a specific point in the freight lifecycle:

| Flow | Event | `isEstimate` |
|------|-------|-------------|
| `pickup-estimated` | Carrier dispatched, ETA calculated | `true` |
| `pickup-actual` | Freight physically picked up | `false` |
| `delivery-estimated` | ETA to destination calculated | `true` |
| `delivery-actual` | Freight delivered, POD confirmed | `false` |

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│             FLOW: status-update-router                          │
│                                                                 │
│  [Scheduler / Event Listener]                                   │
│      │                                                          │
│      ▼                                                          │
│  [HTTP GET / Query] ──► Fetch loads with new departure data    │
│      │                  and unset status-sent flag             │
│      ▼                                                          │
│  [Choice Router]                                                │
│      ├─ Actual departure date present   ──► pickup-actual      │
│      ├─ Expected pickup date present    ──► pickup-estimated   │
│      ├─ Actual delivery date present    ──► delivery-actual    │
│      └─ Expected delivery date present  ──► delivery-estimated │
│                                                                 │
│  [Transform: DWL] ──► Build status event XML payload           │
│  [HTTP POST]      ──► ERP event endpoint                       │
│  [HTTP PATCH]     ──► Mark status flag as sent in TMS          │
└─────────────────────────────────────────────────────────────────┘
```

### DataWeave: Estimated Pickup Event (`msg-pickup-estimated.dwl`)

```dataweave
%dw 2.0
output application/xml writeDeclaration=false
import * from dw::core::Strings
---
FreightStatusEvent: {
  EventContext: {
    DataProvider:    p('status.dataprovider'),
    EnterpriseId:    p('status.enterpriseid'),
    ServerId:        p('status.serverid'),
    ActionPurpose: {
      Code:        p('status.action.pickup.code'),
      Description: p('status.action.pickup.description')
    },
    CompanyCode:   p('status.company.code'),
    EventTypeCode: p('status.eventtype.pickup'),
    SubmittedBy: {
      EmployeeId: vars.loadRecord.createdBy.employeeId default null,
      Name:       vars.loadRecord.createdBy.name       default null
    }
  },
  EventType:   p('status.eventtype.pickup'),
  IsEstimate:  "true",
  // Time resolution logic:
  // If expected date absent         → null
  // If appointment time present     → use start of window
  // If hours range present          → use start of range
  // Fallback                        → 17:00 (end-of-business default)
  EventTime: do {
    var apptTime     = vars.loadRecord.firstStop.shippingAppointmentTime default ""
    var hoursRange   = vars.loadRecord.firstStop.shippingHours default ""
    var expectedDate = vars.loadRecord.firstStop.expectedDate default null
    ---
    if (isEmpty(expectedDate)) null
    else if (!isEmpty(apptTime))
      expectedDate as String ++ "T" ++ (splitBy(apptTime, "-"))[0] ++ ":00"
    else if (!isEmpty(hoursRange))
      expectedDate as String ++ "T" ++ (splitBy(hoursRange, "-"))[0] ++ ":00"
    else
      expectedDate as String ++ "T17:00:00"
  },
  ShipmentReference: {
    ReferenceType:  p('status.reference.type'),
    ReferenceValue: vars.loadRecord.id default null
  }
}
```

### DataWeave: Actual Pickup Event (`msg-pickup-actual.dwl`)

```dataweave
%dw 2.0
output application/xml writeDeclaration=false
---
FreightStatusEvent: {
  EventContext: {
    DataProvider:    p('status.dataprovider'),
    EnterpriseId:    p('status.enterpriseid'),
    ServerId:        p('status.serverid'),
    ActionPurpose: {
      Code:        p('status.action.pickup.code'),
      Description: p('status.action.pickup.description')
    },
    CompanyCode:   p('status.company.code'),
    EventTypeCode: p('status.eventtype.pickup'),
    SubmittedBy: {
      EmployeeId: vars.loadRecord.createdBy.employeeId default null,
      Name:       vars.loadRecord.createdBy.name       default null
    }
  },
  EventType:   p('status.eventtype.pickup'),
  IsEstimate:  "false",
  // Actual departure date and time come directly from the recorded first-stop departure
  EventTime:   vars.loadRecord.firstStop.actualDepartureDate default null,
  ShipmentReference: {
    ReferenceType:  p('status.reference.type'),
    ReferenceValue: vars.loadRecord.id default null
  }
}
```

> **Note:** Delivery estimated and delivery actual follow the same pattern — `firstStop` becomes `lastStop`, and `pickup` action/event codes are replaced with `delivery` codes.

### Status Update Response Parser (`parse-status-response.dwl`)

```dataweave
%dw 2.0
output application/json skipNullOn="everywhere"
import * from dw::core::Strings
var logLines = (payload.processingLog substringBy $ == "|")
---
{
  "id": vars.loadRecord.id,
  "errors": (
    (logLines filter ((line) -> line contains("Error"))) default []
  ) map ((line) ->
    "Load [" ++ vars.loadRecord.id ++ "]: " ++ (line withMaxSize 500)
  )
}
```

---

## 4. Freight ERP → CRM Sync

### Overview

Keeps the sales CRM in sync with the freight ERP by pushing Organization records as Company objects. One-off quotes in the ERP map to Deal records in the CRM. A bidirectional extension allows new CRM companies to flow back into the ERP and auto-populate staff assignments.

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                   FLOW: erp-to-crm-sync                          │
│                                                                  │
│  [Scheduler]                                                     │
│      │                                                           │
│      ▼                                                           │
│  [HTTP GET] ──────► ERP org export endpoint                     │
│      │                                                           │
│      ▼                                                           │
│  [Transform: DWL] ──► Map ERP XML → CRM JSON (Company schema)   │
│      │                                                           │
│      ├──► [HTTP POST/PATCH] ──► CRM Companies API               │
│      │                                                           │
│      └──► [For Each Quote]                                       │
│               ├──► [Transform: DWL] ──► Map quote → Deal JSON   │
│               └──► [HTTP POST] ──────► CRM Deals API            │
└──────────────────────────────────────────────────────────────────┘
```

### DataWeave: ERP Organization → CRM Company (`map-org-to-crm-company.dwl`)

```dataweave
%dw 2.0
output application/json skipNullOn="everywhere"
var orgTypeLabel = { "CUS": "Customer", "CAR": "Carrier", "VEN": "Vendor", "OTH": "Other" }
---
{
  "name":  payload.fullName,
  "phone": payload.phone default null,
  "properties": {
    "erp_org_code":      payload.orgCode,
    "erp_org_type":      orgTypeLabel[payload.orgType] default "Other",
    "is_active":         payload.isActive,
    "city":              payload.address.city     default null,
    "state":             payload.address.state    default null,
    "country":           payload.address.country  default null,
    "zip":               payload.address.postalCode default null,
    "assigned_rep_code": payload.salesRep.staffCode default null,
    "assigned_rep_name": payload.salesRep.name      default null
  }
}
```

---

## 5. EDI X12 214 Status Transmission

### Overview

Sends shipment status updates to a trading partner in **ANSI X12 214 (Transportation Carrier Shipment Status Message)** format. The ERP generates shipment event data in XML; this integration transforms it into a valid X12 214 transaction set and delivers it to the partner's SFTP drop folder.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                FLOW: edi-214-status-send                        │
│                                                                 │
│  [Scheduler]                                                    │
│      │                                                          │
│      ▼                                                          │
│  [HTTP GET] ──────► ERP shipment events endpoint               │
│      │                                                          │
│      ▼                                                          │
│  [Filter] ─────────► Events with qualifying status codes only  │
│      │                                                          │
│      ▼                                                          │
│  [Transform: DWL] ──► Map ERP event → X12 214 segment string  │
│      │                                                          │
│      ▼                                                          │
│  [SFTP Write] ─────► Drop .edi file to partner SFTP folder    │
└─────────────────────────────────────────────────────────────────┘
```

### EDI 214 Segment Reference (all dummy values)

```
ISA*00*          *00*          *ZZ*SENDERID       *ZZ*RECEIVERID     *240615*1200*^*00501*000000042*0*P*>~
GS*QM*SENDERID*RECEIVERID*20240615*1200*42*X*005010~
ST*214*0001~
BSN*00*LOAD-2024-00042*20240615*1200*SH~
AT7*OA*NS***20240615*1200*LT~
MS3*FHTRUCK*M**M~
NM1*CA*2*FastHaul Trucking*****46*FHTRUCK~
LX*1~
AT8*G*L*4500*6~
SE*9*0001~
GE*1*42~
IEA*1*000000042~
```

### DataWeave: ERP Event → X12 214 String (`map-event-to-edi214.dwl`)

```dataweave
%dw 2.0
output application/plain
var today   = now() as String{format: "yyyyMMdd"}
var nowTime = now() as String{format: "HHmm"}
var statusCodeMap = {
  "PickedUp":  "OA",
  "InTransit": "I",
  "Delivered": "D",
  "Delayed":   "X6"
}
var ctrlNum = vars.controlNumber as String
---
"ISA*00*          *00*          *ZZ*$(p('edi.sender.id'))       *ZZ*$(p('edi.receiver.id'))     *$(today)*$(nowTime)*^*00501*$(ctrlNum)*0*P*>~\n" ++
"GS*QM*$(p('edi.sender.id'))*$(p('edi.receiver.id'))*$(today)*$(nowTime)*$(ctrlNum)*X*005010~\n" ++
"ST*214*0001~\n" ++
"BSN*00*$(payload.shipmentId)*$(today)*$(nowTime)*SH~\n" ++
"AT7*$(statusCodeMap[payload.eventType] default 'I')***$(payload.eventDate as String{format:'yyyyMMdd'})*$(payload.eventTime default nowTime)*LT~\n" ++
"MS3*$(payload.carrierCode)*M**M~\n" ++
"NM1*CA*2*$(payload.carrierName)*****46*$(payload.carrierCode)~\n" ++
"LX*1~\n" ++
"AT8*G*L*$(payload.totalWeightLbs)*$(payload.totalPieces)~\n" ++
"SE*9*0001~\n" ++
"GE*1*$(ctrlNum)~\n" ++
"IEA*1*$(ctrlNum)~"
```

---

## 6. SFTP File Intake → ERP Import

### Overview

A file listener polls an SFTP server for inbound JSON files submitted by external partners. Each file is transformed into the ERP's XML shipment format and POSTed via API. Processed files are archived; failed files are moved to an error folder with a companion log.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              FLOW: sftp-file-intake                             │
│                                                                 │
│  [SFTP Listener] ──► /incoming/*.json                          │
│      │               Polling: every 60s                        │
│      │               Post-process: move to /processing/        │
│      ▼                                                          │
│  [Logger] ──────────► Log filename + size                      │
│      │                                                          │
│      ▼                                                          │
│  [Transform: DWL] ──► JSON partner file → ERP Shipment XML    │
│      │                                                          │
│      ├─ SUCCESS ──►  [HTTP POST] ──► ERP import endpoint       │
│      │               [SFTP Move]  ──► /archive/                │
│      │                                                          │
│      └─ FAILURE ──►  [Logger]     ──► Log error details        │
│                      [SFTP Write] ──► /errors/ + error log     │
│                      [SMTP Alert] ──► Ops team notification    │
└─────────────────────────────────────────────────────────────────┘
```

### DataWeave: Partner JSON → ERP Shipment XML (`map-partner-file-to-erp.dwl`)

```dataweave
%dw 2.0
output application/xml writeDeclaration=false
---
FreightShipment: {
  BookingReference: payload.bookingReference,
  ShipmentKey:      payload.shipmentNumber,
  Parties: {
    Shipper @(role: "Origin"): {
      Name:    payload.shipper.name,
      Street:  payload.shipper.address,
      City:    payload.shipper.city,
      State:   payload.shipper.state,
      Country: payload.shipper.country
    },
    Consignee @(role: "Destination"): {
      Name:    payload.consignee.name,
      City:    payload.consignee.city,
      State:   payload.consignee.state,
      Country: payload.consignee.country
    }
  },
  CargoLines: {
    (payload.cargo map (item) -> {
      CargoLine: {
        Description: item.description,
        Quantity:    item.quantity,
        PackageType: item.packageType,
        WeightKg:    item.weightKg
      }
    })
  },
  Charges: {
    (payload.cargo map (item) -> {
      ChargeLine: {
        Code:   item.chargeCode,
        Amount: item.localAmount
      }
    })
  }
}
```

---

## 7. Error Handling & Alerting Pattern

### Overview

A reusable error payload module deployed as a global error handler across all integration flows. When any flow encounters an unhandled exception, this transformation builds a structured error JSON object dispatched as an SMTP alert to the operations team.

### DataWeave: Structured Error Payload (`msg-error-payload.dwl`)

```dataweave
%dw 2.0
output application/json
---
{
  "errorDetails": {
    "applicationName": app.name,
    "environment":     Mule::p('mule.env') as String,
    "timestamp":       now() as String,
    "errorCode":       500,
    "errorType": if (!isEmpty(error.errorType.namespace) and !isEmpty(error.errorType.identifier))
        error.errorType.namespace ++ ":" ++ error.errorType.identifier
      else
        "INTEGRATION:UNKNOWN_ERROR",
    "errorMessage":  error.detailedDescription,
    "correlationId": correlationId,
    // Derive readable metadata from the internal failing component path
    "fileName":      (error.failingComponent default "" as String splitBy ":") [-2],
    "flowName":      (error.failingComponent default "" as String splitBy "/") [0],
    "componentName": ((error.failingComponent default "" as String splitBy "(") [-1] splitBy ")") [0]
  }
}
```

**Sample alert output (dummy):**
```json
{
  "errorDetails": {
    "applicationName": "logistics-integration-api",
    "environment": "prod",
    "timestamp": "2024-06-15T14:32:07.441Z",
    "errorCode": 500,
    "errorType": "HTTP:CONNECTIVITY",
    "errorMessage": "Connection refused after 3 retries",
    "correlationId": "4f2a1b89-c3d0-4e7f-a812-000000000001",
    "fileName": "load-to-shipment",
    "flowName": "load-to-shipment-main",
    "componentName": "post-to-erp"
  }
}
```

**Global Error Handler (Mule XML):**
```xml
<on-error-propagate type="ANY" enableNotifications="true" logException="true">
  <ee:transform>
    <ee:message>
      <ee:set-payload resource="dwl/msg-error-payload.dwl"/>
    </ee:message>
  </ee:transform>
  <email:send config-ref="SMTP_Config"
    toAddresses='#[p("alerts.ops.email")]'
    subject='#["Integration Failure: " ++ app.name ++ " [" ++ Mule::p("mule.env") ++ "]"]'>
    <email:body contentType="application/json">
      <email:content>#[payload]</email:content>
    </email:body>
  </email:send>
</on-error-propagate>
```

---

## 8. CRM ↔ ERP Org Matching Utility

### Overview

A data reconciliation utility that compares organization records between the CRM and the ERP to identify mismatches, inactive records, and orphaned entries — enabling clean bidirectional sync across large datasets.

### Reconciliation Logic

```
Input A: CRM export  → all companies carrying an ERP org code property
Input B: ERP export  → all org records flagged as active Sales type

For each CRM record:
  → ERP org code exists + org is active    → MATCHED
  → ERP org code exists + org is inactive  → INACTIVE_IN_ERP
  → ERP org code not found in ERP at all   → ORPHANED_IN_CRM
  → ERP org code field is blank / invalid  → MISSING_CODE

For each ERP record:
  → Org code not present in any CRM record → MISSING_IN_CRM
```

### DataWeave: Reconciliation Report (`reconcile-orgs.dwl`)

```dataweave
%dw 2.0
output application/json
// vars.crmCompanies = [{ id, name, erpOrgCode }]
// vars.erpOrgs      = [{ orgCode, fullName, isActive }]
var erpByCode = vars.erpOrgs groupBy $.orgCode
var erpCodes  = vars.erpOrgs map $.orgCode
var crmCodes  = vars.crmCompanies map $.erpOrgCode
---
{
  "summary": {
    "totalCrmRecords": sizeOf(vars.crmCompanies),
    "totalErpRecords": sizeOf(vars.erpOrgs),
    "matched":         sizeOf(vars.crmCompanies filter (c) ->  (erpCodes  contains c.erpOrgCode)),
    "orphanedInCrm":   sizeOf(vars.crmCompanies filter (c) -> !(erpCodes  contains c.erpOrgCode)),
    "missingInCrm":    sizeOf(vars.erpOrgs      filter (e) -> !(crmCodes  contains e.orgCode))
  },
  "orphanedInCrm": vars.crmCompanies
    filter  (c) -> !(erpCodes contains c.erpOrgCode)
    map     (c) -> { "crmId": c.id, "name": c.name, "badCode": c.erpOrgCode },
  "inactiveInErp": vars.crmCompanies
    filter  (c) -> ((erpByCode[c.erpOrgCode]?[0].isActive) == false)
    map     (c) -> { "crmId": c.id, "name": c.name, "erpOrgCode": c.erpOrgCode },
  "missingInCrm": vars.erpOrgs
    filter  (e) -> !(crmCodes contains e.orgCode)
    map     (e) -> { "orgCode": e.orgCode, "name": e.fullName }
}
```

---

## 9. CI/CD: Mule App Deploy Pipeline

### Overview

A GitHub Actions workflow that automates testing and deployment of MuleSoft integration projects across three environments — **dev**, **test**, and **prod** — using the MuleSoft Maven plugin and CloudHub 2.0. Feature branch pushes deploy to dev automatically. Merges to `main` deploy to test. Production requires a manual approval gate.

### Branch & Environment Strategy

```
feature/* ──────────────────────────────────────► dev   (auto on push)
               │
              PR + tests pass
               │
               ▼
             main ───────────────────────────────► test  (auto on merge)
               │
          Manual approval
               │
               ▼
          workflow_dispatch ──────────────────────► prod  (gated)
```

### Repository Structure

```
logistics-integration-api/
├── .github/
│   └── workflows/
│       ├── deploy-dev.yml
│       ├── deploy-test.yml
│       └── deploy-prod.yml
├── src/
│   └── main/
│       ├── mule/
│       │   ├── global.xml
│       │   ├── account-to-org.xml
│       │   ├── load-to-shipment.xml
│       │   └── status-updates.xml
│       └── resources/
│           ├── dwl/
│           │   ├── map-account-to-org.dwl
│           │   ├── map-load-to-shipment.dwl
│           │   ├── msg-pickup-estimated.dwl
│           │   ├── msg-pickup-actual.dwl
│           │   ├── msg-delivery-estimated.dwl
│           │   ├── msg-delivery-actual.dwl
│           │   └── msg-error-payload.dwl
│           ├── dev.yaml
│           ├── test.yaml
│           └── prod.yaml
├── src/test/munit/
│   ├── account-to-org-test-suite.xml
│   └── load-to-shipment-test-suite.xml
└── pom.xml
```

### GitHub Actions: Deploy to Dev

**`.github/workflows/deploy-dev.yml`**
```yaml
name: Deploy to Dev

on:
  push:
    branches:
      - 'feature/**'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: dev

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven dependencies
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-maven-

      - name: Run MUnit tests
        run: mvn verify -Dmule.env=dev
        env:
          MULE_ERP_HOST:    ${{ secrets.DEV_ERP_HOST }}
          MULE_ERP_PORT:    ${{ secrets.DEV_ERP_PORT }}
          MULE_CRM_API_KEY: ${{ secrets.DEV_CRM_API_KEY }}

      - name: Deploy to CloudHub Dev
        run: |
          mvn deploy -DskipTests \
            -Dcloudhub.environment=dev \
            -Dcloudhub.application.name=logistics-integration-api-dev \
            -Dcloudhub.worker.type=0.1vCores \
            -Dcloudhub.region=us-east-1 \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }}

      - name: Open issue on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo:  context.repo.repo,
              title: `Deploy Failed: dev — ${context.ref}`,
              body:  `Workflow run failed.\n\nRun: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
              labels: ['deployment-failure', 'dev']
            })
```

### GitHub Actions: Deploy to Test

**`.github/workflows/deploy-test.yml`**
```yaml
name: Deploy to Test

on:
  push:
    branches:
      - main

jobs:
  test-and-deploy:
    runs-on: ubuntu-latest
    environment: test

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven dependencies
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-maven-

      - name: Run full MUnit test suite
        run: mvn verify -Dmule.env=test
        env:
          MULE_ERP_HOST:    ${{ secrets.TEST_ERP_HOST }}
          MULE_ERP_PORT:    ${{ secrets.TEST_ERP_PORT }}
          MULE_CRM_API_KEY: ${{ secrets.TEST_CRM_API_KEY }}

      - name: Upload MUnit test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: munit-results
          path: target/surefire-reports/

      - name: Deploy to CloudHub Test
        run: |
          mvn deploy -DskipTests \
            -Dcloudhub.environment=test \
            -Dcloudhub.application.name=logistics-integration-api-test \
            -Dcloudhub.worker.type=0.2vCores \
            -Dcloudhub.region=us-east-1 \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }}

      - name: Open issue on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo:  context.repo.repo,
              title: `Deploy Failed: test — ${context.sha.substring(0,7)}`,
              body:  `MUnit tests or test deployment failed.\n\nCommit: ${context.sha}\nRun: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
              labels: ['deployment-failure', 'test']
            })
```

### GitHub Actions: Deploy to Production (manual approval)

**`.github/workflows/deploy-prod.yml`**
```yaml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type DEPLOY to confirm production deployment'
        required: true

jobs:
  validate-input:
    runs-on: ubuntu-latest
    steps:
      - name: Confirm deployment intent
        run: |
          if [ "${{ github.event.inputs.confirm }}" != "DEPLOY" ]; then
            echo "Confirmation text did not match. Aborting."
            exit 1
          fi

  deploy-prod:
    runs-on: ubuntu-latest
    needs: validate-input
    environment: prod   # GitHub environment with required reviewers configured

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven dependencies
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-maven-

      - name: Deploy to CloudHub Production
        run: |
          mvn deploy -DskipTests \
            -Dcloudhub.environment=prod \
            -Dcloudhub.application.name=logistics-integration-api \
            -Dcloudhub.worker.type=1vCore \
            -Dcloudhub.region=us-east-1 \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }}

      - name: Tag release
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git tag -a "prod-$(date +'%Y%m%d-%H%M')" -m "Production deploy by ${{ github.actor }}"
          git push origin --tags

      - name: Open issue on failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo:  context.repo.repo,
              title: `🚨 PROD Deploy Failed — ${new Date().toISOString()}`,
              body:  `Production deployment failed.\n\nTriggered by: @${context.actor}\nRun: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
              labels: ['deployment-failure', 'production', 'urgent']
            })
```

### Environment Property Files

```yaml
# src/main/resources/dev.yaml  (safe to commit — no real values, all injected at runtime)
erp:
  host:     "${ERP_HOST}"
  port:     "${ERP_PORT}"
  basePath: "/api/v1"
  timeout:  10000

crm:
  apiKey:  "${CRM_API_KEY}"
  baseUrl: "https://api.crm-placeholder.example.com"

status:
  dataprovider: "INTEGRATION_DEV"
  enterpriseid: "EXAMPLE_ENT_DEV"
  serverid:     "DEV_SERVER"
  company:
    code: "DEV"
  action:
    pickup:
      code:        "PU"
      description: "Pickup"
    delivery:
      code:        "DL"
      description: "Delivery"
  eventtype:
    pickup:   "PICKUP_EVT"
    delivery: "DELIVERY_EVT"
  reference:
    type: "LoadReference"
  time: "17:00"

alerts:
  ops:
    email: "ops-alerts-dev@example.com"
```

### Secret Scoping by Environment

| Secret | Dev | Test | Prod |
|--------|-----|------|------|
| `*_ERP_HOST` | `DEV_ERP_HOST` | `TEST_ERP_HOST` | `PROD_ERP_HOST` |
| `*_ERP_PORT` | `DEV_ERP_PORT` | `TEST_ERP_PORT` | `PROD_ERP_PORT` |
| `*_CRM_API_KEY` | `DEV_CRM_API_KEY` | `TEST_CRM_API_KEY` | `PROD_CRM_API_KEY` |
| `ANYPOINT_USERNAME` | ✅ shared | ✅ shared | ✅ shared |
| `ANYPOINT_PASSWORD` | ✅ shared | ✅ shared | ✅ shared |
| `HEALTH_CHECK_TOKEN` | ✅ shared | ✅ shared | ✅ shared |

---

## 10. Integration Health Monitor

### Overview

A scheduled GitHub Actions workflow that acts as a lightweight **integration health monitor**. Every 30 minutes it calls the `/health` endpoint on each deployed Mule application, evaluates the response, and automatically opens a GitHub Issue if any flow is unhealthy — tagged by environment and severity. This eliminates the need to log into the Anypoint Platform console to discover a down integration.

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│           WORKFLOW: integration-health-monitor.yml              │
│                                                                 │
│  [Cron: every 30 min]                                           │
│      │                                                          │
│      ▼                                                          │
│  [Matrix: each app × each env]                                  │
│      │                                                          │
│      ▼                                                          │
│  [curl /health] ──► check HTTP status + JSON body              │
│      │                                                          │
│      ▼                                                          │
│  [Evaluate]                                                     │
│      ├─ HTTP 200 + status "UP"       ──► log OK, no action     │
│      ├─ HTTP 200 + status "DEGRADED" ──► open WARNING issue    │
│      └─ Non-200 / timeout            ──► open CRITICAL issue   │
│                                                                 │
│  [De-duplicate] ──► skip if matching open issue already exists │
└─────────────────────────────────────────────────────────────────┘
```

### Mule Health Check Endpoint

Each deployed Mule application exposes a `/health` endpoint that returns its status, environment, and version:

```xml
<!-- health-check.xml -->
<flow name="health-check-flow">
  <http:listener path="/health" method="GET" config-ref="HTTP_Listener_Config"/>
  <ee:transform>
    <ee:message>
      <ee:set-payload><![CDATA[
        %dw 2.0
        output application/json
        ---
        {
          status:      "UP",
          application: app.name,
          environment: Mule::p('mule.env'),
          timestamp:   now() as String,
          version:     Mule::p('app.version')
        }
      ]]></ee:set-payload>
    </ee:message>
  </ee:transform>
</flow>
```

**Sample healthy response:**
```json
{
  "status":      "UP",
  "application": "logistics-integration-api",
  "environment": "prod",
  "timestamp":   "2024-06-15T14:00:01.223Z",
  "version":     "1.4.2"
}
```

### GitHub Actions: Health Monitor

**`.github/workflows/integration-health-monitor.yml`**
```yaml
name: Integration Health Monitor

on:
  schedule:
    - cron: '*/30 * * * *'   # every 30 minutes
  workflow_dispatch:           # allow manual trigger

jobs:
  health-check:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        app:
          - name:     "account-to-org"
            url:      "https://logistics-integration-api-prod.cloudhub.io/account-to-org/health"
            env:      "prod"
            severity: "critical"
          - name:     "load-to-shipment"
            url:      "https://logistics-integration-api-prod.cloudhub.io/load-to-shipment/health"
            env:      "prod"
            severity: "critical"
          - name:     "status-updates"
            url:      "https://logistics-integration-api-prod.cloudhub.io/status-updates/health"
            env:      "prod"
            severity: "critical"
          - name:     "sftp-intake"
            url:      "https://logistics-integration-api-prod.cloudhub.io/sftp-intake/health"
            env:      "prod"
            severity: "high"
          - name:     "account-to-org"
            url:      "https://logistics-integration-api-test.cloudhub.io/account-to-org/health"
            env:      "test"
            severity: "high"

    steps:
      - name: Check health endpoint
        id: health
        run: |
          HTTP_STATUS=$(curl --silent --output /tmp/response.json \
            --write-out "%{http_code}" \
            --max-time 10 \
            --header "Authorization: Bearer ${{ secrets.HEALTH_CHECK_TOKEN }}" \
            "${{ matrix.app.url }}" || echo "000")

          echo "http_status=$HTTP_STATUS" >> $GITHUB_OUTPUT

          APP_STATUS=$(cat /tmp/response.json \
            | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('status','UNKNOWN'))" \
            2>/dev/null || echo "PARSE_ERROR")

          echo "app_status=$APP_STATUS" >> $GITHUB_OUTPUT
          echo "Health → HTTP $HTTP_STATUS | App status: $APP_STATUS"

      - name: Check for existing open issue
        id: existing
        if: steps.health.outputs.http_status != '200' || steps.health.outputs.app_status != 'UP'
        uses: actions/github-script@v7
        with:
          script: |
            const issues = await github.rest.issues.listForRepo({
              owner: context.repo.owner,
              repo:  context.repo.repo,
              state: 'open',
              labels: 'integration-health'
            });
            const title  = `[Health] ${{ matrix.app.name }} is unhealthy (${{ matrix.app.env }})`;
            const exists = issues.data.some(i => i.title === title);
            core.setOutput('exists', exists.toString());

      - name: Open GitHub Issue on failure
        if: |
          (steps.health.outputs.http_status != '200' || steps.health.outputs.app_status != 'UP')
          && steps.existing.outputs.exists == 'false'
        uses: actions/github-script@v7
        with:
          script: |
            const httpStatus = '${{ steps.health.outputs.http_status }}';
            const appStatus  = '${{ steps.health.outputs.app_status }}';
            const severity   = '${{ matrix.app.severity }}';
            const emoji      = severity === 'critical' ? '🚨' : '⚠️';

            await github.rest.issues.create({
              owner: context.repo.owner,
              repo:  context.repo.repo,
              title: `[Health] ${{ matrix.app.name }} is unhealthy (${{ matrix.app.env }})`,
              body: [
                `## ${emoji} Integration Health Alert`,
                ``,
                `| Field        | Value |`,
                `|--------------|-------|`,
                `| **App**      | \`${{ matrix.app.name }}\` |`,
                `| **Env**      | \`${{ matrix.app.env }}\` |`,
                `| **HTTP**     | \`${httpStatus}\` |`,
                `| **Status**   | \`${appStatus}\` |`,
                `| **Time**     | ${new Date().toISOString()} |`,
                `| **Severity** | ${severity.toUpperCase()} |`,
                ``,
                `### Suggested Next Steps`,
                `- [ ] Check Runtime Manager logs for this application`,
                `- [ ] Verify the ERP/TMS endpoint is reachable from CloudHub`,
                `- [ ] Review the most recent deployment for this app`,
                `- [ ] Apply the \`resolved\` label to auto-close once healthy`,
                ``,
                `> Auto-opened by the Integration Health Monitor workflow.`
              ].join('\n'),
              labels: ['integration-health', severity, '${{ matrix.app.env }}']
            });

      - name: Log healthy status
        if: steps.health.outputs.http_status == '200' && steps.health.outputs.app_status == 'UP'
        run: echo "✅ ${{ matrix.app.name }} (${{ matrix.app.env }}) is UP"
```

### Auto-Close Resolved Issues

**`.github/workflows/health-auto-close.yml`**
```yaml
name: Auto-Close Resolved Health Issues

on:
  workflow_run:
    workflows: ["Integration Health Monitor"]
    types: [completed]

jobs:
  auto-close:
    runs-on: ubuntu-latest
    steps:
      - name: Close issues marked resolved
        uses: actions/github-script@v7
        with:
          script: |
            const issues = await github.rest.issues.listForRepo({
              owner: context.repo.owner,
              repo:  context.repo.repo,
              state: 'open',
              labels: 'integration-health'
            });

            for (const issue of issues.data) {
              const hasResolved = issue.labels.some(l => l.name === 'resolved');
              if (hasResolved) {
                await github.rest.issues.update({
                  owner:        context.repo.owner,
                  repo:         context.repo.repo,
                  issue_number: issue.number,
                  state:        'closed'
                });
                console.log(`Closed issue #${issue.number}: ${issue.title}`);
              }
            }
```

---

## 🧠 Key Engineering Takeaways

- **DataWeave is a full transformation language.** These scripts use conditional logic, pattern matching, `map` / `filter` / `groupBy`, string manipulation (`splitBy`, `substringBy`, `withMaxSize`), type coercion, dynamic XML attribute injection, and named variables — not drag-and-drop field mapping.
- **Integration is a systems problem.** Every project required understanding the full data model of two or more platforms, failure modes of scheduled vs. event-driven syncs, and strategies to keep records linked bidirectionally without duplicates.
- **Error handling is a first-class feature.** Every flow surfaces structured error details — failing component, flow name, correlation ID, and environment — to make production debugging tractable without a platform console.
- **Logistics data is unpredictable.** Appointment windows arrive as ranges (`"08:00-12:00"`), optional fields need graceful fallbacks, unit codes vary between systems, and a single `null` can break an XML schema. Defensive DataWeave is non-negotiable.
- **CI/CD and monitoring close the loop.** Building the integration is only half the job. Automated deployment pipelines, environment-scoped secrets, and proactive GitHub Issue alerting turn a collection of flows into a maintainable production system.

---

***Built by Keelen Carrera · [LinkedIn](https://linkedin.com/in/keelencarrera) · Houston, TX***
