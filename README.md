# CO₂ per capita: pipeline medalionu w Databricks

Projekt zaliczeniowy: pipeline danych w architekturze medalionu (bronze → silver → gold) w Databricks, napisany w Python/PySpark, z tabelami Delta w Unity Catalog. Pipeline łączy dwa źródła danych i liczy emisję CO₂ na mieszkańca dla krajów i regionów świata.

## Źródła danych

| Źródło | Format | Co zawiera |
|---|---|---|
| [World Bank API](https://api.worldbank.org/v2/country?format=json&per_page=400) | JSON (zagnieżdżony) | lista krajów i agregatów: kod ISO3 (`id`), region (`region.value`) |
| [Our World in Data: CO₂](https://github.com/owid/co2-data) | CSV | wiersz = kraj + rok: `co2` (mln ton), `population` (liczba osób), `iso_code` |

Klucz łączenia: kod ISO3 kraju (`World Bank id` = `iso_code`).

## Architektura

```
World Bank API (JSON) ─► bronze.countries ──┐
                                             ├─► silver.countries_x_pollution ─┬─► gold.co2_tonnes_per_capita_by_country
OWID CSV ─────────────► bronze.pollution ───┘                                  └─► gold.co2_tonnes_per_capita_by_region
```

Wszystkie tabele leżą w katalogu `pollution_per_country_pipeline_project`, w schematach `bronze`, `silver`, `gold`.

### Bronze: surowe dane

Dane trafiają do tabel Delta bez zmian. Zapis `createOrReplace()` jest idempotentny: ponowne uruchomienie nie dubluje wierszy.

| Notebook | Tabela | Opis |
|---|---|---|
| `01_bronze_countries` | `bronze.countries` | odpowiedź API (2. element listy, 1. to metadane), 296 wierszy |
| `02_bronze_pollution` | `bronze.pollution` | cały plik CSV z OWID |

### Silver: dane oczyszczone i połączone

Notebook `03_silver`, tabela `silver.countries_x_pollution` (22 386 wierszy, jeden na kraj i rok).

- **countries:** deduplikacja po `id`, usunięcie agregatów (`region.value = 'Aggregates'`, np. "Africa"), spłaszczenie zagnieżdżonego regionu do kolumny `region`.
- **pollution:** usunięcie wierszy z brakiem `iso_code`, `year`, `co2` lub `population` (to m.in. wiersze zbiorcze typu "World"), odrzucenie `co2 = 0`, deduplikacja po (`iso_code`, `year`).
- **join:** `left join` po `iso_code`, żeby nie tracić prawdziwych terytoriów, których nie ma na liście World Banku (np. TWN, WLF, NIU).
- **brak regionu:** puste wartości zastąpione napisem `Unclassified Region - {kraj}`.

Filtr `co2 = 0` jest potrzebny ze względu na gold: populacja takich wierszy zawyżałaby mianownik wskaźnika regionalnego.

### Gold: tabele analityczne

Notebook `04_gold`. Wskaźnik w obu tabelach to `avg_co2_per_capita` w **tonach CO₂ na osobę**. Surowe `co2` jest w milionach ton, stąd przeliczenie `* 1 000 000`.

| Tabela | Ziarno | Obliczenie |
|---|---|---|
| `gold.co2_tonnes_per_capita_by_country` | kraj + rok | `co2 / population * 1e6` |
| `gold.co2_tonnes_per_capita_by_region` | region + rok | `sum(co2) / sum(population) * 1e6` |

Dla regionów wskaźnik liczony jest jako iloraz sum, a **nie** średnia ilorazów krajów. Średnia z ilorazów traktowałaby mały kraj tak samo jak duży i fałszowałaby wynik.

Grupy `Unclassified Region - {kraj}` zostają w tabeli regionalnej jako grupy jednoelementowe. Odbiorca danych może je odfiltrować.

## Struktura repozytorium

```
notebooks/
  01_bronze_countries.ipynb
  02_bronze_pollution.ipynb
  03_silver.ipynb
  04_gold.ipynb
```

## Uruchomienie

1. Utwórz w Databricks katalog `pollution_per_country_pipeline_project` ze schematami `bronze`, `silver`, `gold` (projekt powstał w Databricks Free Edition, compute: serverless).
2. Zaimportuj notebooki z `notebooks/` do workspace'u.
3. Uruchom je po kolei: `01` → `02` → `03` → `04`.

Źródła są pobierane po HTTP, więc nie trzeba wgrywać żadnych plików ani konfigurować kluczy API.

## Stos

Python, PySpark, Delta Lake, Databricks (Unity Catalog), `requests`, `pandas`.

## Możliwe usprawnienia

- Bronze CSV czytany przez `pandas` (typy zgadywane, brak surowej kopii pliku). Alternatywa: zapis pliku do Volume i `spark.read.csv`.
- Kolumna `is_unclassified` w silver zamiast filtrowania po napisie `Unclassified Region`.
- Orkiestracja notebooków przez Databricks Job.
