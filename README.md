## 1. Projekt áttekintése

A projektben egy ipari szállítószalag- és szortírozórendszer teljes körű automatizálását valósítottam meg. A rendszer egy elektromos hajtású szállítószalagból és egy pneumatikus kitoló hengerből áll, a folyamatokat pedig optikai érzékelők vezérlik.
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
<p> <a href="./conveyorControlSystemElectricalDiagram.pdf"><b>A teljes villamos kapcsolási rajz megtekintése (PDF)</b></a></p>

---
### 3. PLC Vezérlőprogram (CODESYS V3.5)

A teljes vezérlőszoftvert **CODESYS V3.5** környezetben, Létradiagram (**Ladder Diagram / LD**) programnyelven valósítottam meg. A program struktúrája **két külön POU-ra (Program Organization Unit)** épül fel:

* **`S2LD` (Főprogram):** A fő ciklus, amely az üzemmód-választásért (HAND / AUTO / OFF), a szoftveres reteszelésekért, a kézi mozgatásért és az automata alprogram feltételes meghívásáért felel.
* **`PRG_Automata` (Automata alprogram):** Az automata folyamat logikáját tartalmazza, beleértve az optikai érzékelők jelkezelését, a pneumatikus kitoló henger időzítését és a darabszámlálást.

Az alábbiakban a programkódot hálózatonként (Network) haladva részletezzük.

#### 3.1. `S2LD` Főprogram (Üzemmód-választás és biztonsági logika)

A főprogram létradiagramja (1–5. hálózat) a rendszer globális állapotaikért, a kézi mozgatásért és az üzemmódok közötti váltásért felel.

<img width="1564" height="640" alt="{7DCDBEDC-3AAE-4514-8973-1807DF6A95EB}" src="https://github.com/user-attachments/assets/d36fd252-ef03-4974-8409-d704580a8e9c" />
<img width="1565" height="334" alt="{B010B7BD-DE19-4976-A164-4478670E05CF}" src="https://github.com/user-attachments/assets/6644dbfa-78e1-41fa-872e-1406ac24838a" />

*`S2LD` létradiagram hálózatok (1–5. Network)*

* **Network 1 – Automata hívás:** Az `autoMode` jel jelenlétében engedélyezi és meghívja a `PRG_Automata` alprogramot.
* **Network 2–3 – Kézi vezérlés és keresztreteszelés:** A kézi előre/hátra mozgatást `RS` tárolók végzik. A fáziszárlat elleni szoftveres védelmet a szemközti irányok bontóérintkezői (NC) biztosítják. A STOP (`stopPB`) és Vészleállító (`emergencyStop`) gombok azonnal törlik (`RESET1`) a tárolókat.
* **Network 4 – Állapotjelzés:** Amíg a szalag sem előre, sem hátra nem mozog, a `stopLight` jelzőlámpa aktív.
* **Network 5 – OFF / Biztonsági törlés:** OFF állásban (amikor sem `autoMode`, sem `handMode` nem él) a PLC kényszerítetten törli (`Reset coils`) az összes hajtást és beavatkozót (`forwardDirection`, `reverseDirection`, `pusher`), megelőzve a bennragadó állapotokat.

#### 3.2. `PRG_Automata` (Automata és Szortírozási Logika)

A `PRG_Automata` alprogram felelős az automatikus üzemmód felügyeletéért, a visszajelző lámpák kezeléséért, a magas dobozok pneumatikus szortírozásáért, valamint az alacsony dobozok darabszámlálásáért.

**Változó-deklaráció (`PRG_Automata`):**
```pascal
PROGRAM PRG_Automata
VAR
    // Kezelőelemek és nyomógombok
    forwardPB               : BOOL;             // Előremenetel indítógomb
    stopPB                  : BOOL;             // Stop gomb
    reversePB               : BOOL;             // Hátramenetel indítógomb
    emergencyStop           : BOOL;             // Vészleállító (E-Stop)
    counterReset            : BOOL;             // Számláló nullázó gomb

    // Érzékelők és végálláskapcsolók
    presenceSensor          : BOOL;             // Belépő jelenlét-érzékelő
    heightSensor            : BOOL;             // Magasságérzékelő (magas doboz)
    counterSensor           : BOOL;             // Kilépő darabszámláló érzékelő
    pusherFrontLimit        : BOOL;             // Kitoló henger kiadott végállása

    // Belső tárolók, időzítők és számlálók
    forwardBlock            : RS;               // Előre irány RS tároló
    reverseBlock            : RS;               // Hátra irány RS tároló
    rsTimer                 : RS;               // Kitolási időzítés indító tároló
    rsPusher                : RS;               // Kitoló henger RS tároló
    pusherTimer             : TON;              // Kitolási késleltetés időzítő (400 ms)
    smallBoxCounter         : Standard.CTU;     // Alacsony dobozok számlálója
    counterCurrentValue     : WORD;             // Számláló aktuális értéke
    counterOut              : BOOL;             // Számláló határérték elérés állapota
    timerSetter             : RS;               // Időzítő segédtároló
    stopSetter              : RS;               // Stop segédtároló

    // Visszajelzők és állapotlámpák
    counterOutLamp          : BOOL;             // Betelt állapot jelzőlámpa
    isPowerOKLamp           : BOOL;             // Tápellátás rendben lámpa
    safetyCircuitFailedLamp : BOOL;             // Biztonsági kör hiba lámpa
END_VAR

VAR_INPUT
    // Bemeneti paraméterek / Vezérelt kimeneti állapotok
    forwardDirection        : BOOL;             // Előre hajtás állapota
    forwardLight            : BOOL;             // Előre jelzőlámpa állapota
    reverseDirection        : BOOL;             // Hátra hajtás állapota
    reverseLight            : BOOL;             // Hátra jelzőlámpa állapota
    pusher                  : BOOL;             // Kitoló henger kimenete
    stopLight               : BOOL;             // Álló helyzet jelzőlámpa
END_VAR
```

<img width="1356" height="607" alt="{13A9DADC-BD7A-4D6C-947B-14AEB7D7572E}" src="https://github.com/user-attachments/assets/9a1e6a5f-52fc-4adf-bf65-a27366d5da54" />
<img width="1345" height="743" alt="{458321D3-1F0B-4443-96A6-392F97A3B0C1}" src="https://github.com/user-attachments/assets/6571fbd5-442f-4f76-9743-2f1dea18d587" />
<img width="1342" height="218" alt="{BB383D67-F807-4B72-9AD6-8515D6A87BB8}" src="https://github.com/user-attachments/assets/89372c71-ab19-4d08-abc1-2cd1d25da61e" />

---

##### Network 1: Tápellátás visszajelzés (`isPowerOKLamp`)
* **Működés:** Feszültség jelenlétében a PLC közvetlenül aktiválja az `isPowerOKLamp` kimenetet, jelezve a vezérlőfeszültség meglétét a műszerfalon.

---

##### Network 2: Előremenetel és automata leállítás (`forwardBlock`)
* **SET (Indítás):** Az előre irányú `RS` tároló elindul, ha megnyomják a `forwardPB` gombot és a hátramenetel inaktív (`NOT reverseDirection`).
* **RESET1 (Törlés):** A szalag előremenetele azonnal leáll, ha:
  * Megnyomják a STOP gombot (`stopPB`), **VAGY**
  * Kiold a Vészleállító (`emergencyStop`), **VAGY**
  * A darabszámláló eléri a beállított limitet (`counterOut`).
* **Kimenet:** Aktiválja a hajtást (`forwardDirection`) és a műszerfali lámpát (`forwardLight`).

---

##### Network 3: Álló helyzet visszajelzése (`stopLight`)
* **Működés:** Amíg sem az előre (`NOT forwardLight`), sem a hátra irány (`NOT reverseLight`) nem aktív, a PLC felkapcsolja a `stopLight` jelzőlámpát.

---

##### Network 4: Biztonsági kör hiba jelzése (`safetyCircuitFailedLamp`)
* **Működés:** A vészleállító gomb megnyomásakor vagy a biztonsági kör bontásakor (`NOT emergencyStop`) aktiválódik a `safetyCircuitFailedLamp` hibaüzenet-lámpa.

---

##### Network 5: Hátramenetel és automata leállítás (`reverseBlock`)
* **SET (Indítás):** A hátra irányú `RS` tároló elindul a `reversePB` gombbal, ha az előremenetel inaktív (`NOT forwardDirection`).
* **RESET1 (Törlés):** Megegyezik az 2. hálózattal – a STOP gomb (`stopPB`), Vészleállító (`emergencyStop`), vagy a betelt darabszámláló (`counterOut`) azonnal leállítja a hajtást.
* **Kimenet:** Aktiválja a hátra hajtást (`reverseDirection`) és a visszajelzőt (`reverseLight`).

---

##### Network 6: Pneumatikus kitoló henger vezérlése és időzítése (`pusher`)
* **Érzékelés:** Ha a szalag előre mozog (`forwardDirection`), és a belépő optikai szenzor (`presenceSensor`) a magasságérzékelővel (`heightSensor`) együtt **magas dobozt** detektál, az `rsTimer` tároló bekapcsol.
* **Késleltetés (TON):** A `pusherTimer` időzítő **400 ms** (`T#400MS`) késleltetést indít, megvárva, amíg a doboz pontosan a kitolókar elé ér.
* **Kitolás:** A 400 ms letelte után az `rsPusher` tároló aktiválja a pneumatikus hengert (`pusher`), amely lelöki a dobozt a főpályáról.
* **Alaphelyzet:** Amint a henger eléri a kiadott végállást (`pusherFrontLimit`), vagy kiold az `emergencyStop`, mindkét tároló törlődik (`RESET1`), és a henger visszahúzódik.

---

##### Network 7: Alacsony dobozok számlálása (`smallBoxCounter`)
* **Számlálás (CU):** A `CTU` számláló csak azokat a dobozokat regisztrálja, amelyeket a kilépő szenzor érzékel (`counterSensor`), de a magasságérzékelő inaktív maradt (`NOT heightSensor`) előremenet közben (`NOT reverseDirection`).
* **Határérték (PV):** A beállított érték **10 darab** (`PV := 10`).
* **Ciklusvége (Q):** A 10. alacsony doboz áthaladásakor a `counterOut` kimenet high szintre vált:
  * Felkapcsolja a kész állapotot jelző lámpát (`counterOutLamp`).
  * Automatikusan leállítja a szalaghajtást a 2. és 5. hálózatban.
* **Nullázás (RESET):** A `counterReset` gombbal a számláló aktuális értéke (`counterCurrentValue`) tetszőlegesen nullázható.

---
## 4. 3D Szimuláció és Működés (Factory I/O)

A szoftveres vezérlés és a fizikai folyamatok valós idejű validációjához a **Factory I/O** 3D szimulációs környezetet alkalmaztam mint digitális ikret (*Digital Twin*). A PLC és a szimuláció közötti alacsony késleltetésű adatcserét a CODESYS beépített **OPC UA szervere** és a *Symbol Configuration*-ben publikált I/O jelek biztosítják.

### A megvalósított 3D folyamat főbb elemei:

* **Mérnöki szimulációs pálya:** Egy kétirányú hajtott szállítószalagból, optikai érzékelőkből (jelenlét-, magasság- és darabszámláló szenzorok) és egy pneumatikus kitoló hengerből álló szortírozó állomás.
* **Automatizált szortírozási logika:**
  * **Magas dobozok:** A belépő optikai érzékelő és a magasságérzékelő egyidejű jelére a PLC azonosítja a magas dobozt, majd egy 400 ms-os pozicionálási késleltetést követően a pneumatikus hengerrel lelöki azt a gyűjtőpályára.
  * **Alacsony dobozok:** Akadálytalanul továbbhaladnak a főpályán. A kilépő oldali optikai szenzor regisztrálja a darabszámot, és a 10. doboz áthaladásakor a PLC automatikusan leállítja a szalaghajtást.
* **Interaktív kezelőpanel (HMI / Control Panel):** A 3D térben elhelyezett fizikai műszerfal tartalmazza az üzemmód-választó kapcsolót (HAND / AUTO / OFF), a különálló előre/hátra indítógombokat, a STOP és E-STOP gombokat, a számlálónullázót, valamint a teljes állapotjelző láppaparkot (Tápellátás OK, Irányjelzők, Álló helyzet, Számláló betelt, Biztonsági hiba).

### 4.1. Műszerfal és Kezelőfelület (Control Panel)

A Factory I/O-ban megtervezett fizikai műszerfal áttekinthető, az ipari szabványoknak megfelelő elrendezést követ, biztosítva a teljes körű folyamatfelügyeletet és a manuális beavatkozási lehetőséget.

<img width="676" height="785" alt="{C8B05B6A-0BDA-4ADA-B324-08DE2C8A0013}" src="https://github.com/user-attachments/assets/15c83b2d-5dca-4880-a62b-0ce28de4797b" />

*A rendszer fizikai kezelőpanelje (Factory I/O)*

* **Üzemmód-választó kapcsoló (3 állású):**
  * **`I` / HAND:** Kézi üzemmód – az indítógombokkal a szalag manuálisan mozgatható előre és hátra.
  * **`0` / OFF:** Kikapcsolt állapot – a PLC kényszerítetten törli az összes hajtási kimenetet és belső tárolót.
  * **`II` / AUTO:** Automata üzemmód – a `PRG_Automata` POU átveszi a szortírozás, az időzített kitolás és a darabszámlálás vezérlését.
* **Állapot- és hibajelző lámpák:**
  * **`POWER OK` (Kék):** A vezérlőfeszültség meglétét és az aktív üzemi állapotot jelzi (`isPowerOKLamp`).
  * **`MOTOR TRIPPED` (Piros):** Motorvédelem kioldását vagy hajtáshibát visszajelző lámpa.
  * **`SAFETY CIRCUIT FAILED` (Piros):** A biztonsági kör bontását, illetve a vészleállító kioldott állapotát jelzi (`safetyCircuitFailedLamp`).
* **Kezelőgombok:**
  * **`Forward` (Zöld):** Előremenetel indítása kézi üzemmódban (`forwardPB`).
  * **`StopPB` (Piros):** A hajtások azonnali leállítása (`stopPB`).
  * **`Reverse` (Kék):** Hátramenetel indítása kézi üzemmódban (`reversePB`).
* **Vészleállító egység:**
  * **`Emergency Stop`:** Reteszelődő gombafejű vészleállító gomb a veszélyes állapotok azonnali megszüntetésére (`emergencyStop`).
* **Darabszámláló és kijelző:**
  * **`Counter Display`:** Digitális hétszegmenses kijelző, amely az áthaladt alacsony dobozok aktuális számát mutatja (`counterCurrentValue`).
  * **`Reset Counter` (Sárga):** Nyomógomb a darabszámláló alaphelyzetbe állításához (`counterReset`).

 ### 4.2. Működési Demonstráció

<img width="1911" height="1040" alt="{DE243260-CA78-4CCF-8229-4533B5EA822A}" src="https://github.com/user-attachments/assets/5f1a4e83-978f-440b-b825-efb272704439" />


Az alábbi videó bemutatja a rendszer működését az AUTO és HAND üzemi állapotokban:
https://youtu.be/oRk2wWwoCf4

* **1. AUTO (Automata üzemmód):** 
  * A választókapcsoló `II` (AUTO) állásában elindul az automatikus szortírozás.
  * A belépő oldali optikai érzékelők azonosítják a magas dobozokat, amelyeket a pneumatikus kitoló henger 400 ms késleltetéssel letol a másodlagos pályára.
  * Az alacsony dobozok akadálytalanul haladnak tovább a főpályán a számláló szenzorig.

* **2. HAND (Kézi üzemmód):**
  * A választókapcsoló `I` (HAND) állásában az automata logika inaktív.
  * A kezelő a `Forward` (Zöld) és `Reverse` (Kék) nyomógombok segítségével manuálisan, szoftveresen reteszelt módon vezérelheti a szalag előre- és hátramenetelét.
