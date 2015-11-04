#Views
@(devel)[sql,tanul]

Egy `VIEW` elsődleges feladata, hogy lekérdezéseket tároljon el, és ezzel könnyítse meg a felhasználók dolgát.
```sql
CREATE VIEW séma.[view név] [(oszlopok, )]
[WITH [ENCRIPTION] [, SCHEMABINDING] [, VIEW_METADATA]]
AS
[SELECT lekérdezés]
[WITH CHECK OPTION]
```
Egy `VIEW`-n lehet `INSERT`, `UPDATE` és `DELETE` parancsokat is futtatni, amennyiben:  
- a `VIEW` nem tartalmaz joinokat. (ha tartalmaz, akkor csak update utasítást lehet rajta futtatni, ha az csak egy igazi táblát érint)
- ha a `VIEW` csak egy táblából hivatkozik mezőket, és minden kötelező mezőt meghivatkozik, akkor lehet `insert` utasítást futtatni rajta
- egy bizonyos fokig lehet befolyásolni, hogy milyen adatokat szúrhatunk be, vagy frissíthetünk a `view`-ban
- ezeknél az adatmódosító view-knál gyakran használni kell az `instead of` triggert, ami arra képes, hogy az utasítás által elvárt viselkedés helyett egy másik algoritmust indít be. 
##View módosítás és törlése
Kétféleképpen történhet: `ALTER VIEW` és `CREATE VIEW`parancsok használhatóak erre: 
- Az `alter view` számít egy már meglévő view-ra, míg a `create view` nem. 
- az `alter view` megtartja az eredeti view jogosultságait
- az `alter view` megtartja az eredeti view függőségeit
- mind a két parancs teljesen felülírtja az már létező view-t

`VIEW` törlése:
`DROP VIEW <view neve>, <>`

##View kódjának vizsgálata
1. sp_helptext
`EXEC sp_helptext 'view neve'`
2. OBJECT_DEFINITION()
Ez az eljárás lenne a kívánatos, mert ha frissül a program, és vele az adatbázis-objektumokat leíró kifejezések, akkor ez a függvény frissül, és azokat adja vissza. Emellett egyszerűbben használható egy nagyobb program részeként is. 
`OBJECT_DEFINITION(<object id>)`
Az `object id` megtalálása viszont annyira már nem egyszerű, ahhoz is egy beépített függvényt kell használni. 
`SELECT OBJECT_DEFINITON ( OBJECT_ID(N'view_neve));`
3. sys.comments
A sys.comments egy olyan rendszertábla, ami leírja az adatbázis objektumait. És ehhez nem kell azokat közvetlenül meghívnunk. 
```SQL
SELECT sc.text
FROM sys.syscomments sc
JOIN sys.objects so
	ON sc.id = so.object_id
WHERE so.name = 'view neve'
	AND ss.name = 'séma neve';
```
##Particionált táblák és view-k
A gyorsabb feldolgozás érdekében a nagy táblákat szét lehet szabdalni egy dimenzió mentén, amikből utána `union all` használatával lehet olyan view-t késztíteni, ami ismét az összes adatot tartalmazza. Ha ezeket a táblákat `constrain` definiálásával hozzuk létre, amelyek disjunkt halmazokat alkotnak, akkor a `view`-n lehet akár `update`, `insert` és `delete` műveleteket is végrehajtani, amik a megfelelő táblákhoz nyúlnak hozzá. 
```sql
CREATE TABLE OrderPartitionFeb08
  (OrderID		int		NOT NULL,
  OrderDate		date	NOT NULL
	  CONSTRAINT CKIsFebOrder
	  CHECK (OrderDate >= '2008-02-01' 
		  AND OrderDate < '2008-03-01'),
	CustomerID	int		NOT NULL,
	CONSTRAINT PKOrderIDOrderDateFeb
	  PRIMARY KEY (OrderID, OrderDate)
  );
```

