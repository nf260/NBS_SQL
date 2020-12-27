# End of day checks

Ideally these errors would be identified during data entry (and as far as possible I have attempted to prevent or warn about ). However, due to limitations of the current system, this was not always possible.

The following reports are scheduled to print at the end of each working day and attempt to highlight possible errors that can then be correted.


## End of day checklist

### Requirements

### SQL

```sql
SELECT
COUNT(A.EPISODE_NUMBER) AS TOTAL_EPISODES
, MIN (A.EPISODE_NUMBER) AS FIRST_EPISODE_NUMBER
, MAX (A.EPISODE_NUMBER) AS LAST_EPISODE_NUMBER
, SUM(CASE WHEN E.MaternityUnit->Description='Maternity Unit, Hinchingbrooke' THEN 1 ELSE 0 END) as HIN
, SUM(CASE WHEN E.MaternityUnit->Description='Maternity Unit, Ipswich' THEN 1 ELSE 0 END) as IPS
, SUM(CASE WHEN E.MaternityUnit->Description='Maternity Unit, James Paget' THEN 1 ELSE 0 END) as JP
, SUM(CASE WHEN E.MaternityUnit='M4962' THEN 1 ELSE 0 END) as KL
, SUM(CASE WHEN E.MaternityUnit->Description='Maternity Unit, Norwich' THEN 1 ELSE 0 END) as NOR
, SUM(CASE WHEN E.MaternityUnit->Description='Maternity Unit, Peterborough' THEN 1 ELSE 0 END) as PET
, SUM(CASE WHEN E.MaternityUnit->Description='Maternity Unit, Rosie' THEN 1 ELSE 0 END) as ROS
, SUM(CASE WHEN E.MaternityUnit->Description='Maternity Unit, West Suffolk' THEN 1 ELSE 0 END) as WS

FROM 
Dls_Legacy.Episode_Day_Book A
JOIN Dls.Episode E ON (A.Episode_Number = E.Episode)

WHERE
A.DAY_BOOK_DATE = '<%Date|Date|T|D%>'

``` 

### Output

* Insert file here

## A - Gestation check

### Requirements

### SQL

```sql
SELECT
A.EPISODE_NUMBER
, A.DAY_BOOK_DATE
, A.RQ_PTR->PMI_LINK-> NAME
, A.RQ_PTR->PMI_LINK-> NAT_NUMBER
, A.RQ_PTR->PMI_LINK-> DATE_OF_BIRTH
, DATEDIFF(D,%odbcout(A.RQ_PTR->PMI_LINK->DATE_OF_BIRTH), %odbcout(A.RQ_PTR-> DATE_COLLECTED)) AS Age
, PN.GestationAgeByExam 
, (PN.GestationAgeByExam/7) as GA_weeks
, PN.BirthWeight
, CASE
 WHEN (PN.GestationAgeByExam IS NULL AND DATEDIFF(D,%odbcout(A.RQ_PTR->PMI_LINK->DATE_OF_BIRTH), %odbcout(A.RQ_PTR-> DATE_COLLECTED)) < 28) THEN 'Gestation not entered'
 WHEN (PN.GestationAgeByExam < 168 AND DATEDIFF(D,%odbcout(A.RQ_PTR->PMI_LINK->DATE_OF_BIRTH), %odbcout(A.RQ_PTR-> DATE_COLLECTED)) < 28) THEN 'Extremely premature - check'
 WHEN (PN.GestationAgeByExam > 223 AND PN.BirthWeight < 2001) THEN 'Low birth weight but gestation ≥ 32 weeks - check'
 WHEN (PN.GestationAgeByExam < 224 AND PN.BirthWeight > 2000) THEN '< 32 weeks but birth weight > 2000 g - check'
 WHEN (PN.BirthWeight > 5500) THEN 'Very high birth weight - check'
 WHEN (PN.GestationAgeByExam > 301) THEN 'Gestation > 43 weeks - check'
END AS Comment

FROM 
Dls_Legacy.Episode_Day_Book A
JOIN 		Dls_Legacy.Episode DLE 	ON (A.Episode_Number = DLE.Episode_Number)
, Common.Patient P
LEFT OUTER JOIN Common.PATIENT_NEONATAL PN ON (P.PatientIRN = PN.PATIENTIRN)

WHERE
A.RQ_PTR->PMI_LINK->INT_UN = P.PATIENTIRN
AND A.DAY_BOOK_DATE >= '<%Date1|Start Date|T|D%>'
AND A.DAY_BOOK_DATE <= '<%Date2|End Date|T|D%>'
AND A.RQ_PTR->PMI_LINK-> NAME NOT LIKE 'UK NEQAS%'
AND A.RQ_PTR->PMI_LINK-> NAME NOT LIKE 'CDC %'
AND (
(PN.GestationAgeByExam IS NULL AND DATEDIFF(D,%odbcout(A.RQ_PTR->PMI_LINK->DATE_OF_BIRTH), %odbcout(A.RQ_PTR-> DATE_COLLECTED)) < 28)
OR
(PN.GestationAgeByExam < 168 AND DATEDIFF(D,%odbcout(A.RQ_PTR->PMI_LINK->DATE_OF_BIRTH), %odbcout(A.RQ_PTR-> DATE_COLLECTED)) < 28)
OR
(PN.GestationAgeByExam > 223 AND PN.BirthWeight < 2001)
OR
(PN.GestationAgeByExam < 224 AND PN.BirthWeight > 2000)
OR
(PN.BirthWeight > 5500)
OR
(PN.GestationAgeByExam > 301)
)

GROUP BY 
A.Episode_Number

ORDER BY
A.Episode_Number
```



## Child health check

### Requirements

### SQL

```sql
SELECT
A.EPISODE_NUMBER
, COUNT(A.EPISODE_NUMBER) AS TOTAL_EPISODES
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




## Multiple births check

### Requirements
Identification errors are more common for twins, triplets, or other multiple births in newborn screening than for singleton births.

The report aims to identify twins (
### SQL

```sql
SELECT
A.EPISODE_NUMBER
, A.DAY_BOOK_DATE
, A.RQ_PTR->PMI_LINK-> NAME
, A.RQ_PTR->PMI_LINK-> NAT_NUMBER
, A.RQ_PTR->PMI_LINK-> DATE_OF_BIRTH
, DATEDIFF(D,%odbcout(A.RQ_PTR->PMI_LINK->DATE_OF_BIRTH), %odbcout(A.RQ_PTR-> DATE_COLLECTED)) AS Age
, PN.GestationAgeByExam 
, (PN.GestationAgeByExam/7) as GA_weeks
, PN.BirthWeight
, PN.BirthRank
, PN.BirthNumber


FROM 
Dls_Legacy.Episode_Day_Book A
JOIN 		Dls_Legacy.Episode DLE 	ON (A.Episode_Number = DLE.Episode_Number)
, Common.Patient P
LEFT OUTER JOIN Common.PATIENT_NEONATAL PN ON (P.PatientIRN = PN.PATIENTIRN)

WHERE
A.RQ_PTR->PMI_LINK->INT_UN = P.PATIENTIRN
AND A.DAY_BOOK_DATE >= '<%Date1|Start Date|T|D%>'
AND A.DAY_BOOK_DATE <= '<%Date2|End Date|T|D%>'
AND A.RQ_PTR->PMI_LINK-> NAME NOT LIKE 'UK NEQAS%'
AND A.RQ_PTR->PMI_LINK-> NAME NOT LIKE 'CDC %'
AND (
PN.BirthNumber NOT IN ('1')
OR
A.RQ_PTR->PMI_LINK-> NAME LIKE '%TWIN%'
OR
A.RQ_PTR->PMI_LINK-> NAME LIKE '%TRIPLET%'
OR
A.RQ_PTR->PMI_LINK-> NAME LIKE '%QUADRUPLET%'
)

GROUP BY 
A.Episode_Number

ORDER BY
A.Episode_Number
```

## NHS number check

```sql
SELECT DISTINCT
A.EPISODE_NUMBER
, A.DAY_BOOK_DATE
, CLP.NAME
, CLP.NAT_NUMBER
, DLE.Date_Collected
, DLE.Date_Requested
, TR.Spec_Comment
, TR.Test_Set
, E.Card_Number
, CASE
WHEN CLP.NAT_NUMBER IS NOT NULL AND LEFT (CLP.NAT_NUMBER,1) NOT IN ('7') THEN 'NHS number does not start with 7'
WHEN (Spec_Type IN ('C') AND E.Card_Number IS NULL) THEN 'Card number missing'
WHEN (
(Spec_Type IN ('C') AND CAST(E.Card_Number AS INT) > 1599999999)
OR
(Spec_Type IN ('C') AND CAST(E.Card_Number AS INT) < 1500000000)
OR
(Spec_Type IN ('C') AND ISNUMERIC(E.Card_Number) = 0)
) THEN 'Card number error'
WHEN (
(Spec_Type IN ('C') AND CAST(E.Card_Number AS INT) < 1523402300)
) THEN 'Check card expiry date - add as specimen comment if expired (and card not already authorised)'

END AS Explanation

FROM 
Dls_Legacy.Episode_Day_Book A
JOIN 		Dls.Episode E		ON (A.Episode_Number = E.Episode)
JOIN 		Dls_Legacy.Episode DLE 	ON (A.Episode_Number = DLE.Episode_Number)
JOIN 		Dls_Legacy.Tests_Requested TR 	ON (A.Episode_Number=TR.Episode_Number)
LEFT OUTER JOIN 	Common_Legacy.Patient CLP	ON (CLP.Int_Un = DLE.Int_Un)


WHERE
A.DAY_BOOK_DATE >= '<%Date1|Start Date|T-7|D%>'
AND A.DAY_BOOK_DATE <= '<%Date2|End Date|T|D%>'
AND CLP.NAME NOT LIKE 'UK NEQAS%'
AND CLP.NAME NOT LIKE 'CDC %'
AND TR.Test_Set NOT IN ('N099','N940')
AND
(
(CLP.NAT_NUMBER IS NOT NULL AND LEFT (CLP.NAT_NUMBER,1) NOT IN ('7'))
OR
(Spec_Type IN ('C') AND E.Card_Number IS NULL)
OR
(Spec_Type IN ('C') AND CAST(E.Card_Number AS INT) > 1599999999)
OR
(Spec_Type IN ('C') AND CAST(E.Card_Number AS INT) < 1500000000)
OR
(Spec_Type IN ('C') AND ISNUMERIC(E.Card_Number) = 0)
--- CARDS WITH SERIAL NUMBER AFTER 1523402300 HAVE EXPIRY DATE 31/12/2020
OR
(Spec_Type IN ('C')
AND (TR.Spec_comment IS NULL OR TR.Spec_comment NOT LIKE '0311%')
AND (TR.Spec_comment IS NULL OR TR.Spec_comment NOT LIKE '02%')
AND TR.Test_Set NOT IN ('N801')
AND CAST(E.Card_Number AS INT) < 1523402300)
)

ORDER BY
Explanation,
A.Episode_Number
```
