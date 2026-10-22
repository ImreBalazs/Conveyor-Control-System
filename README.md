## 1. Projekt áttekintése

Ebben a projektben egy ipari szállítószalag- és szortírozórendszer teljes körű automatizálását valósítottam meg. A rendszer egy elektromos hajtású szállítószalagból és egy pneumatikus kitoló hengerből áll, a folyamatokat pedig optikai érzékelők vezérlik.
A berendezés célja a szalagon érkező dobozok méret szerinti automatikus válogatása és számlálása:
* **Méret szerinti szortírozás:** A rendszer különbséget tesz az alacsony és a magas dobozok között. 
  * Ha **magas doboz** érkezik, a magasságérzékelő szenzor jelt ad, és a pneumatikus kitoló kar lelöki azt a főpályáról egy gyűjtőhelyre.
  * Az **alacsony dobozokat** a rendszer továbbengedi a szalagon.
* **Darabszámlálás és biztonsági leállítás:** Egy optikai számlálószenzor regisztrálja a folyamaton áthaladó dobozokat. Amint a számláló eléri a **10 darabot**, a rendszer automatikusan leállítja a ciklust.
* **Irányváltás:** A szállítószalag motorja rugalmasan vezérelhető – mind **előre**, mind **hátra** irányba indítható.

A fejlesztés során elkészült a villamos kapcsolási rajz (EPLAN Electric P8), a PLC vezérlőprogram (CODESYS V3.5), valamint a digitális iker szimuláció (Factory I/O), amelyeket OPC UA kommunikációs felület köt össze.

### Vezérlési architektúra (Háromüzemmódú koncepció)

A rendszer működése három jól elkülöníthető üzemmódra épül, amelyeket az üzemmód-választó kapcsoló állása és a PLC-programban definiált logikai feltételek vezérelnek:

#### 1. HAND (Kézi üzemmód)
* **Célja:** Egyedi beavatkozók manuális tesztelése, beállítása és karbantartása.
* **Irányváltás és reteszelés:** A szállítószalag előre/hátra mozgatása relés öntartó körökkel történik. A hardveres és szoftveres villamos reteszelés (`-K1` és `-K2` kontaktorok bontóérintkezői, valamint CODESYS `RS` tárolók) garantálja, hogy a két forgásirány egyidejűleg soha ne kapcsolhasson be.

#### 2. AUTO (Automata üzemmód)
* **Célja:** Ciklikus, emberi beavatkozás nélküli folyamatirányítás.
* **Működése:** A PLC főprogramja feltételesen hívja meg a `PRG_Automata` POU blokkot. A folyamatvezérlés az optikai szenzorok jeleire támaszkodik:
  * **Jelenlét- és magasság-detektálás:** Meghatározza a szalagon érkező dobozok méretkategóriáját.
  * **Aktív szortírozás:** A kiválasztott dobozoknál időzítve működteti a pneumatikus kitoló hengert (`-Y1`).
  * **Számlálás:** Inkrementális logikával regisztrálja az áthaladó és kitolt darabszámot.

#### 3. OFF / Safety (Kikapcsolt és biztonsági állapot)
* **Kényszerített állapot-törlés (State Reset):** A választókapcsoló OFF állásában a PLC ciklusa kényszerítetten törli az összes aktív kimenetet és nullázza a belső tárolókat, megakadályozva a nem kívánt bennragadó állapotokat a következő indításkor.
* **Biztonsági reteszelés:** A Vészleállító (`E-Stop`) kör bontásakor a rendszer azonnal feszültségmentesíti a végrehajtó elemeket és leállítja a hajtásokat.

---

## 2. Villamos terv (EPLAN Electric P8)
A rendszer teljes villamos kapcsolási rajza az **EPLAN Electric P8** szoftverben készült el. Ennek kiemelt része a 4. oldal, amely a kézi üzemmód (HAND) hardveres vezérlési és védelmi logikáját, valamint az üzemmódváltást mutatja be:

* **Relés öntartó körök:** Az indítógombok megnyomásakor a megfelelő irányú kontaktor (`-K1` az előre, `-K2` a hátra irányért) behúz, és a saját segédérintkezőjén keresztül öntartásba kerül. Ez biztosítja, hogy a gomb elengedése után is fennmaradjon a motor tápellátása.
* **Villamos keresztreteszelés:** A fáziszárlat megakadályozása érdekében a két kontaktor vezérlőágába keresztbe vannak beépítve a másik kontaktor bontóérintkezői. Amíg a `-K1` behúzott állapotban van, a bontóérintkezője megszakítja a `-K2` tekercsének áramkörét (és fordítva), így a két forgásirány egyidejű bekapcsolása fizikailag lehetetlen.
* **Soros leállítás:** A vezérlőáramkör elejére sorosan bekötött `-S4` STOP gomb megnyomásakor a táplálás megszakad, az öntartás azonnal kiold, és a motor leáll.
* **Átkapcsolás Automata (AUTO) üzemmódba:** Az üzemmód-választó kapcsolóval a rendszer bármikor átváltható automata üzemmódba. Ebben az állapotban a direkt relés vezérlés helyett a folyamatirányítást a PLC veszi át, és a mozgásokat a CODESYS-ben megírt program szabályozza az optikai érzékelők jelei alapján.
<img width="1239" height="779" alt="{646A23E2-210F-464D-AC1D-F51A5C04ACA7}" src="https://github.com/user-attachments/assets/70e57e85-13bf-4055-b3ba-1fc0c533e337" />
📄 [**A teljes villamos kapcsolási rajz megtekintése (PDF)**](./conveyorControlSystemElectricalDiagram.pdf)

---
## 3. PLC Vezérlőprogram (CODESYS V3.5)
A szoftver az `S2LD` főprogramból (Létradiagram) és a `PRG_Automata` blokkból áll. A kézi üzemmódban az elinduló irányok `RS` tárolókba kerülnek, míg OFF állásban kényszerített törlés fut le a bennragadó állapotok elkerülésére.

*(Ide jön a CODESYS kép)*

---

## 4. 3D Szimuláció és Működés (Factory I/O)
A PLC és a szimuláció közötti adatcserét a CODESYS beépített OPC UA szervere és a Symbol Configuration-ben szűrt I/O jelek biztosítják.

*(Ide jön a szimulációs Videó)*
