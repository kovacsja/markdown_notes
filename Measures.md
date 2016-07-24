#Measures and Measure Groups
A dimenziók megtervezése és létrehozása után érdemes hozzálátni a mértékek megalkotásának, mivel ezek hatással vannak arra, hogy milyen extra számításokat tudnunk megadni. 
Témák: 
 - hasznos tulajdonságok
 - aggregálás típusai
 - több `measure group` használata
 - a kapcsolatok beállítása a `measure group`ok és a dimenziók között

##Mértékek és aggregálásuk
A mértékek a kockának azon része, amiket a aggregálni, szűrni, elemezni akarunk. Az SSAS kockában pedig az értéke mögé üzleti logika rendelhető mögéjük, ami könnyíti a velük való munkát. 

##Jó tudni
**Formázás:** Ez a tulajdonság határozza meg, hogy az adott érték milyen formában jelnjen meg a klienseszközökön. A klienseszközök többsége támogatja a formázási beállításokat. 
Maga az SSAS is számos beépített lehetőséget kínál, de *VBA*-hoz hasoló modon saját formátumokat is meg lehet adni: 
 * a %-jel 100-zal megszorozza az értéket as mögé rak egy % jelet, miközben a cella értéke az eredeti marad a további számolásokban
 * ha a `NULL` értékeket helyettesíteni akarjuk valamivel, akkor arra felesleges egy `MDX` számítást bevezetni, inkább formátum beállításával érdems ezt a helyzetet kezelni: #,#.00;#,#.00;#,#.00;\N\A (+/-/0/null)
 * A *Currency* típus és formátumokkal óvatosan kell bánni, mivel a kocka és a rendszerben beállított lokális válozók is befolyásolhatják, így más gépen máshogyan jelenhet meg.
 **Könyvtárakba rendezés:** Alapértelmezetten minden Measure egy MeasureGrouphoz tartozik, de lehetőség van arra, hogy ezen belül is könyvtárakba rendezzük a mértékeket. Az eredeti csoportokat csak alábontani lehet, átrendezni nem. Az almappák csak a megjelenítést fogják befolyásolni, a hivatkozásokat nem. Egy mértéket több almappában is elhelyezhetünk, ha a mappaneveket ;-vel elválasztva adjuk meg. A mappák között több szintet is létre lehet hozni / vagy \ jel használatával. 

##Beépített aggregáló műveletek
A mértékek legfontosabb tulajdonsága, hogy milyen művelettel aggregálódnak fel. 
###Alaptípusok
 * A `SUM` a legalapvetőbb művelet, ezzel a mértékek felfelé összeadódnak
 * A `COUNT` aggregáció kétféleképpen használható: vagy minden sor rekordot összeszámol a Fact táblából *(Binding Type: Row Binding)*, vagy csak a nem `NULL` értékeket *(Binding Type: Column Binding)*.
 * A `MIN` és a `MAX` egyszerűen a legkisebb és a legnagyobb értéket adja vissza.

###Distinct Count
A `DistinctCount` egy mező egyedi elemeit számolja össze. Ugyanúgy működik, mint a `Count(Distinct <mezőnév> )` az `SQL`-ben. Ez egy nagyon költséges művelet. Az aggregációt elő lehet állítani `MDX`-ben, és many-to-many kapcsolat használtával is, de általában nem javít a kocka teljesítményén. 
A `DistinctCount` kalkulációknak a BIDS mindig új *MeasureGroup*ot hoz létre, ami megkerülhető, de nem ajánlott. 

###None
Amennyiben ezt választjuk, úgy értékek csak a dimenziók legalacsonyabb granularitásában fognak megjelenni, a fact- és a dimenziótáblák kulcsainak metszetében, és magasabb szintre nem összegződnek fel. 

###Semi-additive típusok
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

###By Account
Ez az opció csak az Enterprise Editionben érhető el, és abban segít, hogy az eddigieknél bonyolultabb üzleti logikát és modellezni lehessen az OLAP kockában. Ez jellemzően egy speciális dimenziót jelent.