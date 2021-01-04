## Multiple births check

### Background
Identification errors in newborn blood spot screening are more common for twins, triplets, or other multiple births than for singleton births.

The report aims to identify twins, triplets etc. for manual checking.

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
