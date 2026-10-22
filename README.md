## 1. Projekt áttekintése
Ebben a projektben egy ipari szállítószalag- és szortírozórendszer teljes körű automatizálását valósítottam meg. A rendszer egy szállítószalagból és egy pneumatikus kitoló hengerből áll, ahol a dobozok jelenlét- és magasság-detektálását, valamint a számlálást optikai érzékelők végzik.

A fejlesztés során elkészült a villamos kapcsolási rajz (EPLAN Electric P8), a PLC vezérlőprogram (CODESYS V3.5), valamint a digitális iker szimuláció (Factory I/O), amelyeket OPC UA kommunikációs felület köt össze.

Vezérlési architektúra (Háromüzemmódú koncepció):
A rendszer működése három jól elkülöníthető üzemmódra épül, amelyeket üzemmód-választó kapcsolóval és a PLC-programban definiált logikai feltételekkel vezérlünk:

1. HAND (Kézi üzemmód):

Célja: Egyedi beavatkozók manuális tesztelése, beállítása és karbantartása.

Irányváltás és reteszelés: A szállítószalag előre/hátra mozgatása relés öntartó körökkel történik. A hardveres és szoftveres villamos reteszelés (-K1 és -K2 kontaktorok bontóérintkezői, valamint CODESYS RS tárolók) garantálja, hogy a két forgásirány egyidejűleg soha ne kapcsolhasson be.

2. AUTO (Automata üzemmód):

Célja: Ciklikus, emberi beavatkozás nélküli folyamatirányítás.

Működése: A PLC főprogramja feltételesen hívja meg a PRG_Automata blokkot. A folyamat az optikai szenzorok jeleire támaszkodik:

Jelenlét- és magasság-detektálás: Meghatározza a szalagon érkező dobozok méretkategóriáját.

Aktív szortírozás: A kiválasztott dobozoknál időzítve működteti a pneumatikus kitoló hengert (-Y1).

Számlálás: Inkrementális logikával regisztrálja az áthaladó és kitolt darabszámot.

3. OFF / Safety (Kikapcsolt és biztonsági állapot):

Kényszerített állapot-törlés (State Reset): A választókapcsoló OFF állásában a PLC ciklusa kényszerítetten törli az összes aktív kimenetet és nullázza a belső tárolókat, megakadályozva a nem kívánt bennragadó állapotokat a következő indításkor.

Biztonsági reteszelés: A Vészleállító (E-Stop) kör bontásakor a rendszer azonnal feszültségmentesíti a végrehajtó elemeket és leállítja a hajtásokat.

---

## 2. Villamos terv (EPLAN Electric P8)
A villamos séma az **IEC 81346** szabványnak megfelelő tervjelezést használ (`-K` kontaktorok, `-Y` szelepek, `-B` érzékelők, `-S` nyomógombok). A 4. oldal tartalmazza a kézi üzemmód hardveres logikáját (öntartás, villamos reteszelés a `-K1` és `-K2` bontó érintkezőin keresztül, leállítás a sorba kötött `-S4` STOP gombbal).

*(Ide jön az EPLAN kép)*

---

## 3. PLC Vezérlőprogram (CODESYS V3.5)
A szoftver az `S2LD` főprogramból (Létradiagram) és a `PRG_Automata` blokkból áll. A kézi üzemmódban az elinduló irányok `RS` tárolókba kerülnek, míg OFF állásban kényszerített törlés fut le a bennragadó állapotok elkerülésére.

*(Ide jön a CODESYS kép)*

---

## 4. 3D Szimuláció és Működés (Factory I/O)
A PLC és a szimuláció közötti adatcserét a CODESYS beépített OPC UA szervere és a Symbol Configuration-ben szűrt I/O jelek biztosítják.

*(Ide jön a szimulációs Videó)*
