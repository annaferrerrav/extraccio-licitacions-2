# CLAUDE.md

Instruccions per a Claude Code en aquest repositori.

## Què és aquest projecte

Extracció de dades de licitacions públiques (proveïdor, referència, preus,
quantitats, estat...) a partir dels PDFs de cada expedient, cap a
`files.jsonl` + `control.json` + `metriques.json`. Veure `README.md` per a
l'estructura de carpetes.

## Regla número u

**Quan es demani processar un expedient, segueix `prompt_v1.md` al peu de la
lletra.** És la instrucció de sistema completa (rol, on buscar cada dada,
format de sortida, normalització numèrica, criteris de confiança, control
final...). No la reinterpretis ni la resumeixis: llegeix-la sencera abans de
començar i torna-la a consultar davant de qualsevol dubte durant l'extracció.

Punts que es passen per alt sovint i que val la pena recordar:

- **Un expedient per execució.** Si l'usuari no diu quina carpeta
  d'`input/expedients/` processar, pregunta-ho — no en triïs cap ni les
  processis totes de cop.
- **Obre TOTS els documents de la carpeta**, encara que el nom sembli
  purament administratiu. Un document absent de `documents_processats` és un
  error d'extracció.
- Comprova amb codi (pymupdf/pdfplumber) si cada PDF té capa de text abans de
  triar mètode de lectura. No ho donis per fet pel nom del fitxer.
- El format numèric dels documents és europeu (punt = milers, coma =
  decimals). El que s'escriu a `files.jsonl` mai porta punts.

## Entorn d'aquesta màquina (Windows)

- Bash disponible és Git Bash (POSIX). PowerShell també disponible per a
  tasques natives de Windows.
- Eines Python natives (`python3`) amb `pdfplumber` i `pymupdf` (`fitz`)
  instal·lades — útils per a diagnòstic de capa de text, extracció de taules
  i rasterització de pàgines.
- `pdftotext` (poppler, via mingw64) disponible per Bash. `pdffonts` i
  `pdftoppm` **no** hi són; per rasteritzar pàgines a 300 dpi per a OCR, usa
  `pymupdf` (`page.get_pixmap(dpi=300)`) en lloc de `pdftoppm`.
- **Tesseract OCR pot no estar instal·lat** de sèrie en una màquina nova, ni
  els paquets d'idioma `cat`/`spa` (només ve `eng` per defecte). Si cal fer
  OCR i no hi és:
  - Instal·la'l amb `winget install --id UB-Mannheim.TesseractOCR -e --silent`.
  - Descarrega `cat.traineddata` i `spa.traineddata` de
    `github.com/tesseract-ocr/tessdata_fast` cap a la carpeta `tessdata/` de
    la instal·lació.
  - Executa'l amb `-l cat+spa`.
- Compte amb noms de fitxer llargs si els trunques tu mateix (p. ex. per
  generar noms de sortida d'imatges rasteritzades): dos PDFs diferents poden
  compartir els primers ~30 caràcters del nom i col·lidir. Usa un índex o
  etiqueta única, no un prefix truncat del nom original.

## Fitxers de sortida

Els tres fitxers (`files.jsonl`, `control.json`, `metriques.json`) es
generen sempre dins `output/<CARPETA>/`, mai a `input/`. Si es creen fitxers
temporals de treball (text extret, imatges d'OCR...) dins
`input/expedients/<CARPETA>/` per fer la feina, neteja'ls abans d'acabar:
aquesta carpeta només ha de contenir els PDFs originals.

**`input/` i `output/` estan exclosos del control de versions** (`.gitignore`):
contenen PDFs i dades de contractació real, i no s'han de pujar al
repositori remot de GitHub. Tampoc els `.xlsx` generats. Si es modifica el
`.gitignore`, no s'hi ha de treure aquesta exclusió sense que l'usuari ho
demani explícitament.

## Resposta final

Un cop escrits els tres fitxers, la resposta final a l'usuari no ha de
contenir cap taula Markdown ni explicació ni codi: només les dues línies de
resum que especifica `prompt_v1.md`.
