# End of day checklist

## Requirements

## SQL

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

## Output

* Insert file here
