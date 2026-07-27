# visszaigazolas-szamla/ — Modul 2

Böngészőben, kliensoldali JavaScript-ként futó eszköz (`xlsx.js` + `pdf.js`,
CDN-ről betöltve, nincs backend). Fájl drag-and-drop-pal, minden feldolgozás
a böngészőben történik. Részletes specifikáció a gyökér `README.md` "3. Modul 2"
szakaszában.

## `Cola_ellenorzes.html`

Jelenlegi (2026-07 végén átadott) verzió, 3 füllel:

1. **① Visszaigazolás** — a leadott PM megrendelő excel (üzlet × cikkszám
   mátrix) és a Cola visszaigazolás excel (soronkénti Sold-to Party /
   Material / Order Size / Confirmed Qty lista) összevetése, eltérések
   kiemelése boltonként, várható nettó összeg számítása.
2. **② Számla Ellenőrzés** — a Cola gyűjtőszámla PDF-jének szöveg-alapú
   feldolgozása (`pdf.js` szövegréteg-kiolvasás, **nem** kép-alapú OCR —
   ez a korábbi tervekben szereplő önálló "④ OCR" fület feleslegessé
   tette, szándékosan lett kihagyva), majd összevetés az ① fülön kiszámolt
   várható nettóval.
3. **③ History** — az utolsó 20 feldolgozás naplója, böngésző
   `localStorage`-ban tárolva (nincs szerveroldali mentés).

Megnyitás: egyszerűen böngészőben megnyitva (nincs build lépés). A CDN-es
`xlsx.js`/`pdf.js` betöltéshez internetkapcsolat szükséges.

## Ismert eltérés a közös config-tól (nyitott pont)

Ez az eszköz jelenleg **nem** olvassa be a `../config/parositas.json`-t —
a termékárak (`TERMEK_MAP`) még a HTML fájlba égetve vannak. A README 2.5
kulcskövetelménye szerint ezt is a közös config-ból kellene kiszolgálni.
A termékadatok már át lettek másolva a configba (`ar_ft` mező), de a
tényleges beolvasás (fetch/JS-modul) még nincs bekötve — ehhez döntés kell,
hogyan futtatod ezt a fájlt (pl. `file://` közvetlen megnyitás esetén a
`fetch()` CORS-korlátok miatt nem biztos, hogy működik egy külső JSON-ra,
lokális szerverről futtatva viszont igen).

Emellett az üzletazonosítás is más elven működik itt (SAP "boltkód"), mint
a Modul 1 `uzlet_parositas`-ában (Wildom név / PM-szám) — ld.
`../config/README.md` "Nyitott kérdés" szakaszát. Ehhez egy boltkód ↔
PM-szám megfeleltető tábla szükséges, amit a felhasználó küld át.
