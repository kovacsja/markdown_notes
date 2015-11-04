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
