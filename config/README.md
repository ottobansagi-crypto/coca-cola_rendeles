# config/

Ez a mappa tartalmazza a **közös, kódtól elkülönített alapadatokat**, amiket
a gyökérben lévő `index.html` mindkét fül-csoportja (① Rendelés-generálás,
valamint ②③④ Visszaigazolás/Számla/History) használ. Ld. a gyökér
`README.md` "2.5 Kulcskövetelmény" szakaszát.

## `parositas.json`

Ez a fájl írja le a Wildom ↔ Coca-Cola/PizzaMe párosítást. Szerkesztéséhez
nincs szükség programozói ismeretre, bármilyen szövegszerkesztővel vagy
JSON-t kezelő eszközzel módosítható.

Mezők:

- **`termek_parositas`** — lista, minden elem egy termék:
  - `wildom_nev` — a termék neve a Wildom exportban (ez alapján történik a párosítás a ① Rendelés-generálás fülön); **`null`, ha a termék nincs a Wildom rendelési listán** (csak a ②③④ Visszaigazolás/Számla fülek tartalmazzák — ld. lent)
  - `csomagolas` — dokumentációs infó (Wildom csomagolás-jelölés); `null` a Wildom-ban nem rendelt tételeknél
  - `cikkszam` — a Coca-Cola cikkszáma; **ha ez változik, itt kell frissíteni**
  - `cola_nev` — a hivatalos Cola SKU-név (a sablon "Anyag megnevezése" oszlopában szerepel, dokumentációs célra)
  - `megjelenitett_nev` — rövid, felhasználóbarát terméknév; **ezt jeleníti meg a ② Visszaigazolás fül** a táblázatokban (más, mint a `cola_nev` hivatalos SKU-szöveg)
  - `ar_ft` — nettó egységár Ft/csomagolási egység (zsugor vagy karton); **ezt használja a ② Visszaigazolás fül** a várható végösszeg kiszámításához — ha a Coca-Cola árat változtat, itt kell frissíteni
  - `megjegyzes` — szabad szöveges megjegyzés

  5 tétel (`Fuzetea Eper 0,5l`, `Cappy Barack 0,33l`, `Sprite 0,5l`, `Fanta Narancs 0,5l`, valamint a `Logisztikai díj`) csak a Cola visszaigazoláson/számlán jelenik meg, a Wildom rendelési listában nincs benne — ezeknél `wildom_nev: null`. Ha egyszer megjelennek a Wildom exportban is, itt kell pótolni a `wildom_nev`-et és a `csomagolas`-t.

- **`uzlet_parositas`** — lista, minden elem egy már a sablonban létező telephely-oszlop:
  - `sablon_oszlop_index` — a PizzaMe sablon oszlopának 0-alapú indexe (a fejléc-sor sorrendje szerint); **ne módosítsd, ha nem tudod pontosan, melyik sablon-oszlopnak felel meg**
  - `wildom_nev` — a telephely neve a Wildom exportban
  - `pm_azonosito` — a Wildom "PM száma" (dokumentációs célra)
  - `sablon_felirat` — a sablon jelenlegi oszlopfejléce (dokumentációs célra)
  - `boltkod` — a Coca-Cola/SAP "Sold-to Party" azonosítója; **ezt használja a ② Visszaigazolás fül** az üzletek párosításához a PM megrendelő excel és a Cola visszaigazolás/számla között. `null`, ha egyelőre nincs hozzá boltkód (ld. lent).
  - `megjegyzes` — pl. korábbi átnevezés ténye

  3 üzlethez (`Pizza Me 2- Erzsébet krt. 51.`, `Pizza Me 21- Óbuda`, `Pizza Me 26 Eleven`) a 2026-07-27-én kapott boltkód-listában nem szerepelt boltkód — ezeknél `boltkod: null` maradt, tisztázni kell a felhasználóval.

- **`uj_oszlopok`** — olyan telephelyek, amiknek még nincs oszlopa a sablonban;
  a ① Rendelés-generálás fül automatikusan létrehozza az oszlopot, ha hiányzik. Ha egy ilyen
  telephely oszlopa már bekerült a sablonba, nyugodtan átmozgatható innen
  az `uzlet_parositas` listába (a megfelelő `sablon_oszlop_index` megadásával).

- **`kihagyott_wildom_oszlopok`** — telephelyek, amiket üzleti döntés alapján
  szándékosan nem viszünk át a Cola-fájlba (pl. bezárt telephely, kamion,
  strand).

## Új telephely / termék felvétele, vagy meglévő módosítása

- **Cikkszám vagy ár változik**: keresd meg a terméket `termek_parositas`-ban, írd át a `cikkszam` mezőt.
- **Telephely nyílik**: vedd fel `uj_oszlopok`-ba (`sablon_felirat` + `wildom_nev`).
- **Telephely zár**: mozgasd át `kihagyott_wildom_oszlopok`-ba.
- **Telephely átnevezve a Wildomban**: írd át a megfelelő `wildom_nev` mezőt `uzlet_parositas`-ban (a `sablon_oszlop_index` marad).

Ha a ① Rendelés-generálás fül egy Wildom-oszlopot egyik listában sem talál,
figyelmeztetést jelenít meg futáskor — ez jelzi, ha egy új/átnevezett
telephely lemaradt a konfigurációból.

## Üzletazonosítás a ② Visszaigazolás fülön — `boltkod` mező

A ① Rendelés-generálás fül a Wildom telephelynevet és a "PM-számot"
(pl. `PM4`) használja azonosítóként. A ② Visszaigazolás fül viszont egy
harmadik, SAP-eredetű **"boltkód"** alapján párosít üzleteket (10+ jegyű
szám a PM megrendelő excel egy rejtett sorából és a Cola visszaigazolás
"Sold-to Party" oszlopából). Ez most már be van kötve: minden
`uzlet_parositas` bejegyzésnek van `boltkod` mezője.

**`uzlet_parositas_nem_cola_oszlopok`** — a boltkód-listával együtt kapott,
de más céghez/márkához tartozó oszlopok ("Lángosom", "HTM"), amik nem
PizzaMe/Cola-üzletek. Ezeket **nem** vettük fel semmilyen Cola-listába,
csak dokumentáljuk itt.

**`uzlet_egyeb_cola_helyek`** — két valódi, Cola-rendeléshez kötődő hely,
ami viszont **nem** része a Wildom "Coca Cola" profil 39 rögzített
telephely-csoportjának, ezért nincs `wildom_nev`/`sablon_oszlop_index`-ük
(a ① Rendelés-generálás fül automatikus rendelés-generálása nem kezeli őket):
- `Pizza Me Váci út 31. - Házhozszállítás` — kis házhozszállítás-üzlet, amihez a felhasználó szerint néha külön rendelnek Cola-terméket
- `Pizza Me Váci út 31.` — a központi raktár

Mindkettőnek van `boltkod`-ja, így a ② Visszaigazolás fül tudja párosítani
a PM megrendelő excel és a Cola visszaigazolás/számla között. Ha ezek
rendszeresen a heti Wildom-exporton keresztül is rendelnének, fel kell
venni őket a rendes `uzlet_parositas`/`uj_oszlopok` listába is.

## `Wildom_Cola_parositas.xlsx`

Emberi olvasásra szánt, dokumentációs párosító tábla (nem az `index.html`
olvassa be) — áttekinthető formában mutatja ugyanazt a termék- és
üzletlistát, a Wildom admin felület PM-azonosítóival és a korábbi
átnevezésekkel együtt.
