Wireshark
---------

Um einen Überblick über die Pakete, die im LICSTER Netzwerk unterwegs sind, zu
bekommen, haben wir, mit Wireshark, den Netzwerkverkehr aufgezeichnet.

Als erstes haben wir pcaps während dem Einschalten, Betrieb und Abschalten von dem Testbed, mithilfe des Mirror Ports erstellt,
um zu sehen was während einem Regulären betrieb auf dem Netzwerk passiert.

.. image:: img/wireshark_normal.png

Als nächstes haben wir pcaps von Angriffen, die wir Durchgeführt haben, aufgezeichnet, um Regeln für unser Intrusion-Detection-System entwickeln zu können.

.. image:: img/wireshark_flood.png

Durchgeführte Angriffe
----------------------

Die Skripte für die Angriffe sind im Offiziellen Github Repository zu finden (https://github.com/hsainnos/LICSTER/tree/master/attacks).

Flooding
........

Flooding ist eine einfache und beliebte Art eines Denial-of-Service Angriffs.
Für den Angriff haben wir hping3 benutzt. Ein Einfaches CLI-tool um Pakete zu versenden.

Hier wird eins der beiden RemoteIO's vom LICSTER Testbed geflutet.

.. code-block::

    $ sudo hping3 --flood 192.168.0.51

Angriff auf das Fließband
.........................

Hier haben wir mit einem kleinen Python-Skript das Fließband vom LICSTER Testbed angegriffen und zum stoppen gebracht.
Ausgeführt wird er mit:

.. code-block::

    $ sudo ./conveyer_belt.py 

Angriff auf die manuelle Kontrolle
..................................

Hier wird, mit einem Python-Skript, die manuelle Kontrolle vom LICSTER Testbed übernommen.
Ausgeführt wird er mit:

.. code-block::

      $ python3 -i client.py
      >>>

Anschließend kann man dann Befehle senden um das Testbed zu steuern.

Snort Regel
-----------

Damit Snort Angriffe/Abnormalitäten erkennen kann, müssen Regeln vorhanden sein, wonach Snort sich richten kann. Darauf Achten sollte man, dass man nicht zu viele Regeln implementiert, denn so kann es passieren, das Snort zu viele Alarme wirft, von denen viele  falsch sind und der echte Alarm untergeht (Man sieht den Angriff vor lauter "Angriffe" nicht mehr). Hat man aber zu wenige Regel implementiert, kann es passieren, dass mögliche Angriffe nicht erkannt werden. Es müssen also so wenig Regeln wie möglich, aber so viele wie nötig implementiert werden um Fehlalarme zu vermeiden und dennoch echte Angriffe erkennen zu können.

Regel schreiben
---------------

Damit man Regeln schreiben kann muss man erst verstehen wie eine Regel
aufgebaut ist. Hier Hilft die offizielle Anleitung
(http://manual-snort-org.s3-website-us-east-1.amazonaws.com/node27.html).

Hier in Kurzfassung:

Eine Regel besteht aus drei Teilen:

- Aktion
- Kopf
- Regeloptionen

Aktion
......

Hier wird angegeben was Snort tun soll, wenn es ein Paket findet, das den
Regel Kriterien entspricht. Es gibt 3 verfügbare Standardaktionen in Snort,
alert, log und pass. Wenn Snort im inline-modus ausgeführt wird, stehen drop,
reject und sdrop zur verfügung.

- ``'alert'`` erzeugt einen Alarm mit der gewählten Alarm Methode und protokolliert dann das Paket
- ``'log'`` das Paket protokollieren
- ``'pass'`` das Paket ignorieren
- ``'drop'`` das Paket blockieren und protokollieren
- ``'reject'`` das Paket blockieren, protokollieren und senden eines TCP-Reset, wenn das Protokoll TCP ist, oder eine ICMP-Port-Unerreichbarkeit Meldung, wenn das Protokoll UDP ist.
- ``'sdrop'`` das Paket blockieren, aber nicht protokollieren

Kopf
....

Dieses Feld steht für das Protokoll, die IP Adresse, die Ports und die
Richtungsanweisung.

**Protokolle**

Es gibt vier Protokolle, die Snort auf verdächtiges Verhalten analysiert: TCP,
UDP, ICMP und IP.

**IP Adressen und Ports**

Der nächste Teil des Regelkopfes befasst sich mit der IP-Adresse und den Port
für eine bestimmte Regel. Man kann das Heimnetzwerk in der Konfigurationsdatei
von Snort festlegen. Das Schlüsselwort any kann zur Definition einer beliebigen
Adresse verwendet werden.

Regeloptionen
.............

Alle Regeloptionen werden durch das Semikolon (;) voneinander getrennt.
Es gibt vier Kategorien von Regeloptionen:

- general
    enthält extra Informationen über die Regel, haben aber keine auswirkung während der Erkennung 
- payload
    diese Optionen schauen in den Packet-Payload rein
- non-payload
    diese Optionen schauen für nicht payload Daten
- post-detection
    diese Optionen sind Regelspezifische trigger, die ausgeführt werden, nachdem eine Regel ausgelöst wird 

.. image:: img/rules.png

Für genauere Regeloptionen schaut man hier am besten nach:
http://manual-snort-org.s3-website-us-east-1.amazonaws.com/node32.html

Unsere Snort Regel
"""""""""""""""""""

HTTP Regel
...........

.. code-block::
  
   alert tcp !$HOME_NET any -> 192.168.0.10 $HTTP_PORTS (msg:"HTTP Get from EXTERNAL to 192.168.0.10"; \
   classtype: bad-unknown; content: "HTTP"; sid 1002000; rev: 1;)

Diese Regel ist dafür da, falls von einem Außenstehenden Netzwerk ein HTTP GET request empfangen worden ist.

.. code-block::
  
   alert tcp !192.168.20 any -> 192.168.30 $HTTP_PORTS (msg:"HTTP Get not from 192.168.0.20 to 192.168.0.30"; \
   classtype: bad-unknown; content: "HTTP"; sid 1002005; rev: 1;)

Hier ähnlich wie bei der vorherigen Regel, nur wird hier der Alarm geworfen, falls das GET request vom Heimnetz, aber nicht vom HMI, kommt.

ICMP Regel
...........

**Portscan**

.. code-block::
  
   alert icmp any any -> 192.168.0.10 any (msg:"Ping nmap Portscan 192.168.0.10"; \
   dsize:0; itype:8; classtype: network-scan; sid:1003000; rev:1;)

ICMP-Fehlermeldungen (Protocol/Port Unreachable) können verwendet werden, um die offenen Ports zu einer IP-Adresse herauszufinden.
Da die Paketgröße 0 ist wird hier ``'dsize'`` auf 0 gesetzt und der ``'itype'`` auf 8, da der Typ 8 für Echo Request steht.

**DoS**

.. code-block::
  
   alert icmp any any -> 192.168.0.10 any (msg:"Ping flood detected 192.168.0.10"; \
   itype:8; count 20, seconds 1; classtype: denial-of-service; sid:1003010; rev:1;)

Standard DoS Ping flood.

**DoS Teardrop**

.. code-block::
  
   alert icmp any any -> 192.168.0.10 any (msg:"ICMP Teardrop attack 192.168.0.10"; \
   fragbits:M; classtype: denial-of-service; sid:1003020;rev:1;)

Teardrop-Angriffe senden Fragmentierte Pakete die nicht wieder zusammengesetzt werden können, das zu einem DoS führen kann. Um den Angriff zu erkennen,
wird hier ``'fragbits'`` auf ``'M'`` für more gesetzt, was heißt dass noch mehr Pakete kommen.

**ICMP Router Discovery**

.. code-block::
  
   alert icmp any any -> 192.168.0.10 any (msg:"ICMP Router Discovery 192.168.0.10"; \
   icode:0; itype:9; classtype: network-scan; sid:1003030; rev:1;)

Ähnlich wie beim Portscan, nur werden hier nach Benachbarten Routern gesucht. ``'itype'`` wird auf 9 gesetzt da es für Router Advertisement steht.

**ICMP Too large packet**

.. code-block::
  
   alert icmp any any -> 192.168.0.10 any (msg:"Large ICMP Packet 192.168.0.10"; \
   dsize:>1500; classtype: denial-of-service; sid:1003040; rev:1;)

Diese Regel ist dafür da, falls zu große ICMP Pakete gesendet werden. ``'dsize'`` ist für die Paketgröße und wurde hier auf größer 1500 gesetzt.

Modbus Regel
.............

**DoS**

.. code-block::
  
   alert tcp any any -> 192.168.0.51 502 (msg:"Modbus threshold violation 51"; threshold: \
   type both, track by_dst, count 60, seconds 1; classtype: successful-dos; sid:1001004;)

Diese Regel erkennt einen Denial-of-Service Angriff über das Modbus.

SSH Regel
.........

**Strange Traffic**

.. code-block::
  
   alert tcp !$HOME_NET any -> 192.168.0.10 22 (msg:"SSH Request from EXTERNAL NET to 192.168.0.10"; \
   content:"SSH"; nocase; offset:0; depth:4; classtype: attempted-user; sid:1000101; rev:1;)

Diese Regel erkennt einen SSH Zugriffs versuch aus einem externen Netz.

**Brute Force**

.. code-block::
  
   alert tcp any any -> any 22 (msg:"SSH Brute Force Attempt"; flow:established, to_server; content:"SSH"; \
   nocase; offset:0; depth:4; detection_filter:track by_src, count 30, seconds 1; classtype: attempted-user; sid:1000201; rev:1;)

Diese Regel erkennt einen SSH Brute Force angriff.

**DoS**

.. code-block::
  
   alert tcp any any -> 192.168.0.10 22 (msg:"SSH DOS against 192.168.0.10"; \
   detection_filter:track by_src, count 50, seconds 1; classtype: denial-of-service; sid:1000301; rev:1;)

Diese Regeln erkennt einen SSH Denial-of-Service angriff.

.. code-block::
  
   alert tcp any any -> 192.168.0.10 22 (msg:"SSH DDOS against 192.168.0.10"; \
   detection_filter:track by_dst, count 500, seconds 1; classtype: denial-of-service; sid:1000306; rev:1;)

Gleich wie oben, nur ist diese Regel für das Erkennen eines Distributed-Denial-of-Service Angriffs zuständig. 