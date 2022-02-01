---
title: "Wiederverwendung von  Geschäftslogik in AMDP Transformationsroutinen"
date: "2020-11-16"
categories: 
  - "amdp"
  - "bw-transformations"
  - "bw4hana"
  - "sqlscript"
---

Heute morgen habe ich den großartigen [Artikel von Lars Breddemann](https://www.lbreddemann.org/separate-business-logic-from-tables-and-avoid-dynamic-sql/) über die Trennung von Business Logik von den darunterliegenden Datenbanktabellen ohne die Verwendung von dynamischem [SQLScript](https://www.brandeis.de/category/sqlscript/) gelesen. Das ist eine Problemstellung, die auch im SAP BW/4HANA immer wieder auftaucht: Die Wiederverwendung von Logik in AMDP Transformationsroutinen.

Lars Beispiel ist sehr klein und ich möchte es anhand eines Szenarios, das ich in der Vergangenheit in einem Projekt erlebt habe, in diesem Artikel etwas ausbauen. Darüber hinaus möchte ich noch einen eleganteren Weg des Aufrufs solcher ausgelagerter Funktion zeigen, der die Anwendung erleichtert. Hier die Problemstellung:

## Das Problem - immer die gleiche Logik auf unterschiedliche Tabellen

Ein Kunde hat Niederlassungen in mehreren Ländern. Die Datenmodelle sind strikt nach Ländern getrennt, d.h. sie wiederholen sich also für jedes Land. Im DSO-Namen findet sich an der 4. und 5. Stelle das ISO-Kürzel für das Land:

Beispiele aus dem Profit Center Accounting:

- `PCADE01` - Deutschland, Salden
- `PCADE02` - Deutschland, Statistische Kennzahlen
- `PCAFR01` \- Frankreich, Salden
- `PCAFR02` \- Frankreich, Statistische Kennzahlen
- `PCAUK01` \- Großbritannien, Salden,
- `PCAUK02` \- Großbritannien, Statistische Kennzahlen
- ...

Bei den Stammdaten sind alle InfoObjekte an ein Merkmal Quellsystem geklammert. Die Merkmalsausprägungen entsprechen eindeutig einem Land.

- `DE100` \- Stammdaten Deutschland
- `FR100` \- Stammdaten Frankreich
- ...

Es gibt natürlich auch andere Ansätze, um mehrere Länder abzubilden. Und ob die hier gewählte Modellierung so optimal ist, will ich in diesem Artikel nicht besprechen. In dem Projekt ging es darum, perspektivisch Datenmodelle für mehr als 20 Länder zu versorgen.

Als Entwickler interessiert mich jetzt vor allem die Anforderung, dass die Businesslogik in den unterschiedlichen Transformationsroutinen exakt gleich sein soll. Natürlich soll aber ein Lookup für ein Datenmodell in Frankreich nur auf die französischen Tabellen und der gleiche Lookup in Deutschland auf die deutschen Tabellen gehen. Und die Stammdaten sollten für einen optimalen Zugriff auch jeweils auf das Quellsystem Frankreich oder Deutschland eingeschränkt werden. In ABAP ist das eine Kleinigkeit, da man in einer SELECT-Anweisung auch nur den Tabellennamen dynamisch angeben kann.

## Der Lösungsansatz

In dem Beispiel von Lars wird das in SQLScript so gelöst:

DO BEGIN
    my\_customers = SELECT customer\_name, location, tx\_date
                   from customers\_a
                   where customer\_name = 'ACME'
                     and location = 'NEW YORK';
                        
    SELECT  
         customer\_name, location, tx\_date
    FROM
        earliest\_customer\_transaction (:my\_customers);
END;

Die Business-Logik ist also in einer Funktion `EARLIEST_CUSTOMER_TRANSACTION` gekapselt, und die Daten werden über eine Tabellenvariable übergeben. Der Aufrufer selektiert dazu aus den geeigneten DB-Tabellen. Das funktioniert sehr gut, es lässt sich aber noch vereinfachen. Und natürlich werde ich wie angekündigt auch noch ein komplexeres Beispiel für die Wiederverwendung von Geschäftslogik in AMDP Transformationsroutinen damit aufbauen.

### IN-Parameter

Betrachten wir zunächst die Eingabeparameter der Prozedur ((Zur Vereinfachung beschränke ich mich hier OBDA auf die Prozeduren. Gleiches gilt immer auch für IN-Parameter von Funktionen)). Die Tabellenparameter definierten eine Struktur: Welche Felder mit welchen Datentypen werden benötigt. Das ist sinnvoll, damit der Programmcode innerhalb der Prozedur statisch geprüft werden kann.

#### Untypisierte Tabellenparameter - eine Sackgasse

Es ist aber nicht zwingend. Man kann seit HANA 2.0 SPS04 auch auf die Typisierung von Tabellenparametern verzichten:

CREATE OR REPLACE PROCEDURE do\_something( IN inTab TABLE(...), 
                                          OUT outTab TABLE(...))
...

Diese untypisierten Tabellenparameter sind aber nur dann sinnvoll, wenn man dynamisch auf die Daten zugreifen will. Also wenn die Spaltennamen erst zur Laufzeit bekannt sind. Dynamisches Coding wollten wir aber, wie in der Einleitung geschrieben, in diesem Artikel nicht betrachten.

#### Teilweise typisierte Tabellenparameter

Oft wird übersehen, dass man als Aufrufer einer Prozedur die Struktur der Tabellenparameter nicht exakt treffen muss. Die **Reihenfolge der Spalten ist hier egal**((Das Buch "Thinking in Sets" von Joe Celco beschreibt das sehr schön. Spaltenreihenfolgen sind nur für den Applikationslayer relevant. Intern spielen sie keine Rolle)). Ebenso sind **zuätzliche Spalten irrelevant**, sie werden vom System nicht beachtet und später vom Optimizer entfernt.

Ein anderer interessanter Aspekt ist die Tatsache, dass man nicht nur Tabellenvariablen in einen Tabellenparameter übergeben kann. Auch die **Namen von Datenbanktabellen**, Views oder anderen Tabellenfunktionen sind ein gültiger Tabellenausdruck und können grundsätzlich in SQLScript verwendet werden. Mit der direkten Angabe des Tabellennamens lässt sich das obige Beispiel von Lars so optimieren:

DO BEGIN

    SELECT  
         customer\_name, location, tx\_date
    FROM
        earliest\_customer\_transaction (customers\_a)
    WHERE customer\_name = 'ACME'
      AND location = 'NEW YORK';
      
END;

Leider gibt es in AMDP eine Einschränkung, so dass doch immer eine Tabellenvariable verwendet werden muss.

#### Die Typisierung als eine Art Interface

In dem Artikel von Lars gefällt mir die Betrachtung der Daten als Instanzen von Tabellendefinitionen. Er schreibt:

"_Tables/views are the data types that developers work with, while records (rows) are the instances of the data types._ "

Wenn wir in diesem Bild bleiben, so beschreiben IN-Tabellenparameter nur etwas, das dem Konzept eines Interfaces in objektorientierten Programmiersprachen entspricht: Es legt fest, welche Eigenschaften müssen die übergebenen Daten mindestens haben müssen. Es können aber auch zusätzliche Spalten vorhanden sein.

### OUT-Parameter

Auch die OUT Tabellenparameter einer Prozedur, aber nicht der `RETURNS` Wert einer Tabellen-Funktion, darf untypisiert sein. Das ist aber nur dann nutzbar, wenn diese Prozedur direkt über die SQL-Schnittstelle vom Application-Layer aufgerufen wird.

Der Versuch, den eine solche Prozedur in einem anonymen Block oder einer anderen Prozedur aufzurufen, führt zu dieser Fehlermeldung:

```
Error: (dberror) [7]: feature not supported: nested call on procedure "MULTI_TAB"."EARLIEST_CUSTOMER_TRANSACTION_PROC" has any table output parameter RESULT
```

Die Rückgabetabellen müssen also vollständig typisiert sein. Es ist nicht möglich, mehr Spalten als definiert zurückzugeben. Das bedeutet viel Arbeit in einer AMDP-Transformationsroutine mit 200 InfoObjects. Denn am Ende einer solchen Prozedur muss eine Projektion auf das erwartete Ausgabeformat erfolgen.

## Wiederverwendung von Logik in AMDP Transformationsroutinen

Wenn wir diese Erkenntnisse jetzt auf die Wiederverwendung von Logik in AMDP Transformationsroutinen mit der obigen Problemstellung übertragen, dann könnte eine Lösung wie folgt aussehen:

### Die gemeinsame AMDP Prozedur

Für alle Länder soll die gleiche Geschäftslogik durchlaufen werden. Diese wird in **einer [AMDP Prozedur](https://www.brandeis.de/blog/amdp-prozeduren/)** definiert. Die Daten werden von außen als Tabellenparameter an diese Prozedur übergeben. Dabei werden pro Tabellenparameter immer nur die tatsächlich verwendeten Spalten definiert.

#### Alle Daten oder nur die dynamischen?

Wenn **alle** DB-Tabellen an die Prozedur übergeben werden, dann hat das einen riesigen Vorteil: Es müssen innerhalb in der Prozedur keine Datenbankzugriffe stattfinden. Alle Daten kommen von aussen. Und damit haben wir etwas, das man in der objektorientierten Welt [Dependency Injection](https://de.wikipedia.org/wiki/Dependency_Injection) nennt: Die Abhängigkeiten werden von außen übergeben. Damit können wir unsere Business Logik auch ganz einfach von ABAP aus mit Unit-Tests testen. Wenn man ABAP Entwickler nach einem Unit-Test für ABAP Transformationsroutinen fragt, wird man häufig ausgelacht! (Ist mir auch schon passiert. ;-) ) Hier ist das jetzt ganz einfach möglich.

Also sollten wir auch Standardtabellen und Stammdatentabellen übergeben, falls diese benötigt werden.

CLASS zcl\_tr\_demo DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

  PUBLIC SECTION.
    INTERFACES if\_amdp\_marker\_hdb.
    TYPES: BEGIN OF gty\_s\_totals,
             sysid    TYPE c LENGTH 6,
             ryear    TYPE n LENGTH 4,
             rbukrs   TYPE c LENGTH 4,
             prctr    TYPE c LENGTH 10,
             balance  TYPE p LENGTH 000009 DECIMALS 000002,
             currunit TYPE c LENGTH 5,
           END OF gty\_s\_totals.
    TYPES gty\_t\_totals TYPE STANDARD TABLE OF gty\_s\_totals WITH DEFAULT KEY.

    TYPES: BEGIN OF gty\_s\_stat\_kyf,
             sysid    TYPE c LENGTH 6,
             ryear    TYPE n LENGTH 4,
             rbukrs   TYPE c LENGTH 4,
             prctr    TYPE c LENGTH 10,
             balance  TYPE p LENGTH 000009 DECIMALS 000002,
             currunit TYPE c LENGTH 5,
           END OF gty\_s\_stat\_kyf.
    TYPES gty\_t\_stat\_kyf TYPE STANDARD TABLE OF gty\_s\_stat\_kyf WITH DEFAULT KEY.

    TYPES: BEGIN OF gty\_s\_pplant,
             sysid TYPE c LENGTH 6,
             plant TYPE c LENGTH 4,
             bukrs TYPE c LENGTH 4,
           END OF gty\_s\_pplant.
    TYPES gty\_t\_pplant TYPE STANDARD TABLE OF gty\_s\_pplant WITH DEFAULT KEY.

    TYPES: BEGIN OF gty\_s\_in\_out\_tab,
             recordmode                     TYPE rodmupdmod, " InfoObject: 0RECORDMODE
             logsys                         TYPE rsdlogsys, " InfoObject: 0LOGSYS
             sysid                          TYPE c LENGTH 6,
"            ...
             fiscper                        TYPE n LENGTH 7,
             fiscvar                        TYPE c LENGTH 2,
"            ...
             balance                        TYPE p LENGTH 000009 DECIMALS 000002,
             record                         TYPE c LENGTH 56,
             sql\_\_procedure\_\_source\_\_record TYPE c LENGTH 56,
           END OF gty\_s\_in\_out\_tab.
    TYPES gty\_t\_in\_out\_tab TYPE STANDARD TABLE OF gty\_s\_in\_out\_tab WITH DEFAULT KEY.

    METHODS: do\_transformation\_pca IMPORTING VALUE(it\_intab)    TYPE gty\_t\_in\_out\_tab
                                             VALUE(it\_totals)   TYPE gty\_t\_totals
                                             VALUE(it\_stat\_kyf) TYPE gty\_t\_stat\_kyf
                                             VALUE(it\_pplant)   TYPE gty\_t\_pplant
                                             VALUE(iv\_sysid)    TYPE char6
                                   EXPORTING VALUE(et\_result)   TYPE gty\_t\_in\_out\_tab.
ENDCLASS.

Wie man an dem obigen Beispiel sieht: Das Typsisieren der ganzen Tabellen kann schnell aufwändig werden. Die Rückgabetabellen muss entweder vollständig der Struktur der Transformationsroutinen entsprechen. Oder sie beschränkt sich auf die in der Routine ermittelten Felder. Dann muss sie aber per Join mit den anderen Daten aus der INTAB abgemischt werden. Dazu später mehr...

Der Aufruf der neuen Prozedur in der AMDP-Transformationsroutine gestaltet sich aber vergleichsweise einfach, auch wenn wir hier die Tabellennamen nicht direkt an die Prozedur übergeben dürfen:

CLASS /BIC/SKCP38MSW2ZH49HEXE5D\_M IMPLEMENTATION.

METHOD GLOBAL\_END BY DATABASE PROCEDURE FOR HDB LANGUAGE SQLSCRIPT OPTIONS READ-ONLY
using ZCL\_TR\_DEMO=>DO\_TRANSFORMATION\_PCA
      /BIC/APCADE0022
      /BIC/APCADE0012
      /BIC/PPLANT      .
      
-- \*\*\* Begin of routine - insert your code only below this line \*\*\*

-- Note the \_M class are not considered for DTP execution.
-- AMDP Breakpoints must be set in the \_A class instead.
lt\_stat\_kyf = select \* from "/BIC/APCADE0022";
lt\_totals = select \* from   "/BIC/APCADE0012";
lt\_plant = select \* from    "/BIC/PPLANT"; 

"ZCL\_TR\_DEMO=>DO\_TRANSFORMATION\_PCA"( it\_intab => :intab,
                                      it\_totals => :lt\_totals,
                                      it\_stat\_kyf => :lt\_stat\_kyf ,
                                      IT\_PPLANT => :lt\_plant,
                                      ET\_RESULT => OUTTAB );
                                  
-- \*\*\* End of routine - insert your code only before this line \*\*\*
ENDMETHOD.
ENDCLASS.

Wichtig ist, dass alle DB-Tabellen in der `USING`\-Klausel mit angegeben werden müssen.

## Same Same, but different - Flexibilität und Änderungsrobustheit

### Flexible Eingabe

In Bezug auf die zu lesenden Datenbanktabellen sind wir mit dem obigen Ansatz schon sehr flexibel. Die Tabellen können sich für die einzelnen Länder unterscheiden, solang die in den Tabellenparametern definierten Spalten vorhanden sind. Da wir die Daten mit SELECT \* aus den Quelltabellen übergeben, kann sich der Typ der Tabellenparameter auch nachträglich noch ändern, ohne das die Transformationsroutinen angepasst werden müssen.

### Entkopplung der Ausgabe

Im obigen Beispiel hat die Prozedur eine Ausgabetabelle `ET_RESULT` mit der exakten Struktur der `OUTTAB` der Transformationsroutinen zurückgegeben. Das spart auf der einen Seite etwas Schreibarbeit, es ist aber wenig flexibel. Was passiert, wenn sich die Länder-DSOs marginal unterscheiden? Zum Beispiel durch ein paar zusätzliche Felder, die nichts mit der implementierten Business-Logik zu tun haben. Oder wenn für alle Datenmodelle ein Feld dazu kommen soll. Dann muss das quasi gleichzeitig für alle Länder erfolgen. Der obige Ansatz ist also weniger flexibel.

Wenn wir uns in der Prozedur aber nur auf die zu ermittelnden Felder beschränken, dann können wir deren Ergebnis mit der originalen `INTAB` mittels `INNER JOIN` abmischen. Für die Join-Bedingung bietet sich das Feld `RECORD` an, das für jeden Datensatz in der `INTAB` einen eindeutigen Wert hat. Durch den `INNER JOIN` können auch in der Prozedur Datensätze ausgefiltert werden.

Das Ergebnis könnte dann so aussehen:

\-- \*\*\* Begin of routine - insert your code only below this line \*\*\*

-- Note the \_M class are not considered for DTP execution.
-- AMDP Breakpoints must be set in the \_A class instead.
lt\_stat\_kyf = select \* from "/BIC/APCADE0022";
lt\_totals = select \* from   "/BIC/APCADE0012";
lt\_plant = select \* from    "/BIC/PPLANT"; 

"ZCL\_TR\_DEMO=>DO\_TRANSFORMATION\_PCA"( it\_intab => :intab,
                                      it\_totals => :lt\_totals,
                                      it\_stat\_kyf => :lt\_stat\_kyf ,
                                      IT\_PPLANT => :lt\_plant,
                                      ET\_RESULT => lt\_result );
                                      
outtab = select it.RECORDMODE,
                it.LOGSYS,
                it.RYEAR,
"               ...
                it.CURTYPE,
                res.FISCPER,
                res.FISCVAR,
                it.CHARTACCTS,
"               ...
                it.CURRUNIT,
                res.BALANCE,
                it.RECORD,
                it.SQL\_\_PROCEDURE\_\_SOURCE\_\_RECORD
           from :intab as it
           inner join :lt\_result as res
           on it.record = res.record ;
-- \*\*\* End of routine - insert your code only before this line \*\*\*

## Der Unit-Test

Und nun noch ein kleines Beispiel für das Testen der Prozedur mit Unit-Tests. Natürlich müssen alle Parameter mit passenden Daten versorgt werden. Das kann sehr elegant mit dem [Konstruktorausdruck `VALUE #`](https://help.sap.com/doc/abapdocu_753_index_htm/7.53/en-US/abenconstructor_expression_value.htm) erfolgen. Die drei Punkte müssen in dem Beispiel nur noch durch geeignete Daten ersetzt werden:

\*"\* use this source file for your ABAP unit test classes
CLASS ltcl\_ DEFINITION FINAL FOR TESTING
  DURATION SHORT
  RISK LEVEL HARMLESS.

  PRIVATE SECTION.
    DATA mo\_cut TYPE REF TO zcl\_tr\_demo.
    METHODS:
      setup,
      first\_test FOR TESTING RAISING cx\_static\_check.
ENDCLASS.

CLASS ltcl\_ IMPLEMENTATION.

  METHOD first\_test.

    mo\_cut->do\_transformation\_pca(
      EXPORTING
        it\_intab    = VALUE #( ( ... ) )
        it\_totals   = VALUE #( ( ... ) )
        it\_stat\_kyf = VALUE #( ( ... ) )
        it\_pplant   = VALUE #( ( ... ) )
      IMPORTING
        et\_result   = DATA(lt\_result)
    ).

    cl\_abap\_unit\_assert=>assert\_equals( act = lt\_result
                                        exp = VALUE zcl\_tr\_demo=>gty\_t\_in\_out\_tab( (  ... ) ) ).
  ENDMETHOD.

  METHOD setup.
    CREATE OBJECT mo\_cut.
  ENDMETHOD.

ENDCLASS.

## Fazit

Die Wiederverwendung von Geschäftslogik in AMDP Transformationsroutinen ist auch in SQLScript gut möglich. Das Vorgehen muss aber gründlich durchdacht sein. Und es erfordert relativ viel Schreibarbeit. Deshalb wird es sich wahrscheinlich erst dann lohnen, wenn mehr als zwei Datenflüsse exakt die gleiche Logik brauchen. Der Overhead für eine flexible Lösung ist größer als im ABAP, da hier mehr typisiert und zugeordnet werden muss. Ein positiver Nebeneffekt: man gewinnt die Möglichkeit, aussagekräftige Unit-Test durchzuführen.

In großen BW/4HANA Projekten mit mehreren identischen Datenflüssen geht kein Weg am Auslagern von Geschäftslogik vorbei. Wenn man zu oft das DRY Prinzip ((Don't Repeat Yourself)) verletzt, dann nimmt man technische Schulden in Kauf. Diese werden dann beim Betrieb und der Wartung des Systems zur Zahlung fällig.

* * *

Wir unterstützen Sie gerne in der Planungsphase Ihres BW/4HANA im Bereich Modellierung. Mit Schulungen, Workshops, Beratung oder durch die Vermittlung von spezialisierten Berater. Sprechen Sie uns gerne an.
