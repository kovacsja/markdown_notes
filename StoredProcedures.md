# Stored Procedures

## Alapok
**Basic Syntax**:
```SQL
CREATE PROCEDURE|PROC <sproc name>
	[<parameter name> [schema.]<data type> [VARYING] [=<default value>] [OUT [PUT]]	[READONLY]
    [,<parameter name> [schema.]<data type> [VARYING] [=<default value>] [OUT [PUT]] [READONLY]]
    [, ...]]
[WITH RECOMPILE|ENCRYPTION|[EXECUTE AS {CALLER|SELF|OWNER|<user name>}]
[FOR REPLICATION]
AS
	<code> | EXTERNAL NAME <assembly name>.<assembly class>.<method>
```
A paraméterek használata opcionális, a ténylegesen végrehajtandó kód pedig az `AS` kulcsszó uátn fog jönni.
A sproc futtatása az `EXEC` paranccsal hajtható végre:
`EXEC spEmployee;`
A View-khoz hasonlóan a már elkészült eljárásokat a `CREATE` és az `ALTER` paranccsal is módosíthatjuk. A két módszer közötti különbség, hogy az `ALTER` elvárja a megnevezett sproc létezését, míg a `CREATE` nem. Az `ALTER` nem változtatja meg a meglévő sproc jogosultságait, illtve a már kiosztott objectID-t, valamint a beállított függőségi viszonyokat.
## Paraméterek használata
Amennyiben a paramétereket a sproc-on kívül declaráljuk, akkor azokat akár név, akár pozíció alapján is át lehet adni az eljárásnak.
### Pareméterek deklarálása
A paraméterek deklarálásához ezekre van szükség:
- név
- adattípus
- *alapértélmezett érték*
- *irány*

`@parameter_name [AS] datatype [= defalut|NULL] [VARYING] [OUTPUT|OUT]`

- amennyiben az adattípusnak `CURSOR`-t választunk, úgy a VARYING és az az OUT opciókat is kötelező használni.
- a változóktól eltérően a paraméterek nem inicializálódnak `NULL` értékkel. Amennyiben a `DEFAULT` értéket nem adunk meg a deklaráláskor, akkor a sproc kötelezően megadandónak fogja tekinteni.

>Lehet úgy változót deklarálni, hogy az a változó meghívásával csak egy subsetre fusson le, értékadás nélkül pedig a teljes táblára: 
```sql
CREATE PROCEDURE <proc name>
(@variant bigint = NULL) AS
begin
...
WHERE
	(( @variant IS NULL) OR (t.mezo = @variant ))
	AND ...
```

### Output paraméterek használata
Akkor érdemes használni, ha nem 'recordset' információt szeretnénk visszakapni a futás végén. Ilyen lehet például egy kulcs értékénet visszaadása, amin a sproc múveletet hajtott végre, vagy a sproc futásának eredményére vonatkozó információ.

**Hibajelentés beszúrása egy log táblába:**
Amikor a sproc futása végetér, az `OUTPUT` paraméterben lévő értéket visszakapja a hívó.
```SQL
CREATE PRCEDURE [dbo].[uspLogError]
	@ErrorLogID [int] = 0 OUTPUT --ha 0 eredményt ad vissza, akkor nincs hiba
AS
BEGIN
	SET @ErrorLogID = 0;
BEGIN TRY
	...
    ...
	INSERT [dbo].[ErrorLog]
    	(...)
	VALUES (...);
    SET @ErrorLogID = @@IDENTITY;
END TRY
BEGIN CATCH
...
END CATCH
END;
```
* Az `OUTPUT` használatához a kimenő paramétert a sproc belsejében kell deklarálni.
* Az `OUTPUT` kulcsszót a sproc hívásakor és deklarálásakor is használni kell, hogy a szerver fel tudjon készülni a sproc különleges kezelésére. Ha a híváskor nem adjuk meg a kulcsszót, akkor a paraméternek nem lesznek átadva értékek, és nagy valószínűséggel `NULL` értékkel fog visszatérni a sproc.
* A kimeneti eredményben használt változónak nem kell ugyanazt a nevet adni, mint a sproc belső paraméterének.
* Az `EXEC` kulcsszó használata szükséges, ha a sproc hívása nem az első lépés a batch-ben, de ha akkor is használjuk, akkor legalább következetesek vagyunk.
### Sikeresség, vagy kudarc visszaigazolása visszatérési értékkel
A visszatérési értéket `return value` használatának több célja is lehet. Az egyik, hogy konkrét értéket adjon vissza a sproc futása után, de ez nem ajánlott. Erre a célra inkább az előbb leírt `OUTPUT` kulcsszót érdemes használni. A visszatérési értéket arra érdemes használni, hogy információt kapjuk a sproc futásának állapotáról.
Minden spoc ad vissza értéket, attól függetlenül, hogy azt külön leprogramozzuk-e, vagy sem. A sproc alapból 0 értéket ad vissza sikeres futás esetén. A vissazatérési érétk beállítása a következő utasítás használható:
`RETURN [<integer value to return>]` **A visszatérési értéknek egész számnak kell lennie!**
!Fontos, hogy `RETURN` kulcsszó használata azonnali, és feltétel nélkül megszakítja a sproc futását, és a kulcsszó után egyetlen újabb sor sem fog végrehajtódni.
Hogy a visszatérési értéket felhasználjuk, ahhoz azt egy változóban kell eltárolni: `EXEC @ReturnVal = spMySproc;`
### További hibakezelési gyakorlatok
A leggyakoribb hibatípusok:
* hibák, amik `runtime error`-t okoznak, és megakadályozzák a kód további futását
* olyan hibák, amiket az SQL Server elfog, de nem okoznak `runtime error`-t, ezért nem állítják le a kód futását.
* olyan hibák, amik `runtime error` miatt leállítanák a kód futását, de a kezelésükről gondoskodtunk.
* olyan logikai hibák, amiket az SQL Server nem talál meg
(Hogy az egyes kategóriákba mi tartozik az erősen függ a használt szerver verziójától.)

A hibák kezelésének legalapvetőbb módja a `TRY/CATCH` használata.
#### Inline Errors
Ezek azok a hibák, amik nem állítják le a sproc futását, de mégsem sikerül végrehajtani azt, amit a sproc akart.
#### Az `@@ERROR` használata
Az `@@ERROR` rendszerfüggvény tartalmazza az utolsó T-SQL script futásából származó hiba számát. Ha az értéke 0, akkor nem történt hiba a futás alatt. A működése hasonló az `ERROR_NUMBER()` függvényhez, viszont az utolsó csak a `CATCH` blokk hatáskörében működik, és az értéke nem változik, míg az `@@ERROR` minden parancs futtatásakor új értéket kap. Tehát ha fel akarjuk használni a `@@ERROR`-t, akkor azt folyamatosan egy változóba kell eltárolnunk.
```SQL
DECLARE @Error int;
--nincs olyan BusinessEntityID, vagy PersonID, ami 0 értéket venne fel, ezért idegen kulcsot sért az INSERT
INSERT INTO Person.BusinessEntityContact
	(BusinessEntityID, PersonID, ContactTypeID)
	VALUES (0, 0, 1);
SELECT @Error = @@ERROR
--az @Error változó fogja tartalmazni a felmerült hiba kódját.
PRINT 'The value of @Error is ' + CONVERT(varchar, @Error);
--az @@ERROR értéke már 0 lesz, mert az utolsó parancs, a PRINT resetelte a @@ERROR értékét
PRINT 'The value of @@ERROR is ' + CONVERT(varchar, @@ERROR);
```
#### Az @@ERROR használata eljárásokban
Ha a `@@ERROR`-t használjuk hibakezelésre, akkor azt valószínűleg azért tesszük, hogy kompatibilisek maradjuk a SQL Server 2000-rel. Egyébként a `TRY/CATCH` használata javasolt, ami sokkal rugalmasabb hibakezelést tesz lehetővé.
#### Hibák kezelése, mielőtt azok még bekövetkeznének
Az olyan logikai hibákra, amiknek a felmerülését az SQL szerver nem tudja érzékelni, ezért ezeket az eseteket magunknak kell detektálni, és kezelni. Például ha olyan táblában szeretnénk UPDATE-elni, ahol nincs meg a kívánt PrimaryKey, attól a parancs még lefut, csak nem változtat meg semmit a táblában.

```SQL
...
...
UPDATE HumanResources.Employee
SET JobTitle = @JobTitle,
	HireDate = @HireDate,
    CurrentFlag = @CurrentFlag
WHERE BusinessEntityID = @BusinessEntityID

IF @@ROWCOUNT > 0 --a @@ROWCOUNT az utolsó parancs által módosított sorok számát adja vissza
INSERT INTO HumanResources.EmployeePayHistory
	(BusinessEntityID,
    RateChangeDate,
    Rate,
    PayFrequency)
VALUES (@BusinessentityID, @RateChangeDate, @Rate, @PayFrequency);
ELSE
BEGIN
	PRINT 'BusinessEntityID Not Found';
    ROLLBACK TRAN;
    RETURN @BUSINESS_ENTITY_ID_NOT_FOUND; --azonnal megszakítja a futást
END
...
...
```

#### Manuális hibahívás
Ezzel olyan `runtime error`-t is létre lehet hozni, amit saját magunk definiálhatunk, és magától megállítja a program futását.
```sql
RAISERROR (<message ID | message string | variable>, <severity>, <state>
[, <argument>
[,<...n>]])
[WITH option [,...n]]
```
**Message ID / Message String**
Azt az üzenetet határozza meg, amit a hiba fog visszaadni a felhasználónak. A Message ID-val olyan üzenetet lehet visszaadni, ami előre definiált a `master.sys.messages` táblában. Vagy egyszerűen új stringet is megadhatunk neki:
`RAISERROR('Hi there, I''m an error', 1, 1);`

**Severity**
* Informatív jellegű hibák: 1-18
* Rendszerszintű hibák: 19-25
* Katasztrófális hibák: 20-25

A 19 és annál magasabb komolyságú hibák esetén a `WITH LOG` opció kötelező. A 20 és annál komolyabb hibák pedig automatikusan megszakítják a felhasználó kapcsolatát az adatbázissal.

| Severity | Magyarázat |
|--------:|:--------|
|1-10|Csupán információ átadásra szolgál, de pontos hibakódot fog visszaadni.|
|11-16|Ha nincs beállítva `TRY/CATCH` blokk, akkor megszakítják a program futását, és hibaüzenetet küld a kliensre. A state ilyenkor az lesz, amire a hívásnál be lett állítva. Ha használjuk a `TRY/CATCH` blokkot, akkor mi dönthetjük el, hogy mi történjen a hibahívás esetén.|
|17|Általában csak az SQL szerver használja ezt a szintet. Alapvetően azt jelzi, hogy a szerver kifogyott az erőforrásokból, például a `tempdb` megtelt. A `TRY/CATCH` blokk ezt is előbb elfogja, mint a kliens|
|18-19|Ezek olyan súlyos hibák, amik a rendszeradminisztrátort is érdekelhetik. A 19-esnél a `WITH LOG` opció is kötelező, és a hiba a Windows Event Log-ba is bekerülnek. Ezeket a hibákat még a `TRY/CATCH` blokk le tudja kezelni, a komolyabb hibákat viszont már a kliens kezeli.|
|20-25|Olyan `Fatal Error` események, amik azonnal megszakítják a kapcsolatot. Mivel a `WITH LOG` opció itt már kötelező, a hiba oka az *Event Log*ból visszakereshető.|

**State**
A State egy kitalált érték, ami arra jó, hogy megvizsgáljuk, ugyanaz a hiba történik-e meg a kódban több helyen is. A lényeg, hogy valami információt kapjunk vissza arról, hogy hol keletkezik a hiba. Az értéke 1 és 127 között lehet.

**Error Arguments**
Van olyan előre definiált hiba, aminek különböző argumentumokat is meg lehet adni. Ettől a hibaüzenet dinamikusabb lehet egy statikus szöveg helyett.

| Placeholder | Típus |
|--------------:|:----------|
|%D|Előjeles egész szám|
|%O|Előjel nélküli nyolcasszámrendszerbeli szám|
|%P|Pointer|
|%S|String|
|%U|Előjel nélküli egész szám|
|%X, %x|Előjel nélküli hexadecimális szám|

|Jelölők  |Működés    |
|:--------:|:----------------|
|-|Balra igazít, csak akkor van értelme, ha fix széles a szöveg|
|+|kötelezően kirakja az előjelet, ha maga a típus előjeles|
|0|balról 0-ákkal tölti fel a `WITH`-ben hagyott területet|
|#|csak nyolcas és tizenhatos számrendszerű számoknál használható, hogy a megfelelő prefix kikerüljön: 0, 0x|
|' '|a szám bal oldalát szóközökkel tölti fel, ha a z érték pozitív|

* ++With++: előre be lehet vele állítani, hogy az átadott értéknek mekkora helyet hagyjon a szerver a hibaüzenetben. * esetén automatikusan fogja beállítani.
* ++Precision++: a számoknál a maximális számjegyek számát adja meg.
* ++Long/Short++: h jelenti a rövidet, és I a hosszút, amikor szám típusú paramétert adunk át.

Példa:
`RAISERROR ('This is a sample parameterized %s, along with a zero padding and a sign%+010d', 1, 1, 'string' 12121);`
Ami ezt adja majd vissza:
```
This is a sample paremeterized string, along with a zero padding and a sign+000012121
Msg 50000, Level 1, State 1
```
**WITH LOG**
A hiba bekerül az SQL Server error logjába, és a Windows Application Log-ba is. 19-es komolyság fölött kötelező bekapcsolni.

**WITH SETERROR**
Alapértelmezettként a `RAISERROR` nem módosítja a `@@ERROR` értékét. Ehelyett az `@@ERROR`-ba a `RAISERROR` futásának sikeressége, vagy kudarca kerül bele. A `SETERROR` felülírja ezt a viselkedést, hogy hasonlítson a rendszer hibakezelésére.

**WITH NOWAIT**
Azonnal értesíti a klienst a hibáról, és nem vár a program befejezéséig.

#### Saját hibaüzenetek hozzáadása
Van egy rendszerbe épített sproc, a `sp_addmessage`, amivel saját hibaüzeneteket lehet beállítani a szerveren:
```SQL
sp_addmessage [@msgnum = ] <msg id>,
	[@severity = ] <severity>,
    [@msgtext = ] <'msg'>,
    [, [@lang = ] <'language'>]
    [, [@with_log = ] [TRUE|FALSE]]
    [, [@replace = ] 'replace']
```
**@lang**: Itt adható meg, hogy milyen nyelvhez adja hozzá a sproc az új hibaüzenetet. Az értékkészlet a `syslanguages` táblában található.
**@with_log**: ugyanúgy működik, mint a `RAISERROR` esetén. Ha `TRUE` értéket kap, akkor a hibaüzenet bekerül a szerver logjába.
**@replace**: Ha egy meglévő hibaüzenetet akarunk lecserélni a sajátunkra, akkor ennek az változónak az értékét kell 'REPLACE'-re állítani. Ha ezt kihagyjuk, és egy már meglévő `msgID`-t adunk meg, akkor hibával fog lefutni a parancs.
Akármilyen adatbázison futtatjuk ezt a parancsot, az mindig a master adatbázisban fog végrehajtódni. Ezért ha migráljuk az adatbázisunkat egy másik szerverre, akkor azon az új szerveren is létre kell hoznunk a saját hibaüzeneteinket.
#### Saját hibaüzenet eltávolítása
`sp_dropmessage <message number>`
## Table-Valued Parameters (TVP)
A `TABLE`, mint adattípus SQL Server 2005 óta létezik, de akkor csak annyira voltak képesek, mint a tábla eredményt adó UDF-ek.
A TVP-k hasznát az egy-a-többhöz kapcsolatokkal való munkánál lehet érvényesíteni. Például: egy rendelés header részét és a body részét egy művelettel hozzá lehet fűzni az adott táblákhoz, ahelyett, hogy a rendelés részleteit soronkét, több lépésben kelljen isertálni.
```SQL
CREATE TYPE Person.Address
AS TABLE (
	AddressID int NULL,
    AddressLine1 nvarchar(60) NOT NULL,
    AddressLine2 nvarchar(60) NULL,
    City nvarchar(30) NOT NULL,
    StateProvidenceID int NOT NULL,
    PostalCode nvarchar(15), NOT NULL,
    SpatialLocation geography NULL
);
```

## Mire jók a tárolt eljárások

### A feldolgozási eljárások meghívhatóak lesznek
A sproc egy olyan script, amit az adatbázisban tárolunk. Éppen ezért az adatbázisból meghívható, és nem kell minden egyes alkalommal kézzel betöltni. Tárolt ejárások más eljárásokat is meg tudnak hívni SQL Server 2008-ban ez a beágyazás egészen 32 szintig működik.

### Biztonság
A View-khoz hasonlóan az eljárásokkal is adhatunk úgy információt a felhasználóknak, hogy közben nem fedjük fel a mögöttes adatszerkezetet. Ha valakinek joga van egy sproc végrehajtására, akkor mindenre joga van, amit a sproc tartalmaz. Még akkor is, ha a mögöttes táblákhoz közvetlenül nem férhet hozzá. Ez azt is jelenti, hogy a felhasználó a sproc segítésével módosíthat adatot egy táblában még akkor is, ha a mögöttes táblához csak olvasási joga van.

### Teljesítmény
Tárolt eljárások segítségével növelni lehet a rendszer teljesítményét. `CREATE PROC` paranccsal a rendszer elemzi a kódot, az első futtatás során pedig a query végrehajtási tervét optimalizája, és eltárolja a rendszer. A további futtatások során ezt a végrehajtási tervet fogja használni, hacsak a `WITH RECOMPILE` opciót nem használjuk a futtatáskor.

### Hogyan romlanak el a sproc-ok?
A sprocok egyik hibája, hogy alap esetben csak az optimalizálás csak az első futáskor fog eltárolódni, és a későbbi futtatásokkor azt fogja használni. Ezek az eltárolt adatok azonban idővel elavulhatnak. Különösen igaz ez, ha a sproc dinamikusan áll elő. Ekkor ugyanis sosem fut le kétszer ugyanúgy. Ilyen lehet például, ha a sprocon belül egy `IF ... THEN` állítás dönti el, hogy az adott esetben éppen melyik lekérdezés fog futni.
Ezt a hibát úgy lehet megoldani futtatáskor:
```SQL
EXEC spMySproc [paraméterek]
	WITH RECOMPILE
```
Vagy már állandó opcióként megadhatjuk azt az opciót a sproc előállításakor. Ilyenkor a `WITH RECOMPILE` opció közvetlenül az `AS` kulcsszó elé kerül. Ekkor a végrehajtási terv minden egyes futtatáskor újra fog számolódni.
## Rekurzió
Amikor a kód magát hívja meg, arra is vonatkozik a beágyazott eljárásokra vonatkozó limit. Mivel csak 32 szintet lehet egymásba ágyazni, ezért a rekurzió is csak 32-szer tudja lefuttatni magát. Azt követően hibával lép ki az eljárás.
## Debugging
A debugger a management studióban hasonlóan működik a VB debuggerhez. A *Debug* menüben lehet választani a *Start Debugger* (Alt+F5), vagy a *Step Into* (F11) lehetőségek közül.
Az F11 megnyomásával elindul a debugger. A bal oldalon lévő sárga nyíl mutatja, hogy melyik sor végrehajtása következik. Az eszköztáron lévő ikonok pedig az elérheztő opciókat jelölik:
* **Continue**: ezzel végigfut a sproc, vagy a következő *breaking pont*nál megáll.
* **Step Into**: végrehajtja az aktuális kódsort, majd a következő parancsra ugrik, ahol megáll. Amennyiben az adott sor egy függvény, vagy egy másik sproc, akkor meghívódik az adott kód, és az is része lesz a végrehajtásnak, és a debugger annak a beágyazott kódnak az első sorára fog ugrani, és azon belül folytatódik a végrehajtás.
* **Step Over**: Azt a következő sort hajtja végre, ami a `CALL STACK`-ben azonos szinten van az előző utasítással. Amennyiben a következő utasítás nem egy sproc, vagy UDF hívás, akkor egyenértékű a *Step Into*-val. Sproc vagy UDF esetén viszont már a hívás eredményét adja vissza, és nem részletezi belső kód végrehajtását.
* **Step Out**: Minden kódot végrehajt, ameddig nem ugrik a végrehajtás egy magasabb szintre a `CALL STACK`-ben.
* **Stop Debugging**: Leállítja a végrehajtást, de a Debugging ablak nyitva marad.
* **Toggle Breakpoints and Remove All Breakpoints**: lehetőség van töréspontok beállítására, ahol a kód végrehajtása debugging módban leáll. Lehetőség van külön ablakban megjeleníteni az összes megjelölt töréspontot, ami nagyobb sprocok estében hasznos.

### A debugging végrehajtást segítő ablakok:
* **Locals Window**: ebben az ablakban lehet nyomonkövetni a scopeban aktív változókat. Az ablakban lévő változók, és azoknak az értéke is folyamatosan változhat a végrehajtás szakaszaitól függően. Az ablakban minden változónak megjelenik a neve, a jelenlegi értéke, és az adattípusa is. Emellett futtatás közben is meg lehet az egyes változók értékeit változtatni.
* **Watch Window**: itt lehet beállítani azokat a változókat és kifejezéseket, amiknek az értékét figyelemmel akarjuk kísérni a debug során.
* **Call Stack Window**: ez listázza azokat a sprocokat és függvényeket, amik aktív sprocban jelenleg aktívak. Itt lehet követni, hogy a végrehajtásnak éppen melyik szintéjén járunk, és hogy az adott változónak melyik szinten mi az aktuális értéke.
* **Output Window**: az SQL Server ide írja ki az outputjait: recordset, debug információ, visszatérési értékek, output
* **Command Window**: Parancssoros hozzáférést enged a debugging eszközökhöz. Általában nincs rá szükség.

## Az SQLCLR és a .NET Programozás
A .NET alkalmazásával újabb lehetőségek nyílnak az SQL szerverben:
* alap assembly létrehozása, ami akár nem T-SQL kódot is tartalmazhat
* olyan függvényeket hozhatunk létre, amit az UDF-ben nem lehetne
* komplex adattípusokat hozhatunk létre
* külső alkalmazásokat is meghívhatunk

