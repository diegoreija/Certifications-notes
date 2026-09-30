# 🛠️ NMAP: DESCUBRIMIENTO DE HOSTS, ESCANEO DE PUERTOS Y SCRIPTS NSE
### *Manual Completo y Guía de Referencia Práctica – Certificación eJPT*

---

> **Estructura del Manual:** Organizado cronológicamente según los módulos oficiales de Nmap en TryHackMe. Contiene la explicación técnica profunda, fundamentos de red, sintaxis de comandos, flags TCP, técnicas de evasión de cortafuegos y el uso del motor NSE, todo redactado en texto continuo explicativo sin listas de viñetas para facilitar su consulta directa en exámenes o auditorías.

---

## 🚀 Matriz de Consulta Rápida (Cheat Sheet de Nmap)

| Categoría de Escaneo | Comando Nmap Clave | Mecanismo y Protocolo | Propósito y Aplicación Principal |
| :--- | :--- | :--- | :--- |
| **Descubrimiento de Hosts** | `nmap -sn 192.168.1.0/24` | Ping Sweep (ARP en LAN, ICMP/TCP en WAN) | Identificar hosts activos sin realizar escaneo de puertos. |
| **Escaneo TCP Connect** | `nmap -sT -p- TARGET` | Completa el apretón de manos (SYN-SYN/ACK-ACK) | Escaneo sin privilegios de root; queda registrado en logs. |
| **Escaneo TCP SYN (Stealth)** | `sudo nmap -sS -p 1-1024 TARGET` | Conexión media abierta (SYN $\rightarrow$ SYN/ACK $\rightarrow$ RST) | Escaneo rápido predeterminado; requiere privilegios de root. |
| **Escaneo UDP** | `sudo nmap -sU --top-ports 100 TARGET` | Envía paquetes UDP y espera ICMP Unreachable | Detectar servicios UDP (DNS, SNMP, DHCP, NTP). |
| **Escaneos Sigilosos Anómalos** | `sudo nmap -sN / -sF / -sX TARGET` | Envía paquetes Null, FIN o Xmas (FIN+PSH+URG) | Evadir cortafuegos sin estado en sistemas basados en RFC 793. |
| **Escaneo de Cortafuegos (ACK)** | `sudo nmap -sA -p 80,443 TARGET` | Envía paquetes ACK y analiza respuestas RST | Mapear reglas de cortafuegos e identificar puertos filtrados. |
| **Escaneo con Señuelos (Decoys)**| `sudo nmap -D RND:10 TARGET` | Mezcla la IP real con 10 IPs aleatorias | Enmascarar el origen del escaneo ante analistas e IDS. |
| **Fragmentación de Paquetes** | `sudo nmap -f -p 80 TARGET` | Divide encabezados IP en fragmentos de 8 bytes | Evadir inspección de firmas en cortafuegos simples. |
| **Escaneo Inactivo (Zombie Scan)**| `sudo nmap -sI ZOMBIE_IP TARGET` | Mide incrementos de IP ID en un host inactivo | Escaneo completamente anónimo y suplantado. |
| **Detección de Versión de Servicio**| `nmap -sV --version-intensity 5 TARGET` | Establece conexión e interactúa con el banner | Determinar la versión exacta de la aplicación escuchando. |
| **Detección de Sistema Operativo**| `sudo nmap -O --osscan-guess TARGET` | Compara huella de pila TCP/IP con `nmap-os-db` | Identificar el sistema operativo objetivo y versión de kernel. |
| **Scripts por Defecto y Versión** | `nmap -sC -sV TARGET` | Ejecuta categoría `default` de scripts NSE y `-sV` | Auditoría rápida de vulnerabilidades básicas y servicios. |
| **Escaneo Agresivo Completo** | `nmap -A TARGET` | Combina `-sV -O -sC --traceroute` | Evaluación exhaustiva integral de un solo host objetivo. |

---

## 1. Descubrimiento de Hosts en Vivo con Nmap

### 1.1 Introducción y Arquitectura de Subredes
Network Mapper (Nmap) es la herramienta estándar de la industria para el descubrimiento de activos, mapeo de infraestructura y detección de servicios de red. Creada por Gordon Lyon (Fyodor), Nmap permite a los auditores de seguridad identificar qué equipos están encendidos en una red, qué puertos mantienen abiertos y qué software se encuentra escuchando en ellos. En cualquier auditoría práctica o examen de certificación, el descubrimiento de hosts activos constituye la primera fase operativa antes de lanzar cualquier análisis de puertos.

Para ejecutar un descubrimiento de hosts eficiente, es fundamental comprender la diferencia entre segmento físico y subred lógica. Un segmento de red representa la conexión física entre equipos compartiendo un mismo medio, mientras que una subred IP define la estructura lógica de direccionamiento. La notación CIDR determina el tamaño de la red mediante la máscara de subred. Por ejemplo, una subred `/24` (máscara `255.255.255.0`) abarca 256 direcciones IP totales, de las cuales 254 son asignables a hosts, reservando la primera dirección para la identificación de red y la última para la dirección de difusión o broadcast. En subredes de mayor envergadura como `/16` (`255.255.0.0`), la red puede albergar hasta 65.534 hosts, lo que exige ajustar los parámetros de escaneo de Nmap para evitar tiempos de espera excesivos.

### 1.2 Especificación de Objetivos y Selección de Rango
Nmap ofrece una flexibilidad absoluta para definir el objetivo u objetivos sobre los cuales se lanzará el análisis. Se puede especificar un único host mediante su dirección IP (`192.168.1.100`) o su nombre de dominio FQDN (`objetivo.thm`). Para escanear múltiples objetivos discontinuos, se separan las direcciones por espacios (`192.168.1.1 192.168.1.15 192.168.1.50`). 

Cuando se trabaja con rangos de red dentro de un mismo octeto, se utiliza la notación de guión (`192.168.1.1-50`), o bien la notación CIDR completa para subredes enteras (`192.168.1.0/24`). Nmap también admite combinaciones en múltiples octetos (`10.10.0-255.1-100`). En auditorías de gran escala donde los objetivos se proporcionan en un listado, se utiliza el argumento `-iL lista_objetivos.txt`. Si se requiere excluir ciertos hosts críticos del escaneo, como routers de producción o servidores fuera de alcance, se añade el parámetro `--exclude 192.168.1.1` o se proporciona un archivo de exclusión mediante `--excludefile excluidos.txt`.

### 1.3 Descubrimiento en Capa 2 (ARP Scan)
Cuando el sistema desde el que se ejecuta Nmap se encuentra en la misma subred física que los objetivos (red local o LAN), el protocolo utilizado por defecto para determinar si un host está vivo es ARP (Address Resolution Protocol). La razón de este comportamiento es que las comunicaciones dentro de un mismo segmento Ethernet dependen de las direcciones MAC físicas y no de las direcciones IP.

El escaneo ARP consiste en el envío de peticiones `ARP Request` preguntando qué dirección MAC corresponde a cada IP del rango. Si el host objetivo está encendido, su tarjeta de red responderá inmediatamente con un paquete `ARP Reply`. Este método es el más rápido y preciso en redes locales porque los cortafuegos basados en software del sistema operativo (como Windows Firewall o iptables) no filtran las peticiones ARP de capa de enlace. Nmap fuerza explícitamente el escaneo ARP mediante la opción `-PR`, la cual se activa de forma automática cuando el usuario ejecuta Nmap con privilegios de superusuario (`sudo`) sobre una subred local.

### 1.4 Descubrimiento en Capa 3 (ICMP Scan y Ping Sweep)
Cuando el análisis se realiza a través de un enrutador hacia una subred remota (fuera de la LAN local), el protocolo ARP no puede atravesar el router, por lo que Nmap recurre a sondas de Capa 3 mediante el protocolo ICMP (Internet Control Message Protocol). Para realizar un escaneo de descubrimiento exclusivo sin analizar puertos, se combina la opción `-sn` (anteriormente conocida como `-sP`), la cual le indica a Nmap que ejecute únicamente la fase de comprobación de presencia o Ping Sweep.

En un escaneo ICMP tradicional, Nmap envía tres tipos de solicitudes. En primer lugar, la solicitud de eco ICMP Echo Request (`-PE`, tipo ICMP 8), donde un host activo responde con un ICMP Echo Reply (tipo ICMP 0). Sin embargo, debido a que la gran mayoría de los cortafuegos perimetrales y de host bloquean las peticiones de eco ICMP convencionales, Nmap proporciona sondas ICMP alternativas: la solicitud de marca de tiempo ICMP Timestamp Request (`-PP`, tipo ICMP 13) y la solicitud de máscara de red ICMP Address Mask Request (`-PM`, tipo ICMP 17). Estas sondas alternativas aprovechan configuraciones deficientes donde los cortafuegos bloquean el ping estándar pero permiten responder a consultas de sincronización de tiempo o información de red.

### 1.5 Descubrimiento en Capa 4 (TCP y UDP Ping)
Para superar la restricción de cortafuegos que bloquean por completo todo el tráfico ICMP, Nmap permite enviar sondas de descubrimiento de Capa 4 utilizando los protocolos TCP y UDP. Este enfoque simula el inicio de un servicio estándar para forzar al sistema operativo objetivo a emitir una respuesta que confirme su estado activo.

El escaneo TCP SYN Ping (`-PS`) envía un paquete con la bandera SYN activada a un puerto específico (por defecto el puerto TCP 80). Si el puerto está abierto en el objetivo, este responderá con un paquete SYN/ACK; si el puerto está cerrado, responderá con un paquete RST (Reset). En ambos casos, la recepción de cualquier respuesta TCP confirma categóricamente que el host está encendido. 

Por otro lado, el escaneo TCP ACK Ping (`-PA`) envía un paquete con la bandera ACK activada simulando una conexión ya establecida. Al recibir un paquete ACK sin una conexión previa válida, el sistema operativo objetivo responde de inmediato con un paquete RST para cerrar la anomalía, confirmando su presencia activa y permitiendo atravesar cortafuegos sin estado que dejan pasar tráfico de retorno. 

Finalmente, el escaneo UDP Ping (`-PU`) envía paquetes UDP a puertos inusuales (por defecto el puerto UDP 40125). Si el host objetivo está activo y el puerto UDP no está escuchando, el kernel del sistema responderá con un mensaje de error `ICMP Port Unreachable (Tipo 3, Código 3)`, lo que confirma que el equipo está encendido a pesar de no responder a peticiones ICMP Echo.

### 1.6 Opciones de Configuración y Resolución DNS
Durante la fase de descubrimiento de hosts, Nmap realiza por defecto una consulta de resolución DNS inversa para cada dirección IP identificada como activa, buscando asociar la IP con un nombre de dominio completamente cualificado (FQDN). Aunque esta información es valiosa para la fase de reconocimiento, la resolución de nombres puede ralentizar significativamente el escaneo cuando se analizan subredes de miles de direcciones.

Para optimizar el rendimiento y acelerar el tiempo de ejecución, se utiliza el parámetro `-n`, el cual inhabilita por completo la resolución DNS inversa. Por el contrario, si el objetivo de la auditoría es mapear detalladamente los nombres de dominio internos de la infraestructura, se especifica la opción `-R` para forzar la resolución DNS inversa sobre todas las direcciones IP del rango, incluso sobre aquellas que no hayan respondido a las sondas de descubrimiento. Si se requiere consultar un servidor DNS específico que no sea el predeterminado del sistema operativo, se utiliza la opción `--dns-servers IP_SERVIDOR_DNS`.

---

## 2. Escaneos Básicos de Puertos con Nmap

### 2.1 Puertos TCP y UDP y la Arquitectura de Servicios
Una vez identificados los hosts activos en la red, el siguiente paso operativo consiste en el escaneo de puertos. Mientas que una dirección IP identifica de forma unívoca a un equipo en la red, un puerto TCP o UDP identifica un servicio de red específico que se está ejecutando dentro de ese host. Existen 65.535 puertos para el protocolo TCP y otros 65.535 puertos para el protocolo UDP, divididos convencionalmente en puertos conocidos (0 al 1023), puertos registrados (1024 al 49151) y puertos dinámicos o privados (49152 al 65535).

Un servidor web HTTP se vincula por defecto al puerto TCP 80, mientras que su versión cifrada HTTPS se vincula al puerto TCP 443. Del mismo modo, servicios críticos de administración como SSH escuchan en el puerto TCP 22, FTP en el puerto TCP 21, y DNS en el puerto UDP 53. Un único puerto solo puede ser utilizado por un único proceso de red de forma simultánea en la misma interfaz IP.

### 2.2 Los Seis Estados de Puertos según Nmap
A diferencia de una visión simplista que clasifica los puertos únicamente como abiertos o cerrados, Nmap categoriza los puertos en seis estados precisos para reflejar el comportamiento real de la pila de red y la presencia de dispositivos de seguridad intermedios:

El estado `open` (abierto) indica que una aplicación o servicio de red está escuchando activamente en ese puerto y aceptando conexiones entrantes o procesando peticiones.

El estado `closed` (cerrado) indica que el puerto es plenamente accesible desde la red, pero no hay ningún servicio o aplicación escuchando en él. Al recibir una sonda en un puerto cerrado, la pila TCP/IP del objetivo responde con un paquete RST (en TCP) o un mensaje ICMP Port Unreachable (en UDP).

El estado `filtered` (filtrado) indica que Nmap no puede determinar si el puerto está abierto o cerrado debido a que un cortafuegos, un filtro de paquetes o un dispositivo de seguridad de red está bloqueando las sondas entrantes o impidiendo que las respuestas del objetivo regresen a la máquina del auditor.

El estado `unfiltered` (sin filtrar) indica que el puerto es accesible desde la red, pero Nmap no puede precisar si está abierto o cerrado. Este estado aparece típicamente al utilizar técnicas de escaneo ACK (`-sA`).

El estado `open|filtered` (abierto|filtrado) se asigna cuando Nmap no recibe ninguna respuesta a sus sondas y no puede discriminar si el puerto está abierto y el servicio no respondió, o si un cortafuegos silencioso descartó el paquete de la sonda.

El estado `closed|filtered` (cerrado|filtrado) se utiliza cuando Nmap no puede determinar si el puerto se encuentra cerrado o bloqueado por un filtro.

### 2.3 Banderas TCP y el Apretón de Manos en Tres Vías
El protocolo TCP (Transmission Control Protocol) es un protocolo orientado a conexión que garantiza la entrega ordenada y fiable de datos mediante el uso de encabezados con banderas de control. El encabezado TCP contiene seis bits de control fundamentales: SYN (Synchronize, utilizado para iniciar una conexión y sincronizar números de secuencia), ACK (Acknowledge, utilizado para confirmar la recepción de datos o de un paquete previo), RST (Reset, utilizado para interrumpir o rechazar una conexión de forma abrupta), FIN (Finish, utilizado para cerrar una conexión de forma ordenada), PSH (Push, utilizado para forzar la entrega inmediata de datos al búfer de la aplicación) y URG (Urgent, utilizado para indicar datos prioritarios).

La comunicación en TCP se establece mediante el apretón de manos en tres vías (TCP Three-Way Handshake). En primer lugar, el cliente envía un paquete con la bandera `SYN` hacia el puerto del servidor. Si el puerto está abierto, el servidor responde con un paquete que combina las banderas `SYN/ACK`. Finalmente, el cliente confirma la conexión enviando un paquete `ACK`. En este punto la conexión queda establecida y lista para la transferencia de datos. Si el puerto está cerrado, el servidor responde directamente con un paquete `RST`.

### 2.4 Escaneo TCP Connect (`-sT`)
El escaneo TCP Connect (`-sT`) es la técnica de escaneo más básica e intuitiva. Se basa en utilizar la llamada de sistema nativa `connect()` del sistema operativo para completar íntegramente el apretón de manos en tres vías con el host objetivo.

Cuando Nmap ejecuta un escaneo `-sT`, envía un paquete `SYN` al puerto objetivo. Si recibe `SYN/ACK`, responde con `ACK` completando la conexión y confirmando que el puerto está `open`. Inmediatamente después, Nmap cierra la sesión enviando un paquete `RST` o `FIN/ACK`. La gran ventaja operativa del escaneo TCP Connect es que no requiere privilegios de superusuario (`root`) para ejecutarse, ya que no necesita construir paquetes en bruto (*raw sockets*). Sin embargo, su principal desventaja es que al completar la conexión TCP, el intento de acceso queda registrado indefectiblemente en los archivos de auditoría y logs de la aplicación o del servidor web objetivo, siendo un método altamente ruidoso.

### 2.5 Escaneo TCP SYN o Escaneo Sigiloso (`-sS`)
El escaneo TCP SYN (`-sS`), también conocido como escaneo medio abierto (*Half-Open Scan*) o escaneo sigiloso (*Stealth Scan*), es el método de escaneo predeterminado y más utilizado en Nmap cuando se ejecuta con privilegios de superusuario.

En lugar de completar el apretón de manos de tres vías, el escaneo SYN interrumpe el proceso de conexión justo antes de que se establezca la sesión. Nmap envía un paquete `SYN` al puerto objetivo. Si el puerto está abierto, el servidor responde con `SYN/ACK`. En ese instante preciso, Nmap emite inmediatamente un paquete `RST` para romper la comunicación sin enviar el paquete `ACK` final. Debido a que la conexión TCP nunca llega a completarse, la mayoría de las aplicaciones y servicios de red no registran el intento en sus logs de aplicación. El escaneo SYN es extremadamente rápido, capaz de escanear miles de puertos por segundo, pero requiere ejecutar Nmap con `sudo` para poder fabricar paquetes IP/TCP en bruto a nivel de socket.

### 2.6 Escaneo UDP (`-sU`)
A diferencia de TCP, el protocolo UDP (User Datagram Protocol) es un protocolo no orientado a conexión que no utiliza apretón de manos ni banderas de estado. Por esta razón, el escaneo de puertos UDP (`-sU`) presenta desafíos técnicos particulares y es significativamente más lento que los escaneos TCP.

Para escanear un puerto UDP, Nmap envía un paquete UDP vacío o con un payload específico para el servicio habitualmente asociado a ese puerto (como una consulta de versión DNS para el puerto 53). Si el puerto UDP está cerrado, el sistema operativo objetivo responde con un mensaje de error `ICMP Port Unreachable (Tipo 3, Código 3)`, lo que permite a Nmap clasificar el puerto categóricamente como `closed`. Si el puerto está abierto y el servicio procesa la solicitud, responderá con un paquete de datos UDP, marcando el puerto como `open`. Sin embargo, si el paquete UDP se pierde o el cortafuegos bloquea la respuesta, Nmap no recibe nada y se ve obligado a clasificar el puerto como `open|filtered`. 

Además, los kernels de los sistemas operativos modernos imponen limitaciones de velocidad (*rate limiting*) al número de mensajes de error ICMP que pueden emitir por segundo (por ejemplo, Linux limita la emisión de ICMP a un paquete por segundo), lo que hace que un escaneo UDP completo sobre 65.535 puertos pueda tardar horas si no se limita el alcance.

### 2.7 Ajuste de Rendimiento y Selección de Puertos
Para optimizar el tiempo de ejecución en auditorías con ventanas de tiempo restringidas o en exámenes prácticos, Nmap ofrece controles precisos sobre la selección de puertos y la velocidad de emisión de paquetes.

Por defecto, si no se especifican puertos, Nmap escanea los 1.000 puertos TCP más habituales basándose en su archivo de frecuencias `nmap-services`. Para escanear un puerto específico o una lista de puertos se utiliza la opción `-p` (ejemplo `-p 22,80,443,8080`). Para escanear un rango continuo se especifica `-p 1-1024`. Si se requiere auditar la totalidad de la pila de red, se utiliza el parámetro `-p-`, el cual escanea los 65.535 puertos existentes. Para un análisis ultrarrápido, la opción `-F` escanea únicamente los 100 puertos más frecuentes, mientras que `--top-ports 500` permite definir un número exacto de puertos prioritarios.

La velocidad del escaneo se gestiona mediante las plantillas de tiempo de Nmap, ajustables desde `-T0` hasta `-T5`. Las plantillas `-T0` (Paranoid) y `-T1` (Sneaky) están diseñadas para evadir sistemas de detección de intrusos (IDS) emitiendo sondas con intervalos de varios minutos. La plantilla `-T2` (Polite) reduce la velocidad para no saturar enlaces de red frágiles. La plantilla `-T3` (Normal) es el comportamiento por defecto. La plantilla `-T4` (Aggressive) es la opción recomendada para entornos de laboratorio, CTFs y exámenes, aumentando el paralelismo y reduciendo los tiempos de retransmisión sin perder precisión. La plantilla `-T5` (Insane) envía paquetes de forma extremadamente agresiva, adecuada solo para redes locales gigabit de altísima velocidad.

---

## 3. Escaneos Avanzados de Puertos con Nmap

### 3.1 Escaneos Sigilosos Anómalos (Null, FIN y Xmas)
Los escaneos anómalos o de banderas modificadas aprovechan una sutileza de la especificación RFC 793 del protocolo TCP para determinar el estado de los puertos sin enviar el paquete SYN convencional. Según la norma RFC 793, cualquier segmento TCP recibido en un puerto cerrado que no contenga los bits SYN, RST o ACK activados debe forzar al sistema receptor a responder con un paquete RST. Por el contrario, si el puerto se encuentra abierto, la especificación establece que la sonda anómala debe ser ignorada silenciosamente sin emitir ninguna respuesta.

El escaneo TCP Null (`-sN`) envía un paquete TCP donde todos los bits de control están en cero (sin ninguna bandera activada). 

El escaneo TCP FIN (`-sF`) envía un paquete que contiene únicamente la bandera FIN activada. 

El escaneo TCP Xmas (`-sX`), nombrado así por simular las luces encendidas de un árbol de navidad, envía un paquete con las banderas FIN, PSH y URG activadas de forma simultánea.

En los tres métodos, si el puerto objetivo está cerrado, la pila de red responde con un paquete `RST`, permitiendo a Nmap marcarlo como `closed`. Si el puerto está abierto o filtrado por un cortafuegos, no se recibe ninguna respuesta y Nmap lo clasifica como `open|filtered`. La principal ventaja histórica de estos escaneos era su capacidad para atravesar cortafuegos sin estado (*stateless firewalls*) y sistemas de filtrado sencillos. Sin embargo, estos métodos presentan una limitación insuperable: todos los sistemas operativos Microsoft Windows, así como ciertos dispositivos Cisco y BSD antiguos, no cumplen estrictamente la norma RFC 793 y responden con un paquete RST ante cualquier paquete anómalo, independientemente de si el puerto está abierto o cerrado, haciendo que estos tres escaneos devuelvan falsos positivos masivos sobre plataformas Windows.

### 3.2 Escaneo TCP Maimon (`-sM`)
El escaneo TCP Maimon (`-sM`), desarrollado por el investigador de seguridad Uriel Maimon, es una variante avanzada de los escaneos anómalos. Esta técnica envía paquetes TCP con las banderas FIN y ACK activadas simultáneamente (`FIN/ACK`).

Según la especificación original del protocolo TCP en la RFC 793, un sistema receptor debería responder con un paquete RST ante una sonda FIN/ACK tanto si el puerto está abierto como si está cerrado. Sin embargo, Maimon descubrió que numerosas implementaciones de la pila de red derivadas de BSD poseen un comportamiento anómalo: si el puerto está abierto, la pila de red ignora el paquete FIN/ACK descartándolo en silencio, mientras que si el puerto está cerrado responde con un paquete RST. Esto permite distinguir puertos abiertos en sistemas compatibles. Al igual que ocurre con los escaneos Null, FIN y Xmas, los sistemas Windows modernos responden con RST en todos los casos, neutralizando la eficacia del escaneo Maimon.

### 3.3 Escaneos para Evaluación de Cortafuegos (ACK y Window)
A diferencia de las técnicas orientadas a descubrir puertos abiertos, el escaneo TCP ACK (`-sA`) se utiliza específicamente para mapear la presencia de cortafuegos, determinar si son con estado (*stateful*) o sin estado (*stateless*), y descubrir qué reglas de filtrado están aplicadas.

Cuando se ejecuta un escaneo ACK, Nmap envía paquetes con el bit ACK activado simulando el tráfico de retorno de una conexión establecida. Dado que un paquete ACK nunca puede iniciar una conexión válida, la pila TCP/IP del objetivo responderá de inmediato con un paquete `RST` si el puerto es accesible. Si Nmap recibe el paquete RST, clasifica el puerto como `unfiltered` (sin filtrar), indicando que la sonda logró atravesar el cortafuegos. Por el contrario, si no se recibe respuesta o se recibe un mensaje de error `ICMP Network/Host Unreachable`, el puerto se marca como `filtered`. Si todos los puertos devuelven `unfiltered`, se confirma que el cortafuegos es de tipo sin estado o que no bloquea la entrada de paquetes ACK.

El escaneo TCP Window (`-sW`) utiliza exactamente la misma sonda de paquetes ACK que el escaneo `-sA`, pero examina un detalle adicional en los paquetes RST devueltos: el campo del tamaño de la ventana TCP (*TCP Window Size*). En ciertas implementaciones de sistemas operativos (como algunas versiones de AIX, FreeBSD o OpenBSD), la pila de red devuelve un valor de ventana positivo cuando el puerto está abierto, y un valor de ventana igual a cero cuando el puerto está cerrado. Esto permite diferenciar puertos abiertos de cerrados utilizando únicamente sondas ACK en sistemas compatibles.

Para pruebas avanzadas de evasión, la opción `--scanflags` permite definir combinaciones personalizadas de banderas TCP especificando directamente sus nombres o valores (por ejemplo `--scanflags URGACKSYNFIN` o `--scanflags 0x29`).

### 3.4 Técnicas de Evasión de Cortafuegos e IDS (Spoofing, Decoys y Fragmentación)
Los entornos corporativos modernos emplean sistemas de prevención de intrusos (IPS), Firewalls de Aplicación Web (WAF) y cortafuegos de inspección profunda de estado (SPI) que bloquean los patrones de escaneo automatizados. Nmap integra capacidades nativas para eludir estas defensas.

La suplantación de dirección IP o IP Spoofing (`-S`) permite alterar la cabecera IP de los paquetes enviando el escaneo con la IP de origen de otro equipo. Esta técnica requiere que el auditor pueda monitorear el tráfico de red mediante captura pasiva para leer las respuestas, o bien que se combine con un escaneo ciego. Para forzar el uso de una interfaz de red específica durante el spoofing se añade la opción `-e eth0`.

La técnica de señuelos o Decoys (`-D`) consiste en enviar las sondas de escaneo mezclando la dirección IP real del auditor entre una ráfaga de direcciones IP falsas o señuelos. Por ejemplo, al ejecutar `sudo nmap -D 192.168.1.5,192.168.1.6,ME 192.168.1.100`, el sistema IDS del objetivo registrará peticiones simultáneas desde tres direcciones IP diferentes, imposibilitando que los analistas del SOC identifiquen de inmediato cuál de las tres IPs corresponde al escáner real. Se pueden generar señuelos aleatorios mediante `-D RND:10`.

La fragmentación de paquetes (`-f`) divide los encabezados TCP de las sondas en pequeños fragmentos de datos IP de 8 bytes (o de 16 bytes al usar `-ff`). Al fragmentar los paquetes, los cortafuegos sencillos basados en inspección de firmas no pueden analizar las banderas TCP en el primer fragmento porque los campos del encabezado quedan divididos en paquetes posteriores, logrando evadir el filtro. Para controlar el tamaño exacto de la unidad máxima de transmisión se utiliza la opción `--mtu TAMANIO`, especificando valores múltiplos de 8 (como `--mtu 16` o `--mtu 24`).

### 3.5 Escaneo Inactivo o Zombie Scan (`-sI`)
El escaneo inactivo (`-sI`), también conocido como Zombie Scan o Idle Scan, es la técnica de escaneo de puertos más avanzada y sigilosa existente. Permite realizar un escaneo de puertos completo sobre un objetivo sin enviar un solo paquete IP desde la verdadera dirección IP del auditor, logrando un anonimato absoluto y suplantando la identidad de un equipo tercero inactivo denominado "zombie".

Para que el escaneo inactivo funcione, el equipo zombie debe cumplir dos requisitos estrictos: estar completamente inactivo en la red (para que ningún otro tráfico altere sus contadores de red) y utilizar un algoritmo de asignación de números de identificación IP (IP ID) predictivo de incremento secuencial de uno en uno (común en sistemas Windows antiguos o dispositivos de red embebidos).

El proceso operativo del Zombie Scan consta de tres pasos ejecutados de forma cíclica para cada puerto objetivo:

En el primer paso, el auditor envía un paquete `SYN/ACK` directo al equipo zombie. Al recibir un SYN/ACK sin conexión previa, el zombie responde con un paquete `RST` que contiene su número de identificación IP actual (por ejemplo, `IP ID = 31000`).

En el segundo paso, el auditor envía un paquete `SYN` hacia el puerto del objetivo, pero suplantando la IP de origen con la IP del zombie. Si el puerto del objetivo está `open`, el objetivo enviará un paquete `SYN/ACK` al zombie. Al recibir un SYN/ACK no solicitado, el zombie responderá al objetivo con un paquete `RST`, lo que provocará que su contador interno se incremente en uno (`IP ID = 31001`). Si el puerto del objetivo estuviera `closed`, el objetivo respondería al zombie con un `RST`, el cual es ignorado por el zombie sin incrementar su contador.

En el tercer paso, el auditor vuelve a enviar un paquete `SYN/ACK` al zombie para consultar su número de identificación IP. Si el nuevo valor devuelto es `IP ID = 31002`, significa que el zombie emitió un paquete intermedio hacia el objetivo, confirmando de forma categórica que el puerto del objetivo está `open`. Si el valor devuelto es `IP ID = 31001`, significa que el objetivo no generó respuesta hacia el zombie, confirmando que el puerto está `closed` o `filtered`.

---

## 4. Escaneos Posteriores a Puertos y Scripts NSE con Nmap

### 4.1 Detección de Servicios y Versiones (`-sV`)
Descubrir que un puerto TCP está abierto representa solo la mitad del trabajo de reconocimiento. Para identificar vulnerabilidades explotables en una auditoría, es imprescindible determinar qué aplicación específica y qué versión exacta se está ejecutando en ese puerto. La opción `-sV` habilita el motor de detección de servicios y versiones de Nmap.

Cuando se especifica la opción `-sV`, Nmap no se limita a asumir el servicio asociado por defecto al número de puerto (como asumir que el puerto 22 es SSH). En su lugar, establece el apretón de manos TCP en tres vías, se conecta al puerto abierto e interactúa con la aplicación mediante una serie de sondas de consulta definidas en su archivo de firmas `nmap-service-probes`. Nmap analiza el banner de bienvenida devuelto por la aplicación, los encabezados HTTP o las respuestas a comandos de prueba para extraer el nombre exacto del software y su número de versión (por ejemplo, transformando la lectura genérica `22/tcp open ssh` en `22/tcp open ssh OpenSSH 8.9p1 Ubuntu 3ubuntu0.6`).

Debido a que la detección de versión exige entablar una comunicación completa a nivel de aplicación, el uso del parámetro `-sV` obliga a Nmap a completar el apretón de manos TCP, por lo que no es posible combinar un escaneo sigiloso medio abierto (`-sS`) con la extracción de versiones sin completar la conexión. La intensidad de las sondas de versión se controla con `--version-intensity NIVEL` en una escala del 0 al 9. La opción `--version-light` fija una intensidad de nivel 2 para escaneos rápidos, mientras que `--version-all` fija el nivel 9 ejecutando todas las sondas de prueba disponibles para asegurar la identificación de servicios altamente personalizados.

### 4.2 Detección de Sistema Operativo (`-O`)
La identificación del sistema operativo objetivo es crucial para seleccionar los exploits adecuados durante la fase de explotación. La opción `-O` activa la huella digital de sistema operativo (*OS Fingerprinting*) en Nmap.

El motor de detección de sistema operativo envía hasta cinco sondas TCP, UDP e ICMP diseñadas con combinaciones específicas de opciones de encabezado, tamaños de ventana, valores TTL y secuencias de números de inicio de secuencia (ISN). La pila TCP/IP de cada sistema operativo (Linux, Windows Server, FreeBSD, macOS, Cisco IOS) procesa y responde a estas sondas anómalas de forma única debido a diferencias en el código fuente de sus kernels. Nmap analiza la respuesta recibida, genera una firma o huella digital y la compara contra su base de datos de miles de sistemas conocidos (`nmap-os-db`).

Si la respuesta no coincide al 100% con ninguna firma registrada, Nmap proporciona una estimación porcentual de similitud. Para mejorar la precisión del análisis, la opción `--osscan-limit` restringe la detección de sistema operativo únicamente a objetivos que tengan al menos un puerto abierto y un puerto cerrado, lo que garantiza disponer de las respuestas de control necesarias. Si se requiere forzar una adivinación agresiva del sistema operativo ante firmas ambiguas, se añade el parámetro `--osscan-guess` o `--max-os-tries` para aumentar los intentos de prueba.

### 4.3 Rastreo de Ruta Integrado (Traceroute)
Para determinar la topología física de la red y conocer el número de routers intermedios que separan la máquina del auditor del objetivo final, se utiliza el parámetro `--traceroute`.

A diferencia de la herramienta tradicional `traceroute` del sistema operativo que envía paquetes UDP o ICMP aislados, el rastreo de ruta integrado de Nmap utiliza la mejor sonda TCP identificada durante el escaneo previo. Nmap envía paquetes con valores del campo Tiempo de Vida (TTL) incrementales empezando desde TTL=1. Cada router en la ruta decrementa el valor de TTL en una unidad; cuando el TTL llega a cero, el router descarta el paquete y emite un mensaje `ICMP Time Exceeded (Tipo 11, Código 0)` de vuelta al auditor, revelando su dirección IP. El proceso se repite incrementando el TTL hasta alcanzar el host objetivo, permitiendo mapear gráficamente los saltos de red y detectar cortafuegos intermedios.

### 4.4 El Motor de Scripts de Nmap (Nmap Scripting Engine - NSE)
El motor de scripts de Nmap (NSE) es una de las funcionalidades más potentes y versátiles de la herramienta. Permite a los auditores escribir y ejecutar scripts automatizados escritos en el lenguaje de programación Lua para realizar desde tareas avanzadas de reconocimiento y recolección de banners hasta la detección automatizada de vulnerabilidades críticas y la explotación de fallos.

En sistemas basados en Linux como Kali Linux, la colección completa de scripts NSE se almacena en el directorio `/usr/share/nmap/scripts/`. Los cientos de scripts disponibles se clasifican en trece categorías funcionales principales:

La categoría `auth` agrupa scripts diseñados para verificar credenciales de autenticación o eludir controles de acceso en servicios objetivo.

La categoría `broadcast` ejecuta descubrimiento de hosts y servicios escuchando peticiones de difusión en la red local.

La categoría `brute` realiza ataques de fuerza bruta contra formularios web y servicios de red (SSH, FTP, Database) para descubrir contraseñas válidas.

La categoría `default` contiene scripts optimizados, rápidos y seguros que se ejecutan automáticamente al utilizar la opción `-sC`.

La categoría `discovery` realiza una enumeración profunda de información en servicios activos (como listar recursos compartidos SMB o mapas de sitios web).

La categoría `dos` comprueba si un servicio es vulnerable a ataques de denegación de servicio.

La categoría `exploit` contiene scripts diseñados para aprovechar activamente vulnerabilidades conocidas y obtener acceso.

La categoría `external` consulta servicios de bases de datos públicas en internet (como registradores WHOIS o VirusTotal).

La categoría `fuzzer` envía campos malformados para descubrir desbordamientos de búfer o fallos de software.

La categoría `intrusive` contiene scripts agresivos que pueden saturar servicios o ser detectados por el SOC.

La categoría `malware` busca evidencias de infecciones por troyanos, puertas traseras o gusanos en el objetivo.

La categoría `safe` agrupa scripts no destructivos que no generan fallos de sistema en los servicios auditados.

La categoría `vuln` analiza los servicios en busca de vulnerabilidades conocidas (CVEs) registradas en bases de datos de seguridad.

### 4.5 Ejecución Práctica y Argumentos de Scripts NSE
Para ejecutar la categoría de scripts por defecto de forma rápida, se utiliza la opción `-sC` (o su equivalente `--script=default`). Para ejecutar un script individual específico se pasa su nombre al argumento `--script` (por ejemplo `nmap --script=banner TARGET` para extraer banners de bienvenida).

Es posible ejecutar categorías enteras o conjuntos de scripts mediante patrones de comodín (por ejemplo `nmap --script="http-*" TARGET` para ejecutar todos los scripts de auditoría web, o `nmap --script=vuln TARGET` para lanzar un escaneo completo de vulnerabilidades CVE). 

Muchos scripts NSE requieren parámetros de entrada para personalizar su comportamiento, tales como credenciales, rutas de archivos o comandos personalizados. Los parámetros se pasan mediante el argumento `--script-args` utilizando parejas clave-valor (por ejemplo `nmap --script http-fileupload-exploiter --script-args http-fileupload-exploiter.path=/uploads TARGET`).

Por último, para facilitar auditorías exhaustivas sobre un único objetivo, Nmap ofrece la opción de escaneo agresivo `-A`. Especificar `-A` es un atajo directo que combina simultáneamente la detección de versiones de servicios (`-sV`), la huella digital de sistema operativo (`-O`), la ejecución de la categoría de scripts por defecto (`-sC`) y el rastreo de rutas de red (`--traceroute`), proporcionando una radiografía técnica completa del objetivo en un solo comando.
