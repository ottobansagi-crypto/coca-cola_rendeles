# Coca-Cola Rendelés-kezelő Program – Specifikáció

Státusz: **1. rész (Rendelés-generálás) kész és tesztelve. 2. rész (Visszaigazolás/Számla/History/OCR) vázlat, pontosítás holnap egy frissebb HTML verzió alapján.**

## 1. Háttér és cél

A PizzaMe több telephelyes pizzéria-hálózat, ami a Coca-Cola termékeket (üdítők, vizek) a Wildom nevű belső rendelőrendszeren (admin.manage.pizzame.hu) keresztül rendeli meg telephelyenként. A Wildomból exportált Excel más szerkezetű, mint amit a Coca-Cola beszállítónak le kell adni (a "PizzaMe_megrendelő" nevű, évek óta használt sablon szerkezetében). Eddig ezt valaki kézzel másolta át hétről hétre. A cél egy program, ami ezt automatizálja, és emellett a rendelés utáni folyamatot (visszaigazolás, számla-egyeztetés, naplózás) is egy helyen kezeli.

A programnak két fő modulja van:
1. **Rendelés-generálás** — Wildom export → PizzaMe/Cola formátum (kész, ezt írja le részletesen ez a dokumentum)
2. **Visszaigazolás / Számla-ellenőrzés / History / OCR** — már létezik egy működő HTML eszköz (`COLA_MASTER_v2.html`), amit a felhasználó jelenleg is használ; egy újabb verziót holnap ad át, azzal pontosítjuk ezt a részt.

Fontos, átfogó tervezési elv mindkét modulra: **minden változtatható alapadat (termékek, cikkszámok, árak, üzletlista/párosítás) egy könnyen szerkeszthető, a kódtól elkülönített adatforrásban legyen**, nem a programkódba égetve. Ha a Coca-Cola módosít egy cikkszámot, árat, vagy megnyílik/bezár egy telephely, ezt a business-felhasználónak kódmódosítás nélkül kell tudnia frissíteni.

---

## 2. Modul 1: Rendelés-generálás (Wildom → PizzaMe formátum)

### 2.1 Bemenet: Wildom export

A Wildom admin felület "Beszállítói megrendelések" (Stock Order) menüjéből, a "Coca Cola" profil kiválasztásával (mindig ugyanaz a 39 rögzített telephely-csoport), az "Excel 2007+" export gombbal letölthető egy `.xlsx` fájl (pl. `megrendelés-Coca Cola-20260710-2007.xlsx`).

Szerkezete (egyetlen munkalap):
- 1. sor: fejléc — `Termék`, `Csomagolás`, majd 39 telephely-oszlop (Wildom-beli elnevezéssel, ld. 2.3), végül `Összesen`
- 2–20. sor: termékenként egy sor, cellákban a rendelt mennyiség telephelyenként
- Utolsó sor (`Stand ár`): irreleváns a Cola-rendeléshez, figyelmen kívül hagyandó

### 2.2 Kimenet: PizzaMe_megrendelő sablon formátum

A cél szerkezet egy régóta létező, élő Google Sheet ("PizzaMe_megrendelő"), aminek a felépítése **nem változhat**:
- A–C oszlop: `Kiszerelés`, `Cikkszám`, `Anyag megnevezése` (ez utóbbi kettő a Coca-Cola hivatalos SKU-azonosítója és terméknév-leírása — ez az, amit a beszállítónak ténylegesen le kell adni)
- D-től kezdve: telephelyenként egy oszlop, a "Pizza Me N- Utcanév" vagy hasonló elnevezéssel
- Soronként egy termék, a cellákban a mennyiség
- A munkalapokat eddig dátum szerint nevezték el (pl. "20250414 aktuális"); az elv az, hogy minden rendeléshez egy új, dátummal jelölt lap/fájl készül a sablon másolataként, amibe csak a mennyiségek kerülnek bele — a szerkezet, a termék- és üzletsorok/oszlopok maguk nem változnak.

### 2.3 A párosítás — ez a modul lényegi nehézsége

A Wildom és a PizzaMe sablon **más néven** hivatkozik ugyanarra a termékre és ugyanarra a telephelyre. Két párosító tábla szükséges:

**a) Termék párosítás** (Wildom terméknév + csomagolás → Cola cikkszám + hivatalos SKU-név)
Példa: Wildom "Coca-Cola" (24db/Karton) → cikkszám `1188084`, "330 CAN X24 COCA COLA SLEEK HU".
19 ilyen terméksor van jelenleg, mind egyértelműen párosítható a csomagolás (db/Karton vs db/Zsugor, darabszám) és a névben szereplő méret/típus (Savas=szénsavas/CARB, Mentes=szénsavmentes/STILL stb.) alapján.
Van egy már létező, hivatalos referencia-tábla is a felhasználó Drive-ján ("coca_cola_betétdíjas_termékek_alap"), amit érdemes forrásként kezelni/szinkronban tartani.
**Fontos megfigyelés:** a cikkszámok időről időre változnak Cola-oldalon (pl. egy termék cikkszáma 279045-ről 279044-re változott két különböző dátumú sablon-mentés között) — a párosítás nem lehet égetett/statikus, hanem karbantartható adatforrásból kell jönnie.

**b) Üzlet/telephely párosítás** (Wildom telephelynév/PM-szám → PizzaMe sablon oszlop)
A Wildom rendszerben minden telephelynek van egy "PM száma" (pl. Bazilika = PM4, Törökvész = PM27), ami a Wildom admin "Beszállítói megrendelések" oldalának üzletválasztójában látszik zárójelben a név mellett. A PizzaMe sablon oszlopfejlécei részben ezt a PM-számot és egy (néha elavult) utcanevet tartalmazzák — pl. a sablonban "Pizza Me 4- Sas utca" ma ténylegesen a "Bazilika" nevű telephelyet jelenti (a telephely átnevezve, a PM-szám ugyanaz maradt).
Feltárt átnevezések eddig: PM4 (Sas utca → Bazilika), PM17 (Váci utca → Fővám tér), PM23 (Zugló → Bosnyák tér), PM27/PM28 (korábban üres/foglalt sablon-oszlopok → Törökvész / GoBuda).
Feltárt, valóban új telephelyek (nincs rájuk sablon-oszlop, hozzá kellett adni): **Újpest, K1 Westend** — ezek szerepelnek a rendelésben, kell nekik oszlop.
Üzleti döntés alapján kihagyva (nem kell a Cola-fájlba): Truck 1 Kamion, Budapest Park, PM Plázs, valamint a "Bezárt Móricz"/"Closed Móricz" (zárva lévő telephely, mindig 0).

### 2.4 Működő implementáció

Elkészült és éles adaton tesztelve egy Python script (`rendeles-generalas/wildom_to_cola.py`), ami:
1. Beolvassa a Wildom exportot és a PizzaMe sablon legfrissebb lapját
2. A `config/parositas.json`-ban tárolt termék- és üzlet-párosítás alapján lemásolja a sablon szerkezetét egy új, dátummal jelölt lapként
3. Kitölti a mennyiségeket a helyes cellákba
4. Hozzáadja az új (korábban nem létező) telephely-oszlopokat, ha még nincsenek meg
5. Kihagyja a kihagyandó listában szereplő telephelyeket
6. Ha ismeretlen/új Wildom-oszlopot talál, amit a config egyik listája sem ismer, ezt figyelmeztetésként jelzi (nem hagyja csendben veszni az adatot)

Teszt eredmény: 19 termék, 238 kitöltött mennyiség-cella, minden termék összesített mennyisége pontosan egyezik a Wildom "Összesen" oszlopával (levonva a szándékosan kihagyott telephelyeket). Ez az eredmény a `config/parositas.json`-ra való átállás után szintetikus teszt-adatpárral újra ellenőrizve is stabil (ld. `rendeles-generalas/README.md`).

### 2.5 Kulcskövetelmény a továbbfejlesztéshez: karbantartható alapadatok ✅ (megvalósítva)

A korábbi implementációban a három szótár (`PRODUCT_MAP`, `COL_MAP_BY_NAME`, `NEW_COLUMNS`/`EXCLUDED_WILDOM_COLS`) a Python kódban volt hardkódolva. Ez ki lett emelve a `config/parositas.json` fájlba, hogy:
- ha a Coca-Cola megváltoztat egy cikkszámot vagy árat, ne kelljen a programkódhoz nyúlni — elég a JSON-t szerkeszteni
- ha nyit/zár egy telephely, vagy megváltozik a neve, ugyanez legyen igaz
- a két modul (rendelés-generálás és a holnap részletezendő visszaigazolás/számla modul) **közös** adatforrásból dolgozzon — ne legyen két külön helyen karbantartott termék/ár/üzlet lista.

A `config/parositas.json` szerkezete: `termek_parositas` (Wildom termék → cikkszám + Cola SKU-név), `uzlet_parositas` (Wildom telephelynév → sablon oszlopindex + PM-azonosító + jelenlegi felirat), `uj_oszlopok` (még hozzáadandó telephelyek), `kihagyott_wildom_oszlopok` (üzleti döntéssel kizárt telephelyek). Részletek: `config/README.md`.

---

## 3. Modul 2: Visszaigazolás / Számla-ellenőrzés / History / OCR (vázlat — pontosítás holnap)

A felhasználó már használ egy működő, böngészőben futó HTML eszközt (`COLA_MASTER_v2.html`), aminek felépítése (fülek szerint):
- **① Visszaigazolás** — a leadott rendelés Excel és a Cola által visszaigazolt Excel összevetése, eltérések jelzése
- **② Számla Ellenőrzés** — a beérkező számla (PDF) összevetése a visszaigazolással
- **③ History** — a korábbi egyeztetések naplója
- **④ OCR Ellenőrzés** — feltehetően a PDF számlák OCR-alapú adatkinyerése/ellenőrzése

Ez a modul jelenleg önállóan, kliensoldali JavaScript-ként fut (böngészőben, `xlsx.js` könyvtárral), fájl drag-and-drop-pal. A pontos logika (hogyan párosít terméket/üzletet, hogyan tárolja a historyt, hogyan működik az OCR) a holnap átadott, frissebb verzió alapján kerül ide részletezésre.

**Integrációs cél:** ez a modul ugyanazt a termék/üzlet/ár alapadat-forrást használja, mint a Modul 1 (ld. 2.5), így egyetlen helyen kell karbantartani a Cola-féle SKU-kat, árakat és a telephelylistát mindkét modul számára.

---

## 4. Nyitott kérdések (holnapra)

- A COLA_MASTER_v2.html új verziójának részletei (pontos mezők, jelenlegi adattárolási mód — böngésző localStorage? fájlba mentés?)
- Hol lakjon a közös alapadat-forrás (JSON a repóban? Google Sheet, amit mindkét modul beolvas? Excel fájl?)
- Végleges hosztolás/futtatási mód mindkét modulra (helyi gép, GitHub Actions, kis webes hosting stb.)
- ~~GitHub repó struktúra: egy repóban a két modul almappákban, közös `config/` mappával az alapadatoknak~~ — **megvalósítva**: `rendeles-generalas/` (Modul 1), `config/` (közös alapadatok), `visszaigazolas-szamla/` (Modul 2, előkészítve)
