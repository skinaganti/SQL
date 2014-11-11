SQL
===

SELECT
	C.DisplayName name,
	C.GUID,
	C.IDCode AS MRN,
	CAST (C.BirthMonthNum AS VARCHAR(10))+'/'+CAST (C.BirthDayNum AS VARCHAR(10)) +'/'+ CAST (C.BirthYearNum AS VARCHAR(10)) AS DOB,
	C.GenderCode,
	CONVERT ( VARCHAR(10), CAST (C.CreatedWhen AS date), 101 ) AS CreateDate,
	SSN=(SELECT TOP 1 ID.ClientIDCode
		FROM CV3ClientID ID
		JOIN CV3Client 
		ON ID.ClientGUID = C.GUID 
		AND ID.TypeCode = 'SSN' and IDStatus = 'ACT')

INTO #T1
FROM CV3Client C

WHERE CAST(C.CreatedWhen as date) = '2014-10-30'


SELECT * FROM #T1
ORDER BY name





-- T2 (all clients with SSN and MRN num, including those with unknown and none values for SSN and MRN num)
--
-- pulls all clients from CV3Client table with distinct SSN and MRN num (selected from CV3ClientID table) 
-- where first name is not female or male and last name doesn't start with zz or xx
-- and stores into temp table T2 with index on ClientGUID col
--


SELECT TOP 1000 
	c.IDCode,
	c.GUID clientguid,
	c.BirthDayNum,
	c.BirthMonthNum,
	c.BirthYearNum,
	c.FirstName fname,
	c.LastName lname,
	c.GenderCode,
	c.CreatedBy
,(case when len(c.birthMonthNum) = 1 then '0'+convert(varchar,c.birthMonthNum)
	 else convert(varchar,c.birthMonthNum) end)+'/'+
(case when len(c.birthDayNum) = 1 then '0'+convert(varchar,c.birthDayNum)
	 else convert(varchar,c.birthDayNum) end)+'/'+convert(varchar,c.birthYearNum) AS DOB
,SSN=(SELECT TOP 1 CI.ClientIDCode FROM CV3ClientID CI 
     WHERE CI.ClientGUID=C.GUID
     AND CI.TypeCode='SSN')
,MRN=(SELECT TOP 1 CI.ClientIDCode FROM CV3ClientID CI 
     WHERE CI.ClientGUID=C.GUID
     AND CI.TypeCode='MRN')
     ,CreatedWhen
          	
INTO #T2      
 FROM cv3client C
 WHERE C.FirstName NOT IN ('FEMALE','MALE')
AND ((C.LastName NOT LIKE 'ZZ%') OR (C.LastName NOT LIKE 'XX%'))

create unique clustered index ix_t2 on #T2(clientguid)

SELECT * FROM #T2
ORDER BY lname 


--
-- takes first and last name from temp table T2
-- and compares it to a specific string (calculates Levenstein distance btw name and string)
-- and stores name and Levenstein distance score in temp table ResultTable
-- and ranks names by ascending score and selects top 10 records
-- Levenstein distance - 0 is a match, the higher the distance, the greater the disparity btw the 2 strings
--


DECLARE @name nvarchar(50)

DECLARE Cts CURSOR LOCAL FOR select #T1.name from #T1

OPEN Cts
FETCH NEXT FROM Cts into @name
WHILE @@FETCH_STATUS = 0
BEGIN

		CREATE TABLE #ResultTable
		(
			str1 nvarchar(50),
			str2 nvarchar(50),
			score int
		)

		DECLARE @fname nvarchar(50), @lname nvarchar(50), @s1 nvarchar(50), @s2 nvarchar(50)

		DECLARE Clients CURSOR LOCAL FOR select #T2.fname, #T2.lname from #T2

		OPEN Clients
		FETCH NEXT FROM Clients into @fname, @lname
		WHILE @@FETCH_STATUS = 0
		BEGIN
			DECLARE @result int

--			SET @s1 = 'Rodriguez, Josefina'
			SET @s1 = @name
			SET @s2 = @lname + ', ' + @fname

			EXEC edit_distance @s1, @s2, @c = @result OUTPUT

			INSERT INTO #ResultTable(str1, str2, score)
			SELECT @s1, @s2, @result 

			--PRINT @name + ' - ' + CAST(@result AS nvarchar(100))

			FETCH NEXT FROM Clients into @fname, @lname
		END

		CLOSE Clients
		DEALLOCATE Clients

		Select TOP 5 str1, str2, score 
		from #ResultTable
		ORDER BY score ASC

		drop Table #ResultTable


FETCH NEXT FROM Cts into @name
END

CLOSE Cts
DEALLOCATE Cts


drop Table #T1
drop Table #T2


