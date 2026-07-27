# Coca-Cola Rendelés-kezelő Program – Specifikáció

Státusz: **1. rész (Rendelés-generálás) kész és tesztelve, karbantartható configgal. 2. rész (Visszaigazolás/Számla/History) átadva és a repóba mentve, a közös config-gal való tényleges összekötés (termékárak beolvasása, boltkód ↔ PM-szám megfeleltetés) még hátravan.**

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

## 3. Modul 2: Visszaigazolás / Számla-ellenőrzés / History

A felhasználó már használ egy működő, böngészőben futó HTML eszközt
(`visszaigazolas-szamla/Cola_ellenorzes.html`, korábban `COLA_MASTER_v2.html`
néven), aminek felépítése (fülek szerint):
- **① Visszaigazolás** — a PM megrendelő excel (üzlet × cikkszám mátrix) és a Cola visszaigazolás excel (Sold-to Party / Material / Order Size / Confirmed Qty) összevetése, eltérések jelzése boltonként, várható nettó összeg számítása
- **② Számla Ellenőrzés** — a beérkező gyűjtőszámla PDF szöveg-alapú feldolgozása (`pdf.js`), összevetés az ① fülön kiszámolt várható nettóval
- **③ History** — a korábbi egyeztetések naplója (böngésző `localStorage`)

A korábban tervezett önálló "④ OCR Ellenőrzés" fül **szándékosan kimaradt** — a ② fül szöveg-alapú PDF-feldolgozása kiváltja.

Ez a modul önállóan, kliensoldali JavaScript-ként fut (böngészőben, `xlsx.js` + `pdf.js`, CDN-ről betöltve), fájl drag-and-drop-pal, backend nélkül.

**Integrációs cél és jelenlegi állapot:** ez a modul ugyanazt a termék/üzlet/ár alapadat-forrást kellene használja, mint a Modul 1 (ld. 2.5). A termékadatok (cikkszám, név, egységár) már átkerültek a közös `config/parositas.json`-ba, de a HTML **még nem olvassa be onnan** — ez még hardkódolva van a fájlban (`TERMEK_MAP`), külön munkaként hátravan a bekötés. Az üzletazonosítás is más elven működik itt (SAP "boltkód" a PM-excel és a Cola-visszaigazolás közös oszlopaiból), mint a Modul 1-ben (Wildom név / "PM-szám") — ehhez egy boltkód ↔ PM-szám megfeleltető tábla szükséges, amit a felhasználó ad majd át. Részletek: `config/README.md` és `visszaigazolas-szamla/README.md`.

---

## 4. Nyitott kérdések

- ~~A COLA_MASTER_v2.html új verziójának részletei~~ — **megvalósítva**: átadva, elmentve `visszaigazolas-szamla/Cola_ellenorzes.html` néven, adattárolás böngésző `localStorage`-ban (3 fül, OCR fül szándékosan kimaradt)
- ~~Hol lakjon a közös alapadat-forrás~~ — **megvalósítva**: `config/parositas.json` a repóban
- A Modul 2 tényleges bekötése a közös `config/parositas.json`-ba (jelenleg még nem olvassa be — ld. 3. szakasz)
- **Boltkód ↔ PM-szám megfeleltető tábla** — a Modul 2 SAP "boltkód" alapon azonosítja az üzleteket, a Modul 1 Wildom név/"PM-szám" alapon; ehhez a felhasználó át fogja adni a PM megrendelő excelben szereplő boltkód-listát
- A 4 extra termék (Fuzetea Eper, Cappy Barack, Sprite, Fanta Narancs) és a Logisztikai díj — jóváhagyva, felvéve a közös configba (`wildom_nev: null`, mivel nincsenek a Wildom rendelési listán)
- Végleges hosztolás/futtatási mód mindkét modulra (helyi gép, GitHub Actions, kis webes hosting stb.)
- ~~GitHub repó struktúra: egy repóban a két modul almappákban, közös `config/` mappával az alapadatoknak~~ — **megvalósítva**: `rendeles-generalas/` (Modul 1), `config/` (közös alapadatok), `visszaigazolas-szamla/` (Modul 2)
