# A türelmetlen gyógyszerkereső – hibariport

Repó: https://github.com/95HNR/umfst-bh08-gyogyszerkereso
Készítette: Sandor Hunor, INFO 3.EV

## 1. hiba

* **Hiba leírása és reprodukálása (pontos gépelési sorrend, időzítés):** Gyorsan begépelve az „Algo” szót, a találati listában végül az „A” betűre illeszkedő találatok (pl. Aspirin, Amoxicillin stb.) maradnak meg, nem az „Algo”-ra vonatkozók (Algopyrin, Algoflex).
* **Gyökérok (eseményhurok, Promise-ok időbeli sorrendje):** Versenyhelyzet (race condition) alakult ki. A  fakeFetch  függvény úgy van megírva, hogy a rövidebb keresőszavakra hosszabb késleltetéssel válaszol (az „A” esetén kb. 950 ms, míg az „Algo” esetén kb. 200 ms). Mire a lassan lefutó „A” kérés Promise-a feloldódik (resolve), a gyorsabb „Algo” már megjelent a DOM-ban. Az „A” eredménye ezután felülírja a helyes, újabb állapotot.
* **Javítás (commit üzenet):**  fix: race condition javítása kérésazonosítóval (utolsoKeresId) 
* **Hogyan tesztelted (konzolnapló a kérésekről, fake timer-es egységteszt, Performance panel):** A  fakeFetch.then  ágában elhelyezett  console.log(q, jelenlegiId, utolsoKeresId)  naplóval bizonyítottam, hogy az elavult kérések azonosítója kisebb, mint az aktuális globális  utolsoKeresId , így a  return  miatt a DOM-frissítés már nem fut le rájuk.
* **Tanulság / érintett alapelv (A legutolsó kérés nyer; Debounce; Mindig kezeld a Promise elutasítását):** A legutolsó kérés nyer. Aszinkron hálózati kéréseknél sosincs garancia arra, hogy a kérések a kiküldés sorrendjében érkeznek vissza, ezért az azonosítós (vagy  AbortController -es) érvénytelenítés kötelező.

## 2. hiba

* **Hiba leírása és reprodukálása (pontos gépelési sorrend, időzítés):** Az „Algo” szó begépelése során minden egyes billentyűleütés (A, l, g, o) azonnal egy új  fakeFetch  kérést indít.
* **Gyökérok (eseményhurok, Promise-ok időbeli sorrendje):** Az  input  eseménykezelőben nincs kéréskésleltetés (debounce). A JavaScript Engine szinkron módon, a leütések pillanatában azonnal beleteszi a call stackbe a  fakeFetch  hívást, feleslegesen terhelve a (szimulált) szervert.
* **Javítás (commit üzenet):**  fix: debounce hozzáadása a keresési inputhoz 
* **Hogyan tesztelted (konzolnapló a kérésekről, fake timer-es egységteszt, Performance panel):** A Performance panel hálózat/konzol fülén figyeltem meg, hogy az „Algo” folyamatos gépelése után már csak egyetlen  fakeFetch('Algo')  kérés indult, pontosan 300 ms-mal az utolsó leütés után.
* **Tanulság / érintett alapelv (A legutolsó kérés nyer; Debounce; Mindig kezeld a Promise elutasítását):** Debounce. A felhasználói inputokat (gépelés, ablakméretezés) mindig késleltetve (összevonva) kell szerver felé továbbítani.

## 3. hiba

* **Hiba leírása és reprodukálása (pontos gépelési sorrend, időzítés):** Az „x” karakter beírásakor a felület örökre a „Keresés…” állapotban ragad. A konzolban „Uncaught (in promise) Error: Szerverhiba (503)” hibaüzenet jelenik meg.
* **Gyökérok (eseményhurok, Promise-ok időbeli sorrendje):** A szinkron  try/catch  blokk lefutása már régen befejeződött, mire a  fakeFetch  által visszaadott Promise (a beállított  setTimeout  után) aszinkron módon elutasításra (reject) kerül. Mivel a Promise láncban nem volt  .catch() , illetve az aszinkron hívás nem volt  await -elve, a hiba kezeletlen maradt, és a DOM állapot nem frissült.
* **Javítás (commit üzenet):**  fix: aszinkron szerverhibák helyes lekezelése try/catch-csel és async/await-tel 
* **Hogyan tesztelted (konzolnapló a kérésekről, fake timer-es egységteszt, Performance panel):** Az „x” beírása után a DOM vizuálisan visszaigazolta a hibát: az  #allapot  p tag tartalma azonnal „Hiba: Szerverhiba (503)”-ra változott, a konzolból pedig eltűnt az Uncaught Error.
* **Tanulság / érintett alapelv (A legutolsó kérés nyer; Debounce; Mindig kezeld a Promise elutasítását):** Mindig kezeld a Promise elutasítását! A szinkron hibakezelő nem fogja meg az aszinkron hibákat; ehhez  async/await  kontextus vagy  .catch()  metódus szükséges.

## 4. hiba

* **Hiba leírása és reprodukálása (pontos gépelési sorrend, időzítés):** Bármilyen érvényes keresés (pl. „Aspirin”) esetén a „Találatok listaáron összesen” eredménye mindig 0 lej marad, holott a találatok megjelennek.
* **Gyökérok (eseményhurok, Promise-ok időbeli sorrendje):** Az  osszesit  függvényben a  nevek.forEach(async ...)  szerkezete hibás. A  forEach  nem várja meg ( await ) az aszinkron callbackek lefutását a Microtask Queue-ból, így a JavaScript szinkron folytatja a futást, és azonnal kiírja a DOM-ba a  0  kezdőértékű  osszeg  változót. Az árak csak ezután érkeznek meg a háttérben.
* **Javítás (commit üzenet):**  fix: árak összesítése Promise.all használatával forEach helyett 
* **Hogyan tesztelted (konzolnapló a kérésekről, fake timer-es egységteszt, Performance panel):**  console.log -gal ellenőriztem az idősíkokat: a DOM-ba írás pontosan akkor történik meg, miután a  Promise.all  az összes ár lekérésével befejeződött. Az UI végre helyes számot mutat.
* **Tanulság / érintett alapelv (A legutolsó kérés nyer; Debounce; Mindig kezeld a Promise elutasítását):** Ne használj aszinkron kódot hagyományos  forEach  ciklusban. Ha több független aszinkron műveletet kell bevárni, a  Promise.all()  (vagy iterációkhoz a  for...of ) a megfelelő minta.

## 5. hiba

* **Hiba leírása és reprodukálása (pontos gépelési sorrend, időzítés):** Az „Automatikus frissítés” gombot ötször egymás után megnyomva a találati lista egyre gyorsabban villog, kikapcsolni nem lehet, a böngésző CPU terhelése megnő.
* **Gyökérok (eseményhurok, Promise-ok időbeli sorrendje):** Minden gombnyomás egy új  setInterval  hívást regisztrál az eseményhurokba. Mivel a visszaadott azonosítókat a kód nem tárolja el (memóriaszivárgás), a korábban indított időzítőket sosem lehet  clearInterval -lal törölni. Ráadásul nem veszi figyelembe, ha a böngészőfül nincs fókuszban.
* **Javítás (commit üzenet):**  fix: automatikus frissítés gomb időzítőinek kezelése és document.hidden szűrés 
* **Hogyan tesztelted (konzolnapló a kérésekről, fake timer-es egységteszt, Performance panel):** A Performance panel segítségével megvizsgáltam a CPU-használatot. Gombnyomásra ki/be kapcsolódik az esemény (toggle logika), és a konzol is igazolja, hogy több kattintás ellenére mindig maximum csak egy aktív  setInterval  fut. A  document.hidden  használatával háttérbe tett lapon leáll a hálózati forgalom.
* **Tanulság / érintett alapelv (A legutolsó kérés nyer; Debounce; Mindig kezeld a Promise elutasítását):** Erőforrás-takarítás és állapotkezelés. Az eseménykezelők, intervallumok és feliratkozások referenciáját mindig meg kell őrizni, hogy később megszakíthatók és takaríthatók legyenek (Cleanup).

## Összegzés

* **Mit tanultam:** A modern JavaScript alkalmazásoknál a vizuális hibák mögött szinte mindig az aszinkronitás rossz menedzselése (időzítési problémák, versenyhelyzetek, elveszett scope-ok) áll. Nem elég, har egy kód „szinkronban” működik, tudni kell, hogy az adott feladat az eseményhurok melyik fázisában (Call Stack, Task Queue, Microtask Queue) és mikor hajtódik végre.
* **Melyik alapelv sérült a leggyakrabban:** Az aszinkron műveletek életciklus-kezelésének elve sérült a legtöbbször. Legyen szó a Promise-ok be nem várásáról ( forEach ), hálózati elutasítások kezeletlenül hagyásáról ( try/catch  hiba), felgyülemlő folyamatok megszakításának hiányáról (Timer és Race Condition), a probléma mindig az volt, hogy a program nem uralta a jövőbeli időállapotokat.

---
