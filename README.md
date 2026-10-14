# AAC Membership Merge Tool

A single-file, client-side web app (`index.html`) that merges the AAC Membership and Member Details exports into one cleaned roster. Everything runs in the browser, so no member data is uploaded anywhere. The only outbound call is a postal code sent to OneMap, and only as a last-resort fallback.

## What it does

1. Merges `DataListing-AACMembership.csv` (master roster) with `DataListing-AACMemberDetails.csv` (refresh source).
2. Removes members whose resolved status is **Terminated** or **Withdrawn**.
3. Calculates **Age** from date of birth.
4. Resolves street and building name (see [Address resolution](#address-resolution)).
5. Splits phone numbers into Home and Mobile using the `(H)` / `(M)` tags.
6. Adds **HouseholdID** and **ActivityCount**.
7. Produces `Final_Merged_Members.csv`, plus `Failed_Address_Lookup.csv` if any OneMap fallback lookup fails.

## Deploying to GitHub Pages (free tier)

1. Put `index.html` in the root of a repository.
2. In the repository, go to **Settings → Pages**.
3. Set **Source** to "Deploy from a branch", choose your branch and `/ (root)`, then save.
4. Open `https://<username>.github.io/<repo-name>/`.

There is no build step. Edit `index.html`, push, and Pages redeploys. Open the tool from the Pages URL, not by double-clicking the file.

## Using the tool

| Step | Action |
|---|---|
| 1. Upload | Membership CSV and Member Details CSV (required); Activities CSV (optional) |
| 2. Validate | Checks required columns, flags duplicate `MembershipId` values |
| 3. Merge & Enrich | Progress bars for the merge and for any OneMap lookups |
| 4. Summary | Counts, advisories, and flagged data issues |
| 5. Download | `Final_Merged_Members.csv` and, if needed, `Failed_Address_Lookup.csv` |

## Input files and their headers

Reference headers as exported. The tool reads columns by name, so a missing required column stops the run at Step 2.

### 1. `DataListing-AACMembership.csv` (required, master roster)

Required columns: `MembershipId`, `MembershipStatus`, `NRIC`.

```
Centre1,MembershipId,MembershipStatus,DocumentType,NRIC,Salutation,Surname,FullName,ToAddressAs,NativeName,DOB,Gender,MaritalStatus,BloodGroup,Citizenship,Race,Religion,HomePhone,MobilePhone,OtherPhone,PersonAddressPostalCode,PersonAddressBlockNo,PersonAddressStoreyUnit,PersonAddressStreetName,PersonAddressBuildingName,PersonAddressCountry,ResidesWithinServiceBoundary,PersonAddressTypeOfAccommodation,Address_Ownership,LivingArrangement,OthersLivingArrangement,Language,HighestEducation,CurrentEmploymentStatus,OtherEmploymentStatus,Hobbies,Mobility,LevelOfCares,FinancialStatus,OtherFinancialStatus,NoOfSons,NoOfDaughters,PhysicalHealths,PhysicalHealthOthers,MentalHealths,MentalHealthOthers,SocioEmotional,SocioEmotionalOthers,FollowUps,OtherFollowUps,HealthRemarks,NoNextOfKin,Row_Number,NOKSalutation,NOKFullName,NOKSurname,NOKDocumentType,NOKNRIC,Relationship,TypeOfEmergencyContactPerson,NOKHomePhone,NOKMobilePhone,NOKOtherPhone,NOKPostalCode,NOKBlockNo,NOKStoreyUnitNo,NOKStreetName,NOKBulidingName,NOKCountry,RecordedBy,RegistrationDate,JoinedDate,TerminatedDate,TerminatedBy,TerminationReason,WithdrawnDate,WithdrawnBy,WithdrawnReason,RFID,Remarks,Responder,SelectNOK,DeclarationDate,CreatedOn,CreatedBy,ModifiedOn,ModifiedBy
```

### 2. `DataListing-AACMemberDetails.csv` (required, refresh source)

Required columns: `MembershipId`, `CurrentMembershipStatus`, `RegistrationDocumentNumber`.

```
Centre,MembershipId,CurrentMembershipStatus,JoinedDate,TerminatedDate,RegistrationDocumentType,RegistrationDocumentNumber,Salutation,FullNameNative,ToAddressAs,DateOfBirth,Age,Gender,MaritalStatus,Citizenship,Race,Religion,Phone,ResidesWithinServiceBoundaryStr,NricAddress_PostalCode,NricAddress_BlockNo,NricAddress_UnitNo,FullAddress,NricAddress_Ownership,SpokenLanguage,Mobility,LevelOfCare,IsFaceRegistered,RFID,Hobbies,Remarks,AssignedStaffs,AssignedVolunteers,LastActivity,LastActivityAttendedOn,BnBType,IRConcatenatedString,DeclinedCSStr,AssessmentDate,AUAScore,NextReviewDate,InterRAICSAssessmentDate,InterRAICSSCALES,InterRAICSCAPS,InterRAICSNextReviewDate
```

### 3. Activities CSV (optional, generated from Supabase)

Required columns: `member_registrationdocumentnumber`, `activities_attended_count`.

```
member_registrationdocumentnumber,member_name,cfs_score,activities_attended_count
```

See [Generating the Activities CSV](#generating-the-activities-csv).

## Generating the Activities CSV

Run this in the **Supabase SQL Editor** for the membership database, then export the result as CSV and upload it in Step 1.

**Before running, adjust the two dates** in the `left join` to the reporting period you want to count activities for. The start date is inclusive and the end date is exclusive, so for a full quarter use the first day of the next quarter as the end date.

```sql
select
  m."registrationdocumentnumber" as "member_registrationdocumentnumber",
  m."fullnamenative" as "member_name",
  max(a."cfs") as "cfs_score",
  count(a."id") as "activities_attended_count"
from public."aac_member_details" m
left join public."activities" a
  on a."registrationdocumentnumber" = m."registrationdocumentnumber"
  and a."activitydatedisplay" >= date '2026-04-01'   -- period start (inclusive)
  and a."activitydatedisplay" < date '2026-07-01'    -- period end (exclusive)
where m."resideswithinserviceboundarystr" = true
  and m."dateofbirth" is not null
  and extract(year from age(current_date, m."dateofbirth")) >= 60
group by m."registrationdocumentnumber", m."fullnamenative"
order by "activities_attended_count" desc, "member_registrationdocumentnumber" asc;
```

Example periods:

| Period | Start (inclusive) | End (exclusive) |
|---|---|---|
| Q2 2026 (the dates above) | `2026-04-01` | `2026-07-01` |
| Q3 2026 | `2026-07-01` | `2026-10-01` |
| Q4 2026 | `2026-10-01` | `2027-01-01` |

Notes on this query:

- It only returns members who live within the service boundary, have a date of birth, and are aged 60 or over. Anyone outside that group won't appear in the file and gets an `ActivityCount` of `0`.
- The left join means in-scope members with no activities in the period are returned with a count of `0`.
- The age filter uses `current_date`, so it reflects the day you run it, not the end of the reporting period.

## Output: `Final_Merged_Members.csv`

Columns, in order:

```
MembershipStatus,JoinedDate,NRIC,FullName,ToAddressAs,DOB,Age,Gender,MaritalStatus,Citizenship,Race,Religion,HomePhone,MobilePhone,ResidesWithinServiceBoundary,PersonAddressPostalCode,PersonAddressBlockNo,PersonAddressStoreyUnit,PersonAddressStreetName,PersonAddressBuildingName,PersonAddressCountry,Language,CurrentEmploymentStatus,Hobbies,Remarks,HouseholdID,ActivityCount
```

`MembershipId` is used internally for matching but is not written to the output.

## Merge rules

**Matching.** Each Membership row is matched to a Member Details row by `MembershipId`. If that fails, it falls back to NRIC (`NRIC` ↔ `RegistrationDocumentNumber`, normalised). Member Details rows with no Membership match are ignored. Membership defines the row set.

**Overwrite rule.** For fields present on both sides, a non-blank Member Details value overwrites the Membership value. A blank one keeps the Membership value. Blank means empty, whitespace, or `NA` / `N/A` / `null` / `nil` / `-`.

**Field mapping (Membership ← Member Details):**

| Output column | Membership column | Member Details column |
|---|---|---|
| MembershipStatus | MembershipStatus | CurrentMembershipStatus |
| NRIC | NRIC | RegistrationDocumentNumber |
| FullName | FullName | FullNameNative |
| ToAddressAs | ToAddressAs | ToAddressAs |
| DOB | DOB | DateOfBirth |
| Gender, MaritalStatus, Citizenship, Race, Religion, Hobbies | same name | same name |
| ResidesWithinServiceBoundary | ResidesWithinServiceBoundary | ResidesWithinServiceBoundaryStr |
| PersonAddressPostalCode | PersonAddressPostalCode | NricAddress_PostalCode |
| PersonAddressBlockNo | PersonAddressBlockNo | NricAddress_BlockNo |
| PersonAddressStoreyUnit | PersonAddressStoreyUnit | NricAddress_UnitNo |
| Language | Language | SpokenLanguage |
| MobilePhone, CurrentEmploymentStatus | same name | (none) |
| JoinedDate, Remarks | (none) | same name |

**Status filter.** After the overwrite rule is applied, members whose `MembershipStatus` is **Terminated** or **Withdrawn** (case-insensitive) are removed. Other statuses such as `Joined` and `Pending` pass through. The summary reports the two removal counts separately.

**Age.** Current year minus birth year, as a whole number. Missing or unparseable dates of birth leave Age blank, and the row is kept. This is a year-only calculation, so it can be one year high for members whose birthday hasn't happened yet this year.

### Phone numbers

Member Details' `Phone` field carries `(H)` / `(M)` tags (for example `(M) 94769266`). The tool splits each value and routes it to `HomePhone` or `MobilePhone` based on its tag, with the tag removed. Membership's own `HomePhone` and `MobilePhone` values are the fallback when Member Details doesn't supply that tag, and a Member Details value wins when both exist.

### Address resolution

Street name and building name are resolved in this order:

1. Parsed from Member Details' `FullAddress` (block and unit are stripped using `NricAddress_BlockNo` and `NricAddress_UnitNo`).
2. Membership's own `PersonAddressStreetName` / `PersonAddressBuildingName`.
3. OneMap Search API, only for rows still missing a street name after steps 1 and 2.

`PersonAddressCountry` is taken from Membership. OneMap lookups are cached per postal code, paced to stay under OneMap's rate limit, and retried on HTTP 429. Failures go to `Failed_Address_Lookup.csv` with a `FailureReason` and `Detail`.

### HouseholdID

```
PersonAddressPostalCode _ PersonAddressBlockNo _ PersonAddressStoreyUnit
```

For example `360004_4_08-10`. If the storey/unit is blank (landed property), `LANDED` is used, for example `348126_24_LANDED`.

### ActivityCount

Matched on NRIC against `member_registrationdocumentnumber` in the Activities CSV. No match, or no Activities CSV uploaded, gives `0`. If the same NRIC appears more than once in the Activities CSV, the first row is used and the NRIC is listed as a data issue in the summary.

## Known data-quality points

- **Bad source values pass through.** The tool doesn't correct wrong entries in the source exports (for example a postal code typed into `ToAddressAs`). Fix those at the source.
- **Duplicate IDs.** Duplicate `MembershipId` values in either file are flagged at validation. The first occurrence is used.
- **`ActivityCount = 0` is ambiguous.** It can mean attended nothing in the period, or not in the Activities CSV at all (for example outside the service boundary or under 60).

## Not included

No backend, database, or saved state between runs. Each run starts fresh from whatever files you upload.
