# Extracció de licitacions — fase 2 (OCR) / fase 5

Extracció automatitzada, expedient per expedient, de les dades de línies de
producte (proveïdor, referència, preus, quantitats, estat...) dels documents
PDF d'una licitació pública, mitjançant Claude Code.

## Estructura de carpetes

```
input/expedients/<CARPETA>/   PDFs originals d'un expedient (un per subcarpeta)
output/<CARPETA>/             files.jsonl + control.json + metriques.json
prompt_v1.md                  instrucció de sistema que segueix Claude Code
build_excel.py                consolida tots els output/<CARPETA>/ en un .xlsx
```

Cada subcarpeta de `input/expedients/` és un identificador numèric (p. ex.
`4161038`) i correspon a UN expedient de licitació. Mai es barregen documents
de carpetes diferents.

`input/` i `output/` no es pugen al repositori de GitHub (veure
`.gitignore`): contenen PDFs i dades reals de contractació de l'hospital.
El repositori remot només conté codi i documentació.

## Com s'executa una extracció

1. Obrir el projecte a Claude Code (VS Code).
2. Indicar-li quina carpeta d'`input/expedients/` ha de processar (Claude Code
   no en tria cap pel seu compte).
3. Claude Code segueix les instruccions de `prompt_v1.md` i escriu, per a
   aquella carpeta:
   - `output/<CARPETA>/files.jsonl` — una fila JSON per línia de producte/lot.
   - `output/<CARPETA>/control.json` — metadades de control: expedient
     normalitzat, columnes extra, confiança per fila, incidències...
   - `output/<CARPETA>/metriques.json` — temps i consum de tokens de
     l'execució (sense cost en euros: pla Claude Pro per subscripció).

Es processa **un expedient per execució**. Per processar-ne diversos cal
repetir l'operació indicant cada vegada la carpeta corresponent.

### PDFs escanejats (OCR)

Alguns documents no tenen capa de text (escanejats). `prompt_v1.md` indica
que cal detectar-ho i fer OCR (300 dpi, `cat+spa`) abans d'extraure'n dades.
Això requereix tenir **Tesseract OCR** instal·lat, amb els paquets d'idioma
`cat` i `spa` a `tessdata/` (no només l'`eng` per defecte).

## Generar l'Excel consolidat

```
python build_excel.py                                          # tots els expedients d'output/
python build_excel.py output extraccio_licitacions.xlsx all     # equivalent, explícit
python build_excel.py output sonnet_ok.xlsx 4161038,9252209     # només aquests expedients
```

L'script llegeix cada `output/<CARPETA>/files.jsonl` + `control.json` i genera
un únic `.xlsx` amb una fila per línia extreta, les 13 columnes obligatòries
+ `Notes` + columnes extra detectades + confiança/motiu/font. Es pot tornar a
executar cada cop que s'acaba un expedient nou.

## Convencions clau de les dades

- Claus JSON, valors d'`estat` i el text de `notes` sempre en **català**
  (perquè encaixin amb l'Excel de destí).
- Format numèric: coma decimal, sense cap punt de milers ni símbol d'euro
  (`"1500,50"`, no `"1.500,50"`).
- `estat` només admet: `Adjudicat`, `Oferta`, `Desestimat`, `Desert`.
- Tot valor calculat, deduït o creuat entre documents es declara a `notes`
  i baixa la confiança de la fila com a mínim a `Mitjana`.

Els detalls complets (què és una fila, com resoldre contradiccions entre
documents, criteris de confiança, format de `control.json`...) són a
`prompt_v1.md`.

## Workflow detallat (`Workflow_extraccio_licitacions.md`)

Explicació pas a pas del procés real que segueix Claude Code en processar un
expedient (obertura de PDFs, detecció de capa de text, OCR si cal, extracció
i normalització). Útil com a referència ràpida sense haver de rellegir tot
`prompt_v1.md`. Hi ha també una versió en PDF del mateix document.

## Comparador d'extraccions (`comparador_licitacions.html`)

Pàgina web autònoma per comparar l'Excel generat per `build_excel.py` amb
una resposta correcta feta a mà: nombre de files, files per proveïdor,
import total, import per lot/proveïdor i una comparació fila per fila amb
les diferències ressaltades en vermell.

**Ús:** obrir `comparador_licitacions.html` amb doble clic (s'obre al
navegador per defecte) i pujar-hi els dos Excels. No cal instal·lar res ni
executar cap servidor.

**Requisits:** un navegador modern (Chrome, Edge o Firefox) i connexió a
internet la primera vegada que s'obre, per carregar dues llibreries des de
CDN (lectura d'Excel i tipografia). Tot el processament de les dades passa
dins el navegador: els fitxers pujats **no s'envien enlloc**, per això és
segur pujar-hi la resposta correcta real.
