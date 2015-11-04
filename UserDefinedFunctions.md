#User-Defined Functions (UDF)
[TOC]
##Mire jó az UDF?
Az UDF a tárolt eljárásokhoz hasonlóan egy külön adatbázis objektum, ami rendezett SQL utasításokat tárol, a különbség, a visszaadott értékek kezelésében van. Az UDF visszatérési értéke csak a sikeres futást képes visszaigazoln, nem pedig valamilyen logikai tartalommal bíró értéket. A sproc eredményeként létrejövo recordsetet pedig csak úgy lehet tovább használni, ha előbb egy táblába írjuk ki őket.
Az UDF is képes paramétereket befogadni, visszont visszatérsi paraméter helyett visszatérési értéke van. Ez a kimeneti érték pedig nem csak `int` típus lehet, mint egy sproc, esetében, hanem szinte minden SQL Server adattípus. Sőt, akár még táblákat is lehetnek a függvény eredményei. Az UDF kétfajta lehet, amit a deklaráláskor kell eldönteni:
* olyan, ami **skalár értéket** ad vissza, és
* olyan, ami **táblát** ad vissza.

Az UDF készítés általános szintaxisa, akár skalár értéket, akár táblát ad vissza a függvény:
```SQL
CREATE FUNCTION [<schema name>].<function name> ([<@parameter name> [AS] [<schema name>.]<data type> [= <default value>] [READONLY]]
	[ ,...N ]])
RETURNS {<scalar type> | TABLE [(<table definition>)]}
	[WITH [ENCRYPTION]|[SCHEMABINDING]|[RETUNRNS NULL ON NULL INPUT | CALLED ON NULL INPUT] | [EXECUTE AS {CALLER|SELF|OWNER|<'user name'>}]
]
[AS] {EXTERNAL NAME <external method> | 
BEGIN
	[<funcions statements>]
    {RETURN <type as defined in RETURNS clause> | RETURN (<SELECT statement>)}
END}[;]
```
##Skalár értéket visszaadó UDF
Egy függvénytől általában ez elvárt működés, és az SQL Server legtöbb rendszerfüggvénye is ilyen: `GETDATE()`, `USER()`. Ez is mutatja, hgoy a visszatérési értéknek nem kell `int` típusúnak lennie, sőt még saját adattípus is lehet, kivéve BLOB, cursor, vagy timestamp.
Az UDF előnyei sproccal szemben:
* az UDF legfontosabb funkciója, hogy adatot adjon vissza futás után, a sproc visszatérési értéke viszont csak arra szolgál, hogy jelezze a futás sikerességét.
* egy UDF része tud lenni egy lekérdezésnek is, míg a sproc erre alkalmatlan.

Példa függfény dátumok konvertálására:
```SQL
CREATE FUNCTION dbo.DayOnly(@Date datetime)
RETURNS varchar(12)
AS
BEGIN
	RETURN CONVERT(varchar(12), @Date, 101);
END
```
Egy másik felhasználási lehetésége az UDF-eknek, hogy subquery-k helyett is használhatóak, mivel a belésjük komplett lekérdezéseket lehet ágyazni, így olvashatóbbá tehetik a kódot.
```SQL
USE AdventureWorks2008;

SELECT Name,
	ListPrice,
    (SELECT AVG(ListPrice) FROM Production.Product) AS Average,
    ListPrice = (SELECT AVG(ListPrice) FROM Production.Product) AS Difference
FROM Production.Product
WHERE ProductSubCategoryID = 1;
```
Ezzel egyenértékű, ha a kódokat kiemeljük függvénybe:
```SQL
CREATE FUNCTION dbo.AveragePrice()
RETURNS money
WITH SCHEMABINDING
AS
BEGIN
	RETURN (SELECT AVG(ListPrice) FROM Production.Product);
END
GO

CREATE FUNCTION dbo.PriceDifference (@Price money)
RETURNS money
AS
BEGIN
	RETURNS @Price - dbo.AveragePrice();
END
```
Az itt leírt példából is látható, hogy az UDF-ek egymásba ágyazhatóak. A kikötött `WITH SCHEMABINDING` pedig ugyanúgy működik, ahogyan a `VIEW`-k esetében: tehát a főggőségben lévő objektumok szerkezetét nem lehet módosítani, vagy törölni anélkül, hogy a sémához kötést el nem távolítjuk először.
Ezek alapján át lehet alakítani az előző lekérdezést, hogy olvashatóbb legyen:
```SQL
USE AdventureWorks2008

SELECT Name,
	ListPrice,
    dbo.AveragePrice() AS Average,
    dbo.PriceDifference(ListPrice) AS Difference
FROM Production.Product
WHERE ProductSubCategoryID = 1;
```
Az olvashatóságon kívül ennek a szerkezetnek az előnye továbbá az újrahasznosíthatóság. Ha ügyesen használjuk az UDF-eket, sok időt spórolhatunk.
##Táblákat visszaadó UDF-ek
Az SQL Serverban definiált UDF-ek nem csak skalár értéket, hanem táblákat is vissza tudnak adni. Ezeket a visszaadott táblákat használni lehet `JOIN` és `WHERE` klauzulákban.
```SQL
USE AdventureWorks2008
GO

CREATE FUNCTION dbo.fnContactList()
RETURNS TABLE
AS
RETURN (SELECT BusinessEntityID,
			LastName + ', ' + FirstName AS Name
    	FROM Person.Person);
GO
```
Ezek után használható a
```SQL
SELECT * FROM dbo.fnContactList();
```
kifejezés.
Ezt azonban egy `VIEW` használatával is megoldhattuk volna. Akkor viszont hasznos tud lenni, ha a függvénynek paramétereket is adunk át, amivel az eredmény recordset könnyebben módsítható, mint egy `VIEW` esetében. 
```SQL
CREATE FUNCTION dbo.fnContactSearch(@LastName nvarchar(50))
RETURNS TABLE
AS
RETURN (SELECT p.BusinessEntityID,
			LastName + ', ' + FirstName AS Name,
            ea.EmailAddress
        FROM Person.Person AS p
        LEFT OUTER JOIN Person.EmailAddress ea
        ON ea.BusinessEntityId = p.BusinessEntityID
        WHERE LastName Like @LastName + '%');
GO
```
Amennyiben ezt a függvényt használjuk, akkor gyorsan és dinamikusan végezhetünk el szűréseket:
```SQL
SELECT *
FROM fnContactSearch('Ad');
```
Ezt az eredményt sproc használatával is elérhettük volna, de azt közvetlenül nem használhattuk volna a lekérdezésekben, egy `VIEW` pedig nem ilyen könnnyen módosítható.
Sokszor azonban bonyolultabb művelteket is végre kell hajtani, hogy a megfelelő táblát kapjuk vissza. Az UDF támogatja több SQL paranncs végrehajtását is az eljárásokhoz hasonlóan. Ekkor azonban meg kell neveznünk, és definiálnunk is kell a köztes táblákat úgy, ahogyan azt egy temp-tábla esetében tennénk.
>Hogy ezt is jól átlássuk hierarchikus adatokat fogunk feldolgozni a példában. A relációs adatbázisok egyik gyengepontja a hierarchikus adatok kezelése. Az SQL Server 2008 azonban bevezetett egy `hierachyID` adattípust, és pár beépített függvényt, amik segítenek az adatfa-struktúrák kezelésénél. [prof->] Most azonban a régi módszer szerint fogunk eljárni.

Hierarchikus adatok addig jól kezelhetőek szimpla lekérdezésekkel, amíg a hierarchis mélysége fix. Amint azonban ezt dinamikusan kell kezelni, már nem alkalmazható az előbbi módszer. Ezt rekurzív kóddal lehet a legegyszerűbben megoldani.
**Példa:** Van egy szervezeti felépítést leíró tábla, amiből ki kell nyerni, hogy egy vezető alá hány beosztott tartozik közvetlenül, vagy közvetetten. Lépések:
1. Meg kell találni, hogy ki jelent közvetlenül kijelölt vezetőnek.
2. Meg kell találni, hogy az 1. lépésben megtalált alkalmazottaknak kik jelentenek.
3. Addig kell ismételni a 2. lépést, amíg nem marad több beosztott.

*(Az UDF is rendelkezik a sproc beágyazási küszöbével. Rekurziót itt is csak 32 szintig tudunk elvégezni. Ha ennél mélyebbre kell menni, akkor módosítani kell a kódot.)*
```SQL
CREATE FUNCTION dbo.fnGetReports (@EmployeeID AS int)
	RETURNS @Reports TABLE
    	(
        EmployeeID	int	NOT NULL
        ManagerID	int	NULL
        )
AS
BEGIN
DECLARE @Employee AS int; --hogy számontartsuk, melyik alkalmazottat vizsgáljuk éppen
/*először az első munkavállalató illesztjük be a munkatáblába*/
INSERT INTO @Reports
	SELECT EmployeeID, ManagerID
    FROM HumanResources.Employee
    WHERE EmployeeID = @EmployeeID;
/*értéket adunk a rekurzióban annak, hogy kit vizsgálunk éppen*/
SELECT @Employee = MIN(EmployeeID)
FROM HumanResources.Employee
WHERE EmployeeID = @EmployeeID;
/*a következő alkalmazottra is meghívjuk a függvényt*/
WHILE @Employee IS NOT NULL
	BEGIN
    	INSERT INTO @Reports
        	SELECT *
            FROM fnGetReports(@Employee);
/*tovább ugratjuk a @Employee változó értékét a következő alkalmazottra*/
        SELECT @Employee = MIN(EmployeeID)
        FROM HumanResources.Employee
        WHERE EmployeeID > @Employee
            AND ManagerID = @EmployeeID;
    END
RETURN;
END
GO
```
##Determinizmus
Ha az SQL Server indexelni akar valamit, akkor biztosan kell tudni, hogy mi az, amit indexel. Ez azért fontos itt, mert UDF kerülhet olyan objektumdefinícióba, amit lehet indexelni. Például egy számított mező, vagy `VIEW`. Egy UDF lehet determinisztikus, vagy nem-determinisztikus. Az adott függvény determinisztikusságát nem egy paraméter, hanem a függvény viselkedése határozza meg. Akkor nevezünk valamit determinisztikusnak, ha adott bemeneti paraméterek esetén mindig ugyanaz a vissszatérési értéket kapjuk. A `SUM()` függvény determinisztikus, mert adott inputok esetén mindig ugyanaz lesz az output. A `GETDATE()` függvény viszont nem az, mivel minden hívásnál más és más eredményt fog visszaadni. Egy determinisztikus függvénynek a következő kritériumoknak kell megfelelnie:
- A függvénynek sémakötöttnek kell lennie, tehát a függőségben lévő objektumok közül egyik sem változhat meg anélkül, hogy előbb ezt a kötöttséget fel ne oldanák.
- Minden függvények, amit az UDF használ, szintén determinisztikusnak kell lennie.
- A függvény nem hivatkozhat olyan táblákra, amiket nem a függvényben definiáltak.
- A függvény nem hívhat meg extended sproc-t.

A determinizmus azért fontos, mivel index létrehozása csak determinisztikus adatokon engedélyezett. Az SQL szervernek van beépített függvénye a determinizmus ellenőrzésre, aminek használata a következő: 
```SQL
SELECT OBJECTPROPERTY(OBJECT_ID('<függvény neve>'), 'IsDeterministic');
```
A függvény eredménye 1, ha a függvény determinisztikus, és 0, ha nem.
##Debugging
A hibakeresés ugyanúgy működik, mint egy sproc esetében: Írni kell egy egyszerű scriptet, ami a függvényt használja, majd az F11 segítségével végig lehet lépkedni a parancsokon.
