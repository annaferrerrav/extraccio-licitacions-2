# Workflow d'extracció de licitacions

2026-09-22 · @Someone

Aquest document explica, pas a pas, com processo un expedient de licitació quan m'envies `prompt_v1.md`: des d'obrir els PDFs fins a generar `files.jsonl`, `control.json` i `metriques.json`.

## Esquema del workflow

```
                        ┌───────────────────────────────┐
                        │  Obrir TOTS els PDFs de la     │
                        │  carpeta de l'expedient        │
                        └────────────────┬────────────────┘
                                         │
                        ┌────────────────▼────────────────┐
                        │  Per cada fitxer i cada pàgina:  │
                        │  comptar caràcters amb           │
                        │  page.get_text()                 │
                        └────────────────┬────────────────┘
                                         │
                              ┌──────────▼──────────┐
                              │  Té capa de text?    │
                              └──────────┬──────────┘
                         SI ──────────────┴────────────── NO
                          │                                │
          ┌───────────────▼───────────────┐  ┌─────────────▼─────────────┐
          │  Llegir text existent          │  │  Rasteritzar pàgina a     │
          │  pdftotext -layout             │  │  300 dpi (pymupdf         │
          └───────────────┬───────────────┘  │  get_pixmap)               │
                          │                    └─────────────┬─────────────┘
              ┌───────────▼───────────┐                      │
              │  Taula àmplia o        │        ┌─────────────▼─────────────┐
              │  multipàgina complexa? │        │  OCR amb Tesseract        │
              └───────────┬───────────┘        │  (-l cat+spa)             │
                 SI ────────┴──────── NO         └─────────────┬─────────────┘
                  │                    │                        │
   ┌──────────────▼─────────────┐     │           ┌─────────────▼─────────────┐
   │  pdfplumber.extract_words  │     │           │  Confiança màxima:        │
   │  amb coordenades x0 / top  │     │           │  "Mitjana"                │
   └──────────────┬─────────────┘     │           └─────────────┬─────────────┘
                  │                    │                        │
                  └──────────┬─────────┘                        │
                              │                                   │
                              └───────────────┬───────────────────┘
                                              │
                              ┌────────────────▼────────────────┐
                              │  Extreure dades i normalitzar    │
                              │  (format numèric europeu)        │
                              └────────────────┬────────────────┘
                                               │
                              ┌────────────────▼────────────────┐
                              │  files.jsonl + control.json +    │
                              │  metriques.json                  │
                              └───────────────────────────────────┘
```

La decisió "té capa de text?" mai és a ull ni pel nom del fitxer: surt de comptar caràcters extrets per pàgina.

## Pas 1 — Obertura de tots els documents

- Localitzo la carpeta de l'expedient dins `input/expedients/`. Si no se m'ha indicat quina, la pregunto: mai en trio una ni les proceso totes de cop.
- Obro **tots** els PDFs de la carpeta, encara que el nom sembli purament administratiu (portades, índexs, certificats). Un document absent de `documents_processats` es considera un error d'extracció, no una omissió vàlida.
- No poso a `output/` cap fitxer de treball intermedi: els temporals (text extret, imatges d'OCR) es netegen abans d'acabar; `input/expedients/<CARPETA>` només ha de contenir els PDFs originals.

## Pas 2 — Diagnòstic i lectura de text (sempre, fitxer per fitxer)

Abans de tocar cap dada, per a cada PDF miro si té capa de text (pymupdf, comptant caràcters extrets per pàgina amb `page.get_text()`). Això em diu si el document és "text natiu" o "escaneig d'imatge" — mai a ull ni pel nom del fitxer.

### 2a. Si té capa de text → NO faig OCR

Faig servir dos mètodes de lectura de text ja existent (no reconeixement òptic, el text ja és dins el PDF):

- **`pdftotext -layout`**: extracció ràpida i lineal, intenta preservar l'alineació de columnes amb espais. Serveix per a la majoria de documents (resolucions, actes curtes, text corregut).
- **`pdfplumber.extract_words()` amb coordenades (`x0`, `top`)**: només quan hi ha una taula àmplia o que travessa pàgines, on `pdftotext -layout` pot desalinear columnes (p. ex. el nom del proveidor queda a una línia i el preu que li correspon "salta" a una altra per culpa d'una nota al peu o un marcador de columna que trenca el layout). Extrec cada paraula amb la seva posició exacta i agrupo per banda horitzontal (`top`) per confirmar que proveidor i preu són realment de la mateixa fila física.

No ho faig sempre: el mètode de coordenades només quan la taula és complexa. Per exemple, a l'expedient 15873831, el document d'adjudicació tenia una taula de 20 lots amb múltiples ofertants per lot i una marca d'aigua/text de verificació superposat que desordenava el text pla; vaig fer `extract_words()` pàgina per pàgina (pàgines 4-9) per confirmar quin preu anava amb quin proveidor. Per documents senzills (com les actes curtes de 2 pàgines del mateix expedient) només amb `pdftotext -layout` n'hi ha prou.

### Com "conto" si hi ha text: una eina que no hi és, i la que realment faig servir

- **`pdffonts fitxer.pdf`** (nivell de fitxer sencer): en teoria llegeix els objectes de font incrustats al PDF i diu si el document defineix alguna font (taula buida = típic d'un escaneig pur). Però **en aquesta màquina no està disponible**: el `CLAUDE.md` del repositori ho documenta explícitament — la instal·lació de poppler via mingw64 només inclou `pdftotext`, no `pdffonts` ni `pdftoppm`. No és una fallada intermitent, és una limitació coneguda de l'entorn. A més, encara que hi fos, només donaria una resposta global per a tot el fitxer, no **a quina pàgina** hi ha text i a quina no.
- **`page.get_text()`** (pymupdf, nivell de pàgina) és el mètode que realment faig servir sempre, perquè és l'únic disponible en aquesta màquina i l'únic que baixa a nivell de pàgina: extreu el text real d'una pàgina concreta (el que ve dels operadors de text del content stream, no d'una imatge). Si torna buit o quasi buit (per sota d'un llindar petit, per descartar soroll com un número de pàgina solt), tracto només aquella pàgina com a escaneig i la rasteritzo per OCR; si torna un bloc de text substancial, l'extrec directament. Complemento amb `page.get_fonts()`, que llista les fonts referenciades específicament per aquella pàgina.

### El cas problemàtic: text a dalt, taula escanejada a sota

El criteri "pàgina buida = escaneig" no és suficient: si una pàgina té un paràgraf de text natiu a dalt i una taula escanejada (imatge) a sota, `get_text()` no torna buit — torna el text del paràgraf — i la pàgina es classificaria com "té text", ignorant per complet la taula escanejada. Per evitar perdre-la, hi afegeixo una comprovació addicional encara que la pàgina "tingui text":

1. **`page.get_images()`** llista les imatges incrustades a la pàgina amb la seva mida. Si n'hi ha alguna prou gran (no una icona o logo), és senyal de contingut escanejat encara que hi hagi text a sobre.
2. Comparo l'àrea que cobreix el text extret (`page.get_text("dict")` dona el bounding box de cada bloc) amb l'àrea total de la pàgina. Una zona gran sense blocs de text però amb una imatge a sota és sospitosa de contenir dades escanejades.
3. Si detecto aquest patró, rasteritzo només aquella regió (o tota la pàgina si és més senzill), hi aplico OCR i fusiono el resultat amb el text natiu ja extret.

Sense aquest pas addicional, una taula escanejada sota un paràgraf de text es perdria silenciosament.

### 2b. Si NO té capa de text → Sí faig OCR

Aquí sí: reconeixement òptic real, perquè no hi ha text per extreure directament.

1. **Rasterització**: converteixo la pàgina a imatge a 300 dpi amb pymupdf (`page.get_pixmap(dpi=300)`). No uso `pdftoppm` perquè no està disponible en aquesta màquina.
2. **Noms de fitxer únics**: per evitar col·lisions quan dos PDFs comparteixen els primers caràcters del nom, cada imatge rasteritzada rep un índex o etiqueta única, mai un prefix truncat del nom original.
3. **OCR amb Tesseract**, idiomes català + castellà (`-l cat+spa`). Si Tesseract o els paquets d'idioma no estan instal·lats a la màquina, els instal·lo abans (`winget install --id UB-Mannheim.TesseractOCR`) i descarrego `cat.traineddata`/`spa.traineddata` a la carpeta `tessdata/`.
4. **Confiança limitada**: tota fila que ve d'OCR d'escaneig queda marcada com a màxim "Mitjana" de confiança (o "Baixa" si l'OCR és dubtós), i ho anoto a `notes` (p. ex. "Valor procedent d'OCR d'escaneig").

**En resum**: capa de text → llegeixo el text (pla o per coordenades, segons complexitat de la taula); sense capa de text → OCR. Mai faig OCR sobre un PDF que ja té text natiu — seria més lent i menys fiable que llegir-lo directament.

**Advertència (vist en un cas real)**: aquest diagnòstic es fa amb un llindar de caràcters per pàgina sencera. Si una pàgina té prou text per superar el llindar però també conté una imatge incrustada amb dades (taula escanejada, segell), el criteri bàsic no la detecta com a tal — cal la comprovació addicional d'imatges incrustades (`page.get_images()` + àrea sense text) descrita més amunt, que en la pràctica real no sempre s'ha aplicat fins ara.

## Pas 3 — Extracció de dades i normalització

- Amb el text ja disponible (natiu o via OCR), busco a cada document les dades que demana `prompt_v1.md`: proveidor, referència, preus, quantitats, estat, lots, etc., seguint exactament els criteris d'on buscar cada camp que hi defineix.
- **Normalització numèrica**: els documents originals fan servir format europeu (punt = separador de milers, coma = decimals). El que escric a `files.jsonl` mai porta punts de milers — convertit sempre a format net abans de desar-ho.
- Quan una dada surt de creuar informació entre diversos documents de l'expedient (p. ex. completar una descripció amb l'annex econòmic, o deduir un proveidor del context), ho deixo anotat a `notes`.

## Pas 4 — Confiança i fitxers de sortida

| Confiança | Quan l'aplico |
| --- | --- |
| Alta | Text net, sense ambigüitat, `notes` buida; en taules grans, assignació proveidor↔preu confirmada per coordenades |
| Mitjana | Dada deduida, creuada entre documents, calculada, o d'un OCR raonable però no perfecte |
| Baixa | OCR dubtós, files enganxades separades a mà, dades il·legibles, inferència feble, o assignació proveidor↔preu no confirmada |

No poso "Alta" per defecte: aquest camp és el que decideix què es revisa a mà.

Un cop acabada l'extracció genero, sempre dins `output/<CARPETA>/` (mai a `input/`):

- **`files.jsonl`** — les dades extretes, una fila per registre.
- **`control.json`** — control de l'expedient (documents processats, incidències).
- **`metriques.json`** — mètriques de l'extracció.

La resposta final que et dono només conté les dues línies de resum que especifica `prompt_v1.md`: sense taules Markdown ni explicacions addicionals.
