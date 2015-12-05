#Bonyolultabb dimenziók
##Csoportosítás (Grouping)
A dimenziókon belül viszonylag ritka, hogy csoportosítást kell alkalmazni, mivel nem szokott olyan sok eleme lenni egy dimenziónak, hogy a böngészésnél gondot okozzon. Ha mégis szükség lenne rá, akkor vannak az dimenziószerkesztőben beépített algoritumosok, amik pl. megpróbálnak azonos elemszámú csoportokat képezni. Ez azonban ritkán vezet használható eredményre, ezért érdemesebb inkább az adatbázisban, egy view-ban új, dimenziómezőt létrehozni `CASE ... WHEN` paranccsal, és arra egy új dimenziót építeni.
##Cimkézés (Banding)
Amikor értékekeket akarunk halmazokba csorolni (alacsony-közepes-magas) akkor arra is új mezőt és dimenziót érdemes létrehozni, viszont előfordulhat, hogy az egyes cimkék kategorizálása változhat idővel. Ezért hogy ne kelljen folyamatosan újraszámoltatni az egész adatsort, érdemes a dimezniónak olyan kulcsot adni, ami a besorolás mögöttes értékéhez kapcsolódik. Így a dimenzióban megoldhatjuk a kategóriák megváltozását, és nem kell az egész ETL eljárást újrafuttatni.
##Lassan Váldozó Dimenziók (Slowly Changing Dimensions - SCD)
###I. Típus
Mivel ennél a típusnál a dimenziótáblában a régiértékek felülíródnak az újakkal, ezért ennek a típusnak a használata nem igényel külön odafigyelést a DB tervezése során.
Hibába akkor ütközhetünk az attribútum-hierarchiában egyes kapcsolatok `rigid`-re lettek állítva, és az attribútum helye megváltozott a többi attribútumhoz képest. Vagy ha az attribútum-hierarchiában definiált kapcsolatok megváltoztak.
Amire még oda kell figyelni, hogy azok a mértékek, amik dimenziók mentén épülnek fel, azok is invalidálódhatnak. A dimenziók tulajdonságai között be lehet állítani a `MDXMissingMemberMode`-ben, hogy erre hogyan reagáljon. Az alapértelmezett érték az `Ignore`, így figylemeztetés nélkül széteshetnek jelentéseink. Ha azonban `Error`-ra állítjuk be akkor az AS hibával fog leállni, és manuálisan kell kijavítani a hibát, vagy akár törölni a riportot, és elölről kezdeni a programozását.
###II. Típus
Mivel az Analysis Services OLAP szerver nem tárolja a dimenziók verzióit, ezért a TypeII változtatásoknál a dimenziótáblához új sor adódik hozzá. Abban az esetben, ha csak további adatokat adunk hozzá egy dimenzióhoz, és nem módosítunk meglévő adatokat, elegenedő `Process Add`-t futtatni `Process Update` helyett, ami sokkal gyorsabb feldolgozást jelent.
###III. Típus

##Junk dimensions

##Ragged hierarchies