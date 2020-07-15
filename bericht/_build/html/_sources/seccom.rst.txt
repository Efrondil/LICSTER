.. role:: raw-html-m2r(raw)
   :format: html


.. Secure Communication
.. Raphael Hausmanninger, Muhammet Bilbey

Secure Communication
====================

Als die Secure Communication Gruppe haben wir uns ausschließlich mit der internen Kommunikation des LICSTERs auseinandergesetzt. Als Ziel haben wir es uns gesetzt das bereits implementierte Modbus-Protokoll, das in dieser Form sich als sehr unsicher bewiesen hat, zu verschlüsseln. Dabei haben wir uns dazu entschieden, das TLS-Protokoll einzusetzen. Dieses bietet zusätzlich zu der Verschlüsselung der Anwendungsschicht ein Handshake-Protokoll, das die Authentifizierung der Kommunikationspartner ermöglicht.

.. figure:: ./assets/diagrams/No_Secure_Layer_Overview.png
   :alt: 

   Veranschaulichung der ursprünglichen Kommunikation

Dies wurde realisiert, indem wir eine zusätzliche Softwareschnittstelle (Secure Layer) im PLC implementiert haben. Diese ermöglicht eine beidseitig verschlüsselte Kommunikation zwischen dem PLC und den Remote-IOs, indem sie eingehende Pakete von den Remote-IOs entschlüsselt und von dem PLC ausgehende Pakete verschlüsselt. Die interne Kommunikation im PLC zwischen dem Secure-Layer und dem OpenPLC findet unverschlüsselt statt.

.. figure:: ./assets/diagrams/Secure_Layer_Overview.png
   :alt: 

   Veranschaulichung mit Secure Layer

Secure Layer
------------

Der Secure Layer (sichere Schicht) ist die Komponente die zwischen den Remote-IOs und OpenPLC sitz.

.. figure:: ./assets/diagrams/Secure_Layer_Details.png
   :alt: 

   Detaillierte Veranschaulichung des Secure Layers

Der Secure Layer ist in Python3.6 geschrieben und in 2 Hauptkomponenten zu unterteilen:


* `Bridge <#bridge>`_
* `Bridgemanager <#bridgemanager>`_

Bridge
^^^^^^

Die Bridge (Brücke) besteht aus zwei TCP Verbindungen, eine TLS Verbindung zu einem Remote TCP Server, und einen lokalen TCP Server der auf eine lokale Verbindung wartet.\ :raw-html-m2r:`<br>`
Um dies zu ermöglichen muss die Bridge ein wenig Modbus verstehen, da das Message-Framing in diesem Fall durch Informationen im Header vom Modbus Protokoll vorhanden ist.

Bridgemanager
^^^^^^^^^^^^^

Der Bridgemanager (Brückenverwalter) verwaltet die einzelnen Brücken und startet diese bei Fehlern neu.

Remote-IO
---------


Mbed TLS
^^^^^^^^

Um TLS auf den Remote-IOs zu implementieren haben wurde die Mbed TLS Bibliothek gewählt, da diese kompakt, portabel und einfach verständlich ist. Hierbei handelt es sich um ein Open-Source Projekt, welches auf GitHub (\ `https://github.com/ARMmbed/mbedtls <https://github.com/ARMmbed/mbedtls>`_\ ) zu finden ist. Ein weiterer Grund der für Mbed TLS gesprochen hat ist die Tatsache, dass es innerhalb STM32CubeMX konfiguriert werden kann, und dadurch viel Zeit beim konfigurieren der Bibliothek gespart werden kann.

Heap Speicher
^^^^^^^^^^^^^

FreeRTOS verwaltet den gesamten Heap Speicher, darum schlagen Aufrufe der Standardbibliotheksfunktionen für dynamische Speicherverwaltung fehl. Das heißt jeder Aufruf an ``calloc`` oder ``malloc`` geben ``NULL`` zurück. Jedoch benötigt Mbed TLS dynamischen Speicher. Explizit benötigt Mbed TLS ``calloc`` und ``free``. Alternativ kann Mbed TLS auch mit einem statischen Block an Speicher arbeiten, jedoch wurde den dynamischen Ansatz gewählt. Aufgrund der schwer vorhersehbaren Speicheraufwand wurde dieser Ansatz gewählt, da mit dem statischen Block eine vordefinierte Menge an Speicher reserviert wird und dieser meist zu groß oder zu klein ist.

Diese Implementation ist in ``heap_mem.h`` deklariert und in ``heap_mem.c`` definiert.

``heap_mem.c`` beinhaltet Folgenden Quellcode:

.. code-block:: c

   #include "heap_mem.h"
   #include "cmsis_os.h"

   extern SemaphoreHandle_t alloc_mutex;

   void* rmalloc(size_t size)
   {
       void* ptr = NULL;

       if(size > 0)
       {
           while (xSemaphoreTake(alloc_mutex, 10) != pdTRUE);
           ptr = pvPortMalloc(size);
           xSemaphoreGive(alloc_mutex);
       }
       return ptr;
   }

   void* rcalloc(size_t num, size_t size)
   {
       size_t total = num * size;
       void* ptr = NULL;

       if(total > 0)
       {
           while (xSemaphoreTake(alloc_mutex, 10) != pdTRUE);
           ptr = pvPortMalloc(total);
           xSemaphoreGive(alloc_mutex);
           for(size_t i = 0; i < total; i++)
               ((uint8_t *) ptr)[i] = 0;
       }
       return ptr;
   }

   void rfree(void* ptr)
   {
       if(ptr)
       {
           while (xSemaphoreTake(alloc_mutex, 10) != pdTRUE);
           vPortFree(ptr);
           xSemaphoreGive(alloc_mutex);
       }
   }

Um diese problemlose Nebenläufigkeit in unserer eigenen Implementierung zu gewährleisten werden alle kritischen Vorgänge mit einem Mutex, bzw. Semaphore abgesichert. Der Code der das gewährleistet ist Folgender:

.. code-block:: c

   /* 
    * Warte bis die Lock von dem Mutext genommen werden kann,
    * d.h. bis es sicher ist das kein (anderer) Thread
    * die Lock vom Mutex besitzt.
    */ 
   while (xSemaphoreTake(net_mutex, 10) != pdTRUE);

   /*  
    * Gebe die Lock von dem Mutext ab,
    * sodass sie von einem (anderen) genommen werden kann.
    */ 
   xSemaphoreGive(net_mutex);

Mbed TLS stellt die folgende Funktion bereit um eigene Implementationen der ``calloc`` und ``free`` Funktionen innerhalb Mbed TLS zu verwenden:  

.. code-block:: c

   int mbedtls_platform_set_calloc_free( void * (*calloc_func)( size_t, size_t ),
                                         void (*free_func)( void * ) );

Jedoch gilt zu beachten, dass diese Funktion nur aufgerufen werden kann wenn ``MBEDTLS_PLATFORM_C`` und ``MBEDTLS_PLATFORM_MEMORY`` definiert sind! Diese wurden in unserem Fall über STM32CubeMX konfiguriert.

Diese Funktion wird in ``mbedtls.c`` aufgerufen:  

.. code-block:: c

   #include "mbedtls.h"
   #include "mbedtls/platform.h"
   #include "heap_mem.h"

   void MX_MBEDTLS_Init(void)
   {
       mbedtls_platform_set_calloc_free(rcalloc, rfree);
   }

Network Stack
^^^^^^^^^^^^^

Um Mbed TLS in den aktuellen LWIP Network Stack einzubringen mussten einige Anpassungen gemacht werden.

LWIP verwendet für die Adressierung der Sockets einen Index.
Dieser Index wird standardmäßig in Mbed TLS nicht richtig adressiert, was dazu führt das alle ``mbedtls_net_context`` auf den gleichen Socket in LWIP verweisen. Diese Socketverwaltung musste somit selbst implementiert werden.
Dies wurde durch ein einfaches Array umgesetzt:  

.. code-block:: c

   char socks[MEMP_NUM_NETCONN] = {0};

``MEMP_NUM_NETCONN`` beschreibt hierbei die Präprozessor für die maximale Anzahl an simultanen Netzwerkverbindungen.\ :raw-html-m2r:`<br>`
Wenn ein Socketindex aktiv, bzw. reserviert ist wird dieser auf eine Wert der nicht ``0`` ist (in diesem Fall wird er auf ``1`` gesetzt).

``net_sockets.c``\ :  

.. code-block:: c

   void mbedtls_net_init( mbedtls_net_context *ctx )
   {
       while (xSemaphoreTake(net_mutex, 10) != pdTRUE);
       if(!lwip_initialized)
       {
           MX_LWIP_Init();
           lwip_initialized = 1;
       }
       ctx->fd = -1;
       for(int i = 0; i < MEMP_NUM_NETCONN; i++)
       {
           if(socks[i] == 0)
           {
               ctx->fd = i;
               socks[i] = 1;
               break;
           }
       }
       xSemaphoreGive(net_mutex);
   }

Beim freigeben eines ``mbedtls_net_context`` wird dessen Socketindex auch wieder auf ``0`` gesetzt, somit ist dieser wieder frei von einem anderen Socket benutzt zu werden.

.. code-block:: c

   void mbedtls_net_free( mbedtls_net_context *ctx )
   {
       if( ctx->fd == -1 )
           return;
       while (xSemaphoreTake(net_mutex, 10) != pdTRUE);
       socks[ctx->fd] = 0;
       xSemaphoreGive(net_mutex);
       shutdown( ctx->fd, 2 );
       close( ctx->fd );

       ctx->fd = -1;
   }

Sowohl beim initialisieren als auch beim freigeben wurde Nebenläufigkeit berücksichtig. Um diese problemlose Nebenläufigkeit in unserer eigenen Implementierung zu gewährleisten werden alle kritischen Vorgänge mit einem Mutex, bzw. Semaphore abgesichert. Der Code der das gewährleistet wird unter `Heap Speicher <#heap-speicher>`_ erläutert.

Modbus
^^^^^^

Um die TLS Implementierung optional zu halten wurde viel mit Präprozessoren gearbeitet. Wenn eine bestimmte Präprozessor definiert wird werden bestimmte Sektionen an Code ausgeführt, dadurch kann TLS einfach an- bzw. abgeschaltet werden. Zur Veranschaulichung wie das konkret in Code funktioniert dient folgendes Beispiel:

.. code-block:: c

   #ifdef USE_TLS
   // Code in diesem Bereich wird nur ausgeführt wenn USE_TLS definiert ist.
   #else
   // Code in diesem Bereich wird nur ausgeführt wenn USE_TLS *nicht* definiert ist.
   #endif

Konkret wird diese Präprozessor über die Makefile gesetzt, lediglich nur wenn ``make`` mit ``config=tls`` aufgerufen wird.

Generell hat sich strukturell nicht viel geändert zur ursprünglichen Modbus Implementation, es wurden lediglich LWIP Funktionen mit denen von Mbed TLS ersetzt, und wenn ``USE_TLS`` definiert ist wird zusätzlich der TLS Handshake durchgeführt.

Zertifikate
-----------

In ``./tools/`` wurde ein Bash Skript mit dem Namen ``create_new_certs_with_ca.sh`` erstellt.
Dieses Skript erstellt eine CA, sowie alle benötigten Zertifikate.
Zur Erstellung dieser Daten werden von Mbed TLS bereitgestellte Programme verwendet (\ ``gen_key`` und ``cert_write``\ ). Diese sind als Sourcecode auf GitHub zu finden: 
`https://github.com/ARMmbed/mbedtls/tree/development/programs <https://github.com/ARMmbed/mbedtls/tree/development/programs>`_

Im Anschluss werden die erstellten Zertifikate mit der CA signiert.
Die CA (Certificate Authority) und ihre signierten Zertifikate werden in Ordnern des `Secure Layers <#secure-layer>`_ gespeichert.
Daraufhin werden die für die Remote-IOs benötigten Schlüssel und Zertifikate in eine Makefile exportiert, wodurch beim Bauen der Remote-IO Binaries diese alle benötigten Informationen erhalten.

Fazit und Ausblick
------------------


.. raw:: html

   <!-- evtl. überarbeiten -->


Durch die zusätzlichen Implementierungen kann nun optional zwischen der ursprünglich unverschlüsselten Modbus Verbindung und der durch das TLS-Protokoll verschlüsselten Verbindung ausgewählt werden. Zusätzlich zu der verschlüsselten Verbindung übernimmt das Protokoll auch die Überprüfung der Authentizität der Kommunikationspartner. So muss bei einem Verbindungsaufbau das Remote-IO mit einem Zertifikat belegen, dass dieser dem LICSTER-Netzwerk zugehörig ist. 

Nach der durch die Verschlüsselung der Kommunikationswege zwischen dem PLC und der Remote-IOs erzielten Sicherheit kann an der Beschleunigung des TLS-Handshakes gearbeitet werden. Durch verwenden eines Secure Elements kann der momentan sehr langsame Verbindungsaufbau von etwa 10 Sekunden beschleunigt werden. Solch ein Microchip würde zusätzliche Sicherheit mit sich bringen, da die Privat Keys dieser unzugänglich sind. Nach der Verbesserung der Performance des Protokolls könnte die sichere Modbus Verbindung auch auf die weiteren Komponenten (HMI und SCADA) des Netzwerkes ausgeweitet werden. Um eine höhere Authentizität im LICSTER-Netzwerk zu erreichen könnte man die Client Authentifizierung derartig erweitern, dass zusätzlich zu den Remote-IOs auch das PLC mithilfe von Zertifikaten seine Zugehörigkeit bestätigen muss.
