## Child health check

### Background
Newborn blood spot screening results are reported to child health records departments based on the clinical commissioning group (CCG) for the child's home address.

Where the home address is recorded in the laboratory IT system, and the CCG is within the laboratory's normal catchment area, then the system usually automatically assigns the sample to the correct child health department.

The report highlights samples where the child health department has not been entered or is potentially incorrect. This may occur due to the following reasons:
* The postcode is not recognised, for example due to the address being a new development. As an alternative, the report provides the CCG for the GP practice, if available.
* The child lives outside the laboratory's normal catchment area. Normal process in newborn screening is for the results to be sent to the child's local screening laboratory - the report provides the relevant code, where possible.


### SQL

```sql
SELECT
A.EPISODE_NUMBER
, COUNT(A.EPISODE_NUMBER) AS TOTAL_EPISODES
### If there are a large number of records this may signal that postcode tables need to be updated.
, CASE
WHEN COUNT(A.EPISODE_NUMBER) > 5 THEN 'Large number of episodes on end of day check - please inform Nick Flynn'
ELSE ''
END AS WARNING_MESSAGE
, A.DAY_BOOK_DATE
, A.RQ_PTR->PMI_LINK-> NAME
, A.RQ_PTR->PMI_LINK-> NAT_NUMBER
, DLE.Doc_Ref_Ptr->Hospital_Doctor_Code as CHRD
, DLE.Doc_Ref_Ptr->Doctor_Name 
, HCP.PCTCode AS GP_CCG
, PC.Primary_Care_Trust->Code AS PATIENT_ADDRESS_CCG
, PC.Address_Postcode
, CASE
WHEN (PC.Primary_Care_Trust->Code IS NOT NULL
          AND PC.Primary_Care_Trust->Code IN('06H','06Y','06W','07J','06V','06L','07K','06M','07H','06T','06Q','07G','99E','99F','99G','06K','06F','06N','06P','04F','03T','04D','99D','04Q','13Q')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code <> PC.Primary_Care_Trust->Code
          ) THEN PC.Primary_Care_Trust->Code
WHEN (PC.Primary_Care_Trust->Code IN('04X','04Y','05A','05C','05D','05F','05G','05H','05J','05L','05N','05P','05Q','05R','05T','05V','05W','05X','05Y','06A','06D','13P')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('BIRM')
          ) THEN 'BIRM'
WHEN (PC.Primary_Care_Trust->Code IN('07L','07M','07P','07R','07T','07W','07X','07Y','08C','08D','08E','08F','08G','08H','08M','08N','08V','08W','08Y','09A')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('GOSH')
          ) THEN 'GOSH'
WHEN (PC.Primary_Care_Trust->Code IN('07N','07Q','08A','08K','08L','08Q','09C','09D','09E','09F','09J','09P','09W','09X','10A','10D','10E','99J','99K')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('SET')
          ) THEN 'SET'
WHEN (PC.Primary_Care_Trust->Code IN('02P','02Q','02X','03H','03K','03L','03N','03V','03W','03X','03Y','04C','04E','04G','04H','04J','04K','04L','04M','04N','04R','04V')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('SHEFF')
          ) THEN 'SHEFF'
WHEN (PC.Primary_Care_Trust->Code IS NOT NULL
          AND PC.Primary_Care_Trust->Code NOT IN('06H','06Y','06W','07J','06V','06L','07K','06M','07H','06T','06Q','07G','99E','99F','99G','06K','06F','06N','06P','04F','03T','04D','99D','04Q','13Q')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('BIRM','BRIS','GOSH','LEEDS','LIV','MAN','NEW','OXF','POR','SCOT','SET','SHEFF','SWT','WALES')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code <> PC.Primary_Care_Trust->Code
          ) THEN 'Enter external screening lab code'
WHEN (PC.Primary_Care_Trust->Code IS NULL
          AND HCP.PCTCode IN('06H','06Y','06W','07J','06V','06L','07K','06M','07H','06T','06Q','07G','99E','99F','99G','06K','06F','06N','06P','04F','03T','04D','99D','04Q','13Q')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code <> HCP.PCTCode 
          ) THEN HCP.PCTCode
ELSE 'Refer to clinical scientist'
END
AS RequiredCode


FROM 
Dls_Legacy.Episode_Day_Book A
JOIN 		Dls.Episode E		ON (A.Episode_Number = E.Episode)
JOIN 		Dls_Legacy.Episode DLE 	ON (A.Episode_Number = DLE.Episode_Number)
, Common.Patient P
LEFT OUTER JOIN Common.HCPPractice HCP ON (P.GPPracticeCode = HCP.PracticeCode)
LEFT OUTER JOIN Common.Patient_Contact PC ON (P.PatientIRN = PC.PATIENTIRN AND PC.%ID = P.DisplayHomeContact)

WHERE
A.RQ_PTR->PMI_LINK->INT_UN = P.PATIENTIRN
AND A.DAY_BOOK_DATE >= '<%Date1|Start Date|T-7|D%>'
AND A.DAY_BOOK_DATE <= '<%Date2|End Date|T|D%>'
AND A.RQ_PTR->PMI_LINK-> NAME NOT LIKE 'UK NEQAS%'
AND A.RQ_PTR->PMI_LINK-> NAME NOT LIKE 'CDC %'
AND
(
(DLE.Doc_Ref_Ptr->Hospital_Doctor_Code IN ('UNK'))
OR
(DLE.Doc_Ref_Ptr->Hospital_Doctor_Code IS NULL)
OR
-- NOT PATIENT ADDRESS CCG
(PC.Primary_Care_Trust->Code IS NOT NULL
          AND PC.Primary_Care_Trust->Code IN('06H','06Y','06W','07J','06V','06L','07K','06M','07H','06T','06Q','07G','99E','99F','99G','06K','06F','06N','06P','04F','03T','04D','99D','04Q','13Q')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code <> PC.Primary_Care_Trust->Code)
OR
-- INCORRECT EXTERNAL SCREENING LAB (BIRMINGHAM)
(PC.Primary_Care_Trust->Code IN('04X','04Y','05A','05C','05D','05F','05G','05H','05J','05L','05N','05P','05Q','05R','05T','05V','05W','05X','05Y','06A','06D','13P')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('BIRM')
)
OR
-- INCORRECT EXTERNAL SCREENING LAB (GOSH)
(PC.Primary_Care_Trust->Code IN('07L','07M','07P','07R','07T','07W','07X','07Y','08C','08D','08E','08F','08G','08H','08M','08N','08V','08W','08Y','09A')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('GOSH')
)
OR
-- INCORRECT EXTERNAL SCREENING LAB (SE THAMES)
(PC.Primary_Care_Trust->Code IN('07N','07Q','08A','08K','08L','08Q','09C','09D','09E','09F','09J','09P','09W','09X','10A','10D','10E','99J','99K')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('SET')
)
OR
-- INCORRECT EXTERNAL SCREENING LAB (SHEFFIELD)
(PC.Primary_Care_Trust->Code IN('02P','02Q','02X','03H','03K','03L','03N','03V','03W','03X','03Y','04C','04E','04G','04H','04J','04K','04L','04M','04N','04R','04V')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('SHEFF')
)
OR
-- OTHER EXTERNAL SCREENING LAB AREA
 (PC.Primary_Care_Trust->Code IS NOT NULL
          AND PC.Primary_Care_Trust->Code NOT IN('06H','06Y','06W','07J','06V','06L','07K','06M','07H','06T','06Q','07G','99E','99F','99G','06K','06F','06N','06P','04F','03T','04D','99D','04Q','13Q')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code NOT IN('BIRM','BRIS','GOSH','LEEDS','LIV','MAN','NEW','OXF','POR','SCOT','SET','SHEFF','SWT','WALES')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code <> PC.Primary_Care_Trust->Code
          )
OR
-- PATIENT ADDRESS CCG NOT AVAILABLE AND NOT EQUAL GP CCG
(PC.Primary_Care_Trust->Code IS NULL
          AND HCP.PCTCode IN('06H','06Y','06W','07J','06V','06L','07K','06M','07H','06T','06Q','07G','99E','99F','99G','06K','06F','06N','06P','04F','03T','04D','99D','04Q','13Q')
          AND DLE.Doc_Ref_Ptr->Hospital_Doctor_Code <> HCP.PCTCode
          )
)

ORDER BY
A.Episode_Number, Primary_Care_Trust
```
