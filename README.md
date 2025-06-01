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




