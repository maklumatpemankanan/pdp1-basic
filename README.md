# PDP-1 Tiny BASIC

Ein ganzzahliger BASIC-Interpreter für den DEC PDP-1 mit 16K Kernspeicher
(Speicherbänke 0–3), geschrieben in MACRO-1-Assembler, mit Punktgrafik auf dem
Type-30-Bildschirm. Entwickelt und getestet auf dem PiDP-1 und im Emulator
*blincolnlights*.

```
PDP-1 TINY BASIC
6143 WORDS FREE
>10 FOR I=1 TO 5
>20 PRINT I, I×I
>30 NEXT I
>RUN
1	1
2	4
3	9
4	16
5	25
>
```

---

## Inhalt

1. [Überblick](#1-überblick)
2. [Schnellstart](#2-schnellstart)
3. [Sprachreferenz](#3-sprachreferenz)
4. [Programme auf Lochstreifen](#4-programme-auf-lochstreifen)
5. [Speicherbelegung](#5-speicherbelegung)
6. [Aufbau des Interpreters](#6-aufbau-des-interpreters)
7. [Dateien und Werkzeuge](#7-dateien-und-werkzeuge)
8. [Entwicklungsgeschichte](#8-entwicklungsgeschichte)
9. [Einschränkungen](#9-einschränkungen)
10. [Ausblick](#10-ausblick)

---

## 1. Überblick

| Merkmal | Wert |
|---|---|
| Zielsystem | PDP-1 mit Speichererweiterung, 16K Wörter zu 18 Bit (Bank 0–3) |
| Zahlen | ganzzahlig, 18 Bit Einerkomplement, −131071 … +131071 |
| Variablen | A–Z |
| Programmspeicher | 6143 Wörter (Bank 2 und untere Hälfte von Bank 3) |
| Grafik | Type 30, 1024 × 1024 Punkte, bis zu 1024 Punkte gleichzeitig |
| Ein-/Ausgabe | Flexowriter (Schreibmaschine), Type-30-Bildschirm, Lochstreifenleser, Lochstreifenstanzer |
| Optionen | Multiply/Divide-Option (EAE) wird beim Start erkannt und genutzt; ohne sie rechnet der Interpreter in Software |
| Codegröße | Interpreter 0100–4027 in Bank 0, Daten 355 Wörter in Bank 1 |
| Ladeband | `basic.tape`, 8410 Zeilen Lochstreifen, ca. 21 s Lesezeit |

Der Interpreter folgt dem Vorbild der Tiny-BASIC-Dialekte der 1970er Jahre:
Zeilennummern, eine Anweisung pro Zeile, direkter Modus und Programmmodus.
Programmzeilen werden beim Eingeben tokenisiert und können auf Lochstreifen
gestanzt und wieder eingelesen werden. Mit `PLOT` zeichnen Programme Punkte
auf den Type-30-Bildschirm; der Interpreter hält das Bild stehen, solange das
Programm läuft.

---

## 2. Schnellstart

### 2.1 Bauen

Voraussetzungen: `macro1_1` (aus `pidp1/src/macro`) und Python 3.

```sh
macro1_1 -d basic0.mac                              # Bank 0, Blockformat für READ-IN
macro1_1 -r -d basic1.mac                           # Bank 1, RIM, bank-relativ
python3 xtape.py --check basic0.lst 1:basic1.lst    # Adressen beider Teile abgleichen
python3 xtape.py -o basic.tape basic0.rim 1:basic1.rim
```

`macro1_1` nennt seine Ausgabe immer `.rim`, auch im Blockformat. Auf die
Maschine kommt nur `basic.tape`.

### 2.2 Laden

1. Schalter **EXTEND aus**, Extension-Adressschalter auf **0**.
2. `basic.tape` in den Lochstreifenleser legen.
3. **READ-IN** drücken.

Nach etwa 21 Sekunden erscheint der Begrüßungstext auf der Schreibmaschine.
Der Interpreter schaltet selbst in den Extend Mode.

**Neustart** ohne neu zu laden: START bei Adresse **0100**. Programm und
Variablen werden dabei gelöscht.

### 2.3 Bedienung

| Eingabe | Wirkung |
|---|---|
| `10 PRINT "HALLO"` | Zeile 10 eintragen oder ersetzen |
| `10` | Zeile 10 löschen |
| `PRINT 2+3` | sofort ausführen (direkter Modus) |
| Rückschritt | letztes Zeichen der Eingabezeile zurücknehmen |
| Wagenrücklauf | Zeile abschließen |
| beliebige Taste während RUN, LIST, SAVE, LOAD | Abbruch mit `BREAK` |

Schlüsselwörter dürfen klein oder groß geschrieben werden. LIST zeigt sie
einheitlich in Großbuchstaben, ebenso die Variablennamen. Leerzeichen
außerhalb von Texten entfernt der Interpreter beim Eintippen; LIST setzt um
die Schlüsselwörter wieder je ein Leerzeichen. Aus `20 print "x", a * b` wird
also `20 PRINT "x",A×B`. Text in Anführungszeichen und alles hinter `REM`
bleibt so, wie es getippt wurde.

---

## 3. Sprachreferenz

### 3.1 Befehle

Befehle werden in der Regel direkt eingegeben, dürfen aber auch im Programm
stehen.

| Befehl | Beschreibung |
|---|---|
| `RUN` | Variablen löschen, Programm ab der ersten Zeile ausführen |
| `LIST` | ganzes Programm ausgeben |
| `LIST n` | Programm ab Zeile *n* ausgeben |
| `NEW` | Programm, Variablen und Bildschirm löschen |
| `SAVE` | Programm auf den Lochstreifenstanzer ausgeben (Abschnitt 4) |
| `LOAD` | Programm vom Lochstreifenleser einlesen (Abschnitt 4) |

### 3.2 Anweisungen

| Anweisung | Beschreibung |
|---|---|
| `[LET] v = ausdruck` | Wert zuweisen; `LET` darf entfallen |
| `PRINT element, element, …` | ausgeben; *element* ist ein Ausdruck oder `"text"` |
| `PRINT` | Leerzeile |
| `INPUT v [, v …]` | Werte einlesen, Eingabeaufforderung `? ` |
| `IF ausdruck THEN anweisung` | bedingt ausführen; `THEN` darf entfallen |
| `IF ausdruck THEN zeile` | bedingt springen |
| `GOTO ausdruck` | springen; im direkten Modus Programmstart ab dieser Zeile |
| `GOSUB ausdruck` | Unterprogramm aufrufen |
| `RETURN` | aus dem Unterprogramm zurückkehren |
| `FOR v = a TO b [STEP c]` | Zählschleife, `STEP` auch negativ |
| `NEXT [v]` | Schleifenende |
| `PLOT x, y` | Punkt auf den Bildschirm setzen (Abschnitt 3.5) |
| `CLS` | Bildschirm löschen |
| `REM text` | Kommentar |
| `END` | Programm beenden |

Einzelheiten:

- **PRINT:** Ein Komma setzt einen Tabulator. Steht das Komma am Zeilenende,
  entfällt der Zeilenvorschub.
- **INPUT:** Jeder Wert ist ein vollständiger Ausdruck, zum Beispiel `A+1` oder
  `RND(6)`. Mehrere Werte dürfen durch Komma getrennt in einer Zeile stehen.
  Fehlende Werte werden mit einem neuen `? ` erfragt. Eine leere Eingabe (nur
  RETURN) setzt 0.
- **IF:** Jeder Wert ungleich 0 gilt als wahr. Ist die Bedingung falsch, geht es
  mit der nächsten Zeile weiter.
- **GOTO und GOSUB** akzeptieren berechnete Zeilennummern, etwa `GOTO 100+10*K`.
- **FOR/NEXT:** Schleifen dürfen verschachtelt werden. Eine Schleife läuft
  mindestens einmal. Bei positivem Schritt wird fortgesetzt, solange
  *v* ≤ *b* ist, bei negativem Schritt, solange *v* ≥ *b* ist. Ein erneutes `FOR`
  mit derselben Variablen ersetzt die alte Schleife; ein Sprung aus einer
  Schleife heraus belegt daher keinen Speicher dauerhaft. FOR ist nur im
  Programm erlaubt.
- **GOSUB** darf rekursiv sein, bis zu 170 Ebenen.

### 3.3 Ausdrücke

Vorrang von oben nach unten:

| Stufe | Operatoren |
|---|---|
| Klammern, Funktionen, Vorzeichen | `( )` `RND()` `ABS()` `-x` `+x` |
| Multiplikation, Division | `×` (oder `*`, `x`, `X`) und `/` |
| Addition, Subtraktion | `+` `-` |
| Vergleich | `=` `<` `>` `<=` `>=` `<>` `#` |

- **Vergleiche** liefern 1 (wahr) oder 0 (falsch) und sind nur einmal je
  Ausdruck erlaubt.
- **Division** schneidet zur Null hin ab: `7/2` = 3, `-7/2` = −3.
- **Rest** einer Division: `A-A/B*B`.
- **Überlauf** über ±131071 bricht mit `?OVERFLOW` ab.

### 3.4 Funktionen

| Funktion | Ergebnis |
|---|---|
| `RND(n)` | Zufallszahl von 1 bis *n*; *n* ≥ 1 |
| `ABS(x)` | Betrag von *x* |

Der Zufallsgenerator ist der aus Spacewar!. Sein Startwert läuft weiter,
solange der Interpreter auf eine Taste wartet, daher ist jeder Programmlauf
anders.

### 3.5 Grafik

`PLOT x, y` setzt einen Punkt auf den Type-30-Bildschirm. Die Koordinaten folgen
dem eingebauten Koordinatensystem des Type 30:

| | |
|---|---|
| Bereich | −511 … +511 für x und y |
| Ursprung | Bildmitte |
| Richtung | x nach rechts, y nach oben |
| Auflösung | 1023 × 1023 Punkte |
| Punkte gleichzeitig | 1024 |

- **Bild stehen lassen:** Der Type 30 hat keinen Bildspeicher, ein Punkt
  verblasst nach etwa einer Viertelsekunde. Deshalb merkt sich der Interpreter
  jeden Punkt in einer Anzeigeliste und zeichnet sie ständig nach, solange das
  Programm läuft, auch während es druckt oder bei `INPUT` auf eine Eingabe
  wartet.
- **Verblassen:** Kehrt der Interpreter zur Eingabe `>` zurück, wird die
  Anzeigeliste geleert, und das Bild verblasst wie auf einem echten Bildschirm.
  Das geschieht am Programmende, bei `END`, nach einer Fehlermeldung, nach
  `BREAK` und nach jedem direkt getippten Befehl. Ein direkt getipptes
  `PLOT x, y` blitzt also einmal auf und verlöscht.
- **Bild am Programmende zeigen:** Ein Programm, dessen Bild stehen bleiben
  soll, wartet am Ende mit `INPUT`. Dabei zeichnet der Interpreter mit voller
  Rate nach, und RETURN beendet das Programm:

  ```basic
  900 PRINT "ENDE MIT RETURN"
  910 INPUT Z
  ```

- **Löschen während des Laufs:** `CLS` löscht das Bild.
- **Fehler:** Koordinaten außerhalb von ±511 ergeben `?ILLEGAL VALUE`, der
  1025. Punkt ergibt `?DISPLAY FULL`.
- **Wiederholrate:**

  | Situation | nachgezeichnete Punkte pro Sekunde (Emulator) |
  |---|---|
  | Warten auf `INPUT`, Druckausgabe | ca. 10 000 |
  | während das Programm rechnet (32 Punkte pro Programmzeile) | ca. 4 000 |

  Bei vielen Punkten flimmert das Bild deshalb während des Rechnens etwas und
  steht ruhig, sobald das Programm bei `INPUT` wartet.
- **Geschwindigkeit:** Programme ohne Grafik werden durch die Auffrischung
  nicht langsamer. Bei Programmen mit vielen Punkten geht dagegen ein großer
  Teil der Zeit ins Nachzeichnen: Bei der Lissajous-Figur mit 1001 Punkten
  ist es knapp die Hälfte.

Beispiel, ein Kreis nach Minskys Kreisalgorithmus, der auf dem PDP-1
entstanden ist:

```basic
10 CLS
20 X=400
30 Y=0
40 FOR I=1 TO 101
50 PLOT X,Y
60 X=X-Y/16
70 Y=Y+X/16
80 NEXT I
90 INPUT Z
```

**Programme aus anderen BASICs anpassen.** Viele Grafikprogramme sind für
Fließkomma-BASIC und Bildschirme mit 40 bis 320 Punkten Breite geschrieben.
Hier wird ganzzahlig gerechnet, und `B/X` ergibt 0, solange B kleiner als X ist.
Solche Programme werden deshalb hochskaliert:

- Alle Amplituden und Längen um einen Faktor wie 10 bis 20 vergrößern, dann
  bleiben die Divisionen genau genug.
- Den Ursprung in die Bildmitte legen, Verschiebungen wie `32+…` also
  weglassen.
- Auf Zwischenwerte achten: `X*X` überläuft schon bei X = 363.
  `(X/2)*(X/2)/100` liefert dasselbe wie `X*X/400` ohne Überlauf.

`lissajous.bas` ist so ein angepasstes Programm. Im Original standen die
Amplituden 15 und 6 und der Mittelpunkt (32, 22); das ergab hier nur ein
30 Punkte großes Quadrat. Mal 20 und um 0, 0 zentriert entsteht die richtige
Figur aus 1001 Punkten.

### 3.6 Zeichen auf dem Flexowriter

Der Flexowriter kennt keinen Stern, kein Semikolon und keinen Doppelpunkt.

| gewünscht | tippen | Hinweis |
|---|---|---|
| mal | `×` bzw. `*` am PC | `x` und `X` werden an dieser Stelle ebenfalls als „mal“ gelesen |
| ungleich | `<>` | `#` geht auch, wird aber als `⊃` gedruckt |
| Trennzeichen in PRINT | `,` | `;` gibt es nicht |
| mehrere Anweisungen pro Zeile | — | nicht möglich |

Hinter einem Wert kann in einem Ausdruck nie ein Buchstabe stehen. Deshalb ist
`DXD` eindeutig „D mal D“, und die Variable X bleibt trotzdem nutzbar.

### 3.7 Fehlermeldungen

Im Programmlauf folgt die Zeilennummer, zum Beispiel `?DIVISION BY ZERO IN 60`.

| Meldung | Ursache |
|---|---|
| `?SYNTAX ERROR` | Anweisung nicht verständlich |
| `?OVERFLOW` | Ergebnis oder Zahl außerhalb ±131071 |
| `?DIVISION BY ZERO` | Division durch 0 |
| `?UNDEFINED LINE` | GOTO/GOSUB auf eine nicht vorhandene Zeile |
| `?RETURN WITHOUT GOSUB` | RETURN ohne offenes GOSUB |
| `?NEXT WITHOUT FOR` | NEXT ohne passendes FOR |
| `?ILLEGAL VALUE` | Zeilennummer 0, `RND(n)` mit *n* < 1, PLOT außerhalb ±511 |
| `?ILLEGAL DIRECT` | FOR im direkten Modus |
| `?OUT OF MEMORY` | Programmspeicher voll |
| `?DISPLAY FULL` | mehr als 1024 Punkte auf dem Bildschirm |
| `?STACK OVERFLOW` | zu tief verschachtelt (GOSUB, FOR oder Ausdruck) |
| `BREAK` | Abbruch durch Tastendruck |

---

## 4. Programme auf Lochstreifen

### 4.1 SAVE

`SAVE` stanzt das Programm als lesbaren Text in dieser Reihenfolge:

1. 40 Zeilen Leerband als Vorlauf
2. Umschaltung auf Kleinschrift (072)
3. das Programm, genau wie LIST es ausgibt
4. Stoppcode (013)
5. 40 Zeilen Leerband als Nachlauf

Ein Tastendruck bricht ab, danach schreibt der Interpreter wieder auf die
Schreibmaschine.

### 4.2 LOAD

`LOAD` löscht Programm, Variablen und Bildschirm und liest dann Zeile für Zeile vom
Leser. Jede Zeile wird eingetragen, als wäre sie getippt worden.

- Am **Stoppcode** endet das Einlesen mit der Meldung `n LINES LOADED`.
- **Zeilen ohne Nummer** und Leerzeilen werden übersprungen.
- **Fehlt der Stoppcode,** beendet ein Tastendruck das Einlesen. Die bis dahin
  gelesenen Zeilen bleiben erhalten.
- `LOAD` darf getippt werden, **bevor** das Band eingelegt ist. Der Interpreter
  wartet auf das Band.
- **Fehler** beim Einlesen, etwa `?ILLEGAL VALUE` bei Zeile 0, beenden LOAD.

### 4.3 Bandformat

Das Format entspricht den echten PDP-1-Textbändern. Zum Vergleich diente das
Lisp-Band `lisp-defs.pt` der PiDP-1.

| Element | Lochung |
|---|---|
| Zeichen | 6-Bit-FIO-DEC-Code in Loch 1–6 |
| Loch 8 | ungerade Parität über die sechs Datenlöcher |
| Leerzeichen | 200 (Code 0 mit Paritätsloch) |
| Leerband | 0, wird überlesen |
| ausgestanzte Zeile | 377, wird überlesen |
| Groß-/Kleinschrift | Umschaltcodes 074 / 072 |
| Zeilenende | 077 (277 mit Parität) |
| Bandende | Stoppcode 013 |

### 4.4 Austausch mit dem PC

`bastape.py` wandelt in beide Richtungen um:

```sh
python3 bastape.py zahlenraten.bas zahlenraten.tape    # Textdatei → Band
python3 bastape.py -d gesichert.tape gesichert.bas     # Band → Textdatei
```

- `*` wird zu `×` und beim Zurückwandeln wieder zu `*`.
- Ein mit `SAVE` gestanztes Band ist Byte für Byte identisch mit dem, das
  `bastape.py` aus derselben Datei erzeugt.
- Die Werkzeuge `encode_fiodec` und `decode_fiodec` der PiDP-1 verwenden
  dasselbe Format. Bei `encode_fiodec` die Frage nach dem Stoppcode mit `y`
  beantworten.

---

## 5. Speicherbelegung

Der Interpreter läuft im **Extend Mode**:

- Direkte Adressen erreichen nur die Bank, in der der Code läuft, also Bank 0.
- Daten in Bank 1–3 werden über Zeigerwörter mit voller 16-Bit-Adresse
  angesprochen (`lac i zeiger`). Indirektion ist in diesem Modus einstufig.

### 5.1 Übersicht

| Bank | Adressen | Inhalt | auf dem Band |
|---|---|---|---|
| 0 | 00000–07777 | Interpreter, Konstanten, Variablen, Lader | ja |
| 1 | 10000–17777 | Tabellen, Meldungen, Puffer, Stapel | Tabellen und Meldungen |
| 2 | 20000–27777 | Programmspeicher | nein |
| 3 | 30000–33777 | Programmspeicher (Fortsetzung) | nein |
| 3 | 34000–37777 | Anzeigeliste für PLOT | nein |

### 5.2 Bank 0: Interpreter

| Adressen | Inhalt |
|---|---|
| 0000–0003 | reserviert für den Sequence Break (unbenutzt) |
| 0004–0077 | frei |
| 0100–1647 | Start mit EAE-Erkennung, Hauptschleife, Sprungtabelle, Fehlerbehandlung, Zeileneditor, alle Anweisungen einschließlich LIST, PLOT, CLS, SAVE und LOAD |
| 1650–2472 | Ausdrucksauswertung (`expr`, `aexpr`, `trm`, `factor`), Vergleich, Multiplikation, Division, Zufallszahlen |
| 2473–3502 | Zahlen ein- und ausgeben, Zeileneingabe, Zeichenklassen, Tokenisierung (`crunch`, `gettok`, `prtok`, `kwfind`), Schreibmaschine und Stanzer (`inch`, `outch`, `rawch`, `punch`), Textausgabe, Bildauffrischung (`clsdl`, `dslice`) |
| 3503–3634 | Konstanten und Zeiger auf Bank 1–3 |
| 3635–4027 | Variablen des Interpreters |
| 4030–7677 | **frei, 1960 Wörter** |
| 7700–7737 | `xld`, Lader für Bank 1–3 (nur beim Einlesen gebraucht) |
| 7740–7750 | frei |
| 7751–7777 | Bootstrap-Lader von macro1_1, **nicht belegen** |

Die Grenzen der Abschnitte stammen aus dem Listing `basic0.lst` und
verschieben sich bei jeder Codeänderung. `xtape.py --check` meldet das Ende
des Codes (`endcod`) und warnt, wenn es `xld` erreicht.

### 5.3 Bank 1: Daten

| Adressen | Inhalt | Größe | auf dem Band |
|---|---|---|---|
| 10000–10031 | Variablen A–Z | 26 Wörter | nein (RUN/NEW löschen) |
| 10100–10377 | Meldungen, je 16 Wörter | | ja |
| 10400–10602 | Schlüsselworttabelle (23 Einträge), Platz bis 10777 | 131 Wörter | ja |
| 11000–11177 | Eingabepuffer für Programmzeilen | 126 Zeichen | nein |
| 11200–11377 | Eingabepuffer für INPUT | 126 Zeichen | nein |
| 11400–11577 | weitere Meldungen | | ja |
| 11600–11777 | Zeichenklassen für alle 128 Zeichen | 128 Wörter | ja |
| 12000–12777 | GOSUB-Stapel, 3 Wörter pro Ebene | 170 Ebenen | nein |
| 13000–13777 | FOR-Stapel, 5 Wörter pro Schleife | 102 Schleifen | nein |
| 14000–17777 | Stapel der Ausdrucksauswertung | 2048 Wörter | nein |

Die Bereiche, die nicht auf dem Band stehen, werden beim Start gelöscht oder
erst zur Laufzeit beschrieben.

**Format der Schlüsselworttabelle:** Jeder Eintrag besteht aus den Buchstaben
des Worts (ein FIO-DEC-Code pro Wort, ohne Fallbit), dann `777777` und dem
Token. `777776` beendet die Tabelle.

**Zeichenklassen:** Der Index ist das 7-Bit-Zeichen.

| Wert | Klasse |
|---|---|
| `100+n` | Buchstabe *n* (0 = A … 31 = Z) |
| `200+n` | Ziffer *n* |
| `400` | fallunabhängiges Zeichen (Leerzeichen, Tabulator, Rückschritt, Wagenrücklauf) |
| `0` | sonstiges Zeichen |

### 5.4 Bank 2–3: Programmspeicher und Anzeigeliste

Die Programmzeilen liegen aufsteigend sortiert ab 20000 und dürfen bis 33777
reichen. Jede Zeile ist ein
Eintrag:

```
+0   Zeilennummer
+1   Länge des Eintrags in Wörtern (2 + Text einschließlich 077)
+2   Text, ein Wort je Element:
       Zeichen      0 … 177
       Token        200 + Nummer
       Zahl         400000 + Wert (Wert 0 … 131071)
     ...
     077  Zeilenende
```

Nach der letzten Zeile folgt ein Wort 0 als Endemarke. Der Zeiger `ptop` zeigt
darauf, alles dahinter bis 33777 ist frei. Mit der Längenangabe springen GOTO,
GOSUB und der Zeilenwechsel ohne Textsuche von Eintrag zu Eintrag.

Beispiel: `20 PRINT I*12` belegt 7 Wörter, nämlich `24` (20 dezimal), `7`,
`201` (PRINT), `171` (I), `173` (×), `400014` (Zahl 12) und `77`.

Zahlen über 131071 bleiben als Ziffern stehen und melden zur Laufzeit
`?OVERFLOW`.

**Anzeigeliste (34000–37777):** Jeder Punkt belegt zwei Wörter, erst Y, dann
X, beide schon um 8 Bit nach links verschoben. In diesem Format erwartet `dpy`
die Koordinate in den oberen 10 Bits von AC und IO. Der Zeiger `dlend` zeigt
auf den nächsten freien Platz, `dcur` auf den nächsten nachzuzeichnenden
Punkt. 2048 Wörter ergeben 1024 Punkte.

### 5.5 Tokens

| Token | Wort | Token | Wort | Token | Wort |
|---|---|---|---|---|---|
| 1 | PRINT | 10 | NEXT | 17 | THEN |
| 2 | LET | 11 | INPUT | 20 | TO |
| 3 | IF | 12 | REM | 21 | STEP |
| 4 | GOTO | 13 | END | 22 | RND |
| 5 | GOSUB | 14 | RUN | 23 | ABS |
| 6 | RETURN | 15 | LIST | 24 | SAVE |
| 7 | FOR | 16 | NEW | 25 | LOAD |
| | | | | 26 | PLOT |
| | | | | 27 | CLS |

Alle Werte sind oktal. Im Speicher steht ein Token als 200 + Nummer.

---

## 6. Aufbau des Interpreters

### 6.1 Ladevorgang in zwei Stufen

`macro1_1` erzeugt nur 12-Bit-Adressen und kann deshalb Bank 1–3 nicht direkt
auf ein Band schreiben. Das Ladeband besteht daher aus zwei Teilen:

1. **Blockband für Bank 0**, von `macro1_1` erzeugt. READ-IN lädt den Bootstrap
   nach 7751–7777. Der Bootstrap lädt den Interpreter und `xld`. Das letzte
   Wort springt nach `xld` (7700).
2. **X-Blöcke für Bank 1–3**, von `xtape.py` angehängt. `xld` schaltet mit
   `eem` in den Extend Mode und liest die Blöcke:

   ```
   n  adresse16  daten[n]  prüfsumme      (XOR über n, Adresse und Daten)
   ```

   Ein Wort `0` beendet das Band, danach springt `xld` zum Start bei 0100.

   Bei einem Prüfsummenfehler hält die Maschine bei 7733 an. CONTINUE liest den
   nächsten Block.

Die Quellen für Bank 1 werden bank-relativ übersetzt (0000–7777). `xtape.py`
verschiebt sie beim Anhängen um 10000. Bank 0 kennt die Bank-1-Adressen über
Gleichungen wie `kwtab=10400`. `xtape.py --check` vergleicht diese mit dem
Listing von `basic1`.

### 6.2 Aufrufkonventionen

| Art | Aufruf | Ablauf |
|---|---|---|
| ohne Argument | `jsp sub` | `sub, dap ret` legt den Rücksprung in den Ausgang |
| mit Argument | `jda sub` | Argument im AC; `jda` speichert es in `sub` und springt nach `sub+1` |
| rekursiv | `jsp expr` | Rücksprung mit dem Makro `pushm` auf den Software-Stapel, Ausgang über `rexit` mit `popm` und `jmp i` |

Das Ergebnis steht immer im AC. Der PDP-1 hat weder Stapel noch
Indexregister. `pushm` (`dac i sp`, `idx sp`) und `popm` (`law i 1`,
`add sp`, `dac sp`, `lac i sp`) sind Makros, also direkt in den Code
eingesetzt statt aufgerufen. Den Stapelüberlauf prüft nur `factor`, durch das
jede tiefere Verschachtelung läuft. Nur die vier Routinen der Ausdrucksauswertung (`expr`, `aexpr`,
`trm`, `factor`) sind reentrant, alle anderen Unterprogramme nicht. Diese vier
legen auch linke Operanden und Operatoren auf den Stapel. Im Extend Mode
enthält die von `jsp`/`jda` gelieferte Rücksprungadresse die Bankbits, und
`jmp i` springt mit voller 16-Bit-Adresse zurück.

### 6.3 Tokenisierung

1. Nach jeder Eingabe bearbeitet `crunch` die Zeile im Puffer selbst. Das
   Ergebnis ist nie länger als die Eingabe.
   - Schlüsselwörter werden zu Tokens.
   - Zahlen werden in ein Zahlwort umgerechnet (400000 + Wert). Das gesetzte
     Vorzeichenbit unterscheidet es von allen Zeichen und Tokens.
   - Leerzeichen und Tabulatoren entfallen.
   - Kleinbuchstaben werden zu Großbuchstaben.
2. Texte in Anführungszeichen und alles hinter REM bleiben unverändert.
3. Beim Ausführen erkennt der Interpreter ein Token oder eine Zahl mit einem
   einzigen Vergleich. Eine Zahl muss nicht mehr umgerechnet werden, und
   Leerzeichen müssen nicht mehr übersprungen werden.
4. LIST schreibt Tokens mit `prtok` und Zahlen mit `prnum` wieder aus und
   setzt die Leerzeichen um die Schlüsselwörter.

### 6.4 Geschwindigkeit

Gemessen im Emulator, Primzahlprogramm bis 200 ohne Druckausgabe und
`lissajous.bas` bis zur Eingabeaufforderung:

| Stand | Primzahlen | Lissajous |
|---|---|---|
| Text bei jeder Ausführung neu durchsucht (Schritt 2/3) | 74,5 s | – |
| Schlüsselwörter tokenisiert, Zeilenlänge (Schritt 4) | 15,8 s | – |
| mit Grafik (Schritt 6) | 16,0 s | 52,0 s |
| Zahlen und Leerzeichen beim Eintippen, Stapel als Makros, ohne EAE (Schritt 7) | 10,1 s | 43,6 s |
| dasselbe mit EAE | **8,9 s** | **40,1 s** |

Wo die Zeit hingeht, zeigten Stichproben des Befehlszählers über den
Debug-Dienst:

- **Vor Schritt 7:** gut ein Drittel der Zeit für `push`/`pop`/`rexit` und
  knapp ein Fünftel in `skpsp`.
- **Bei der Lissajous-Figur nach Schritt 7:** knapp die Hälfte im
  Nachzeichnen der Anzeigeliste (`dslice`).

Ein BASIC-Befehl kostet trotzdem einige hundert Maschinenbefehle. Gegenüber
Assembler bleibt ein Faktor in der Größenordnung von 100.

### 6.5 Arithmetik

- **Einerkomplement:** Es gibt +0 (`000000`) und −0 (`777777`). Nach jedem
  `cma` folgt `sad m0 / cla`, damit Vergleiche und `sza` stimmen.
- **Überlauf** erkennt das Overflow-Flag, das `szo` abfragt und dabei löscht.
- **EAE-Erkennung:** Beim Start rechnet der Interpreter `3 × 5 / 5` mit
  `mul` und `div`. Ergibt das 3 und überspringt `div` wie vorgesehen den
  Folgebefehl, ist die Multiply/Divide-Option vorhanden (`eae` = 1). Ohne
  Option sind dieselben Befehlscodes die Schrittbefehle `mus`/`dis`, die auf
  jedem PDP-1 harmlos sind; der Test schlägt dann fehl.
- **Multiplikation mit EAE:** `mul`, danach `scr 1s`. Das Produkt steht dann
  als Doppelwort in AC:IO und passt, wenn AC = 0 und IO positiv ist.
- **Division mit EAE:** Dividend als Doppelwort `0 : Wert` mit `scl 1s` in die
  EAE-Lage bringen, dann `div`. Der Quotient steht im AC, der Rest im IO
  (verwendet für `RND`).
- **Ohne EAE:** Multiplikation durch Schieben und Addieren. Division:
  den Divisor verdoppeln, bis er nicht mehr in den Dividenden passt, dann
  stellenweise subtrahieren.
- **Vorzeichen** behandelt der Interpreter in beiden Fällen getrennt und
  rechnet mit den Beträgen.
- **Vergleich:** Bei verschiedenen Vorzeichen entscheidet das Vorzeichen,
  sonst die Differenz. So kann der Vergleich nicht überlaufen.
- **Dezimalausgabe:** Wiederholtes Abziehen der Zehnerpotenzen 100000 bis 1.

### 6.6 Grafik und Bildauffrischung

- **Zeichnen:** `PLOT` prüft beide Koordinaten (`pchk`), verschiebt sie mit
  `sal 8s` in die oberen 10 Bits und hängt sie an die Anzeigeliste an. Danach
  zeichnet es den Punkt sofort einmal. `sal` füllt mit dem Vorzeichen auf und
  passt damit zum Einerkomplement des Type 30.
- **Nachzeichnen:** `dslice` zeichnet bei jedem Aufruf bis zu 16 Punkte ab
  `dcur` und fängt am Ende der Liste wieder vorne an. Bei leerer Liste kehrt es
  sofort zurück. Aufgerufen wird es an drei Stellen:

  | Aufrufer | Gelegenheit |
  |---|---|
  | `inch` | in der Warteschleife auf einen Tastendruck |
  | `rawch` | während eines Anschlags: `tyo` ohne Warten, dann Auffrischen, bis `cks` die Schreibmaschine wieder als bereit meldet |
  | `runat` | am Anfang jeder Programmzeile, zweimal (32 Punkte) |

- **Verblassen:** `loop`, die Rückkehr zur Eingabe, leert die Anzeigeliste
  mit `clsdl`. Ohne Nachzeichnen verlöschen die Punkte nach etwa einer
  Viertelsekunde von selbst.
- **Punktausgabe:** `dpy` ohne Warten (`720307`) mit Helligkeit 3, der
  hellsten Stufe. Die Schleife braucht pro Punkt etwa 95 µs und damit länger
  als die 50 µs, die der Type 30 für einen Punkt benötigt.

### 6.7 Schreibmaschine und Zeichensatz

- **Gemeinsamer Umschaltzustand:** Der Flexowriter hat einen einzigen
  Umschaltzustand für Tippen und Drucken. Die Variable `tcase` verfolgt ihn.
  Sie wird von `inch` (Eingabe), `rawch` (Ausgabe) und `brk` (verworfene Taste
  bei BREAK) nachgeführt.
- **Zeichenformat:** Intern ist jedes Zeichen 7 Bit groß: Fallbit 100 plus
  FIO-DEC-Code. Leerzeichen, Tabulator, Rückschritt und Wagenrücklauf stehen
  ohne Fallbit.
- **Ausgabeziel:** Bei SAVE leitet `rawch` die Ausgabe auf `punch` um, mit
  eigenem Umschaltzustand. `unpm` stellt danach die Schreibmaschine wieder her,
  auch im Fehlerfall.
- **Leser:** LOAD liest mit `rpa` ohne Warten und fragt den Status mit `cks`
  ab, so dass ein Tastendruck jederzeit abbrechen kann. Ein liegengebliebenes
  Zeichen wird verworfen. Kommt etwa 0,1 s lang nichts, stößt `rdch` den Leser
  neu an. So bleibt LOAD auch nach einem Abbruch nicht hängen.

---

## 7. Dateien und Werkzeuge

| Datei | Inhalt |
|---|---|
| `basic0.mac` | Interpreter und Lader, Bank 0 (MACRO-1, ca. 2300 Zeilen) |
| `basic1.mac` | Meldungen, Schlüsselwörter, Zeichenklassen, Bank 1 |
| `xtape.py` | setzt das Ladeband zusammen, prüft Adressen (`--check`) |
| `bastape.py` | BASIC-Programme zwischen Textdatei und Lochstreifen umwandeln |
| `basic.tape` | fertiges Ladeband des Interpreters |
| `primzahlen.bas` / `.tape` | Beispiel: Primzahlen bis 200, zehn pro Zeile |
| `zahlenraten.bas` / `.tape` | Beispiel: Zahlenraten mit `RND` und `INPUT` |
| `kreis.bas` / `.tape` | Beispiel: Kreis nach Minsky auf dem Type 30 |
| `parabel.bas` / `.tape` | Beispiel: Funktionsgraph mit Achsen auf dem Type 30 |
| `lissajous.bas` / `.tape` | Beispiel: Lissajous-Figur aus zwei gekoppelten Schwingungen |

### Beispielprogramm Zahlenraten

```basic
10 REM ZAHLENRATEN
20 N=RND(100)
30 T=0
40 PRINT "ICH DENKE MIR EINE ZAHL VON 1 BIS 100."
50 INPUT G
60 T=T+1
70 IF G<N THEN PRINT "ZU KLEIN"
80 IF G>N THEN PRINT "ZU GROSS"
90 IF G<>N THEN 50
100 PRINT "RICHTIG NACH", T, "VERSUCHEN"
110 END
```

### Testumgebung

Entwickelt wurde mit dem Emulator `pdp1` aus *blincolnlights*, gestartet
mit `-t`. Er läuft ohne Bildschirm und ohne Panel, mit Schreibmaschine auf
Port 1041 und Debug-Dienst auf Port 1040. Über den Debug-Dienst werden Bänder
eingelegt (`r datei`, `p datei`), READ-IN ausgelöst und Breakpoints auf
Labels aus dem Listing gesetzt. Jeder Entwicklungsschritt wurde dort mit
geladenem Band und echten Tastatureingaben geprüft.

---

## 8. Entwicklungsgeschichte

### Schritt 1: Gerüst und Lader

- Speicheraufteilung über die vier Bänke festgelegt.
- Zweistufiges Laden mit `xld` und `xtape.py`; der Bootstrap bei 7751–7777
  bleibt unangetastet.
- Zeileneingabe mit Rückschritt, Zahlenein- und -ausgabe, `PRINT`, `REM`,
  `NEW`, `END`.
- Software-Stapel als Vorbereitung für die Rekursion.

### Schritt 2: Die Sprache

- Rekursiver Ausdrucksparser mit Vorrang, Vergleichen, `RND` und `ABS`.
- Multiplikation und Division in Software.
- Programmspeicher mit Einfügen, Ersetzen und Löschen von Zeilen.
- `LIST`, `RUN`, `GOTO`, `IF/THEN`, `GOSUB/RETURN`, `FOR/NEXT`, `INPUT`, `LET`.
- Abbruch per Tastendruck.
- Erste lauffähige Programme: Primzahlen und Zahlenraten.

### Schritt 3: Anpassungen nach dem ersten Einsatz auf dem PiDP-1

- **Malzeichen:** Die Schreibmaschinen-Anbindung des Anwenders lieferte für
  `*` ein `x`. `x` und `X` gelten seitdem an Operatorstelle als „mal“.
- **Fehlermeldungen:** Die knappen Meldungen `WHAT?`, `HOW?` und `SORRY`
  wurden durch zehn eindeutige Meldungen ersetzt.

### Schritt 4: Tokenisierung, SAVE und LOAD

- Schlüsselwörter werden beim Eingeben zu Tokens.
- Jede Programmzeile speichert ihre Länge.
- Das Primzahlprogramm wurde dadurch etwa 4,7-mal so schnell.
- SAVE und LOAD im echten FIO-DEC-Bandformat.
- `bastape.py` für den Austausch mit dem PC.

### Schritt 5: Grafik auf dem Type 30

- `PLOT x, y` und `CLS`.
- Anzeigeliste für 1024 Punkte in Bank 3; der Programmspeicher wurde dafür auf
  6143 Wörter verkleinert.
- Ständige Bildauffrischung beim Warten, beim Drucken und während des
  Programmlaufs.
- Die Zeichenklassen-Tabelle zog nach 11600 um, weil die gewachsene
  Schlüsselworttabelle sonst in sie hineingeragt hätte. `xtape.py` hatte die
  Doppelbelegung beim Bandbau gemeldet.
- Geprüft wurde mit Aufnahmen des Bildschirm-Datenstroms (Port 3400): Lage
  der Punkte, Vollständigkeit des Bildes während LIST und während einer
  Rechnung, `CLS`, `NEW`, `?DISPLAY FULL`.

### Schritt 6: Grafik nach dem ersten Einsatz

Die erste Fassung ließ jedes Bild stehen, bis `CLS` oder `NEW` kam. Auch
direkt getippte Punkte blieben, und ein zweiter Lauf eines Programms mit
vielen Punkten endete mit `?DISPLAY FULL`. Das entsprach weder dem Verhalten
eines Type 30 noch der Erwartung beim Testen.

- Die Anzeigeliste gilt jetzt nur noch während des Programmlaufs. Bei der
  Rückkehr zur Eingabe verblasst das Bild.
- `INPUT` akzeptiert eine leere Eingabe (0). Damit kann ein Programm sein Bild
  bis zu einem RETURN stehen lassen.
- Pro Programmzeile werden 32 statt 16 Punkte nachgezeichnet.
- Das Lissajous-Testprogramm wurde für Ganzzahl-Arithmetik hochskaliert.

### Schritt 7: Geschwindigkeit

- Zahlen werden beim Eintippen in Binärwerte umgerechnet.
- Die Multiply/Divide-Option wird erkannt und genutzt.
- Ein Profil der Befehlszähler-Stichproben zeigte danach die eigentlichen
  Bremsen: die Stapelverwaltung und das Überspringen von Leerzeichen. Deshalb
  wurden `push`/`pop` zu Makros, und Leerzeichen entfallen schon beim
  Eintippen.
- Ergebnis: Das Primzahlprogramm läuft in 8,9 statt 16,0 s.
- Nebenwirkung: LIST zeigt die Leerzeichen normiert.

### Gefundene und behobene Fehler

| Fehler | Ursache | Behebung |
|---|---|---|
| `BREAK` griff nicht bei `20 GOTO 20` | Abfrage nur beim Zeilenwechsel, nicht nach Sprüngen | Abfrage in `runat`, das jede Zeile durchläuft |
| nach BREAK `/` statt `?` und `8` statt `>` | verworfene Taste war ein Umschaltcode, `tcase` stimmte nicht mehr | `brk` wertet Umschaltcodes aus |
| LOAD hing nach einem abgebrochenen LOAD | offener Lesevorgang; ein weiteres `rpa` schaltet die Leserkupplung wieder ab | altes Zeichen verwerfen, Leser nach Zeitablauf neu anstoßen |
| Fehlermeldung direkt hinter Ausgabe | kein Zeilenwechsel vor der Meldung | Spaltenmerker `col`, Meldung beginnt immer auf neuer Zeile |

---

## 9. Einschränkungen

- Nur ganze Zahlen von −131071 bis +131071, keine Fließkommazahlen.
- Variablen sind einzelne Buchstaben, es gibt keine Felder und keine
  Zeichenkettenvariablen.
- Eine Anweisung pro Zeile.
- Höchstens ein Vergleich pro Ausdruck, keine logischen Operatoren AND/OR.
- Eingabezeilen haben höchstens 126 Zeichen.
- Schlüsselwörter werden auch ohne Leerzeichen erkannt. `GO TO` mit
  Leerzeichen ist deshalb nicht erlaubt, und eine Buchstabenfolge, die ein
  Schlüsselwort ergibt, wird zu diesem Schlüsselwort.
- Leerzeichen außerhalb von Texten werden nicht gespeichert. LIST zeigt das
  Programm deshalb mit normierten Abständen, nicht genau wie getippt.
- Wer schneller tippt, als der Interpreter verarbeitet, kann Zeichen verlieren,
  etwa beim Einfügen eines ganzen Programms per Zwischenablage. Für längere
  Programme ist LOAD vom Band der vorgesehene Weg.

---

## 10. Ausblick

Mögliche nächste Schritte, für die in Bank 0 noch 2146 Wörter frei sind:

- `LINE x1, y1, x2, y2` zum Zeichnen von Strecken
- `MERGE`, um ein Programm vom Band zum vorhandenen hinzuzufügen
- Trigonometrie mit der Sinus-Routine aus Spacewar!, skaliert auf ganze Zahlen
