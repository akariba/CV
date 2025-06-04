https://usosweb.wne.uw.edu.pl/kontroler.php?_action=katalog2/przedmioty/pokazPrzedmiot&kod=2400-QFU1MMF

Input: (IssuerID, ProductType ["CDS"/"Bond"], Seniority, Currency, Region, Sector, Entity-specific flag)
   |
   |--> Step 1: Identify Entity
   |--> Step 2: Determine FCY/LCY (foreign/local currency)
   |--> Step 3: Identify Country, Sector, Group, SubGroup
   |--> Step 4: Extract Ratings (S&P, Moody’s, Fitch)
   |         └──> Use SD_RATING_IDX, SD_TRM_OVERRIDE, MD_SECURITY_RATINGS
   |
   |--> Step 5: Apply Overrides (from SD_TRM_CROVERRIDE or SD_TRM_OVERRIDE)
   |
   |--> Step 6: Derive Internal Rating (from SD_SCI_MAIN)
   |         └──> Standardize using SD_RATING_IDX and sector-specific mappings
   |
   |--> Step 7: Pick final external rating (rating precedence, default logic)
   |
   |--> Step 8: Map entity to SN/CDS curve using multi-key match strategy
             └──> Wildcard fallback → Generic curves → Region/Country/Sector/Rating match


🧩 DETAILED LOGIC BREAKDOWN
📌 1. Entity Identification
Functions: getEntityIssuerRating, getCDSIssuerRating, getBondIssueRating

Sources:

SD_ENTITY_ID

SD_CDS_ID

SD_ENTITY.MASTER

SD_SCI_MAIN

Goal: Establish the master ID for the entity and associated currency

getCDSEntity cdsName = ...
→ entityNick ← tokenize the name
→ get MASTERID from CVA.ENTITY.MASTER
→ get CCY from SD_CDS

🌍 2. Country (FCY/LCY) Determination
Logic:

Uses SD_TRM_CROVERRIDE and SD_MICR_REGION

Determines if the issuer’s country matches currency

Returns "FCY" or "LCY"

if isLocal then LCY else FCY

🏢 3. Sector, Group, SubGroup Classification
Function: getIndustry

Sources:

SD_INDUSTRY

SD_SCI_MAIN

SD_SCI_MAIN_EX

Goal: Map issuer to industry levels (L1, L2, L3), with fallbacks via Bloomberg/Overrides

🎯 4. Extract Ratings
Functions:

getBondSpRating

getBondMdyRating

getBondCRIssueRating
getRatingForEntity

Sources:

MD_SECURITY_RATINGS

SD_RATING_IDX

SD_TRM_OVERRIDE

Fields Used:

For Sovereign: SP_LT_LC_IR, MDY_LT_LC_IR, etc.

For Non-Sovereign: SP_LONG_RATING, MDY_LT_FC_DEBT, etc

rating = ratingPrecedence spRating mdyRating fitchRating

🛑 5. Apply Overrides
Overrides from:

SD_TRM_CROVERRIDE (entity-specific)

SD_TRM_OVERRIDE (BB_UNIQUE level override)

Checks if:

"RATING_OVERRIDE" is defined → use it instead of raw rating

"INDUSTRY" override tag exists → affects sector/group

🧠 6. Internal Rating Logic
Function: getInternalRating

Steps:

Get internal rating for entity from SD_SCI_MAIN

If found, standardize using standardizeInternalRating

Mapping depends on sector:

Banks → "INT-BANKS"

Government → "INT-SOVEREIGN"

Others → "INT-CORPS/MBFI"

Standardization Source: SD_RATING_IDX

🧪 7. Rating Precedence and Defaulting
Rating selection order:

Use override if available

Else use median of external ratings

If not available, fallback:

"NR" or blank → default "b" rating

Default logic:

case rating1 of
  Just r1 | r1 == "NR" || r1 == "" → "b"
  Nothing                         → "b"

          ┌──────────────┐
          │ Input Params │
          └────┬─────────┘
               │
               ▼
   ┌────────────────────────┐
   │ Entity Identification  │
   └────┬───────────────────┘
        ▼
   ┌────────────────────────┐
   │ Country Risk FCY/LCY   │
   └────┬───────────────────┘
        ▼
   ┌────────────────────────┐
   │ Industry & Sector Map  │
   └────┬───────────────────┘
        ▼
   ┌────────────────────────┐
   │ Get SP/Moody/Fitch     │
   └────┬───────────────────┘
        ▼
   ┌────────────────────────┐
   │ Apply Overrides        │
   └────┬───────────────────┘
        ▼
   ┌────────────────────────┐
   │ Internal Rating Fetch  │
   └────┬───────────────────┘
        ▼
   ┌────────────────────────┐
   │ Rating Precedence      │
   │ & Defaulting ("b")     │
   └────┬───────────────────┘
        ▼
   ┌────────────────────────┐
   │ Waterfall Curve Match  │
   └────┬───────────────────┘
        ▼
   ┌────────────────────────┐
   │ Final Curve Selection  │
   └────────────────────────┘


→ if curveType == "GN" then use ZSPD
→ if curveType == "SN" then use CRSPD

"SR_GEN_CORP_XXX_USD_SN" ++ rating

=IFERROR(
  INDEX(issuer_details!H:H, MATCH(Sheet20!L2, issuer_info_map!E:E, 0)),
  INDEX(issuer_details!H:H, MATCH(Sheet20!I2, issuer_details!F:F, 0))
)




🧾 Meeting Summary – Equity Onboarding, RISK-3 Integration, and XVA VAR Validation
Date: [Assumed recent]
Attendees: TRM, Strat, Risk, AP, Quant, GMV teams
Key Topics:

Equity onboarding challenges

RISK-3 batch stage development

XVA VAR integration for equity trades

Market data readiness and validation strategy

Governance and go-live timeline planning

📍 1. Production Onboarding Issues
🔹 Aging Stage
Issue: Enabled in Production but not in Development, causing discrepancies during onboarding.

Impact: Unexpected behavior for newly onboarded clients; under investigation.

Action: Investigate configuration alignment between environments.

🔹 Market Data Anomalies
Issue: Minor market data issues observed, nature unclear (transient vs structural).

Impact: Could affect pricing/sensitivity accuracy.

Action: Monitor for reoccurrence and raise fixes if persistent.

📍 2. Sensitivities vs VAR Behavior
Observation: Currently only sensitivities are run in production batches.

Critical Note: Trades onboarded for sensitivities will automatically be included in XVA VAR if that functionality is enabled.

Implication: Careful coordination is needed to avoid including unvalidated trades in VAR.

📍 3. RISK-3 Batch Onboarding – Equity and Others
🔹 Objective
Introduce a new stage (RISK-3) in the risk batch pipeline to output additional risk sensitivities not covered in RISK-1 or RISK-2.

🔹 Expected Outputs
File Type	Sensitivities Included
RISK-Result	IN-VEGA, IR-VEGA (exotic ccy), FX-Gamma, EQT-VEGA (bucketed)
RISK-View	IR-Delta (G10 IMM Projections)

🔹 EQ-Delta Location
EQ-Delta remains in RISK-1 and is not duplicated in RISK-3.

🔹 Status
Owner: Rupel

Action: Running ~1,500-trade test batch from multiple product types.

ETA for Sample Files: By end-of-day; will be distributed tomorrow.

Purpose: Validate format, consistency, measure names before AP ingestion.

🔹 Decision
RISK-3 general enhancements (e.g., inflation) will be treated separately.

Sven to set up a dedicated call for RISK-3 format upgrades beyond equity.

📍 4. AP Team Reporting Integration
🔹 Objective
Update existing AP dashboards and risk reports to include equity sensitivities from RISK-3.

🔹 Clarifications
Sample CSV Report (Bespoke Credit Solutions) was shared for format reference only.

Final production reports will be generated by AP, based on feeds from SEPA / RISK-ETL.

Report type: Enhancement to existing, not a new standalone report.

🔹 Requirements & Timeline
Equity sample data needed within 2 weeks to:

Validate format and completeness.

Raise tickets for July development cycle.

Milestone	Date Target
Go-Live	30 Sept 2025
UAT Completion	Mid-August 2025
Dev Handover	Early July 2025

📍 5. Equity XVA VAR Integration – Scope & Governance
🔹 Context
Current XVA VAR does not include equity trades.

Inclusion of equity in sensitivities → automatic inclusion in XVA VAR if active.

🔹 Governance Implications
Scope Expansion: Inclusion of equity is not covered by current XVA VAR model approval.

GMV Involvement:

Next Week: Meet Jamie from GMV to classify this as major or minor model change.

Additional TDD (Technical Design Document) review may be required.

Assumption: Approval for XVA VAR does not automatically cover equity instruments.

📍 6. Market Data Considerations
🔹 Requirements for XVA VAR
Unlike Fair Value VAR, XVA VAR requires volatilities (e.g., for TRS/options) to be available and shocked.

Dependencies:

Equity vols must be stored and perturbed in HisSim.

Shocks are required even if those volatilities are not used in Fair Value.

🔹 Risk of Data Gaps
Illiquid equity products or FX pairs may lack vol or shock history.

Requires validation during testing and quant review phases.

📍 7. Validation & Testing Requirements
🔹 For Equity XVA VAR Sign-Off
Likely need at least 5 COP-day runs with equity exposure.

Output must be verified across:

Vol coverage and perturbation logic.

Reasonability of VAR metrics.

Comparability with Fair Value runs.

🔹 SimIE Model Relevance
SimIE (regression-based) is used for equity modeling similar to credit.

Confirm applicability for equity XVA VAR (currently unclear).

🔹 Actions
Aladin to prepare test plan.

Input: Awaiting GMV guidance on expected validation scope.

📍 8. Action Item Tracker
Owner	Task	Due Date
Rupel	Deliver RISK-3 test output files (incl. EQ Vega)	Tomorrow
Roy & Alvin	Validate TRM equity report requirements against RISK-3 data	Within 2 weeks
Sven	Schedule separate call for general RISK-3 enhancements	ASAP
AP Team	Include equity report enhancements in July release scope	Early June
Aladin	Draft test plan for equity XVA VAR validation	Post-GMV meeting
Jamie/GMV	Determine if equity XVA VAR needs new approval	Next Week
Meg / Namin	Confirm equity vols are available and shocked in HisSim	ASAP

🧭 Closing Summary
RISK-3 batch work is progressing on schedule.

Equity onboarding introduces non-trivial implications for XVA VAR and risk governance.

Sample data delivery within the next 24 hours is a key dependency for downstream validation and reporting integration.

The go-live timeline of 30 September 2025 remains feasible but depends on early July ticket creation and mid-August UAT completion.

GMV validation will determine whether model sign-off for equity XVA VAR requires formal re-approval.

=IF(
  MID(R2,FIND("CDS_",R2)+4,2) = MID(S2,FIND("CDS_",S2)+4,2),
  "OK",
  "Mismatch: System 1 = " & MID(R2,FIND("CDS_",R2)+4,2) & "; System 2 = " & MID(S2,FIND("CDS_",S2)+4,2)
)
