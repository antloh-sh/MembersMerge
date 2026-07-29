# AAC Membership Merge Tool — README

## Overview

The **AAC Membership Merge Tool** is a client-side browser utility designed to merge Active Ageing Centre (AAC) **Membership** and **Member Details** CSV exports. It drops terminated members, computes member ages, handles split/tagged phone numbers, resolves street and building names using OneMap, and integrates activity attendance counts.

---

## Input Files & Reference Headers

The tool accepts three CSV files (two required, one optional):

### 1. Membership CSV (`DataListing-AACMembership.csv`) — *Required*

Primary roster export containing core member records and employment details.

```csv
Centre1, MembershipId, MembershipStatus, DocumentType, NRIC, Salutation, Surname, FullName, ToAddressAs, NativeName, DOB, Gender, MaritalStatus, BloodGroup, Citizenship, Race, Religion, HomePhone, MobilePhone, OtherPhone, PersonAddressPostalCode, PersonAddressBlockNo, PersonAddressStoreyUnit, PersonAddressStreetName, PersonAddressBuildingName, PersonAddressCountry, ResidesWithinServiceBoundary, PersonAddressTypeOfAccommodation, Address_Ownership, LivingArrangement, OthersLivingArrangement, Language, HighestEducation, CurrentEmploymentStatus, OtherEmploymentStatus, Hobbies, Mobility, LevelOfCares, FinancialStatus, OtherFinancialStatus, NoOfSons, NoOfDaughters, PhysicalHealths, PhysicalHealthOthers, MentalHealths, MentalHealthOthers, SocioEmotional, SocioEmotionalOthers, FollowUps, OtherFollowUps, HealthRemarks, NoNextOfKin, Row_Number, NOKSalutation, NOKFullName, NOKSurname, NOKDocumentType, NOKNRIC, Relationship, TypeOfEmergencyContactPerson, NOKHomePhone, NOKMobilePhone, NOKOtherPhone, NOKPostalCode, NOKBlockNo, NOKStoreyUnitNo, NOKStreetName, NOKBulidingName, NOKCountry, RecordedBy, RegistrationDate, JoinedDate, TerminatedDate, TerminatedBy, TerminationReason, WithdrawnDate, WithdrawnBy, WithdrawnReason, RFID, Remarks, Responder, SelectNOK, DeclarationDate, CreatedOn, CreatedBy, ModifiedOn, ModifiedBy

```

* **Key required headers for merge:** `MembershipId`, `MembershipStatus`, `NRIC`

---

### 2. Member Details CSV (`DataListing-AACMemberDetails.csv`) — *Required*

Enrichment file used to update demographic, address, and contact fields.

```csv
Centre, MembershipId, CurrentMembershipStatus, JoinedDate, TerminatedDate, RegistrationDocumentType, RegistrationDocumentNumber, Salutation, FullNameNative, ToAddressAs, DateOfBirth, Age, Gender, MaritalStatus, Citizenship, Race, Religion, Phone, ResidesWithinServiceBoundaryStr, NricAddress_PostalCode, NricAddress_BlockNo, NricAddress_UnitNo, FullAddress, NricAddress_Ownership, SpokenLanguage, Mobility, LevelOfCare, IsFaceRegistered, RFID, Hobbies, Remarks, NextOfKin, AssignedStaffs, AssignedVolunteers, LastActivity, LastActivityAttendedOn, BnBType, IRConcatenatedString, InteractionContactDate, InteractionStatus, DeclinedCSStr, AssessmentDate, AUAScore, NextReviewDate, InterRAICSAssessmentDate, InterRAICSSCALES, InterRAICSCAPS, InterRAICSNextReviewDate

```

* **Key required headers for merge:** `MembershipId`, `CurrentMembershipStatus`, `RegistrationDocumentNumber`

---

### 3. Member Activities CSV (`Supabase Snippet Members Activities Count.csv`) — *Optional*

SQL export containing activity counts matched by member NRIC. If omitted, `ActivityCount` defaults to `0` for all rows.

```csv
member_registrationdocumentnumber, member_name, cfs_score, activities_attended_count

```

* **Key required headers for merge:** `member_registrationdocumentnumber`, `activities_attended_count`

---

## SQL Query for Member Activities CSV (Supabase)

To generate the **Member Activities CSV**, execute the following SQL query in your **Supabase Membership Database**.

> ⚠️ **Note:** Adjust the start date (`2026-04-01`) and end date (`2026-07-01`) to match your reporting quarter/period.

```sql
select
  m."registrationdocumentnumber" as "member_registrationdocumentnumber",
  m."fullnamenative" as "member_name",
  max(a."cfs") as "cfs_score",
  count(a."id") as "activities_attended_count"
from public."aac_member_details" m
left join public."activities" a
  on a."registrationdocumentnumber" = m."registrationdocumentnumber"
  and a."activitydatedisplay" >= date '2026-04-01'
  and a."activitydatedisplay" < date '2026-07-01'
where m."resideswithinserviceboundarystr" = true
  and m."dateofbirth" is not null
  and extract(year from age(current_date, m."dateofbirth")) >= 60
group by m."registrationdocumentnumber", m."fullnamenative"
order by "activities_attended_count" desc, "member_registrationdocumentnumber" asc;

```

Export the query output directly as CSV and upload it to Step 1 of the tool.

---

## Output Specifications

### Generated Output Files

1. **`Final_Merged_Members.csv`**: Main enriched roster ready for operational use.
2. **`Failed_Address_Lookup.csv`**: Generated only if one or more postal codes could not be resolved via OneMap.

### Final Output Column Structure (27 Columns)

| Column Name | Source / Rule |
| --- | --- |
| `MembershipStatus` | Overwritten by `MemberDetails.CurrentMembershipStatus` if present |
| `JoinedDate` | From `MemberDetails.JoinedDate` |
| `NRIC` | Overwritten by `MemberDetails.RegistrationDocumentNumber` if present |
| `FullName` | Overwritten by `MemberDetails.FullNameNative` if present |
| `ToAddressAs` | Overwritten by `MemberDetails.ToAddressAs` if present |
| `DOB` | Overwritten by `MemberDetails.DateOfBirth` if present |
| `Age` | Calculated dynamically from resolved DOB |
| `Gender` | Overwritten by `MemberDetails.Gender` if present |
| `MaritalStatus` | Overwritten by `MemberDetails.MaritalStatus` if present |
| `Citizenship` | Overwritten by `MemberDetails.Citizenship` if present |
| `Race` | Overwritten by `MemberDetails.Race` if present |
| `Religion` | Overwritten by `MemberDetails.Religion` if present |
| `HomePhone` | Parsed from `MemberDetails.Phone` `(H)` tag; falls back to Membership `HomePhone` |
| `MobilePhone` | Parsed from `MemberDetails.Phone` `(M)` tag; falls back to Membership `MobilePhone` |
| `ResidesWithinServiceBoundary` | Overwritten by `MemberDetails.ResidesWithinServiceBoundaryStr` if present |
| `PersonAddressPostalCode` | Overwritten by `MemberDetails.NricAddress_PostalCode` if present |
| `PersonAddressBlockNo` | Overwritten by `MemberDetails.NricAddress_BlockNo` if present |
| `PersonAddressStoreyUnit` | Overwritten by `MemberDetails.NricAddress_UnitNo` if present |
| `PersonAddressStreetName` | Parsed from `FullAddress` or resolved via OneMap API |
| `PersonAddressBuildingName` | Parsed from `FullAddress` or resolved via OneMap API |
| `PersonAddressCountry` | Set to `"Singapore"` upon valid address match |
| `Language` | Overwritten by `MemberDetails.SpokenLanguage` if present |
| `CurrentEmploymentStatus` | From `Membership.CurrentEmploymentStatus` |
| `Hobbies` | Overwritten by `MemberDetails.Hobbies` if present |
| `Remarks` | From `MemberDetails.Remarks` |
| `HouseholdID` | Formatted as `{PostalCode}_{BlockNo}_{StoreyUnit}` (`LANDED` if unit is blank) |
| `ActivityCount` | Matched via NRIC from Activities CSV (`0` if unmatched or skipped) |

---

## Key Processing Rules

1. **Record Matching:** Records are primary-matched on `MembershipId`. Unmatched rows fall back to NRIC matching.
2. **Terminated Filter:** Any record with a resolved status of `Terminated` is automatically removed from the final export.
3. **Address Resolution & OneMap Fallback:** The tool first parses `FullAddress` from Member Details. If street info is missing, it calls OneMap's API using the 6-digit postal code (rate-limited at 200 calls/min to prevent throttling).
4. **Privacy:** Runs entirely client-side in the browser. No personal data is sent to external servers.
