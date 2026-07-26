# rendeles-generalas/ — Modul 1

Wildom export → PizzaMe_megrendelő sablon automata feltöltés. Részletes
specifikáció a gyökér `README.md` "2. Modul 1" szakaszában.

## Használat

```
python3 wildom_to_cola.py <wildom_export.xlsx> <pizzame_sablon.xlsx> [uj_lap_nev]
```

- `<wildom_export.xlsx>` — a Wildom "Beszállítói megrendelések" → "Coca Cola"
  profil → "Excel 2007+" exporttal letöltött fájl.
- `<pizzame_sablon.xlsx>` — a PizzaMe_megrendelő fájl legutóbbi állapota,
  amiből a script egy új lapot másol.
- `[uj_lap_nev]` — opcionális; ha nem adod meg, a mai dátum lesz (`YYYYMMDD`).

A kimenet egy `..._Cola_<uj_lap_nev>.xlsx` fájl a sablon mellett.

## Függőség

```
pip install openpyxl
```

## Párosítási adatok

A termék- és üzletlista **nem** ebben a scriptben van, hanem a közös
`../config/parositas.json` fájlban — ha cikkszám, ár vagy telephely
változik, azt a JSON-t kell szerkeszteni, nem ezt a kódot. Ld.
`../config/README.md`.

## Teszt

A script viselkedése ellenőrizve lett szintetikus teszt-adatpárral: 19
termék, 238 kitöltött mennyiség-cella — megegyezik a `config/parositas.json`-ra
való átállás előtti, hardkódolt verzió eredményével.
