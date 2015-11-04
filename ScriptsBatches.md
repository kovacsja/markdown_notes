#Scripts and Batches
@(devel)[sql,tanul]

>**Script**: fájlban tárolható, hogy újra és újra felhasználható legyen, és általában egyszerre lehet végrehajtani az egész fájlt. 
>**Batch**: T-SQL parancsok egy logikai egységbe rendezése. Ha az parse-time alatt hiba lép fel, akkor semmi sem fut le, míg runtime alatt merül fel hiba, akkor a hibáig lefutnak a parancsok. 

Hogy egy script-et batch-ekre bontsuk, ahhoz a `GO` parancsot kell használni: 
- a GO parancsnak egyedül kell a sorban lennie (csak komment lehet mellette)
- a script elejétől, vagy az előző GO utasításig található parancsokból egyetlen végrehajtási terv készül, és a többi batch-től függetlenül küldi el a szerverre
- ez nem a T-SQL saját parancsa, de főleg az SQL Server termékcsaládban használják

A Batch-ek akkor hasznosak, ha valaminek időben előbb, vagy külön kell megtörténnie a script többi részétől. 
Kénytelenek vagy Batch-eket használni és `GO` paranccsal elválasztani a script többi részétől az alábbi utasításokat: 
  - `CREATE DEFAULT`
  -  `CREATE FUNCTION`
  -  `CREATE PROCEDURE`
  -  `CREATE RULE`
  -  `CREATE SCHEMA`
  -  `CREATE TRIGGER`
  -  `CREATE VIEW`
A `DROP` utasításoknak csak akkor kell külön batch-be kerülniük, ha a törölni kívánt objektumot újra létre akarjuk hozni. Ekkor számítani kell nevek ütközésére. 

##SQLCMD
Az `splcmd` egy olyan parancs, amivel a windows parancssorából lehet SQL lekérdezéseket futtatni. 
Általános szintakszis: 
```
sqlcmd
[
{ { -U <login id> [ -P <password> ] } | –E }
]
[-S <server> [ \<instance > ] ] [ -H <workstation> ] [ -d <database> ]
[ -l <time out> ] [ -t <time out> ] [ -h <headers> ]
[ -s <col separator> ] [ -w <col width> ] [ -a <packet size> ]
[ -e ] [ -I ]
[ -c <cmd end> ] [ -L [ c ] ] [ -q "<query>" ] [ -Q "<query>" ]
[ -m <error level> ] [ -V ] [ -W ] [ -u ] [ -r [ 0 | 1 ] ]
[ -i <input file> ] [ -o <output file> ]
[ -f <codepage> | i:<codepage> [ <, o: <codepage> ]
[ -k [ 1 | 2 ] ]
[ -y <display width> ] [-Y <display width> ]
[ -p [ 1 ] ] [ -R ] [ -b ] [ -v ] [ -A ] [ -X [ 1 ] ] [ -x ]
[ -? ]
]
```

##Dinamikus parancsok generálása menet közben, és az EXEC utasítás
Van, hogy csak menet közben derül ki, hogy mit is kell a *runtime* alatt futtatni. Ilyenkor az `EXEC()` vagy az `EXECUTE()` parancs használható. A két utasítás végrehajtása között nincs különbség. 
Az `EXEC`utasításnak szövegként kell átadni az összefűzött lekérdezést.
`EXEC ({<string variable>|’<literal command string>’})`

###Trükkök
 - az EXEC-en belüli utasításnak más a hatóköre, mint az azt hívó SQL utasításnak. Az EXEC utasításban deklarált változót nem tudja meghívni a futtató utasítás
```sql
DECLARE @InVar varchar(200);

SET @InVar = 'DECLARE @OutVar varchar(50)
	SELECT @OutVar = FirstName FROM Person.Person
	WHERE BusinessEntityID = 1
SELECT ''The Value Is ''+ @OutVar'; --ha két aposztrof kerül egymás mellé, akkor az első excape karakterként működik

EXEC (@Invar);
```
 - Alapértelmezett állapot szerint az EXEC az aktuális felhasználó jogosultságaival rendelkezik. Ha ezt felül akarjuk írni, akkor az `EXECUTE AS` parancsot kell használni
 
 - Ugyanazokat a kapcsolatokat és tranzakciós kontextusokat használja, amit a hívó SQL parancs
 
 - Azokat a szövegrészleteket, amik függvényhívást is tartalmaznak, azokat még az EXEC előtt meg kell hívni, és úgy belefoglalni az EXEC hívásba. 
```sql
DECLARE @NumberOfLetters AS int;
SET @NumberOfLetters = 3;

DECLARE @str AS varchar(255);
SET @str = 'SELECT LEFT(LastName,' + CAST(@NumberOfLetters AS varchar) + ') AS
FilingName FROM Person.Person';

EXEC(@str);
```
 - EXEC nem használható felhasználó által létrehozott függvényben *(user-defined function - UDF)*

##Control Flow

###IF ... ELSE ...
```sql
IF <boolean kifejezés>
	<SQL kifejezés> | BEGIN <kód blokk> END
[ELSE
	<SQL kifejezés> | BEGIN <kód blokk> END]
```
Ha a boolean kifejezésben `NULL`-t akarunk használni akkor annak a biztos módja: `IF @myvar IS NULL`, amennyiben az ANSI_NULLS opció ON-ra van állítva. 
Amennyiben olyan helyzet állna elő, hogy `NULL`-hoz kell bármit hasonlítani, akkor az eredmény mindig `FALSE`-ként fog kiértékelődni. 
####A kód blokkokba rendezése
A blokkokba rendezés lényege, hogy a kód minden részét vagy lefuttatjuk, vagy nem. Az `IF` parancs alapesetben csak az első SQL utasítást tekinti az utasítás részének. Emiatt ha több utasítást is futtatni akarunk az `IF` bármelyik ágán, akkor azokat `BEGIN ... END`parancsok közé kell helyezni, és a blokkba is el lehet helyezni további `IF` elágazásokat. 
####A CASE utasítás
A `CASE` utasítást többféle szintaxissal is meg lehet írni. Vagy egy kifejezés több értékére lehet elágazásokat felépíteni:
```sql
CASE <kiértékelendő kifejezés>
	WHEN <eredmény 1> THEN <akkor kifejezés>
	WHEN <eredmény 2> THEN <akkor kifejezés>
	[...n]
	[ELSE <akkor kifejezés>]
END
```
vagy soronként megadni a kiértékelendő kifejezéseket, és azokra építeni fel az elágazást:
```sql
CASE
	WHEN <boolean-ra kiértékelhető kifejezés 1> THEN <eredmény 1>
	WHEN <boolean-ra kiértékelhető kifejezés 2> THEN <eredmény 2>
	[...n]
	[ELSE <eredmény>]
END
```
Amennyibe az `ELSE` ág kimarad, és a `CASE` egyik ága sem igaz, akkor a kifejezés `NULL`-t ad eredményül. 
Amennyiben a `WHEN` ágakból több is igaz eredményt ad vissza, akkor mindig az első igazhoz tartozó `THEN` hajtódik végre, és utána befejeződik a `CASE` végrehajtása. 

###Hurkok és a WHILE utasítás
Alap szintaxis: 
```sql
WHILE <boolean kifejezés>
	<SQL parancs> 
	| [BEGIN
		<kód blokk>
		[BREAK]
		<SQL parancs>|<kód blokk>
		[CONTINUE]	
	END]
```
A `BRAKE` megtöri a ciklust, mielőtt az a végére érne, és a `WHILE`-ban lévő kifejezés még egyszer kiértékelődne. 
A `CONTINUE` parancs szintén megtöri az éppen aktuális kód végrehajtását, de nem lép ki a ciklusból, hanem visszaugrik a `WHILE`-ra és újra kiértékeli azt (és kilép, ha az az kifejezés már nem IGAZ értéket ad vissza). 

###WAITFOR
Ez a parancs arra szolgál, hogy késleltessük, vagy időzítsük egy parancs futásának indulását.
```
WAITFOR 
	DELAY <'time'> | TIME <'time'>
```
A késleltetésre a `DELAY`paraméter szolgál, aminek óra:perc:másodperc értéket lehet átadni szövegként, de maximum 24 órát. 
Az időzítésnél a `TIME` paramétert kell használni, ami a nap egy bizonyos időpontjában fogja elindítani az utána következő kódblokkot. 24-órás formátumban kell megadni a nap egy időpontját. 

###TRY/CATCH
Az SQL Server 2005 óta támogatott. A hibakeresést és a futás közbeni hibák jobb kezelését segíti. 
```sql
BEGIN TRY
	{ <SQL kifejezés/ek> }
END TRY
BEGIN CATCH 
	{ <SQL kifejezés/ek> }
END CATCH [ ; ]
```
A szerkezet futtatásakor a szerver elkezdi futtatni a `try` blokkban lévő kódot, és ha 11-19 hibaszintű hiba merül fel, akkor azonnal megszakítja a futást, és átugrik a `catch` blokkra. 
> - hibaszint 1-10: csak tájékoztatás. pl: megváltoztak a beállítások, számítás közben NULL értéket talált a program
> - hibaszint 11-19: viszonylag súlyos hibák, amik azonban a kódon belül javíthatóak (pl. idegen kulcs hibák, memória túlcsordulás). Nem feltétlenül kell mindent a kódon belül megoldani, de tisztességesen meg lehet így állítani a futást. 
> - hibaszint 20-25: nagyon súlyos hibák, amik általában rendszerszintűek. Ezekről sosem kapunk értesítést, mivel a rendszer azonnal megszakítja a futást, és a `catch` blokk sosem hajtódik végre. 

A `try... catch` módszer nem minden hibaesetre alkalmazható, de a többség a 11-19-es szakaszba esik. 
```sql
BEGIN TRY
	<SQL parancs>
END TRY
BEGIN CATCH
DECLARE	@ErrorNo 	int,
		@Severity	tinyint,
		@State		smallint
		@LineNo		int,
		@Message 	nvarchar(4000)
SELECT
	@ErrorNo = ERROR_NUMBER(),
	@Severity = ERROR_SEVERITY(),
	@State = ERROR_STATE(),
	@LineNo = ERROR_LINE(),
	@Message = ERROR_MESSAGE()
IF @ErrorNo = 2714 -- obeject exists error
	PRINT 'WARNING TEXT'
ELSE
	RAISEERROR (@Message, 16, 1 )
END CATCH
```
> - ERROR_NUMBER() : A hiba tényleges száma. Ha egy rendszer hibáról van szó, akkor a bejegyzés készül róla a `sys.messages` táblába, ahol további információk is találhatóak a hibával kapcsolatban. 
> - ERROR_SEVERITY(): Ez a függvény a hiba szintjét adja vissza. 
> - ERROR_STATE(): Az értéke 1 a rendszer hibák esetén. Saját hibaesemények esetén mi adhatunk neki értéket. 
> - ERROR_PROCEDURE(): Csak tárolt eljárások, függvények, és triggerek esetén van értelme. Azt az eljárást adja eredményül, amelyik a hibát kiváltotta. 
> - ERROR_LINE(): Melyik sorban következett be a hiba. 
> - ERROR_MASSAGE(): A hiba szövege, ami rendszer hibák esetén a `sys.messages`-be kerül. Saját hibaesemények esetén mi adhatjuk meg a RAISEERROR() függvény paramétereként. 

