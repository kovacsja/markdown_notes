#Tranzakció adatok kezelése

>Molap / Rolap dimenziók
>Many-to-many dimenziók


##Tranzakció-részletek
Az eddigi módszerrel létre kellene hozni egy olyan dimenziót, aminek a granularitása megegyezik a Fact tábláéval, és az egyes Tranzakcióból létrehozott ID-kat dimenzióba rendezve adnánk listát a felhasználónak, hogy szűrhessen az adatokban. 
Ez azonban több problémát is okozhat. A legfőbb a klienseszközön jelentkező teljesítményprobléma. Ilyen esetben jelentős lassulást okozhat egy nagy dimenziótáblából szűrögetni az adatokat. 
Amikor ezzel a problémával szembesülünk, akkor el kell döntni, hogy egy adott ponton csak a *részletek érdekelnek* bennünket, vagy *navigálni is akarunk* az adott ID dimenzió alapján. Amennyiben csak az adatokra akarjuk megjeleníteni, akkor a `AttributeHierarchyEnabled` tulajdonságot `False`-ra állítva jeleníthejük meg egyszerűbben az adatokat. A tételes lekérdezésre szolágl a `Drillthrough`.

##Drillthrough - lefúrás
A hagyományos Pivot táblás lefúrástól abban különbözik, hogy a pivot táblánál a lefúrás az előre definiált pivot szerkezetben történik, itt viszont automatikusan a kocka legkisebb granularitásán adaja vissza az adatokat. 

###Actions
Alapvetően arra szolgál, hogy a felhasználó műveleteket hatjtson végre a klienseszközön kontextushoz igazodóan. 
Háromfajta `Action` létezik: 
 * Generic Action: URL, Rowset, Dataset, Proprietary, Statement
 * Reporting Action: A Reporting Servicenek ad át egy linket attól függően, hogy hová kattintottunk, amiből újabb riportot generál
 * Drillthrough Action: A kattintott cella alapján végrehajt egy `DRILLTHROUGH` lekérdezést.

###Drillthrough Actions
Alapvetően metaadatokat kell megadni, ami alapján a klienseszköz összeállítja a lekrédezést a kocka felé. 
Az `Action Target` mezőben definiálhatjuk, hogy melyik `Measure Gruop`hoz akarjuk társítani az eseményt. Van lehetőség az `All` opciót választani, amikor is mindegyik csoportban elérhető lesz ez a lehetőség. 
A `Condition` részben azt lehet beállítani, hogy milyen esetekben ne hajtódjon végre az esemény. Nem egy gyakran használt opció. Akkor lehet hasznos, ha például nem akrjuk, hogy üres cellákra futtassák a drillthrough-t. 
A `Drilltrhough Columns` érsz a leglényegesebb itt lehet beállítani, hogy a visszatérő adathalmazba milyen mezők szerepeljenek. Minden dimenziót használhatunk, ami az adott kockához társítva van, de csak azokat attribútumkoat, amelyeknél az `AttributeHierarhyEnabled` tulajdonság `True`ra van állítva. 
Az `Additional Properties` alatt a `Default` kapcsolót `Measure Gruop`onként csak egy Actionnál lehet `True` értékre állítani! Excel kliensben a jelentéscellára duplán kattintva aktiválhatjuk a `Default`ra állított actiont. Ha ez nincs beállítva, akkor az alapértelmezett `Drillthrough` fog végrehajtódni, ami a legkisebb granularitáson kilistáz minden dimenzió id-t. 
>Biztonsági okokból érdemes a kockában a Drillthrough lehetőséget letiltani. Ez viszont nem akadályozza, hogy a klienseszköz DRILLTHROUGH lekérdezéseket küldjün, így a magunk által beállított Action-ök működőképesek maradnak így is. 

Amennyiben csak `Action`öket adtunk hozzá a kockához, akkor ezt Deploy-olhatjuk anélkül, hogy újra kellene processzálni az egész kockát. 

####Drillthrough Columns order
A lefúrás során az oszlopsorrendet sajnos nem lehet a fellületen megváltoztatni. Sem a használt dimenziók sorrendjét, sem azon belül az attribútumokat. Ez a korlátozás azonban csak a felhasználói felületen létezik. A kocka definíciójában, a `.cube` fájl valójában egy `xml`, amiben a kézzel át lehet variálni az oszlopok sorrendjét az `<Actions>` tag alatt. Az egyed dimenziókhoz tartozó mezők nincsenek csoportba foglalva, így ez sem korlátozza a rendezést. 
Ezt követően a változtatások meg fognak jelenni a felhasználói felületen is, de a dimenziónkénti csoportosítás megmarad a felületen, bár működni nem így fog. Ezután ügyelni kell rá, hogy a felület bármilyen módosítás felülírja a kézzel beállított sorrendet. 

####Drillthrough and calculated members

###Drillthrough modeling

