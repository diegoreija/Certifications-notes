<h1>
  <img src="https://assets.tryhackme.com/img/modules/nmap.png" width="65px" align="absmiddle">
  <span>    NMAP: DESCUBRIMIENTO DE HOSTS Y ESCANEO DE PUERTOS</span>
</h1>

### *Guía de Referencia y Explotación Extensiva – Certificación eJPT*

---

> **Estructura del Manual:** Organizado exactamente según las fuentes de Notion de TryHackMe sobre Nmap (**Nmap Live Host Discovery**, **Nmap Basic Port Scans** y **Nmap Advanced Port Scans**). Cada sección contiene explicaciones conceptuales detalladas, fundamentos de red, mecanismos de transmisión de paquetes, técnicas de evasión de cortafuegos y análisis paso a paso en texto corrido sin listas.

---

## Matriz de Consulta Rápida (Cheat Sheet de Nmap)

| Tipo de Escaneo / Función | Parámetro en Nmap | Banderas TCP / Mecanismo | Respuesta Esperada e Interpretación |
| :--- | :--- | :--- | :--- |
| **Descubrimiento ARP** | `nmap -PR` | Consultas ARP de Capa 2 | Respuesta ARP revela MAC e indica host activo en misma subred. |
| **TCP Connect Scan** | `nmap -sT` | SYN $\rightarrow$ SYN/ACK $\rightarrow$ ACK $\rightarrow$ RST | Completa apretón de 3 vías. Único para usuario no privilegiado. |
| **TCP SYN Scan (Sigiloso)** | `nmap -sS` | SYN $\rightarrow$ SYN/ACK $\rightarrow$ RST | Modo por defecto como root. Rompe la conexión tras el SYN/ACK. |
| **Escaneo UDP** | `nmap -sU` | Envío de datagramas UDP | Sin respuesta indica Abierto\|Filtrado; ICMP Tipo 3 Código 3 indica Cerrado. |
| **Escaneo Nulo (Null)** | `nmap -sN` | Ninguna bandera activa (0) | Sin respuesta indica Abierto\|Filtrado; RST indica Cerrado. |
| **Escaneo FIN** | `nmap -sF` | Bandera FIN activa | Sin respuesta indica Abierto\|Filtrado; RST indica Cerrado. |
| **Escaneo Navidad (Xmas)** | `nmap -sX` | Banderas FIN, PSH y URG | Sin respuesta indica Abierto\|Filtrado; RST indica Cerrado. |
| **Escaneo Maimon** | `nmap -sM` | Banderas FIN y ACK | Diseñado para sistemas derivados de BSD. Descartar paquete expone puerto. |
| **Escaneo ACK** | `nmap -sA` | Bandera ACK activa | Devuelve RST siempre. Mapea reglas de firewall (Filtrado vs No Filtrado). |
| **Escaneo de Ventana (Window)** | `nmap -sW` | Bandera ACK + Campo Window | Examina el tamaño de la ventana TCP en respuestas RST para inferir puertos. |
| **Escaneo Personalizado** | `nmap --scanflags` | Combinación manual de banderas | Permite especificar combinaciones arbitrarias como `RSTSYNFIN`. |
| **Suplantación de IP (Spoofing)** | `nmap -e <iface> -Pn -S <IP_falsa>` | Paquetes forzados con IP origen | Envía peticiones usando otra IP; requiere capturar tráfico para ver respuesta. |
| **Señuelos (Decoys)** | `nmap -D <IP1,IP2,ME>` | Mezcla de tráfico desde varias IPs | Oculta la IP real del atacante entre múltiples fuentes falsas. |
| **Fragmentación de Paquetes** | `nmap -f` o `-ff` | Fragmenta el encabezado IP/TCP | Divide datos en bloques de 8 o 16 bytes para evadir inspección IDS/WAF. |
| **Escaneo Inactivo (Zombie)** | `nmap -sI <IP_Zombie>` | Monitorización de secuencia IP ID | Usa un host inactivo para sondear de forma 100% anónima. |
| **Razón del Estado** | `nmap --reason` | Diagnóstico del paquete | Muestra el tipo de paquete recibido (ej. `arp-response`, `syn-ack`). |

---

<br><br>

<h1>
  <img src="https://cdn-images.tryhackme.com/room-icons/62a7685ca6e7ce005d3f3afe-1778939441152" width="40px" align="absmiddle">
  <span> Descubrimiento de Hosts en Vivo</span>
</h1>

### 1.1 Introducción a Nmap y Arquitectura de Red
Nmap, abreviación de Network Mapper, es una herramienta gratuita y de código abierto creada por Gordon Lyon, conocido en la comunidad como Fyodor. Es el estándar absoluto de la industria para mapear redes, identificar sistemas activos y descubrir servicios en ejecución. A través de su motor de scripting, Nmap permite extender sus capacidades desde la simple huella dactilar de servicios hasta la detección y explotación automatizada de vulnerabilidades.

Para comprender adecuadamente el descubrimiento de sistemas, es necesario distinguir entre un segmento de red físico y una subred lógica. Un segmento de red hace referencia a la infraestructura física donde los ordenadores están conectados mediante un medio compartido, como un conmutador Ethernet o un punto de acceso Wi-Fi. Por otro lado, una subred es una división lógica dentro de una red IP conectada a través de un enrutador.

En el direccionamiento IP, la máscara de subred determina la cantidad de hosts que pueden coexistir en un mismo segmento. Por ejemplo, una subred expresada con la notación `/16` posee una máscara `255.255.0.0` y puede albergar aproximadamente 65.000 sistemas. En contraste, una subred con notación `/24` utiliza una máscara `255.255.255.0` y permite un máximo aproximado de 250 hosts activos.

<br>

### 1.2 Métodos de Descubrimiento de Hosts y Protocolos de Capa
Cuando el equipo del auditor se encuentra dentro del mismo segmento físico o subred de capa de enlace que el objetivo, Nmap utiliza por defecto consultas del Protocolo de Resolución de Direcciones, conocido como ARP. Una petición ARP solicita la dirección física MAC vinculada a una dirección IP para permitir la comunicación local. Debido a que un equipo encendido debe responder a las peticiones ARP para funcionar en la red local, recibir una respuesta ARP confirma con absoluta certeza que el host está activo.

Es crucial entender que los paquetes ARP pertenecen estrictamente a la capa de enlace de datos de la arquitectura de red y no se enrutan más allá del enrutador o puerta de enlace predeterminada. Si el objetivo se ubica en una subred diferente, las consultas ARP no cruzarán el enrutador. En tales escenarios entre subredes distantes, Nmap redirige el tráfico a través del enrutador utilizando sondas de capa de red y transporte, como peticiones de eco ICMP, paquetes TCP SYN o ACK a puertos específicos, y datagramas UDP.

<br>

---

<br>

<h1>
  <img src="https://cdn-images.tryhackme.com/room-icons/62a7685ca6e7ce005d3f3afe-1778940381346" width="40px" align="absmiddle">
  <span> Escaneo Básico de Puertos</span>
</h1>

### 2.1 Puertos TCP y UDP y los Seis Estados de Nmap
Así como una dirección IP identifica un equipo único en la red, un puerto TCP o UDP identifica un servicio de red específico que se ejecuta dentro de ese equipo. Por defecto, servicios estándares se asocian a puertos fijos, como el puerto TCP 80 para servidores HTTP web sin cifrar o el puerto TCP 443 para conexiones cifradas HTTPS, aunque los administradores pueden cambiar estos números según sus necesidades. En una misma dirección IP, ningún puerto puede ser escuchado por más de un servicio simultáneamente.

El encabezado de un segmento TCP ocupa un mínimo de 24 bytes según la especificación RFC 793 y contiene los campos de puerto de origen, puerto de destino, número de secuencia, número de acuse de recibo y seis banderas de control de 1 bit. Estas banderas son URG para indicar datos urgentes, ACK para confirmar la recepción de datos, PSH para forzar el procesamiento inmediato en la aplicación, RST para reiniciar o rechazar una conexión, SYN para iniciar el apretón de manos sincronizando números de secuencia, y FIN para finalizar la transmisión.

Para clasificar los puertos analizados, Nmap define exactamente seis estados posibles basados en la respuesta obtenida del sistema y la presencia de dispositivos de seguridad intermediarios:

* El estado Abierto (Open) confirma que hay un servicio escuchando activamente y aceptando conexiones en ese puerto.

* El estado Cerrado (Closed) indica que el puerto es accesible y no está bloqueado por ningún cortafuegos, pero ningún servicio está escuchando en él, por lo que el objetivo responde rechazando la conexión.

* El estado Filtrado (Filtered) significa que Nmap no puede determinar si el puerto está abierto o cerrado porque no puede comunicarse con él. Ocurre cuando un cortafuegos o filtro de paquetes bloquea las sondas salientes de Nmap o impide que las respuestas del servidor regresen al auditor.

* El estado Sin Filtrar (Unfiltered) indica que el puerto es accesible, pero Nmap no puede concluir si existe un servicio escuchando. Este estado es característico de las respuestas obtenidas durante un escaneo de tipo ACK.

* El estado Abierto|Filtrado (Open|Filtered) se asigna cuando Nmap no recibe ninguna respuesta y no puede distinguir si el puerto está abierto sin responder o si un cortafuegos silencioso está descartando los paquetes sin enviar notificaciones.

* El estado Cerrado|Filtrado (Closed|Filtered) indica que Nmap no puede determinar si el puerto está totalmente cerrado o si existe un dispositivo intermedio bloqueando la comunicación.

<br>

### 2.2 Escaneo TCP Connect (`-sT`)
El escaneo TCP Connect, seleccionado mediante el parámetro `-sT`, funciona completando de forma íntegra el apretón de manos TCP de tres vías. La secuencia comienza cuando Nmap envía un paquete con la bandera SYN activa hacia el puerto objetivo. Si el puerto está abierto, el servidor responde con las banderas SYN y ACK activadas. Finalmente, Nmap confirma la conexión enviando un paquete ACK y, habiendo verificado la existencia del servicio, rompe inmediatamente la sesión enviando un paquete RST/ACK.
Si el puerto escaneado está cerrado, el servidor responde directamente con un paquete RST/ACK para indicar que no hay ningún servicio escuchando. Esta secuencia de rechazo se repite para cada puerto cerrado analizado.
El escaneo TCP Connect es el único método de escaneo de puertos TCP disponible para usuarios que ejecutan Nmap sin privilegios administrativos de root o sudo en el sistema operativo, ya que utiliza las llamadas a la API de sockets del sistema operativo subyacente. Por defecto, Nmap analiza los 1000 puertos más comunes. Es posible añadir la opción `-F` para activar el modo rápido que escanea únicamente los 100 puertos más frecuentes, o incluir la opción `-r` para analizar los puertos en orden numérico consecutivo en lugar de en orden aleatorio.

<br>

### 2.3 Escaneo TCP SYN o Sigiloso (`-sS`)
El escaneo TCP SYN, seleccionado con el parámetro `-sS`, es el modo de escaneo predeterminado cuando Nmap se ejecuta con privilegios de usuario root o sudo. Se conoce popularmente como escaneo semiabierto o sigiloso porque rompe la conexión TCP antes de completar el apretón de manos de tres vías.

Durante un escaneo SYN, Nmap transmite un paquete con la bandera SYN. Si el puerto objetivo está abierto, el servidor responde con un paquete SYN/ACK. En ese instante preciso, en lugar de responder con un paquete ACK para cerrar el establecimiento de la sesión como haría una aplicación convencional, Nmap envía inmediatamente un paquete RST para cancelar la conexión. Debido a que la sesión TCP nunca llega a establecerse formalmente a nivel de aplicación, muchas aplicaciones y registros de auditoría tradicionales no registran el intento de conexión, resultando en un análisis mucho más discreto.

En caso de que el puerto escaneado esté cerrado, el comportamiento es idéntico al escaneo TCP Connect: el servidor objetivo responde con un paquete RST/ACK confirmando que no hay ningún servicio escuchando.

<br>

### 2.4 Escaneo UDP (`-sU`)
El protocolo UDP es un protocolo no orientado a conexión que no realiza ningún apretón de manos inicial para establecer una sesión. Esta naturaleza genera una dinámica de análisis completamente diferente a la de TCP. Cuando Nmap envía un datagrama UDP a un puerto abierto, la mayoría de los servicios no devuelven ninguna respuesta a menos que la sonda contenga un payload específico esperado por la aplicación.

Debido a esta falta de respuesta, Nmap marca inicialmente los puertos UDP silenciosos como Abierto|Filtrado. Por el contrario, si un datagrama UDP llega a un puerto cerrado en el objetivo, el sistema operativo genera y devuelve obligatoriamente un mensaje de error ICMP de Tipo 3, Código 3, correspondiente a Puerto Inalcanzable (Destination Unreachable, Port Unreachable).

Por lo tanto, en un escaneo UDP activado con la opción `-sU`, la ausencia total de respuesta sugiere un puerto abierto o filtrado, mientras que la recepción de un error ICMP Tipo 3 Código 3 confirma de manera categórica que el puerto está cerrado. El escaneo UDP puede combinarse en un mismo comando con escaneos de tipo TCP para obtener una auditoría integral de la máquina objetivo.

<br>

### 2.5 Ajuste de Alcance, Tiempos y Rendimiento
Nmap permite definir con precisión el rango de puertos a analizar. Se pueden especificar puertos individuales separados por comas como `-p22,80,443`, rangos específicos como `-p1-1023`, la totalidad de los 65535 puertos mediante `-p-`, o los diez puertos más atacados con `--top-ports 10`.

Para controlar la agresividad y velocidad del tráfico, Nmap ofrece seis plantillas de tiempo predefinidas desde `-T0` hasta `-T5`. La plantilla Paranoid (`-T0`) analiza un puerto a la vez esperando 5 minutos entre el envío de cada paquete, siendo ideal para evadir sistemas de detección de intrusos en entornos altamente supervisados. La plantilla Sneaky (`-T1`) se utiliza frecuentemente en auditorías reales donde el sigilo es prioritario. La plantilla Polite (`-T2`) reduce la velocidad para no saturar el ancho de banda del objetivo. La plantilla Normal (`-T3`) es el comportamiento estándar por defecto. La plantilla Aggressive (`-T4`) acelera la transmisión y es la más utilizada en laboratorios y competiciones CTF. Por último, la plantilla Insane (`-T5`) envía sondas a máxima velocidad a costa de reducir la precisión si la red sufre pérdida de paquetes.

Adicionalmente, se puede limitar el flujo exacto de paquetes por segundo utilizando parámetros como `--max-rate 10` para garantizar que la herramienta nunca supere las diez sondas por segundo, o ajustar la paralelización de las consultas mediante `--min-parallelism 512` para forzar a Nmap a mantener al menos 512 sondas en ejecución concurrente.

<br>

---

<br>

<h1>
  <img src="https://cdn-images.tryhackme.com/room-icons/62a7685ca6e7ce005d3f3afe-1778952120935" width="40px" align="absmiddle">
  <span>Escaneo Avanzado de Puertos y Evasión</span>
</h1>

### 3.1 Escaneos Nulo (`-sN`), FIN (`-sF`) y Navidad (`-sX`)
Los escaneos avanzados de puertos manipulan combinaciones inusuales de banderas TCP para analizar la respuesta de los sistemas y eludir reglas de filtrado de cortafuegos sin estado (stateless firewalls).

El escaneo Nulo (Null Scan), seleccionado con la opción `-sN`, envía paquetes TCP donde ninguno de los seis bits de banderas está activado, enviando un valor de cero. Según la especificación técnica de TCP, si un puerto está abierto y recibe un paquete sin banderas, no debe generar ninguna respuesta. Por lo tanto, la ausencia de respuesta indica a Nmap que el puerto está Abierto|Filtrado. 

Si el puerto está cerrado, el objetivo responde con un paquete RST.
El escaneo FIN, seleccionado mediante `-sF`, transmite un paquete con únicamente la bandera FIN activada. Al igual que en el escaneo nulo, un puerto abierto no genera ninguna respuesta, clasificándose como Abierto|Filtrado, mientras que un puerto cerrado responde devolviendo un paquete RST.

El escaneo Navidad (Xmas Scan), activado con el parámetro `-sX`, recibe su nombre porque los paquetes enviados encienden las banderas FIN, PSH y URG simultáneamente, recordando a las luces de un árbol navideño. La lógica de respuesta es exactamente la misma: los puertos abiertos no responden, mientras que los puertos cerrados devuelven un paquete RST.

Estos tres tipos de escaneo son especialmente eficientes al auditar sistemas protegidos por cortafuegos sin estado. Un cortafuegos sin estado inspecciona únicamente si los paquetes salientes o entrantes contienen la bandera SYN para bloquear intentos de conexión no autorizados. Al enviar combinaciones de banderas distintas a SYN, las sondas atraviesan la regla del firewall e interactúan directamente con el sistema interno. Sin embargo, frente a cortafuegos con estado (stateful firewalls) que rastrean el estado de las sesiones activas, estos paquetes anómalos son descartados automáticamente.

<br>

### 3.2 Escaneos TCP Maimon (`-sM`), ACK (`-sA`) y Ventana (`-sW`)
El escaneo TCP Maimon, denominado así por su descubridor Uriel Maimon y seleccionado mediante `-sM`, envía paquetes con las banderas FIN y ACK activadas simultáneamente. En la mayoría de los sistemas operativos modernos, el objetivo responde con un paquete RST tanto si el puerto está abierto como si está cerrado, imposibilitando la diferenciación. Sin embargo, ciertos sistemas derivados del código histórico BSD descartan silenciosamente el paquete si el puerto está abierto, permitiendo descubrir puertos activos en esas plataformas específicas.

El escaneo TCP ACK, seleccionado con la opción `-sA`, envía paquetes con la bandera ACK activada. Dado que un paquete ACK solo debe enviarse legítimamente para confirmar la recepción de datos en una sesión previa, el objetivo siempre responderá enviando un paquete RST indicando que no existe tal conexión, sin importar si el puerto está abierto o cerrado. Por lo tanto, el escaneo ACK no sirve para descubrir puertos abiertos, sino para mapear reglas de cortafuegos. Si se recibe un paquete RST en respuesta, se confirma que el puerto está Sin Filtrar (Unfiltered) y que el firewall permite el paso del tráfico. Si no se recibe respuesta o se obtiene un error ICMP, el puerto está Filtrado (Filtered) por el cortafuegos.

El escaneo de Ventana TCP (TCP Window Scan), seleccionado con la opción `-sW`, utiliza la misma estructura que el escaneo ACK enviando paquetes con la bandera ACK activa, pero analiza en detalle el valor devuelto en el campo de la Ventana TCP (TCP Window) del paquete RST de respuesta. En ciertos sistemas operativos, los puertos abiertos devuelven un paquete RST con un valor de ventana positivo, mientras que los puertos cerrados devuelven un valor de ventana igual a cero, permitiendo distinguir puertos abiertos detrás de reglas de filtrado.

Si el auditor desea probar combinaciones de banderas totalmente personalizadas no contempladas en las opciones integradas, puede utilizar el parámetro `--scanflags` seguido del nombre de las banderas deseadas, como `--scanflags RSTSYNFIN`.

<br>

### 3.3 Suplantación (Spoofing) y Señuelos (Decoys)
La suplantación de dirección IP permite a un auditor enviar paquetes sondeando un objetivo mientras coloca una dirección IP falsa como origen en los encabezados de red. Se ejecuta mediante el comando `nmap -e INTERFAZ -Pn -S IP_FALSA IP_OBJETIVO`, especificando obligatoriamente la interfaz de red y desactivando el escaneo previo de ping. Sin embargo, la suplantación pura solo es útil si el auditor se encuentra en una posición de red que le permita interceptar las respuestas que el objetivo enviará de vuelta a la IP suplantada.

Cuando el auditor y el objetivo coexisten dentro de la misma subred física de red local Ethernet o Wi-Fi, también es posible suplantar la dirección física del hardware especificando la opción `--spoof-mac MAC_FALSA`.

Para ocultar la dirección IP real del auditor sin perder la capacidad de recibir respuestas en evaluaciones remotas, se utiliza la técnica de Señuelos (Decoys) mediante el parámetro `-D`. Esta técnica genera tráfico hacia el objetivo simulando que el escaneo proviene simultáneamente de múltiples direcciones IP distintas. Por ejemplo, al ejecutar `nmap -D 10.10.0.1,10.10.0.2,RND,RND,ME IP_OBJETIVO`, la máquina analizada recibirá sondas provenientes de las direcciones especificadas, dos direcciones IP asignadas aleatoriamente por Nmap y la dirección real del auditor designada por la palabra clave `ME`. Esto satura los registros de seguridad y dificulta la identificación del origen real de la auditoría.

<br>

### 3.4 Fragmentación de Paquetes y Evasión de Cortafuegos e IDS
Un cortafuegos filtra el tráfico inspeccionando las cabeceras IP y de transporte, mientras que un Sistema de Detección de Intrusos (IDS) analiza adicionalmente el contenido del payload buscando firmas de ataque conocidas. Para evitar que estos dispositivos reconozcan el patrón característico de los paquetes de Nmap, se puede utilizar la fragmentación de paquetes IP.

Al incluir el parámetro `-f` en el comando, Nmap divide los datos del encabezado IP y TCP en fragmentos de 8 bytes o menos. Si se especifica el parámetro dos veces como `-ff` o `-f -f`, los datos se dividen en fragmentos de 16 bytes. Al fragmentar el encabezado TCP en múltiples paquetes IP independientes, los cortafuegos e IDS simples incapaces de reensamblar fragmentos en memoria no pueden inspeccionar el paquete completo ni aplicar sus firmas de detección, dejando pasar el tráfico. También se puede ajustar el tamaño del bloque mediante `--mtu` seleccionando un múltiplo de 8.

Adicionalmente, para alterar la firma de tamaño de paquete habitual de Nmap y simular tráfico de red legítimo, se puede añadir el parámetro `--data-length NUMERO`, el cual anexa una cantidad especificada de bytes de datos aleatorios basura al final de cada paquete enviado.

<br>

### 3.5 Escaneo Inactivo o Zombi (`-sI`)
El escaneo inactivo, conocido también como Zombie Scan y seleccionado mediante la opción `nmap -sI IP_ZOMBIE IP_OBJETIVO`, es la técnica de escaneo más sigilosa de Nmap, ya que permite escanear un objetivo de forma totalmente anónima haciendo que todas las sondas provengan de un tercer equipo expuesto en la red denominado host zombi.

Esta técnica aprovecha el comportamiento del campo de Identificación de IP (IP ID) presente en el encabezado de la capa de red, el cual se incrementa de forma secuencial en ciertos sistemas operativos cada vez que transmiten un paquete. Para que el ataque sea preciso, el host zombi debe ser una máquina totalmente inactiva en la red que no esté generando tráfico por sí misma.

El procedimiento de explotación consta de tres pasos secuenciales ejecutados internamente por Nmap:

En el primer paso, el auditor envía un paquete SYN/ACK al host zombi. El zombi responde obligatoriamente enviando un paquete RST y revelando su número de Identificación de IP actual.

En el segundo paso, Nmap envía un paquete con la bandera SYN hacia el puerto deseado en la máquina objetivo, pero suplantando la dirección IP de origen para que parezca provenir de la dirección IP del host zombi. Si el puerto en el objetivo está abierto, la máquina objetivo responderá enviando un paquete SYN/ACK hacia el host zombi. Como el zombi no solicitó dicha conexión, responderá al objetivo enviando un paquete RST e incrementando en 1 su contador de Identificación de IP. Si el puerto en el objetivo está cerrado o filtrado, el objetivo responderá con un RST o descartará el paquete, por lo que el zombi no enviará ninguna respuesta y su contador de Identificación de IP no se incrementará.

En el tercer paso, el auditor envía un nuevo paquete SYN/ACK al host zombi para obtener su nuevo número de Identificación de IP en la respuesta RST.

Si al comparar la cifra de Identificación de IP inicial con la final se observa que el contador del zombi aumentó en 2 unidades, Nmap concluye que el puerto del objetivo está Abierto. Si el contador aumentó únicamente en 1 unidad, significa que el puerto del objetivo está Cerrado o Filtrado.

<br>

### 3.6 Obtención de Detalles Adicionales y Depuración
Para obtener un diagnóstico preciso del motivo exacto por el cual Nmap clasifica el estado de un puerto o sistema, se puede agregar el parámetro `--reason`. Esta opción muestra el paquete específico recibido desde el servidor, indicando por ejemplo si el host se considera activo por haber devuelto un `arp-response` o si un puerto se marca como abierto tras haber recibido un paquete `syn-ack`.

Para incrementar la cantidad de información mostrada en la consola durante la ejecución del análisis, se puede utilizar el parámetro de verbosidad `-v` o `-vv`. Si se requiere inspección técnica detallada del intercambio de paquetes a nivel de socket, se añaden los parámetros de depuración `-d` o `-dd`.
