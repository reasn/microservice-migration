# Basics

## Anforderungen

 1. Eventual consistency reicht
	> ACID skaliert nicht (-> ???)
"In der Realität gibt es kein ACID. Auf mehrere Gebäude verteilte Mitarbeiter, die schriftlich Anträge bearbeiten sind schon nur noch eventually consistent. Und in mancher Stadtverwaltung nicht mal das"
 2. Infrastructure as Code (+ CI)
 	> Mit manueller Provisionierung zu arbeitsintensiv (-> terraform, ansible, kubernetes) 
 3. Open Mind
 	> Microservices und FP sind keine Regeln für Monolithisches OOP sondern erfordern anderes Denken
"Wenn man sich das erste mal mit funktionaler Programmierung besschäftigt denkt man leicht Funktionen sind ja toll, aber ohne Objekte fehlt doch was, weil man sich mit dem Wissen, das man hat orientiert. Man muss sich erst tief einarbeiten um zu verstehen, dass mit FP bereits die Modellierung anders aussieht, also Fragen anders formuliert werden"


# Ausgangssituation

## Schritt 1 - Auslagern, was sowieso asynchron läuft
Monolith + LCE-Facade + Worker + Persister 

## Schritt 2 - Downstream definieren
'Je weniger Kreise der Datenflussgraph hat, desto leichter verständlich und weniger fehleranfällig ist er' - Eminem
Bild eines Monolithen mit bidirektionalen Verbindungen zur Datenbank
"Verabschiedet euch davon, Daten zentral vorzuhalten und versucht sie eher Schrittweise anzureichern"

Hilfe: Fluss als Metapher, grundsätzliche Richtung der Daten definieren. Datenquellen minimieren

Am Ende des Flusses sitzt der Nutzer

Monolith + LCE-Facade + Broker    ----> Image Processor + Broker
            |				|	
           \/				\/
          Persister + Webapp-DB


(!) Das ist der entscheidenste mentale Schritt: Ich verabschiede mich davon, alle Daten immer zugänglich zu haben und beginne, Services so anzuordnen, dass jeder exakt das konsumiert, was er braucht.

Products            --> Scoring
   |    \                 |     
   |     _|               |
   |        Monolith      |
   |       /             /
   \/   |_              /
Webapp       <----------  


### Schritt 3 - unvermeidbare Kreise

Orders  ---->
 |	/\	<----    Products            --> Scoring
 |	|			   |    \                 |     
 |	|			   |     _|               |
 |	|			   |        Monolith      |
 |	|			   |       /             /
 |	|-------	   \/   |_              /
  ---------->	Webapp       <----------

+ Verbindung der Webapp zu monolithischen DBs

(!) Wir haben verständliche Angst vor Dateninkonsistenzen. Es ist aber meistens leichter und natürlicher, Konfliktlösungsstrategien zu implementieren als monolithen zu skalieren


### Noch mehr unvermeidbare Kreise

              ---->    Products
           <----         /       \
                        /         \
                      |_           _|
Orders   <---- Monolith                 Scoring
 |	/\	                
 |	|			   |                      |   /\ 
 |	|			   |     	              |   |
 |	|			   |				      |   |
 |	|			   |                     /    |
 |	|-------	   \/                   /     |
  ---------->	Webapp       <----------      |
                      \                       /
                        --------> Tracking ---

+ Verbindung der Webapp zu monolithischen DBs

// Möglicherweise auf orders verzchten um Beispiele einfacher zu halten?!


Ziele: Keine geteilten Datenbanken (hart), so wenig synchrone Kommunikation wie möglich (weich)

Was haben wir gewonnen?
 - Asynchrone Microservices müssen nur noch entsprechend der zu verarbeitenden Message-Rate skalieren
 - Alles außer der Webapp darf ausfallen, ohne dass der User es direkt merkt

Horizontale Skalierung- - Last x10000
-> Webapp-Instanzen++ (leicht, da kein gemeinsamer State existiert)
-> Webapp-DB-Instanzen++ (z.B. elasticsearch-Cluster)
-> Tracking: Alternative Skalierungsstrategien, z.B. Bulk-Inserts -> Bedarft steigt nicht linear!!

-> Orders-Instanzen++ (Dank Trennung vom Restsystem viel leichter machbar, Verwendung z.B. von redis)

Warum geht das?
Minimierung von gemeinsamem Datenzugriff und Synchronisierung
