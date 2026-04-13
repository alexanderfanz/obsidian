# Account (Customer)

> **Auto-synced from Obsidian** — Do not edit this page directly. Your changes will be overwritten on the next sync. If you need to add information, create a new page and link to this one.

The customer profile entity. Represents a shopper who interacts with a dispensary storefront.

## Where it lives

| Store | Repo | Purpose |
|-------|------|---------|
| DynamoDB (per-store table) | [[e-com-srv]], [[e-com-cognito-worker]] | Source of truth. Key prefix `c#` + EntityID. Shares table with [[Order]]s. |
| Cognito User Pool | [[Store CDK]], [[e-com-cognito-worker]] | Authentication identity. Custom attributes map to Account fields. |
| Jane platform | [[e-com-cognito-worker]] | Synced on signup for stores with Jane integration. |
| CRM platforms | [[e-com-app-worker]], [[e-com-cognito-worker]] | Subscribed on signup (Klaviyo, ListTrack, AlpineIQ, Omnisend). |

## Key fields

`EntityID`, `Email`, `FirstName`, `LastName`, `Phone`, `DateOfBirth`, `CountryCode`, address fields, `AcceptMarketing`, `TotalOrderExpense` (lifetime spend), `Type` (ADULT or MEDICAL), `MedicalID`, `MedExpirationDate`, `medical_id_photo_uri`, cross-system links (`treez_id`, `alpine_id`), loyalty points, activity log.

## How each service touches it

| Service | Operations |
|---------|-----------|
| [[e-com-cognito-worker]] | **Creates** account on PostConfirmation. Maps Cognito custom attributes → Account struct. Upserts in DynamoDB. Syncs to Jane. Subscribes to CRMs. |
| [[e-com-order-worker]] | **Updates** on every order: aggregates lifetime spend, copies medical ID and DOB from order. |
| [[e-com-srv]] | CRUD via GraphQL mutations. Photo upload to S3. ID verification (Berbix, IDScan). |
| [[gap-dashboard]] | Operators view customer records, purchase history. |
| [[treez-tlp]] | Customers manage their profile, view order history. |

## Medical vs. Adult

`Type` is set from `custom:ACCOUNT_TYPE` Cognito attribute at signup. Medical accounts carry `MedicalID` and `MedExpirationDate` — compliance-relevant fields surfaced in CRM subscriptions and Treez order submissions. The `treez_medical_id_resend` feature flag on [[Store]] controls medical ID resend flows.

## BusinessAccount

A separate B2B entity (distinct from retail Account). Has its own service interface in [[e-com-lib]]. Used for wholesale/business customers.