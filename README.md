# Coca-Cola Rendelés-kezelő Program – Specifikáció

Státusz: **Egyesített, élő, böngészőben futó oldal (`index.html`) elkészült és GitHub Pages-en fut, PizzaMe brand-kinézettel. A boltkód ↔ PM-szám megfeleltetés 2 üzletnél még hiányos.**

## 1. Háttér és cél

A PizzaMe több telephelyes pizzéria-hálózat, ami a Coca-Cola termékeket (üdítők, vizek) a Wildom nevű belső rendelőrendszeren (admin.manage.pizzame.hu) keresztül rendeli meg telephelyenként. A Wildomból exportált Excel más szerkezetű, mint amit a Coca-Cola beszállítónak le kell adni (a "PizzaMe_megrendelő" nevű, évek óta használt sablon szerkezetében). Eddig ezt valaki kézzel másolta át hétről hétre. A cél egy program, ami ezt automatizálja, és emellett a rendelés utáni folyamatot (visszaigazolás, számla-egyeztetés, naplózás) is egy helyen kezeli.

A programnak két fő funkciója van, **egyetlen élő, böngészőben futó oldalon** (`index.html`), fülekre bontva:
1. **① Rendelés-generálás** — Wildom export → PizzaMe/Cola formátum
2. **② Visszaigazolás / ③ Számla-ellenőrzés / ④ History** — a Cola visszaigazolás és a beérkező számla ellenőrzése, korábbi egyeztetések naplója

Fontos, átfogó tervezési elv: **minden változtatható alapadat (termékek, cikkszámok, árak, üzletlista/párosítás) egy könnyen szerkeszthető, a kódtól elkülönített adatforrásban legyen** (`config/parositas.json`), nem a programkódba égetve. Ha a Coca-Cola módosít egy cikkszámot, árat, vagy megnyílik/bezár egy telephely, ezt a business-felhasználónak kódmódosítás nélkül kell tudnia frissíteni. Mindkét fül **ugyanabból** a config-fájlból olvas.

Az oldal teljesen kliensoldali (böngészőben futó JavaScript, `ExcelJS`/`xlsx.js`/`pdf.js` könyvtárakkal, CDN-ről betöltve) — nincs backend, nincs build lépés, a fájlfeltöltés/feldolgozás/letöltés mind a böngészőben történik.

---

## 2. Rendelés-generálás (Wildom → PizzaMe formátum)

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
Feltárt átnevezések eddig: PM4 (Sas utca → Bazilika), PM17 (Váci utca → Fővám tér), PM23 (Zugló → Bosnyák tér), PM27/PM28 (korábban üres/foglalt sablon-oszlopok → Törökvész / GoBuda), PM2 (Erzsébet krt. 51. → a Wildom exportban ma "PM13N" néven/azonosítóval fut, 2026-07-27-én felfedezve).
Feltárt, valóban új telephelyek (nincs rájuk sablon-oszlop, hozzá kellett adni): **Újpest, K1 Westend** — ezek szerepelnek a rendelésben, kell nekik oszlop.
Üzleti döntés alapján kihagyva (nem kell a Cola-fájlba): Truck 1 Kamion, Budapest Park, PM Plázs, valamint a "Bezárt Móricz"/"Closed Móricz" (zárva lévő telephely, mindig 0).

### 2.4 Működő implementáció

A ① Rendelés-generálás fül (`index.html`), böngészőben futó JavaScript (`ExcelJS` könyvtárral):
1. Beolvassa a feltöltött Wildom exportot és a PizzaMe sablon legfrissebb lapját
2. A `config/parositas.json`-ban tárolt termék- és üzlet-párosítás alapján lemásolja a sablon szerkezetét (stílusokkal, oszlopszélességgel együtt) egy új, dátummal jelölt lapként
3. Kitölti a mennyiségeket a helyes cellákba
4. Hozzáadja az új (korábban nem létező) telephely-oszlopokat, ha még nincsenek meg
5. Kihagyja a kihagyandó listában szereplő telephelyeket
6. Ha ismeretlen/új Wildom-oszlopot talál, amit a config egyik listája sem ismer, ezt figyelmeztetésként jelzi (nem hagyja csendben veszni az adatot)
7. A kész fájlt letölthető linkként ajánlja fel (nincs szerver, nincs mentés máshova)

Ez a logika korábban egy külön Python script (`wildom_to_cola.py`, `openpyxl` könyvtárral) volt — **JavaScriptre lett portolva**, hogy ugyanazon az élő oldalon fusson, mint a Modul 2. A port pontosságát szintetikus teszt-adatpárral ellenőriztük: a JS-verzió **cellaértékre pontosan (0 eltérés)** ugyanazt adja, mint a Python-verzió (19 termék, 343 kitöltött cella a teszt-adaton), és a stílusmásolás (font, kitöltés, keret, oszlopszélesség, új oszlop létrehozása) is helyesen működik valódi formázott mintafájlon tesztelve.

**Ismert korlát:** a stílusmásoláshoz használt `ExcelJS` könyvtár nem feltétlenül reprodukál 100%-ban minden Excel-formázási finomságot (pl. feltételes formázás, egyes ritka szám-formátumok) úgy, mint az `openpyxl` — a font/kitöltés/keret/oszlopszélesség viszont tesztelten helyesen öröklődik át.

### 2.5 Kulcskövetelmény: karbantartható alapadatok ✅ (megvalósítva)

A termék- és üzletpárosítás **nem** a kódba van égetve, hanem a `config/parositas.json` fájlban van, hogy:
- ha a Coca-Cola megváltoztat egy cikkszámot vagy árat, ne kelljen a programkódhoz nyúlni — elég a JSON-t szerkeszteni
- ha nyit/zár egy telephely, vagy megváltozik a neve, ugyanez legyen igaz
- **mindkét fül-csoport** (Rendelés-generálás és Visszaigazolás/Számla) **közös** adatforrásból dolgozzon — ne legyen két külön helyen karbantartott termék/ár/üzlet lista

A `config/parositas.json` szerkezete: `termek_parositas` (Wildom termék → cikkszám + Cola SKU-név + megjelenített név + egységár), `uzlet_parositas` (Wildom telephelynév → sablon oszlopindex + PM-azonosító + boltkód + jelenlegi felirat), `uj_oszlopok` (még hozzáadandó telephelyek), `kihagyott_wildom_oszlopok` (üzleti döntéssel kizárt telephelyek), `uzlet_egyeb_cola_helyek` (Cola-rendeléshez kötődő, de a Wildom 39-es listáján kívüli helyek). Részletek: `config/README.md`.

---

## 3. Visszaigazolás / Számla-ellenőrzés / History

A ②③④ fülek (`index.html`), ugyanazon az oldalon, mint a Rendelés-generálás:
- **② Visszaigazolás** — a PM megrendelő excel (üzlet × cikkszám mátrix) és a Cola visszaigazolás excel (Sold-to Party / Material / Order Size / Confirmed Qty) összevetése, eltérések jelzése boltonként, várható nettó összeg számítása
- **③ Számla Ellenőrzés** — a beérkező gyűjtőszámla PDF szöveg-alapú feldolgozása (`pdf.js`), összevetés a ② fülön kiszámolt várható nettóval
- **④ History** — a korábbi egyeztetések naplója (böngésző `localStorage`)

A korábban tervezett önálló "OCR Ellenőrzés" fül **szándékosan kimaradt** — a ③ fül szöveg-alapú PDF-feldolgozása kiváltja.

Ez a funkció eredetileg egy önálló HTML eszköz volt (`COLA_MASTER_v2.html`, amit a felhasználó korábban külön fájlként használt), amit összefésültünk a Rendelés-generálással egyetlen oldalba.

**Integráció a közös configgal — elkészült:** a termékárak/nevek (korábban a fájlba égetett `TERMEK_MAP`) most a `config/parositas.json`-ból töltődnek be (`fetch('config/parositas.json')`), nem hardkódoltak.

**Üzletazonosítás — részben nyitott:** a ② fül SAP "boltkód" alapon párosít üzleteket (a PM megrendelő excel és a Cola-visszaigazolás közös oszlopa), ami más azonosító-tér, mint a ① fül Wildom név/"PM-szám" rendszere. A felhasználó átadott egy boltkód-listát, ami alapján 31/33 meglévő Cola-üzlethez sikerült rögzíteni a `boltkod` mezőt a configban. **2 üzlethez még hiányzik** (Óbuda, Eleven Center) — ld. 6. szakasz.

---

## 4. Architektúra és hosztolás

- **Egyetlen fájl, `index.html`**, a repó gyökerében — ez GitHub Pages-kompatibilis (statikus, nincs build lépés).
- **`config/parositas.json`** — közös alapadat, ugyanonnan (`fetch('config/parositas.json')`) tölti be mindkét fül-csoport.
- Külső könyvtárak CDN-ről: `xlsx.js` (SheetJS, a ②③ fülek excel/pdf-beolvasásához), `pdf.js` (③ fül PDF-szövegkinyeréshez), `ExcelJS` (① fül stílus-hű Excel-íráshoz).
- Nincs szerver, nincs adatbázis — minden feldolgozás a böngészőben történik, a History a böngésző `localStorage`-ában él (eszközönként/böngészőnként külön).

---

## 5. GitHub Pages beüzemelése — 2 manuális lépés a felhasználótól

Ezeket a Claude Code jelenlegi jogosultságai (GitHub MCP eszközök) **nem tudják elvégezni** — a repó tulajdonosának kell megtennie a GitHub felületén:

1. **A repó publikussá tétele** — *Settings → General → Danger Zone → Change repository visibility → Public.* (A felhasználó ezt jóváhagyta: az ingyenes GitHub Pages csak publikus repóhoz jár; ez azt jelenti, hogy a `config/parositas.json`-ban lévő cikkszámok, árak és üzletlista bárki számára láthatóvá válik, aki ismeri a repó/oldal URL-jét.)
2. **GitHub Pages bekapcsolása** — *Settings → Pages → Source: "Deploy from a branch" → válaszd ki azt a branch-et, amiről publikálni szeretnéd (pl. `main`, a `/ (root)` mappával).* Az `index.html` és a `config/` mappa a repó gyökerében van, tehát nincs szükség extra build-lépésre vagy Actions workflow-ra.

Ezután az oldal élesben elérhető lesz a `https://<felhasználónév>.github.io/coca-cola_rendeles/` URL-en (pár perc alatt frissül, ha változás kerül a publikált branch-re).

---

## 6. Nyitott kérdések

- **Boltkód ↔ PM-szám megfeleltetés 2 üzlethez hiányzik**: Óbuda, Eleven Center — tisztázandó, hogy aktívak-e még, és ha igen, mi a boltkódjuk.
- **GitHub Pages beüzemelése** — a felhasználónak el kell végeznie a 5. szakaszban leírt 2 manuális lépést (repó publikussá tétele + Pages bekapcsolása).
- **Stílus-hűség valós sablonon** — a JS-portot szintetikus és kézzel formázott mintafájlon teszteltük; érdemes az első éles futásnál ellenőrizni, hogy a generált fájl a valódi PizzaMe_megrendelő sablonon is jól néz ki (font/szín/keret), mert az `ExcelJS` könyvtár nem garantáltan 100%-ban azonos az `openpyxl`-lel minden formázási esetben.
