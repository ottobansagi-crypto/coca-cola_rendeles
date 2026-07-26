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
  - `wildom_nev` — a termék neve a Wildom exportban (ez alapján történik a párosítás)
  - `csomagolas` — dokumentációs infó (Wildom csomagolás-jelölés)
  - `cikkszam` — a Coca-Cola cikkszáma; **ha ez változik, itt kell frissíteni**
  - `cola_nev` — a hivatalos Cola SKU-név (a sablon "Anyag megnevezése" oszlopában szerepel, dokumentációs célra)
  - `megjegyzes` — szabad szöveges megjegyzés

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

## `Wildom_Cola_parositas.xlsx`

Emberi olvasásra szánt, dokumentációs párosító tábla (nem a script olvassa
be) — áttekinthető formában mutatja ugyanazt a termék- és üzletlistát, a
Wildom admin felület PM-azonosítóival és a korábbi átnevezésekkel együtt.
