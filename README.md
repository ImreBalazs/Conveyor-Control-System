## 1. Projekt áttekintése
Ebben a projektben egy ipari szállítószalag- és szortírozórendszer teljes körű automatizálását valósítottam meg. A rendszer egy szállítószalagból és egy pneumatikus kitoló hengerből áll, ahol a dobozok jelenlét- és magasság-detektálását, valamint a számlálást optikai érzékelők végzik.

A fejlesztés során elkészült a villamos kapcsolási rajz (EPLAN Electric P8), a PLC vezérlőprogram (CODESYS V3.5), valamint a digitális iker szimuláció (Factory I/O), amelyeket OPC UA kommunikációs felület köt össze.

* **Vezérlési architektúra:** 
  * **HAND (Kézi):** Relés öntartó és villamosan reteszelt irányváltó logika.
  * **AUTO (Automata):** Feltételes POU-hívás (`PRG_Automata`) érzékelőalapualapú dobozszortírozáshoz.
  * **OFF / Safety:** Kényszerített kimenet-törlés és biztonsági reteszelések.

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
