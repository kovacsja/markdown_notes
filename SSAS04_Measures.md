# Measures and Measure Groups
A dimenziók megtervezése és létrehozása után érdemes hozzálátni a mértékek megalkotásának, mivel ezek hatással vannak arra, hogy milyen extra számításokat tudnunk megadni. 
Témák: 
 - hasznos tulajdonságok
 - aggregálás típusai
 - több `measure group` használata
 - a kapcsolatok beállítása a `measure group`ok és a dimenziók között

## Mértékek és aggregálásuk
A mértékek a kockának azon része, amiket a aggregálni, szűrni, elemezni akarunk. Az SSAS kockában pedig az értéke mögé üzleti logika rendelhető mögéjük, ami könnyíti a velük való munkát. 

### Jó tudni
**Formázás:** Ez a tulajdonság határozza meg, hogy az adott érték milyen formában jelnjen meg a klienseszközökön. A klienseszközök többsége támogatja a formázási beállításokat. 
Maga az SSAS is számos beépített lehetőséget kínál, de *VBA*-hoz hasoló modon saját formátumokat is meg lehet adni: 
 * a %-jel 100-zal megszorozza az értéket as mögé rak egy % jelet, miközben a cella értéke az eredeti marad a további számolásokban
 * ha a `NULL` értékeket helyettesíteni akarjuk valamivel, akkor arra felesleges egy `MDX` számítást bevezetni, inkább formátum beállításával érdems ezt a helyzetet kezelni: #,#.00;#,#.00;#,#.00;\N\A (+/-/0/null)
 * A *Currency* típus és formátumokkal óvatosan kell bánni, mivel a kocka és a rendszerben beállított lokális válozók is befolyásolhatják, így más gépen máshogyan jelenhet meg.
 **Könyvtárakba rendezés:** Alapértelmezetten minden Measure egy MeasureGrouphoz tartozik, de lehetőség van arra, hogy ezen belül is könyvtárakba rendezzük a mértékeket. Az eredeti csoportokat csak alábontani lehet, átrendezni nem. Az almappák csak a megjelenítést fogják befolyásolni, a hivatkozásokat nem. Egy mértéket több almappában is elhelyezhetünk, ha a mappaneveket ;-vel elválasztva adjuk meg. A mappák között több szintet is létre lehet hozni / vagy \ jel használatával. 

### Beépített aggregáló műveletek
A mértékek legfontosabb tulajdonsága, hogy milyen művelettel aggregálódnak fel. 
#### Alaptípusok
 * A `SUM` a legalapvetőbb művelet, ezzel a mértékek felfelé összeadódnak
 * A `COUNT` aggregáció kétféleképpen használható: vagy minden sor rekordot összeszámol a Fact táblából *(Binding Type: Row Binding)*, vagy csak a nem `NULL` értékeket *(Binding Type: Column Binding)*.
 * A `MIN` és a `MAX` egyszerűen a legkisebb és a legnagyobb értéket adja vissza.

#### Distinct Count
A `DistinctCount` egy mező egyedi elemeit számolja össze. Ugyanúgy működik, mint a `Count(Distinct <mezőnév> )` az `SQL`-ben. Ez egy nagyon költséges művelet. Az aggregációt elő lehet állítani `MDX`-ben, és many-to-many kapcsolat használtával is, de általában nem javít a kocka teljesítményén. 
A `DistinctCount` kalkulációknak a BIDS mindig új *MeasureGroup*ot hoz létre, ami megkerülhető, de nem ajánlott. 

#### None
Amennyiben ezt választjuk, úgy értékek csak a dimenziók legalacsonyabb granularitásában fognak megjelenni, a fact- és a dimenziótáblák kulcsainak metszetében, és magasabb szintre nem összegződnek fel. 

#### Semi-additive típusok
*(AverageOfChildren, FirstChild, LastChild, FirstNonEmpty, LastNonEmpty)*
>Alapvetően ezek is úgy viselkedenek, mint a `SUM` típusú mértékek, kivéve az idődimenzió esetében. Ehhez a SSAS-ben megfelelően kell beállítani a dimenzió típusát (Type: Time).

A `Semi-additive` mértékek mindig egy idődimenzióhoz kapcsolódnak. Ha több idődimenzió is kapcsolódik a kockához, akkor az elsőhoz kötődik a mérték. Erről bővebben [itt](http://www.artisconsulting.com/blogs/greggalloway/2009/2/19/lastnonempty-gotchas).
Ezek a számítások akkor hasznosak, ha a Fact táblában állományi adatok vannak letárolva, amiket nincs értelme összeadni. 
* **AverageOfChildren**: A granularitás alján lévő értékek átlagát jeleníti meg. Így ha egy évet választunk ki, akkor napokhoz tartozó értékek átlagát fogjuk megkapni, és nem a hónapokét.
* **FirstChild**: A legalasó granularitási szinten lévő első értéket jeleníti meg. 
* **LastChild**: A legalsó graunálaritási szinthez tartozó utosló időszakhhoz tartozó értéket jeleníti meg. 
* **FirstNonEmpty**: A `FirstChild`hoz hasonlóan működik, de kihagyja azokat az időpontokat, amihez `NULL` érték tartozik.
* **LastNonEmpty**: A `LastChild`hoz hasonló, de kihagyja a `NULL` értékeket. 

A semi-additive mértékek csak egy idődimenzióval együtt működnek. A számítások csak magasabb hierachiaszinten érvényesülnek. A semi-additíve mértékek és a `None` csak az Enterprice Editionben állnak rendelkezésre, de `MDX` szkripttel is lehet helyettesíteni őket, bár a teljesítmény nem lesz ugyanolyan. `LastNonEmpty` helyettesíthető így:

```MDX
SCOPE([Measures].[Sales Amount]);
	THIS = TAIL(
			NONEMPTY(
				{EXISTING [Date].[Date].[Date].Members}
				* [Measures].[Sales Amount])
			,1).ITEM(0);
END SCOPE
```

#### By Account
Ez az opció csak az Enterprise Editionben érhető el, és abban segít, hogy az eddigieknél bonyolultabb üzleti logikát és modellezni lehessen az OLAP kockában. Ez jellemzően egy speciális dimenziót jelent.
Ezt a dimenziót be lehet állítani `Define Account Intelligece wizard` segítségével: Dimension\Add Business Intelligence.

### Dimension calculations
A beépített aggregálófüggvényeken kívül más módon be lehet állítani a mértékek számítási módját. Ezek egyik fajtája ez.

**Unary operators and weights**
A `By Account` típushoz hasonlóan egy dimnezió hierarchiában is lehetőség van arra, hogy a megadjuk, az egyes elemek hogyan aggregálódjank fel a következő (magasabb) szintre. Ehhez a dimenziótáblában egy új mezőt kell létrehozni, ami ezt a műveletet fogja tartalmazni, és a mező típusát `UnaryOperatorClolun`-ra kell állítani. A támogatott műveletek:
 * +: szimpla összeadás
 * -: az adott érték negatívja kerül be magasabb szintre
 * *: az ehhoz tartozó érték összeszorzódik a hierarchiában előtte állóval
 * /: az ehhez tarozó érték le lesz oszva az előtte lévő értékkel
 * ~: az adott érték ki lesz hagyva a felaggregálásból
 * akármilyen szám: az adott elemhez tartozó érték megszorzódik a számértékkel, majd összeadással aggregálódik tovább
 Az így beállított számítosok nagyobb hatékonysággal futnak, mint az erre a célre megírt `MDX` kifejezések.

**Custom Member Formulas**
Ha az Unary műveletek nem elegek ahhoz akkor a dimenziótábla nem csak műveleteket, hanem `MDX` kifejezéseket is be lehet állítani. Ekkor a mező attribútumát `CustomRollupColumn`ra kell állítani. 
Ezzel a módszerrel hasonló eredményt lehet elérni, mint `MDX` szkriptekkel. A kettőt összehasonlítva:
 * tejesítményben a kettő ugyanolyan hatékony
 * a fő különbség az `Custom Member Fromula` és az `MDX` kifejezések között, hogy azok `Closest Pass Wins` szabályt használják `Last Pass Wins` helyett, mikor egy cella értéke két különböző módon is kiszámítható két formulából(??)[^1]. 
 * Amennyiben a számítások benne vannak a dimenzióban, akkor a dimezió magát dokumentálja, és a felhasználók könnybben megérthetik a működésüket.
 * Ha ugyanazt a dimenziót több kockában is alkalmazzuk, akkor a számítások is a dimezióval együtt mozognak, és nem kell az `MDX` számításokat lemásolni.
 * A legfőbb hátrány viszont, hogy így nehezebb a számításokat debuggolni, és a képletek meg lesznek osztva a dimeznió és a kocka `MDX` képletei között. 

[^1]: see the book Microsoft SQL Server Analysis Services 2008 Unleashed (by Gorbach, Berger and Melomed), Chapter 13

**Non-aggregatable values**
Ezt elég ritkán kell alkalmazni. Akkor hasznos, ha egy elemet nem a hierarchiában alatta lévő elemekből kell kiszámítani, hanem közvetlenül az adattárházból kell kiolvasni. Ez gyakran annak köszönhető, hogy az aggrágálást már az ETL alatt elvégzi az adatbázis.
Ha `non-aggregatable` mértéket akarunk használni, akkor azt a legegyszerűbben egy parent/child hierarchiával lehet elérni. Minden `non-leaf member`nek van egy alapértelmezett *datamember*je, aminek a láthatóságát állíthatjuk a `MembersWithData` tulajdonsággal. Alapesetben, ha egy `non-leaf member`höz tartozik érték az adattárházban, akkor az a többi gyerekhez hasonlóan felaggregálódik a hierarchia mentén. Egy `MDX` képlet segítégével viszont árírhatjuk ezt a viselkedést:

```MDX
SCOPE([Measures].[Sales Amount Quota]);
	THIS=[Employee].[Employees].CURRENTMEMBER.DATAMEMBER;
END SCOPE;
```

Ekkor minden gyerek megjelenik a hierachiában a hozzá tartozó értékkel, a szülőhoz viszont nem egy aggregált érték, hanem csak a sajátja fog tartozni. 

## Measure groups
Egy kockát több fact és dimenziótáblát is tartalmazhat, így több `Measure Group`ot is. Ezek a csoportok más-más dimenzióval, és granularitással rendelkezhetnek. 

### Új csoport létrehozása
Új csoport létrehozásakor ki kell választani azt az egyetlen Fact táblát, amire a csoport épülni fog. A DSV-ben definiált minden dimenziókapcsolat meg be lesz állítva a `Dimension Usage` fülön, lehet ezeket átállítani. Teljesítmény szempontjából nem okoz különbséget, hogy hány csoportba vannak a mértékek rendezve. 
**Az egy kocka megközelítés előnyei:**
 - minden adat egy helyen van. ha a felhasználóknak több `Measure Group`ból is lenne szüksége adatra, akkor azt `MDX` kalkulációkkal kell megoldani, de akkor is minden egy helyen van. 
 - a biztonsági beállításokat és a számításokat csak egyszer kell mindig módosítani.
 
**A több kokca megközelítés előnyei:**
- Ha egy komplex kockát kell megvalósítani, de csak a `Standard Edition` áll rendelkezésre, amiben nem lehet `Perspective`kat létrehozni, akkor a több, kisebb kockákkal a felhasználók jobban elboldogulhatnak.
- Biztonsági szempontból egyszerűbb lehet kezelni több kockát, mivel a kockához való hozzáférés könnyen állítható. 
- bonyolultabb kalkulációknál előfordulhat, hogy olyan részére is hatással van a kockának, amire nem számítottunk. Ez kisebb kockáknál könnyebben elkerülhető. 

### Measure Grupe készítése dimenizótáblából
Measure Groupot létre lehet hozni Dimenziótáblából is. Ez akkor hasznos, ha meg kell számolni, hogy hány gyerek van a kiválasztott hierarchiában. Például: a kiválasztott időszak hány napot tartalmaz? Ezt a feladatot egy `COUNT()` aggregációval meg lehet oldani. 
> Amikor el kell dönteni, hogy az `ETL` során készítsünk elő egy számítást, vagy `MDX`-ben írjuk le a logkát, akkor érdemes szem előtt tarani, hogy az MDX sokkal rugalmasabban változtatható, mivel ha változik az üzleti logika, akkor csak a kódot kell megváltoztatni, a teljesítménye viszont a legtöbbször elmarad az előre kódolt aggregátumokétól.

### Különböző dimenzionalitás használata
A `Measure Group`ok létrehozásának általában a fő indoka, hogy más dimenziókapcsolatokat állíthassunk be ugyanazokhoz a mértékekhez. Ha két csoportnak ugyanazokat a dimenziókapcsolatokat álltjuk be, akkor érdemes azokat inkább összevonni. 
Ha egy dimenziónak nincs kapcsolata egy Mértékcsoporttal, akkor kétfajta viselkedésmódot állíthatunk be a Measure Group tulajdonságaiban: 
* `IgnoreUnrelatedDimensions=False`: esetén akkor az `All` tag alatti minden szinten `Null` érétkek fognak megjelenni, kivéve az `Unknown` tagot.
* `IgnoreUnrelatedDimensions=True`: ez az alapértelmezett viselkedés. Ebben az esetben a gyökérelemnél számított érték ismétlődik a dimenzió minden eleménél. 

Ezt a tulajdonságot érdemes `True` állásban hagyni, hiszen ha egy olyan dimenzió alapján szűrik le a kockát, amivel nem minden mértéknek van kapcsolata, akkor olyan helyeken is `NULL` értébe futunk, ahol nem számítunk rá, ezért ezt érdemes óvatosan kezelni. 

### Különböző granularitások használata
Mégha az egyes `Measure Group`ok ugyanazokkal a dimenziókapcsolatokkal rendelkeznek, a granularitásuk eltérhet. Példátul tényadatink lehetnek akár napi bontásban is, de tervek csak havi, vagy negyedévesek szoktak lenni. Ezért az idődimenzióval mind a két Fact táblának lesz kapcsolata, csak más granularitási szinten. 
Hogy a granularitás alatt mi jelenjen meg egy lekérdezésben ugyanúgy beállítható, mint az eltérő dimenziók esetében. Az alapértelmezett `IgnoreUnrelatedDimesions:True` beállítást azonban itt érdemes megváltoztatni, hogy tervet csak arra a szintre kapjunk, amire az vonatkozik.  
### Non-aggretable dimenziók - egy másik megoldás
Ha valamilyen speciális számításra van szükéség, akkor az `MDX` számítás definiálása mindig opció, csak nem minden esetben hatékony. A példában a hierarchia különböző szintjein különböző számítási módszert kellene használni, hogy az adott szint a tervet megkapjuk. A példában 3féle terv van: *manager*, *team lead*, *sales person*, és ezek külön, nem aggregált mértékekben vannak nyivántartva. Az alábbi script ezt kombinálja össze egy mértékbe: 

```MDX
SCOPE([Measures].[Sales Amount Quota]);
	SCOPE([Employee].[Salesperson].[All]);
		THIS=[Measures].[Sales Amount Quota_TeamLead];
	END SCOPE;
	SCOPE([Employee].[Team Lead].[All]);
		THIS=[Measures].[Sales Amount Quota_Manager];
	END SCOPE;
END SCOPE;
```

### Linked Dimensions és Measure Groups
Egy linked objektum létrehozásával több `SSAS` adatbázis is használhatja ugyanazt a dimenziót, vagy Measure Gruopot. Abban az esetben lehet hasznos, ha több egy csapat több adatbázison is dolgozik egyszerre. Az előnye, hogy ezeket az objektumokat csak egyszer kell létrehozni, és leprocesszálni, hogy minden adatbázisban működjenek, és aktuálisak maradjanak. 

A gyakrolatban azonban nem gyakran használják az alábbi okok miatt: 
 * a csatolt objektumban eltárolódnak a forrástábla metaadatia is, így ha egy új attribútomot kellene létrehozni a dimenzióban, akkor nincs lehetőség a dimenzió frissítésére. Törölni kell és újra létrehozni. 
 * a Measure Groupokban található `MDX` számítások is bekereülnek az objektumba, így ha ott olyan hivatkozás is található, ami csak az egyik kockában érvényes, akkor az máshol gondolkat fog okozni. Ezért azokat kézzel kell törölni a csatolt objektum létrehozása előtt. 
 * a megosztott dimenziókat csak egy adatbázis több kockája között lehet használni, más adatbázisban nem. 
 * mesasure groupokat más adatbázisból is át lehet hivatkozni, de amikor erre történik lekérdezés, akkor az a forrás adabázsra fog visszahivatkozni, ami teljesítményromlással jár. 
 
### Role-playing dimenziók
Hasznos és sokat használt funkció, hogy egy dimenziót több kapcsolattal is hozzá lehet csatolni egy Measure Grouphoz. 
>Például az idő dimenziót elég egyszer létrehozni, és utána többször, több különböző kapcsolattal használhatjuk fel a kockában. 

Az egyik hátulütője ennek, hogy a dimenzióban létrehozott hierachiáknak minden egyes kapcsolatnál ugyanaz marad a neve. Így a klienseszközön a felhasználóknak gondot okozhat, hogy éppen melyiket látják a riportban. 

## Dimenzió/Measure Group kapcsolatok
A dimenziók és a mértékcsoportok kapcsolati a következő fajták lehetnek: 
 * [nincs kapcsolat]
 * Regular
 * Fact
 * Referenced
 * Many-to-Many
 * Data Mining

### Fact 
Ebben az esetben a dimenzió nem egy külön táblából épül, hanem a Fact tábla egyik mezőjéből. Az SSAS szempontjából nem sok különbség van a `Regular` és a `Fact` dimenziókkapcsolatok között: 
 * A Fact kapcsolat hajszálnyival jobban teljesít a lefúrásokkor
 * A Fact kapcsolatokat a klienseszközök is érzékelik, ezért azokat esetleg máshogyan jelenítik meg. 
 * csak olyan dimezióra vagy mesasure groupra lehet beállítani, ami a DSV-ben ugyanabban a táblában van (?)
 * A measure group can only have a fact relationship with one database
dimension. It can have more than one fact relationship, but all of them have
to be with cube dimensions based on the same database dimension.

### Referenced
Ebben az esetben egy dimenzió egy másik dimenzión keresztül kapcsolódik a Fact táblához. Egy közvetlen dimenzióban például lehetnek olyan adatok (pl. vevők besorolásai: életkor, lakóhely, stb), amiknek a besorolása egy másik dimenziótáblában van tárolva. 
A kapcsolat létrehozásakor a `Materialize` válsztógomb automatikusan kipipálódik, ami biztosítja az optimális teljesítményt, ami annyit tesz, hogy a processzálás alatt fog a join művelet megtörténni, és nem lekérdezéskor. Az `Materialize` opció elhagyásakor a számítás a lekérdezési teljesítményt fogja rontani. 

### Data mining
(erről többet majd valamikor később...)
Amikor egy `Mining Structure`t állítunk be a kockában, akkor lehetőség van ezt dimenziókétn is csatolni a Measure Grouphoz.

