#Bonyolultabb dimenziók
##Csoportosítás (Grouping)
A dimenziókon belül viszonylag ritka, hogy csoportosítást kell alkalmazni, mivel nem szokott olyan sok eleme lenni egy dimenziónak, hogy a böngészésnél gondot okozzon. Ha mégis szükség lenne rá, akkor vannak az dimenziószerkesztőben beépített algoritumosok, amik pl. megpróbálnak azonos elemszámú csoportokat képezni. Ez azonban ritkán vezet használható eredményre, ezért érdemesebb inkább az adatbázisban, egy view-ban új, dimenziómezőt létrehozni `CASE ... WHEN` paranccsal, és arra egy új dimenziót építeni.
##Cimkézés (Banding)
Amikor értékekeket akarunk halmazokba csorolni (alacsony-közepes-magas) akkor arra is új mezőt és dimenziót érdemes létrehozni, viszont előfordulhat, hogy az egyes cimkék kategorizálása változhat idővel. Ezért hogy ne kelljen folyamatosan újraszámoltatni az egész adatsort, érdemes a dimezniónak olyan kulcsot adni, ami a besorolás mögöttes értékéhez kapcsolódik. Így a dimenzióban megoldhatjuk a kategóriák megváltozását, és nem kell az egész ETL eljárást újrafuttatni.
##Lassan Váldozó Dimenziók (Slowly Changing Dimensions - SCD)
###I. Típus
###II. Típus
###III. Típus

##Junk dimensions

##Ragged hierarchies