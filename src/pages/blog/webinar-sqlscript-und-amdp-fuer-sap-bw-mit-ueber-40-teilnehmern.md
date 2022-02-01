---
title: "Fragen und Antworten aus dem Webinar SQLScript und AMDP für SAP BW"
date: "2019-09-13"
categories: 
  - "news"
  - "amdp"
  - "bw-transformations"
  - "sqlscript"
---

Am 13.9.2019 hat das Webinar SQLScript und AMDP für SAP BW stattgefunden.

Am Ende der Veranstaltung bestand die Möglichkeit, Fragen zu stellen bzw. während der Veranstaltung sind im Chat unterschiedliche Fragen aufgetaucht.

[![](images/2019-09-16_20-15-18.png)](https://www.brandeis.de/sqlscript-amdp-im-sap-bw/)

Die Aufzeichnung der Veranstaltung und den zugehörigen Foliensatz können  Sie über den Link hinter dem Bild anfordern:

## Kommentare und Antworten zum Chat

Im Nachgang des Webinars habe ich noch die Fragen bzw. Anmerkungen aus dem Chat zusammengetragen, und möchte diese hier für alle Interessierten beantworten bzw. kommentieren.

- Wahrscheinlich zum Punkt, **warum bislang AMDP so wenig genutzt wird:** _"Weil es sehr aufwendig und riskant ist, eine bestehende, laufende ABAP-TRFN auf SQLScript umzustellen!"_

Die vollständige Migration bestehender ABAP Routinen kann aufwändig sein. Da stimme ich voll mit Ihnen überein. Das gilt insbesondere für solche Routinen, die komplexes ABAP Coding enthalten und das über einen langen Zeitraum gewachsen ist. Sicherlich wird man diese Routinen nur dann migrieren, wenn es ein akutes Problem gibt.  
Aber was spricht dagegen, neue Routinen als AMDP anzulegen?  
Und die meisten ABAP Routinen sind eher klein und lassen sich ganz einfach umstellen. Dafür biete ich meinen Analyseworkshop an, damit man hier ein klares Bild bekommt, wo Aufwand und Nutzen in einem guten Verhältnis liegen.  
Das es riskant ist, auf AMDP umzustellen, kann ich nicht bestätigen.

- **Wie sieht es mit re-use und Modularisierung von SQLScript aus?**

In AMDP Routinen kann man andere AMDP Prozeduren aufrufen. Diese müssen aber voll typisiert sein. Das ist ein Nachteil, der hoffentlich demnächst behoben wird. Mit SPS04 der HANA 2.0 sind Tabellenparameter ohne festen Typ möglich, weiter unten sind noch mal Details, insbesondere zu generischen Prozeduren.   
Eine andere Technik der Modularisierung in SQL ist das Anlegen von Views. Oft verwendete Abfragen können so zentral gepflegt werden. Dafür können auch Calculation Views verwendet werden, wobei dies einige Vor- und Nachteile mit sich bringt, die hier den Rahmen sprengen.

- **Da man beim SQLScript auf Datenmengen arbeitet finde ich das Debugging mühsam. Ich finde nur schwer den Datensatz der aus fachlicher oder technischer Sicht Probleme macht. Haben Sie hierzu Tipps / Empfehlungen?**

Meine Empfehlung ist, dass man den Code der Routine in die SQL-Konsole kopiert und dort einen anonymen Block darum baut. Hier ist ein Bild von einer Folie aus meiner Schulung dazu, der gelb hinterlegt Code ist hinzugefügt worden. Dieses Vorgehen hat sich sehr für die Analyse bewährt.

![](images/2019-09-13_15-37-08-1024x578.png)

- **Fehlerbehandlung in AMDP läuft „nicht wirklich gut“, Unterstützung error stack?**

Das ist völlig richtig und da muss ich mich entschuldigen. Diesen Punkt habe ich in den Folien einfach vergessen. Details finden Sie in [SAP Hinweis 2580109](https://launchpad.support.sap.com/#/notes/2580109). Zusammenfassung: Errorstack funktioniert nicht mit HANA Ausführung. Aber soll mit BW/4HANA 2.0 gelöst werden. 

- **Ein möglicher weiterer Nachteil:  Berechtigungskonzept für direkte Entwicklung auf HANA Ebene .. und „Gefahr“ von Zugriffen auf die Objekte die eigentlich im „Hoheitsbereich“ des BW-Schemas / ABAP-Users liegen.**

Das ist **nicht** richtig. Der Zugriff bei der AMDP Entwicklung erfolgt mit einem normalen NetWeaver Benutzer, ganz ohne spezieller HANA Berechtigung oder einen HANA User. Das hatte der Teilnehmer später bei der Live-Demo auch erkannt. Allerdings: Wenn Calculation Views verwendet werden sollen, dann wird natürlich wieder ein HANA Zugriff mit entsprechenden Berechtigungen benötigt. 

- **ABAP Methoden zu AMDP, wie Kursumrechnungen etc… Momentan noch wenig bis gar keine Codeschnipsel vorhanden**

Es gibt nicht wie im ABAP eine riesige Bibliothek an Funktionsbausteinen und Methoden, die genutzt werden können. Das ist richtig. Für die Kurs- und Mengenumrechnung gibt es aber eine SQL-Funktion. Wenn Sie etwas Spezielles suchen, lassen Sie es mich bitte wissen. 

- **Wie lautet die Quelle für die Aussage, dass AMDPs zukünftig dynamische Programmierung erlauben? Und welche konkreten Änderungen möchte SAP hier unternehmen?**

Die Einschränkung von AMDP-Routinen ist Read-Only. In den Releasenotes für SPS04 für HANA 2.0 steht, dass dynamische SQL-Anweisungen auch als Read-Only markiert werden können. Damit müssten diese in ReadOnly AMDPs verwendbar sein. Siehe [Releasenotes](https://help.sap.com/viewer/42668af650f84f9384a3337bcd373692/2.0.04/en-US/a8e63229aab44836941a2605804ac51c.html) und [Doku zu dynamischem SQL](https://help.sap.com/viewer/de2486ee947e43e684d39702027f8a94/2.0.04/en-US/093c4fd307064f838cb582555c187b9e.html).  Darüber hinaus kann SQLScript mit SPS04 auch ANY TYPE Tabellenparameter, [siehe hier](https://help.sap.com/viewer/de2486ee947e43e684d39702027f8a94/2.0.04/en-US/9abb5bede2ac4d47abe3311062a7eb47.html).  Was jetzt noch fehlt ist, dass man auch AMDP Methoden mit Type ANY TABLE definieren kann. Das habe ich bislang nicht in der Doku gefunden. Und leider habe ich kein System mit dem entsprechenden HANA Stand, um das auszuprobieren.  Es fehlt also nur noch ein kleiner Baustein, damit man in SQLScript generische Prozeduren implementieren kann.  

- **Nein, es war sicher eine TRNF mit einer InfoSource dazwischen, und ein Tal hatte ABAP-code, der zweite nichts**

Da ging es um die anschließende Diskussion mit der Partitial Execution auf HANA. Das kann natürlich gut sein, dass es eine gemischte Ausführung mit ABAP und HANA war. Diese ist grundsätzlich möglich, wenn unten HANA und oben ABAP verwendet wird.

Soweit meine Anmerkungen und Antworten zu den Fragen aus dem Chat. Vielen Dank für diese Beiträge und auch für die vielen positiven Rückmeldungen zu dem Webinar.
