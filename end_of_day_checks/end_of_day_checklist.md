# End of day checklist

## Background

Information recorded in the laboratory IT system must be accurate to ensure that every child’s newborn blood spot screening result is correctly interpreted and reported. Previously, a list of data from around 100 samples was printed at the end of every day and manually checked to identify errors. This process was time consuming and errors were often missed.

The end of day checklist lists several custom SQL reports that were developed to identify potential data entry errors. Reports are printed at the end of each working day. This system targeted checking to a small subset of samples and improved the efficiency and accuracy of error detection.

Ideally these errors would be identified during data entry (and as far as possible I have attempted to prevent or warn about ). However, due to limitations of the current system, this was not always possible.

The end of day checklist provides some basic information on the number of samples received in total, and from each maternity unit that usually sends samples to the Cambridge laboratory (to identify any transit issues). It also provides a simple checklist to confirm that each report has been checked.

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
