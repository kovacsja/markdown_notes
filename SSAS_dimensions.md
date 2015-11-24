#Egyszerű dimenzió- kockaépítés
##Új AS projekt létrehozása
Alapvetően kétféleképpen dolgozhatunk a BIDS-ben (Business Intelligence Developement Studo):
* Projekt módban: amikor lokálisan készíted elő a programot, és csak akkor kerül fel a szerverre, amiror már készen vagyunk a publikálásra, illetve
* Online módban: amikor élesben lehet módosítani az AS adatbázist, és minden mentés alkalmával a változások a szerverre kerülnek mentésre.

Általában javasolt projekt módban dolgozni, és ehhez egy-két tulajdonságot be kell állítani. Ezt a `Solution Explorer`ben a Projekt nevére kattintva lehet megtenni:
* Build
 *  Amennyiben a szerver és BIDS nem ugyanaz a változat (Standard/Developer/stb.) akkor érdemes Standardra állítani, és akkor a deployment során hibaüzenetet ad kapunk, ha olyan funkciót akarunk használni, ami a szerveren nem elérhető.
* Deployment
 * Processing Option: ennek az opciónak a beállításával lehet szabályozni, hogy deployment után az adatbázis automatikusan leprocesszálja-e magát.
 * Server: itt lehet beállítani, hogy a deployment hová történjen. Alapértelmezetten ez a localhost.
 * Database: Ez annak az adatbázisnak a neve, ahová a deployment történik.

##Adatforrás (data source) létrehozása
Annak ellenére, hogy lehetőség van több DataSoruce objektum létrehozására, ez ellenjavalt. Könnyebben karbantartható ugyanis egy olyan modell, ahol minden adat egyetlen adatpiacon megtalálható.
##Data Source View (DSV) létrehozása
Ha következetesen neveztük el a Fact és a Dimenziótáblában a mezőket, akkor a táblák hozzáadása után a DSV automatikusan felismer egyes kapcsolatokat a névkonvenciók alapján. Mielőtt azonban bármilyen táblát hozzáadnánk a DSV-hez, édemes bizonyos dolgokat előre beállítani. Ehhez egy üres DSV-területen jobb gomb menüből kell indítani:
* Retrieve Relationships: alapértelmezetten `TRUE` értéket kap, ami azt befolyásolja, hogy a táblák közötti kapcsolatokat a BIDS próbálja-e meg automatikusan kitalálni a nevek és az idegen kulcsok alapján.
* SchemaRestriction: itt lehet vesszőkkel megadni, hogy melyik schema táblái jelenjenek meg az Add/Remove párbeszédablakban.
* NameMatchingCriteria: ha a Retrieve Relationships True-ra van állítva, akkor itt lehet finomhangolni, hogy a táblák közötti kapcsolatok finomhangolása milyen algoritmusok mentén történjen.
Rendkívül hasznos tud lenni, ha view-kat hivatkozunk be a DSV-be, mivel ekkor nincsenek idegen kulcsok a táblában, így itt rugalmasabban kapcsolhatóak egymáshoz.
`Named Query`-k és `Named Calculation`-ok segítségéven a DSV-ben lehet külön számításokat és táblákat készíteni, de ez csak abban az esetben javasolt, ha az alap adatszerkezeteket nincs jogunk módosítani. Ezért is javallott, hogy minden AS objektum egy VIEW-ra hivatkozzon, mivel azoknak a módosítása sokkal egyszerűbb, mint a tábláké.
##Egyszerű dimenziók tervezése
###Varázsló használata
Ha varázslóval is hozunk létre egy dimenziót, még utána is szerkeszthetjük, és finomhangolhatjuk. A varázsló első kérdése, hogy magunk választjuk ki a dimenzió forrásadatait, vagy a BIDS hozzon létre dimenzió, és töltse fel adatokkal automatikusan.
Az első lépésben ki kell választani a dimenzió fő tábláját, és azt a mezót, ami közvetlenül kapcsolódik a ténytáblákhoz, valamint a dimezió legalacsonyabb granularitását tartalmazó mezőt. (A kettőnek nem feltétlenül kell ugyanannak lennie.)
*A Kulcs.* Ez az a mezőhalmaz, ami egyedileg azonosít egy dimnezióelememet. Általában ez a tábla surrogate kulcsa. *A Név* az az oszlop a dimenziótáblában, amit a végfelhasználó a felületen látszi szeretne. A névnek a kulccsal ellentétben nem kell egyedinek lennie. Ha az adatszerkezet hópehely mintát követ, akkor ezen túl még a kapcsolódó táblákat is be kell állítani
>A dimenzióvarázsló folyamatában több ponton is lehetőség van az elnevezések megváltoztatására. Érdemes elnevezési konvenciókat a végfelhasználókkal közösen kialakítani. Bármilyen átnevezés a fejlesztés során hatással lehet már elkészített kalkulációkra.
>Ugyanakkor az objektum létrejöttekor a Name tulajdonsága - ami bekerül a projektet leíró XMLA dokumentuma - az először adott név lesz. Ezért érdemes már elsőre jó nevet választani az objektumoknak, mert később sok félreértést el lehet ezzel kerülni.

###Dimenziószerkesztő használata
####Új attribútumok hozzáadása
Újabb attribútomokat a hivatkozott táblából bármikor hozzáadhatunk a dimenzióhoz, és ezeket az következő tulajdonságokkal lehet ellátni:
- *Keycolumns, NameColumn*: a kulcs, vagy kulcs mezők tesznek egyedivé egy adott attribútumot. Ha nem a tábla kulcsáról van szó, akkor általában ez szolgál névnek is, azonban egy kevésbé leíró kulcsmező helyett külön mezőt is meg lehet adni, amiből az elnevezéseket fogja tölteni.
- *AttributeHierarchyEnabled*: alapesetben `True`-ra van állítva, ami azt jelenti, hogy kocka böngészésénél hozzá lehet adani valamilyen csoportosító mezőként. A `False` érték viszont azt jelenti, hogy csak értékként lehet megjeleníteni egy másik attribútum tulajdonságaként. Ilyen lehet például egy személy telefonszáma, mire valószínűleg sosem fogunk szűrni, de adatként szükség lehet rá a kimutatásokban.
- *AttributeHierarchOptimizedState*: ez a tulajdonság szabályozza, hogy az adott dimenzióra épüljön-e index. Ez természtesen megnöveli a process idejét, de növeli a lekérdezések teljesítményét is.
- *OrderBy, OrderByAttribute*: Ez szabályozza, hogy az attribútum értékei milyen sorrendben jelenjenek meg a kimutatásokban. A hónapok nevinél például nem ABC-rendet akarunk követ, hanem a sorrendet egy másik attribútum, vagy kulcs alapján akarhatjuk meghatározni.
- *AttributeHierarchyOrdered*: ha a dimenzió elemeinek sorrendje nem számít, viszont nagy a dimenzió, akkor a rendezés kikapcsolása gyorsít a feldolgozáson.
- *IsAggregatable, DefaultMember*: ha az attribútumra engedélyezve van akkor ez alapból igazra van állítva, és ekkor az attrubútumnak van `All Member` tagja, ami felösszesíti az attribútum minden elemét. És ez az elem lesz a `Default Member` hacsak nem jelölünk ki egy másikat. Ha van egy külön `Default Member` - például az aktuális év - akkor a dimenzió automatikusan az le lesz szűrve az alapértelmezett elemre. Az esetek többségében azonban csak zavaró, ha nem a felhasználók választhatják ki a megfelelő paramétereket.

####Idő dimenzió beállítása
Egy idő dimenzió létrehozása nem sokban különbözik egy szimpla dimenzió létrehozásától, de van egy-két speciális paraméter, amit be kell állítani. A varázsló helyett érdemes manuálisan létrehozni a dimenziót. Így ugyanis magunk állíthatjuk be a neveket és formátumokat. A legfontosabb, hogy a dimenzió typusának *Time* értéket adjunk, hogy a BIDS is tudja, hogy idő dimenzióról van szó. Egyes függvények, csak így működnek. Ezuátán pedig ezek után pedig a többi attribútumot kell beállítani a legközelebbi megfelelőjénak (Év, hónap, stb.).
####Hierarchiák létrehozása
Arra szolgál, hogy a dimenzió attribútumait újabb csoportba tudjuk rendezni. A saját hierachiák létrehozása segíti a felhasználókat kötött lefúrási struktúrák létrehozásával, valamint egyes MDX művelet is ki tudják használni a hierachia szülő-gyerek kapcsolatát, valamint a processzálás miatt az előre definiált hierachiák mentén jobb teljesítménnyel lehet lekérdezni. A saját hierarchiák használatánál meg kell válogatni, hogy melyiket tesszük láthatóvá a végső felhasználó számára, hogy az ne legyen zavaró.
####Attribútum-kapcsolatok beállítása
Itt van lehetőség az attribútumok között egy-a-többhöz kapcsolatok beállítására. A kapcsolatok beállítása opcionális, viszont a felhasználói felületen nem okoz változást, és így a végfelhasználót nem zavarja össze.
Egy dimenzió attribútumai többféleképpen is összekapcsolhatóak, ám nem minden változat optimális. Egy dimenzió létrehozásakor az előre definiált kapcsolatrendszer helyes, de nem feltétlenül optimális. Ebben minden nem kulcs attrubútum a kulcs attribútumhoz kapcsolódik. Azokat a hierarchiákat, amiknél minden szint között be van állatva az egy-a-többhöz kapcsolat *természetes* hierarhciának nevezzük. A nem természetes hierarchiák esetében a tervezőben figyelmeztetés jelenik meg. Ez nem azt jelent, hogy a beállítás hibás, csak hogy nincs megfelelően beállítva. Nem-természtes hierarchiákat is létre lehet hozni, bár ezek teljesítménye elmarad a természtes hierarchiákétól.
> Gyakori hiba a hierarchiák beállításakor, hogy az évek és a hónapok között nincs egy-a-többhöz kapcsolat, mivel Január van minden évben. Hogy az idő dimenzió jól működjön, ahhoz korrigálni kell az alapadatokat, és létrehozni egy `YYYY-MMM` mezőt, amire már igaz lesz, hogy egy-a-több kapcsolatban van az év attribútummal.

A `RelationshipType` tulajdonság beállítással az lehet elérni, hogy a kapcsolat az egyes attribútumelemek között ne változhasson (Rigid), vagy változhasson (Flexible). A Rigid beállítás dátum dimenzióknál jellemező, ahol egy adott dátum mindig ugyanabban a hónapban fog szerepelni. Míg a egy-egy személy lakhelye változhat. Amennyiben nem vagyunk biztosak a dolgunkban, érdemes mindig a Flexible lehetőséget választani. A Rigid beállítás előnye, hogy a gyorsabb feldolgozást tesz lehetővé, mert a hozzá tartozó aggregátumok nem fognak újraszámolódni.
Az attribútumok kapcsolatainak beállítása további jelentősége, hogy a tag-tulajdonságokat is szabályoz: Ha egy hónapban sok dátum van, akkor a dátumoknak van egy hónap tulajdonsága. Alapértelmezetten minden kapcsolat látható, és így a tulajdonságok szűrhetőek, és riportálhatóak. Ha ezt el akarjuk rejteni, akkor a kapcsolat `Visible` tulajdonságát kell false-ra állítani.
###Egyszerű kocka
####Az Új Kocka varázsló
A *Select Creation Method* lépésben válasszuk forrásnak a meglévő táblát. Ha előre és jól megterveztük az adatbázist a kocka mögött, akkor mindig érdemes ezt választani. Ezután ki kell választani, hogy a használni kívánt *Measure*ök melyik táblában találhatóak, valamint, hogy mely mezőket akarjuk mértékként használni SUM() művelettel. Majd ki lehet választani a kockához használni kívánt új dimenziókat. Az új dimenzió létrehozását inkább ne használjuk itt. Az utolsó lépésben pedig adjunk nevet a kockának.
####Deployment
A Deploy a Build menüből indítható, és egy két fázisú folyamatot takar:
 - Először egy build készül a projektből egy XMLA fálj készül, ami magának az AS adatbázisnak a definíciója.
 - A második lépésben pedig ez az XMLA kerül bele egy `ALTER` parancsba, és kerül végrehajtásra az AS szerveren. Ez létrehozza vagy módosítja a meglévő adatbázist.
Amennyiben a Deployment hibára fut, akkor az error logból lehet kiolvasni a hiba forrását és javítani azt.
####Processing
Amennyiben a `Process Option`-t `Do Not Process`-re állítottuk akkor a Deployment után még nem tudjuk böngészni az adatbázist, mivel az még csak egy üres struktúra a Processzálás nélkül. Az objektumok adatokkal történő feltöltéséhez a `Database` menüből kell a `Process` parancsot választani, ahol az egész adatbázisra le lehet futtani a feldolgozást. Az AS adatbázis processálása objektumonkét is lefuttatható, de a teljes processzálás ezeket megpróbálja párhuzamosan futtani, hogy időt nyerjen. A processzálás során minden korábbi adat kiürül az adatbázisból, és felülíródik az új futtatás eredményével. A processzálás alatt nem lehet hozzányúlni a feldolgozás alatt álló AS adatbázishoz.
**A processzálás során fellépő leggyakoribb hibák:**
 - Ha a mögöttes séma úgy változik meg, hogy érvénytelen referenciák kerülnek bele. Ilyen lehet például egy DSV-ben használt tábla/mező átnevezése, vagy megváltozása.
 - Kulcs-hibák: Akkor állnak elő, ha a dimnenzió- és ténytáblák közötti dimenziókulcsok `inner join` kapcsolataiban hiba van: ha a ténytáblákban olyan dimezenziókulcs van, ami a dimenziótáblákból hiányzik. Ennek a hibának kezelésére több lehetőség közül lehet választani a processzálás párbeszédablakban a `Change Settings` menüpont alatt. Ezt a fajta hibát ignorálhatjuk, és ekkor a ténytáblából a "kakukktojás" adatok nem kerülnek feldolgozásra, vagy az adott dimenzióból lehet ezekhez egy előre definiálrt értéket választani. Ezek használata azonban ellenjavalt, és a kulcsok konzinsztenciájáról érdemes az ETL folyamat során gondoskodni.
 - Az objektumok processzálásának rossz sorrendje is kulcshibát válthat ki. Például ha egy ténytáblában azért nem talál egy érték dimenziót, mert az adott dimenzió még nem került frissítésre.
 - MDX Script hibák: könnyen előfordulhat, hogy az AS adatbázisban bekövetkezett strukturális változtatások, akkor azok tönkretehetik az MDX számításokat is, amik átnevezett mezőkre/táblákra hivatkoznak. Ekkor pedig az egész processzálás kudarccal végződik.