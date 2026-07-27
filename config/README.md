# config/

Ez a mappa tartalmazza a **közös, kódtól elkülönített alapadatokat**, amiket
mindkét modul (`rendeles-generalas/` és a jövőbeli `visszaigazolas-szamla/`)
használ. Ld. a gyökér `README.md` "2.5 Kulcskövetelmény" szakaszát.

## `parositas.json`

Ez a fájl írja le a Wildom ↔ Coca-Cola/PizzaMe párosítást. Szerkesztéséhez
nincs szükség programozói ismeretre, bármilyen szövegszerkesztővel vagy
JSON-t kezelő eszközzel módosítható.

Mezők:

- **`termek_parositas`** — lista, minden elem egy termék:
  - `wildom_nev` — a termék neve a Wildom exportban (ez alapján történik a párosítás a Modul 1-ben); **`null`, ha a termék nincs a Wildom rendelési listán** (csak a Modul 2 visszaigazolás/számla tartalmazza — ld. lent)
  - `csomagolas` — dokumentációs infó (Wildom csomagolás-jelölés); `null` a Wildom-ban nem rendelt tételeknél
  - `cikkszam` — a Coca-Cola cikkszáma; **ha ez változik, itt kell frissíteni**
  - `cola_nev` — a hivatalos Cola SKU-név (a sablon "Anyag megnevezése" oszlopában szerepel, dokumentációs célra)
  - `ar_ft` — nettó egységár Ft/csomagolási egység (zsugor vagy karton); **ezt használja a Modul 2 (visszaigazolás/számla-ellenőrzés) a várható végösszeg kiszámításához** — ha a Coca-Cola árat változtat, itt kell frissíteni
  - `megjegyzes` — szabad szöveges megjegyzés

  5 tétel (`Fuzetea Eper 0,5l`, `Cappy Barack 0,33l`, `Sprite 0,5l`, `Fanta Narancs 0,5l`, valamint a `Logisztikai díj`) csak a Cola visszaigazoláson/számlán jelenik meg, a Wildom rendelési listában nincs benne — ezeknél `wildom_nev: null`. Ha egyszer megjelennek a Wildom exportban is, itt kell pótolni a `wildom_nev`-et és a `csomagolas`-t.

- **`uzlet_parositas`** — lista, minden elem egy már a sablonban létező telephely-oszlop:
  - `sablon_oszlop_index` — a PizzaMe sablon oszlopának 0-alapú indexe (a fejléc-sor sorrendje szerint); **ne módosítsd, ha nem tudod pontosan, melyik sablon-oszlopnak felel meg**
  - `wildom_nev` — a telephely neve a Wildom exportban
  - `pm_azonosito` — a Wildom "PM száma" (dokumentációs célra)
  - `sablon_felirat` — a sablon jelenlegi oszlopfejléce (dokumentációs célra)
  - `megjegyzes` — pl. korábbi átnevezés ténye

- **`uj_oszlopok`** — olyan telephelyek, amiknek még nincs oszlopa a sablonban;
  a script automatikusan létrehozza az oszlopot, ha hiányzik. Ha egy ilyen
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

Ha a script egy Wildom-oszlopot egyik listában sem talál, figyelmeztetést ír
ki futáskor ("FIGYELEM - ismeretlen uj Wildom oszlop(ok)...") — ez jelzi, ha
egy új/átnevezett telephely lemaradt a konfigurációból.

## Nyitott kérdés: üzletazonosítás a Modul 2-ben

A `uzlet_parositas` jelenleg a Wildom telephelynevet és a "PM-számot"
(pl. `PM4`) használja azonosítóként — ezt a Modul 1 (`rendeles-generalas/`)
használja. A Modul 2 (`visszaigazolas-szamla/Cola_ellenorzes.html`) viszont
egy harmadik, SAP-eredetű **"boltkód"** alapján párosít üzleteket (10+
jegyű szám a PM megrendelő excel egy rejtett sorából és a Cola
visszaigazolás "Sold-to Party" oszlopából). Ez a boltkód-lista még nincs
összekötve a `pm_azonosito`/`sablon_oszlop_index` mezőkkel — ehhez egy
boltkód ↔ PM-szám megfeleltető táblára van szükség, amit a felhasználó
küld át.

## `Wildom_Cola_parositas.xlsx`

Emberi olvasásra szánt, dokumentációs párosító tábla (nem a script olvassa
be) — áttekinthető formában mutatja ugyanazt a termék- és üzletlistát, a
Wildom admin felület PM-azonosítóival és a korábbi átnevezésekkel együtt.
