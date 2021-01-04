## Gestation check

### Background
[Newborn screening for congenital hypothyroidism](https://www.gov.uk/government/publications/congenital-hypothyroidism-screening-laboratory-handbook) relies on the measurement of Thyroid Stimulating Hormone (TSH). However, interpretation of results also [depends on the baby's gestational age](https://www.gov.uk/government/publications/congenital-hypothyroidism-screening-laboratory-handbook/a-laboratory-guide-to-newborn-blood-spot-screening-in-the-uk-for-congenital-hypothyroidism#pre-term), and a repeat sample is requested for babies who are born at < 32 weeks gestation.

Laboratories receive demographic information on a baby's gestational age on the blood spot sample or via the [Newborn Blood Spot Screening Failsafe Solution](https://www.gov.uk/government/publications/newborn-blood-spot-screening-failsafe-solution-user-guide/newborn-blood-spot-screening-failsafe-solution-user-guide#:~:text=The%20NBSFS%20is%20an%20IT,in%20the%20laboratory%20on%20time).

This information may be incorrectly recorded by the midwife or incorrectly transcribed from the blood spot sample.

Since almost all babies who are born at less than 32 weeks gestation have a birth weight of less than 2000g, the report aims to detect potentially incorrectly recorded gestation by comparing birth weight and gestational age.

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
