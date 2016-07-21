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

###None

###Semi-additive típusok

