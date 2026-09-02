# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Mi ez a projekt

A PizzaMe pizzéria-hálózat Coca-Cola rendelés-kezelő eszköze: egyetlen,
kliensoldali HTML-oldal, ami (1) a Wildom belső rendelőrendszer Excel-exportját
átalakítja a Coca-Cola beszállítónak leadható "PizzaMe_megrendelő" formátumba, és
(2) kezeli a rendelés utáni folyamatot (Cola visszaigazolás összevetése,
gyűjtőszámla PDF ellenőrzése, napló). A részletes üzleti specifikáció a
gyökér `README.md`-ben van — **azt olvasd el először**, ha a domain-logikához nyúlsz.

A projekt nyelve magyar: a UI, a kódkommentek, a config-mezőnevek és a
commit-üzenetek is magyarul vannak. Ezt tartsd meg.

## Nincs build, nincs teszt, nincs backend

Nincs `package.json`, nincs build lépés, nincs CI, nincs szerveroldali kód, és
nincs automatizált teszt-suite. A külső könyvtárak CDN-ről jönnek
(`xlsx.js`/SheetJS, `pdf.js`, `ExcelJS`), a repó GitHub Pages-ről szolgálja ki
magát statikusan.

Helyi futtatás — **`file://`-ként megnyitva nem működik**, mert az oldal
`fetch('config/parositas.json')`-nal tölti be az alapadatot (CORS), ezért mindig
HTTP-n keresztül nyisd meg:

```bash
python3 -m http.server 8000   # majd http://localhost:8000/
```

Az oldal maga jelzi ezt a hibát: ha a config nem töltődik be, a
`lib-warning-config` sáv magyarázó szöveggel megjelenik, és az ① fül letiltva marad.

Tesztelés = manuális, a böngészőben, valódi vagy szintetikus Excel/PDF
mintafájlokkal. Ha a generálási logikához nyúlsz, egy Wildom export + egy
PizzaMe sablon párral futtasd végig, és cellaértékre hasonlítsd össze a
kimenetet a változtatás előtti verzióval.

## Architektúra

- **`index.html`** (~1000 sor) — a teljes alkalmazás: stílus, markup és az összes
  JavaScript egy fájlban. Négy fül (`switchTab`), fülenként külön panel és külön
  state: `wildomFile`/`sablonFile` az ①-hez, `pmWb`/`colaWb`/`lastResult` a ②-höz.
- **`config/parositas.json`** — az **egyetlen** alapadat-forrás, mindkét
  fül-csoport innen olvas (`loadConfig()` → globális `CONFIG` +
  `TERMEK_MAP`). Termék, cikkszám, ár, üzlet, boltkód **soha nem kerül a kódba**;
  ez a projekt legfontosabb tervezési elve (README 2.5), mert a business-felhasználó
  kódmódosítás nélkül karbantartja. A mezők leírása: `config/README.md`.
- **`assets/pizzame-logo.png`**, **`.nojekyll`** (hogy a GitHub Pages ne
  Jekyll-ként dolgozza fel a repót).
- **`config/Wildom_Cola_parositas.xlsx`** — a párosítás eredeti, kézi
  munkatáblája; nem a program olvassa, csak referencia.

### Két, egymástól független azonosító-tér

Ezt fontos fejben tartani, mert ez a leggyakoribb hibaforrás:

- Az **① Rendelés-generálás** a Wildom **telephelynevet** és a "PM-számot"
  (pl. `PM4`) használja, és a sablon **oszlopindexét** (`sablon_oszlop_index`).
- A **② Visszaigazolás / ③ Számla** a Coca-Cola/SAP **boltkódot** ("Sold-to
  Party", 10 jegyű szám) használja — ez a `boltkod` mező a configban, és 3
  üzletnél jelenleg `null` (ld. README 6. szakasz).

Termékoldalon a közös kulcs mindenhol a **`cikkszam`**.

### ① Rendelés-generálás (`generateColaOrder`)

`ExcelJS`-sel dolgozik (nem SheetJS), mert stílus-hű Excel-írás kell. A logika
egy korábbi Python script (`wildom_to_cola.py`, `openpyxl`) portja — a Python
verzió már nincs a repóban, a git history-ban keresd, ha referencia-viselkedést
kell tisztázni.

A sablon szerkezetéről **égetett feltevései** vannak, amiket a config nem ír le:
fejléc a **5. sorban**, adatsorok a **6. sortól**, cikkszám a **B (2.) oszlopban**,
az első telephely-oszlop a **D (4.)**, és a `getCell(5,4)`/`getCell(6,4)` cella a
stílus-minta az új oszlopokhoz. Ha a sablon szerkezete változik, ezeket kell
átírni. A legfrissebb sablon-lapot a `findLatestSheet()` a `\d{6,8}` nevű
(dátumos) lapok közül a legnagyobbként választja ki.

Ismeretlen Wildom-oszlop (ami a config egyik listájában sem szerepel) `unmapped`
figyelmeztetésként jelenik meg — **ezt sose nyeld el csendben**, ez a védőhálója
annak, hogy egy új telephely rendelése ne veszhessen el.

### ②③ Beolvasás (`parsePM`, `parseCola`, `parseInvoicePdfText`)

Mindhárom parser **heurisztikusan keresi meg a fejlécet**, nem fix pozícióról
olvas — a bemeneti fájlok formája ingadozik:

- `parsePM`: az első 15 sorban keresi a "Cikkszám" szót a B oszlopban; a boltkód-sort
  az alapján találja meg, hogy egy fejléc fölötti sorban 3-nál több > 1e9 szám van.
- `parseCola`: az első 10 sorban keresi azt a sort, amiben egyszerre van "material"
  és "sold"; utána nevek szerinti (substring) oszlopkeresés.
- `parseInvoicePdfText`: `pdf.js`-szel kinyert **szövegből** vág blokkokat
  `Boltkód: <szám>` találatok szerint, és blokkonként regexeli az összegeket
  ("Nettó összesen", "ÁFA", "Bruttó összesen", …). Egy bolt több egymást követő
  oldalon átfolyó blokkja egybe kerül.

Az azonosítókat mindig a `normId()`-n keresztül hasonlítsd (stringesít, levágja a
`.0` végződést, amit az Excel-beolvasás okoz); a magyar formátumú számokat a
`parseHuNum()` értelmezi (`1.234,56`).

### ④ History

A böngésző `localStorage`-ában, `cola-history` kulcs alatt, legutóbbi 20 bejegyzés
(`saveHistory`). Eszközönként/böngészőnként külön, nincs szinkronizálás.

## Amire figyelj módosításkor

- Új termék, ár, cikkszám, telephely vagy boltkód **csak** a
  `config/parositas.json`-ba kerül, és ha a szerkezete változik, a
  `config/README.md`-t is frissítsd.
- A "PizzaMe_megrendelő" sablon szerkezete (A–C oszlop, soronként egy termék,
  telephelyenként egy oszlop) **nem változtathatható** — élő Google Sheetről van szó.
- A generált fájl a böngészőben letöltésként keletkezik; nincs szerver, nincs
  máshova mentés.
- A CDN-ekről érkező könyvtárak verzióra pinneltek az `index.html` fejében; ha
  frissítesz, a `pdf.js` `workerSrc` URL-jét is együtt kell léptetni.
