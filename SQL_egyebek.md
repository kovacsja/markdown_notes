#SQL jegyzetek 
@(devel)[sql|tanul|manual]
##INTERSECT vs EXISTS
```SQL
SELECT * FROM [tábla]
INTERSECT
SELECT * FROM [tábla]

SELECT * FROM [tábla]
WHERE EXISTS
	(SELECT 1
	FROM [lekérdezés ugyanannyi mezővel, mint a szülő lekérdezés]
	WHERE [szülő lekérdezés első mezője]=[beágyazott lekérdezés első mezője])
```

##EXCEPT vs NOT EXISTS
```SQL
SELECT * FROM [tábla]
EXCEPT
SELECT * FROM [tábla]

SELECT * FROM [tábla]
WHERE NOT EXISTS
	(SELECT 1
	FROM [lekérdezés ugyanannyi mezővel, mint a szülő lekérdezés]
	WHERE [szülő lekérdezés első mezője] = [beágyazott lekérdezés első mezője])
```

##Common table expressions (CTE)

 - ideiglenes halmazokra lehet vele hivatkozni név szerint
 - külün, még a használat előtt kell definiálni
 - a definiálás után úgy lehet rá hivatkozni, mint egy táblára
 - ha több cte-t akarunk használni, akkor egy with-en belül kell definiálni őket
 - egy cte-t egy lekérdezésben többször is lehet használni, és cte-t lehet más cte-ben is felhasználni. viszont minden select után a cte definíció elvész

```sql
    WITH [cte_name_1] ( [mezők,] )
    AS ( [query_1] ),
    
    [cte_name_2] ( [mezők,] )
    AS ( [query_2] )
```
bizonyos klauzulák nem használhatóka  CTE definiálásakor: COMPUTER, COMPUTE BY, ORDER BY, INTO, FOR XML, FRO BROWSE, OPOTION

ref: http://technet.microsoft.com/en-us/library/ms190766(v=sql.105).aspx

##Rekurzív lekérdezések
A hierarchikus kapcsolatokkal rendezett adatok kezelését könnyíti meg. (Ezt sp-ben, vagy függvény is meg lehet valósítani.)

Rekurzív lekérdezésekkel nem kiegyensúlyozott fák is kezelhetőek. 

A rekurzív lekérdezéseknél a CTE két részből áll: egy horgonyból, ami a hierarchia tetejét határozza meg, és egy rekurzív részből, ami UNION ALL -lal kapcsolódik az előzőhöz. 

```sql
WITH [cte_name] ([mezők, ]) AS
	(
	SELECT id, parent_id, 0 AS level, * 
	FROM [tálba]
	WHERE [root = 1]
	
	UNION ALL 

	SELECT id, parent_id, r.level + 1, *
	FROM tábla JOIN [cte_name] 
	  ON parent_id = tábla.id
	)

SELECT * 
FROM [cte_name] JOIN ...
WHERE ...
```

##MERGE
Az SQL Server 2008-tól érhető el ez a parancs. Egyszerre lehet vele INSERT, UPDATE és DELETE parancsokat végrehajtani, ezért hatékonyabb, mint a felsoroltak külön-külön. 

```sql
MERGE [tábla] AS target
USING (
	[query]
) as source
on target.id = source.id
WHEN MATCHED THEN
	[UPDATE / INSERT / DELETE]
WHEN NOT MATCHED THEN
	[UPDATE / INSERT / DELETE];
```
A ;-t kötelező kitenni a MERGE utasítás után.
