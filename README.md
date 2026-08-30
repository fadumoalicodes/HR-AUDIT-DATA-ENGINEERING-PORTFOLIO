# T-SQL Data Engineering Portfolio

This repository contains a master collection of business reporting scripts built inside Microsoft SQL Server (T-SQL). It shows how to combine split workforce tables, clean data entry errors, and calculate advanced payroll metrics row-by-row without slowing down the server.

### Technical Skills
* **Database Language:** T-SQL (Microsoft SQL Server)
* **Core Skills:** Stored Procedures, Temporary Tables,  Views, Multi-Table JOINS, Chained CTEs, Nested Subqueries, UNION ALL, Window Functions, and Data Cleaning.

## Part 1: Staging and Foundational Data Cleaning

### QUESTION 1: Creating Corporate Emails
* **Business Question:** The onboarding team is manually typing employee logins, causing formatting errors and accidental spacing issues. Can we build an automated database tool that unifies everyone's records and outputs a clean, lowercase email string (`firstname.lastname@company.com`) for all staff?

```sql


use SQLTUTORIAL
go

drop table if exists  #temp1combineddemodata
go

create table #temp1combineddemodata
(FirstName varchar(50), LastName varchar(50)) 

INSERT INTO #temp1combineddemodata 
EXEC COMBINEDSTAFFDATA 


select cleanedcombineddata.firstname, cleanedcombineddata.lastname, concat(firstname, lastName,  '@gmail.com') as email
from
(SELECT trim(lower(FIRSTNAME)) as firstname, trim(lower(lastname)) as lastname
FROM #temp1combineddemodata) as cleanedcombineddata

```
### QUESTION 2: Masking Last Names for Privacy
* **Business Question:** For data privacy compliance, our security team needs a personnel roster where employee last names are masked, but the output must be completely capitalized for uniform administrative filing. Can we pull a secure list showing the first three letters of their last name followed by `_SECURE`?

```sql
use SQLTUTORIAL
go

drop table if exists #tempcombineddata

create table #tempcombineddata
(FirstName varchar(50), LastName varchar(50))

insert into #tempcombineddata 
exec combinedstaffdata

select upper(FirstName) as FirstName, concat(left(upper(lastname), 3), 'SECURE') as maskedlastname
from #tempcombineddata



```

### QUESTION 3: Creating an Employee Initials Table
* **Business Question:** Our analytics team wants a fast, temporary scratchpad containing the combined roster of both our office and warehouse staff, showing their customized, joined initials for a new tracking dashboard. Can we merge the lists and extract their exact initials?

```sql
use SQLTUTORIAL
go

drop table if exists #temp2
go

create table #temp2
(EmployeeID int, FirstName varchar(50), LastName varchar(50), INITIALS VARCHAR(50))



Insert into #temp2
Select combinedstaffdata.EmployeeID, combinedstaffdata.FirstName, combinedstaffdata.LastName,  concat(substring(FirstName, 1,1), substring(LastName, 1,1)) as INITIALS
FROM (
select  ed.employeeID, ed.FirstName, ed.LastName
from [SQLTUTORIAL].dbo.EMPLOYEEDEMOGRAPHICS as ed
union all
select wd.EmployeeID, wd.FirstName, wd.LastName
from [SQLTUTORIAL].dbo.WarehouseEmployeeDemographics as wd) as combinedstaffdata
where EmployeeID is not NULL

select *
FROM #temp2
```

### QUESTION 4: Standardizing Job Titles
* **Business Question:** There are spelling inconsistencies in our payroll tracking table where human resources typed out variations of job titles. Can we standardize the title 'accountant' to 'Finance Specialist' and pull a clean budget list without any trailing space errors disrupting the records?

```sql
WITH MERGEDDATA AS 
(Select CombinedStaff.EmployeeID, CombinedStaff.FirstName, CombinedStaff.LastName, ES.Jobtitle, ES.Salary
from
(SELECT ED.EmployeeID, ED.FirstName, ED.LastName
FROM [SQLTUTORIAL].DBO.EMPLOYEEDEMOGRAPHICS AS ED
UNION ALL
SELECT WD.EmployeeID, WD.FirstName, WD.LastName
from [SQLTUTORIAL].dbo.WarehouseEmployeeDemographics as WD) as combinedstaff
left join [SQLTUTORIAL].dbo.EmployeeSalary as ES
on combinedstaff.EmployeeID = ES.EmployeeID
where combinedstaff.EmployeeID is not NULL)
,
CLEANEDDATA AS (
SELECT MERGEDDATA.EmployeeID, MERGEDDATA.FirstName, MERGEDDATA.LastName, RTRIM(Replace(MERGEDDATA.Jobtitle, 'Accountant', 'Finance Specialist')) AS CorrectedJobTitle
from MERGEDDATA)


Select*
from CLEANEDDATA


```

### QUESTION 5: Finding Specific Name Patterns
* **Business Question:** The auditing team wants to scan our payroll pool to identify high earners whose first names contain specific letter patterns ('an') so they can cross-reference them against an automated system check. Can we locate these workers and label them cleanly?

```sql
use SQLTUTORIAL
go

drop table if exists #temp3
create table #temp3
(EmployeeID int, FirstName varchar(50), LastName varchar(50), Salary int)

insert into #temp3
exec combineddatasalarystaff

select EmployeeID, FirstName, Salary,  charindex('Sa', FirstName) as NamesWithAN, Case
WHEN charindex('Sa', FirstName) >=1 then 'MATCH FOUND'
ELSE 'MATCH NOT FOUND'
END AS MATCHES
from #temp3
where salary in (select #temp3.Salary
from #temp3
where salary > 45000)
order by MATCHES 
```
---

## Part 2: Intermediate Pipeline Engineering

### QUESTION 6: Generating Security Badge Codes
* **Business Question:** Security operations needs an automated process that generates security badge codes for warehouse workers by blending pieces of their names with the end of their employee identification numbers. Can we build a reusable tool to output these codes?

```sql
use SQLTUTORIAL 
go

drop procedure if exists WarehouseEmployees
go

create procedure WarehouseEmployees
as
begin 
Select EmployeeID, FirstName, LastName
from [SQLTUTORIAL].dbo.WarehouseEmployeeDemographics 
end



use SQLTUTORIAL
go

drop table if exists #temp4
go

create table #temp4
(EmployeeID varchar(50), FirstName varchar(50), LastName varchar(50))

insert into #temp4 
exec WarehouseEmployees

SELECT TRIM(Right(EmployeeID, 2)) AS Last2DigitsOfID, Trim(Left(FirstName, 2)) as First2LettersFromFirstName, concat(TRIM(Right(EmployeeID, 2)), Trim(Left(FirstName, 2))) as badgecode
from #temp4
```

### QUESTION 7: Cleaning Text Typos in Subqueries
* **Business Question:** A data entry bug corrupted some text fields in our system. Without touching the raw tables permanently, can we write a query that runs multiple text-replacement layers at once to clean up specific vowels before displaying the report?

```sql
select EmployeeID, Replace(Replace(FirstName, 'Y', 'A'), 'e', 'a') as FixedStrings
from (
SELECT ED.EmployeeID, ED.FirstName, ED.LastName
FROM [SQLTUTORIAL].DBO.EMPLOYEEDEMOGRAPHICS AS ED
UNION ALL
SELECT WD.EmployeeID, WD.FirstName, WD.LastName
from [SQLTUTORIAL].dbo.WarehouseEmployeeDemographics as WD) as combinedstaff
where FirstName is not null
```
### QUESTION 8: Combining Rosters and Making Audit Keys
* **Business Question:** Management requires a temporary, consolidated master table of all corporate locations. The last name must be combined with the first name in a specific format, and an audit tracking key must be generated from the middle of that combined text. Can we stage this data safely?

```sql
USE SQLTUTORIAL
GO

DROP PROCEDURE IF EXISTS COMBINEDSTAFF
GO

CREATE PROCEDURE COMBINEDSTAFF
AS
BEGIN
SELECT ED.EmployeeID, ED.FirstName, ED.LastName
FROM [SQLTUTORIAL].DBO.EMPLOYEEDEMOGRAPHICS AS ED
UNION ALL
SELECT WD.EmployeeID, WD.FirstName, WD.LastName
FROM [SQLTUTORIAL].dbo.WarehouseEmployeeDemographics AS WD
END 



USE SQLTUTORIAL
GO

DROP TABLE IF EXISTS #TEMP1
GO

CREATE TABLE #TEMP1
(EmployeeID varchar(50), FirstName varchar(50), LastName varchar(50))

INSERT INTO #TEMP1 
EXEC COMBINEDSTAFF


SELECT COMBINEDNAMESTABLE.COMBINEDNAMES, SUBSTRING(COMBINEDNAMESTABLE.COMBINEDNAMES, 2, 4) AS AUDITTRACKINGKEY
FROM (
SELECT CONCAT(LastName,',', ' ' ,FirstName) AS COMBINEDNAMES
FROM #TEMP1 as #t1
WHERE #T1.LastName IS NOT NULL) AS COMBINEDNAMESTABLE
```

### QUESTION 9: Finding Workers in High Budget Departments
* **Business Question:** We need to isolate the complete details of workers whose departments meet massive overall spending thresholds, and the output job titles must be printed in screaming uppercase text for executive presentation slides. Can we pull this roster?

```sql

Select CombinedStaff.FirstName, Combinedstaff.LastName, upper(es.jobtitle) as CAPITALISEDJOBTTILE
from 
(select ed.EmployeeID, ed.FirstName, ed.LastName
from [SQLTUTORIAL].dbo.EMPLOYEEDEMOGRAPHICS as ed
union all
select wd.EmployeeID, wd.FirstName, wd.LastName
from [SQLTUTORIAL].dbo.WarehouseEmployeeDemographics as wd) as CombinedStaff
left join  [SQLTUTORIAL].dbo.EmployeeSalary as es
on combinedstaff.EmployeeID = es.EmployeeID
where es.JobTitle IN ( 
Select es.JobTitle
from [SQLTUTORIAL].dbo.EmployeeSalary as es
group by es.JobTitle
having sum(es.salary) > 100000 )

```

### QUESTION 10: Cleaning Roster Details Step by Step
* **Business Question:** Human resources requires a multi-stage data cleaning pipeline. First, all invisible leading and trailing spaces must be stripped from the raw data. Second, the last names must be made lowercase for file sorting. Finally, it must connect to the active financial tables. Can we build this step-by-step?

```sql

USE SQLTUTORIAL
GO

DROP PROCEDURE IF EXISTS COMBINEDSTAFFSALARIES
GO

CREATE PROCEDURE COMBINEDSTAFFSALARIES
AS
BEGIN
SELECT COMBINEDSTAFF.EMPLOYEEID, COMBINEDSTAFF.FIRSTNAME, COMBINEDSTAFF.LASTNAME, ES.SALARY, ES.JOBTITLE
FROM
(SELECT ED.EMPLOYEEID, ED.FirstName, ED.LastName
FROM [SQLTUTORIAL].DBO.EMPLOYEEDEMOGRAPHICS AS ED
UNION ALL
SELECT WD.EmployeeID, WD.FirstName, WD.LastName
FROM [SQLTUTORIAL].DBO.WarehouseEmployeeDemographics AS WD) AS COMBINEDSTAFF
LEFT JOIN 
[SQLTUTORIAL].DBO.EmployeeSalary AS ES
ON COMBINEDSTAFF.EmployeeID = ES.EmployeeID
END



USE SQLTUTORIAL
GO

DROP TABLE IF EXISTS #MANAGEMENTSALARIES
GO

CREATE TABLE #MANAGEMENTSALARIES
(EmployeeID varchar(50), FirstName varchar(50), LastName varchar(50), Salary int, JobTitle varchar(50))

INSERT INTO #MANAGEMENTSALARIES
EXEC COMBINEDSTAFFSALARIES

SELECT #M1.EmployeeID, #M1.FirstName, #M1.LastName, case 
when CHARINDEX('manager', JobTitle) >=1 then 'managerinrole'
else 'not manager'
end as Management,
#M1.Salary,  avg(salary) over () as AveragePayrollSpend, ((avg(salary) over ()) - #m1.Salary) as DifferenceBetweenAverageAndSalary
FROM #MANAGEMENTSALARIES AS #M1


```

---

## Part 3: Deeper Analytics

### QUESTION 11: Flagging Managers and Pay Differences
* **Business Question:** Our finance team needs a re-runnable tool that blends our entire workforce roster together and automatically flags which employees hold management positions based on their job titles. Right next to that flag, we need to show the exact difference between each employee's salary and the overall corporate average pay, calculated dynamically in a single pass without slowing down the database server.

```sql
use SQLTUTORIAL
go

drop view if exists V_MANAGEMENTVIEW
GO


CREATE VIEW V_MANAGEMENTVIEW
AS
SELECT COMBINEDSTAFFSALARIES.EmployeeID, CombinedStaffSalaries.FirstName, COMBINEDSTAFFSALARIES.LastName, case 
when CHARINDEX('manager', COMBINEDSTAFFSALARIES.JobTitle) >=1 then 'managerinrole'
else 'not manager'
end as Management,
COMBINEDSTAFFSALARIES.Salary,  avg(COMBINEDSTAFFSALARIES.salary) over () as AveragePayrollSpend, ((avg(COMBINEDSTAFFSALARIES.salary) over ()) - COMBINEDSTAFFSALARIES.Salary) as DifferenceBetweenAverageAndSalary
FROM 
(SELECT COMBINEDSTAFF.EMPLOYEEID, COMBINEDSTAFF.FIRSTNAME, COMBINEDSTAFF.LASTNAME, ES.SALARY, ES.JOBTITLE
FROM
(SELECT ED.EMPLOYEEID, ED.FirstName, ED.LastName
FROM [SQLTUTORIAL].DBO.EMPLOYEEDEMOGRAPHICS AS ED
UNION ALL
SELECT WD.EmployeeID, WD.FirstName, WD.LastName
FROM [SQLTUTORIAL].DBO.WarehouseEmployeeDemographics AS WD) AS COMBINEDSTAFF
LEFT JOIN 
[SQLTUTORIAL].DBO.EmployeeSalary AS ES
ON COMBINEDSTAFF.EmployeeID = ES.EmployeeID) AS COMBINEDSTAFFSALARIES


GO


SELECT *
FROM V_MANAGEMENTVIEW

```
### QUESTION 12: Big Nested Query for Salary Ranges
* **Business Question:** We need to build a complex, layered report that unifies data, cleans text, runs company-wide average salary math, subtracts deviations, and groups employees into range brackets completely within a single, inside-out nested layout. Can we map this entire flow?

```sql
USE SQLTUTORIAL
GO

DROP VIEW IF EXISTS V_SALARYRANGEANALYSIS
GO

CREATE VIEW V_SALARYRANGEANALYSIS
AS
SELECT CombinedData.FirstName, CombinedData.LastName, ES.SALARY, avg(ES.Salary) over () as AverageSalaryCompanyWide, ((avg(ES.Salary) over ()) - ES.Salary) as DeviationFromAverageSalary, CASE
WHEN ((avg(ES.Salary) over ()) - ES.Salary) > 8000 THEN ' WAY ABOVE AVERAGE'
WHEN ((avg(ES.Salary) over ()) - ES.Salary) < -8000 THEN 'WAY BELOW AVERAGE'
ELSE 'SALARY CLOSE TO COMPANY AVERAGE'
END AS SALARYDEVIATIONRANGES
From (
SELECT ED.EmployeeID, ED.FirstName, ED.LastName
FROM [SQLTUTORIAL].DBO.EMPLOYEEDEMOGRAPHICS AS ED
UNION ALL 
SELECT WD.EmployeeID, WD.FirstName, WD.LastName
FROM [SQLTUTORIAL].DBO.WarehouseEmployeeDemographics AS WD
) AS CombinedData
LEFT JOIN [SQLTUTORIAL].DBO.EMPLOYEESALARY AS ES
ON CombinedData.EmployeeID = ES.EmployeeID

GO



SELECT FIRSTNAME, Salary, DeviationFromAverageSalary, SALARYDEVIATIONRANGES, DENSE_RANK() OVER (PARTITION BY SALARYDEVIATIONRANGES ORDER BY SALARY DESC) AS  RANKOFSALARY
FROM V_SALARYRANGEANALYSIS
WHERE SALARY IS NOT NULL









```

### QUESTION 13: Splitting Accountant Salaries with CTEs
* **Business Question:** HR wants a sequential data pipeline that isolates a specific department's average pay without using slow self-joins, strips out spacing defects, and fuses the employee details with a custom flag indicating their status against that benchmark. Can we link these metrics?

```sql

USE SQLTUTORIAL
GO

DROP VIEW IF EXISTS V_ACCOUNTANTSALARYAVERAGE 
GO

CREATE VIEW V_ACCOUNTANTSALARYAVERAGE
AS
WITH COMBINEDDATA AS (
SELECT ED.EmployeeID, ED.FirstName, ED.LastName, ED.Age, ED.Gender
FROM [SQLTUTORIAL].DBO.EMPLOYEEDEMOGRAPHICS AS ED
UNION ALL 
SELECT WD.EmployeeID, WD.FirstName, WD.LastName, WD.Age, WD.Gender 
FROM [SQLTUTORIAL].DBO.WarehouseEmployeeDemographics AS WD ) 
,
COMBINEDDATASALARY AS (
SELECT  COMBINEDDATA.EmployeeID,  COMBINEDDATA.FirstName, COMBINEDDATA.LastName, COMBINEDDATA.Age, COMBINEDDATA.Gender, es.JobTitle, es.Salary
FROM COMBINEDDATA
Left Join [SQLTUTORIAL].dbo.EmployeeSalary as ES
on COMBINEDDATA.EmployeeID = ES.EmployeeID)
,
CLEANEDSALARYDATA AS (
SELECT COMBINEDDATASALARY.EmployeeID, COMBINEDDATASALARY.FirstName, COMBINEDDATASALARY.LastName, COMBINEDDATASALARY.Age, COMBINEDDATASALARY.Gender, RTRIM(LTRIM(COMBINEDDATASALARY.JobTitle)) AS CleanedJobTitle,  COMBINEDDATASALARY.Salary
FROM COMBINEDDATASALARY) 
,
AVERAGESALARY AS (
SELECT CLEANEDSALARYDATA.EmployeeID, CLEANEDSALARYDATA.FirstName, CLEANEDSALARYDATA.LastName, CLEANEDSALARYDATA.Age, CLEANEDSALARYDATA.Gender, CLEANEDSALARYDATA.Salary,  CLEANEDSALARYDATA.CleanedJobTitle, avg(CLEANEDSALARYDATA.Salary) over (partition by CLEANEDSALARYDATA.CleanedJobTitle)  as AverageSalaryPerJobTitle 
FROM CLEANEDSALARYDATA)
,
DeviationFromAverageSalary AS (
SELECT AVERAGESALARY.EmployeeID, AVERAGESALARY.FirstName, AVERAGESALARY.LastName, AVERAGESALARY.Age, AVERAGESALARY.Gender, AVERAGESALARY.Salary, AVERAGESALARY.AverageSalaryPerJobTitle, (AVERAGESALARY.AverageSalaryPerJobTitle-AVERAGESALARY.Salary) as DeviationFromAverage, CASE 
WHEN (AVERAGESALARY.AverageSalaryPerJobTitle-AVERAGESALARY.Salary) > 5000 THEN 'FAR FROM AVERAGE SALARY' 
WHEN (AVERAGESALARY.AverageSalaryPerJobTitle-AVERAGESALARY.Salary) <-5000 then 'FAR FROM AVERAGE SALARY'
ELSE 'CLOSE TO AVERAGE SALARY'
END AS AVERAGERANKING
FROM AVERAGESALARY)

Select *
from DeviationFromAverageSalary;
GO

SELECT*
FROM V_ACCOUNTANTSALARYAVERAGE
where salary is not null

```

### QUESTION 14: Optimizing Large Data in Temp Tables
* **Business Question:** To mimic high-volume production speeds, we need a data staging script that unifies our workforce, transforms text fields into uniform uppercase formats, extracts tracking markers from the right side of the IDs, and filters fields based on string patterns using an optimal script lifecycle. Can we build this staging environment?

```sql


USE SQLTUTORIAL
GO

DROP PROCEDURE IF EXISTS COMBINEDSTAFF
GO

CREATE PROCEDURE COMBINEDSTAFF
AS
BEGIN
SELECT ED.EmployeeID,  ED.FirstName, ED.LastName
FROM [SQLTUTORIAL].DBO.EMPLOYEEDEMOGRAPHICS AS ED
UNION ALL
SELECT WD.EmployeeID, WD.FirstName, WD.LastName
FROM [SQLTUTORIAL].DBO.WarehouseEmployeeDemographics AS WD
END

USE SQLTUTORIAL
GO

DROP TABLE IF EXISTS #STAGINGTABLE
GO

CREATE TABLE #STAGINGTABLE
(EmployeeID varchar(50), FirstName varchar(50), LastName varchar(50))

INSERT INTO #STAGINGTABLE
EXEC COMBINEDSTAFF
GO

SELECT RIGHT(EmployeeID, 3) as TRACKINGNUMBER, UPPER(FirstName) as CapitalisedFirstName, UPPER(LastName) as CapitalisedLastName
FROM #STAGINGTABLE
where FirstName like 's%'
```

### QUESTION 15: The Master HR Control Script
* **Business Question:** Management wants the definitive capstone script for our reporting suite. It must be an automated procedure that runs a three-stage data pipeline to combine facility records, clean spacing, make text lowercase, calculate peak company limits, stage everything in a temporary table, and assign range tiers. Can we deploy this master hub?

```



use SQLTUTORIAL
go


drop procedure if exists combinedstaffsalary
go

create procedure combinedstaffsalary
as
begin 
select combinedstaff.EmployeeID, combinedstaff.FirstName, combinedstaff.LastName, es.Salary
from 
(select ed.EmployeeID, ed.FirstName, ed.LastName
from [SQLTUTORIAL].dbo.EMPLOYEEDEMOGRAPHICS as ed
union all
select wd.EmployeeID, wd.FirstName, wd.LastName
from [SQLTUTORIAL].dbo.WarehouseEmployeeDemographics as wd) as combinedstaff
left join [SQLTUTORIAL].dbo.EmployeeSalary as es
on combinedstaff.EmployeeID = es.EmployeeID
end

go

use SQLTUTORIAL
go

drop table if exists #tieredsalaries
go

create table #tieredsalaries
(EmployeeID varchar(50), FirstName varchar(50), LastName varchar(50), Salary int)

insert into #tieredsalaries
exec combinedstaffsalary
go


select row_number() over (partition by t1.MaxSalaryDeviationTiers order by Salary DESC) as RowNumber,  t1.TrimmedEmployeeID, t1.TrimmedFirstName, t1.TrimmedandLowerCaseLastName, t1.Salary, t1.MaxSalaryCompanyWide, t1.DeviationFromTheMaxSalaryCompanyWide, t1.MaxSalaryDeviationTiers, dense_rank() over (partition by t1.MaxSalaryDeviationTiers Order by Salary DESC) as RankInEachTier
from  
(
select trim(EmployeeID) as TrimmedEmployeeID, trim(FirstName) as TrimmedFirstName, trim(lower(lastname)) as TrimmedandLowerCaseLastName, Salary,   Max(Salary) over () As MaxSalaryCompanyWide, ((Max(Salary) over ()) - Salary) as DeviationFromTheMaxSalaryCompanyWide, Case 
WHEN ((Max(Salary) over ()) - Salary) < 25000 then 'Executive Tier'
ELSE 'Standard Tier'
END as MaxSalaryDeviationTiers
from #tieredsalaries
where salary is not null) as t1
```


