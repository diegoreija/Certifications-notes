# 🛡️ GUÍA COMPLETA DE RECONOCIMIENTO Y PROTOCOLOS DE RED
### *Manual Extensivo y Guía de Consulta para Exámenes y Auditorías – Certificación eJPT*

---

> **Estructura del Manual:** Organizado estrictamente en orden cronológico según los contenidos oficiales de TryHackMe y las notas de Notion. Cubre desde las fases iniciales de recolección de inteligencia pública y sondeo activo hasta la anatomía interna de los protocolos de red tradicionales, sus vulnerabilidades de texto claro, los vectores de ataque en tránsito y la implementación de mecanismos modernos de cifrado y autenticación.

---

## 🚀 Matriz de Consulta Rápida (Cheat Sheet de Reconocimiento y Protocolos)

| Fase / Protocolo | Puerto / Comando Clave | Herramientas Principales | Indicadores y Propósito de Auditoría |
| :--- | :--- | :--- | :--- |
| **Reconocimiento Pasivo** | `whois domain`, `dig domain MX`, `crt.sh` | WHOIS, RDAP, dig, DNSDumpster, Shodan | Identificar registros de dominio, subdominios ocultos, SANs de certificados y exposición en Shodan sin enviar paquetes al objetivo. |
| **Reconocimiento Activo** | `ping -c 4 IP`, `traceroute -T IP`, `nc -vnlp 4444` | Navegador, DevTools, Ping, Traceroute, Netcat | Detectar hosts activos, estimar SO por TTL (Linux ~64, Windows ~128), mapear saltos de red e inspeccionar banners de servicios. |
| **Protocolo HTTP / HTTPS** | Puerto 80 (HTTP), 443 (HTTPS) | Telnet, Netcat, Curl, Burp Suite | Peticiones manuales `GET / HTTP/1.1`, identificación de cabeceras de servidor (`Server:`, `X-Powered-By`) y auditoría de certificados TLS. |
| **Protocolo FTP / FTPS / SFTP** | Puerto 21 (FTP), 22 (SFTP), 990 (FTPS) | Cliente FTP, Telnet, Netcat, FileZilla | Autenticación en texto plano, comprobación de acceso anónimo (`anonymous`), modo activo vs pasivo y transferencia de archivos. |
| **Protocolos de Correo (SMTP/POP3/IMAP)** | Ports 25/587 (SMTP), 110/995 (POP3), 143/993 (IMAP) | Telnet, Netcat, Thunderbird | Envío manual de correos, comprobación de relés abiertos, suplantación de identidad (Spoofing) y captura de credenciales en tránsito. |
| **Olfateo e Intercepción (Sniffing / MITM)** | `tcpdump -i any port 110 -A`, `bettercap` | Tcpdump, Wireshark, Bettercap, Responder | Captura de credenciales en texto claro, ARP Spoofing, DNS Spoofing y degradación de cifrado (SSL Stripping). |
| **Seguridad de Capa de Transporte (TLS)** | Puertos dedicados o STARTTLS | `testssl.sh`, Sslyze, SSL Labs, Nmap | Verificación de suites de cifrado, versiones soportadas (TLS 1.2/1.3), vigencia de certificados y configuración HSTS. |
| **Administración Remota SSH** | Puerto TCP 22 | `ssh`, `ssh-keygen`, `sftp`, `rsync` | Autenticación mediante claves públicas (Ed25519/RSA), verificación de `known_hosts`, trasferencia segura de archivos y endurecimiento de `sshd_config`. |
| **Ataques a Contraseñas** | `hydra -l user -P wordlist.txt IP service` | THC Hydra, Medusa, Ncrack, RockYou | Fuerza bruta y ataques de diccionario sobre servicios de red (SSH, FTP, POP3, IMAP, HTTP-POST) y mitigaciones con MFA o políticas de bloqueo. |

---

## 1. Reconocimiento Pasivo (Passive Reconnaissance)

### 1.1 Introducción y Diferencia entre Reconocimiento Pasivo y Activo
El reconocimiento representa la fase inicial indispensable en cualquier auditoría de seguridad o prueba de penetración, posicionándose como el primer eslabón en marcos de ataque metodológicos como la Cadena de Destrucción Cibernética (Cyber Kill Chain) y la Cadena de Destrucción Unificada (Unified Kill Chain). La premisa operativa expresada históricamente en el Arte de la Guerra se traduce directamente a la ciberseguridad: el auditor debe comprender detalladamente la superficie expuesta del objetivo antes de planificar cualquier interacción. Desde la perspectiva del equipo de defensa (Blue Team), entender qué información es accesible de forma pública permite reducir la huella expuesta y mitigar vectores de entrada antes de que sean explotados.

La distinción entre el reconocimiento pasivo y el activo radica en la interacción directa con la infraestructura objetivo. El reconocimiento pasivo se fundamenta de forma exclusiva en la recolección de inteligencia a partir de fuentes públicas y registros abiertos, sin transmitir un solo paquete de red a los servidores del objetivo. Esta metodología es equivalente a observar un edificio a distancia con binoculares sin pisar su propiedad. Al no generar tráfico directo, el reconocimiento pasivo es indetectable por los sistemas de monitorización del objetivo como cortafuegos, Sistemas de Detección de Intrusiones (IDS) o Firewalls de Aplicación Web (WAF), minimizando los riesgos legales y operativos. Por el contrario, el reconocimiento activo implica interactuar directamente con los sistemas, enviando sondas y paquetes para descubrir hosts vivos, puertos abiertos y servicios, lo que genera registros de auditoría y posibles alertas de seguridad. Cabe destacar que cualquier interacción directa con el personal de la organización, incluso a través de conversaciones informales o ingeniería social presencial, se clasifica rigurosamente como reconocimiento activo debido al contacto directo establecido.

### 1.2 Protocolo WHOIS y la Transición hacia RDAP
El protocolo WHOIS, definido en el RFC 3912, opera tradicionalmente en el puerto TCP 43 mediante un esquema de consulta y respuesta simple. Cuando se registra un nombre de dominio, el registrador almacena información detallada que puede ser consultada públicamente. Entre los datos clave que proporciona una consulta WHOIS se encuentran la empresa registradora (como Namecheap o GoDaddy), los datos de contacto del registrante, las fechas críticas de creación, última actualización y caducidad, los servidores de nombres autorizados para el dominio, los códigos de estado del dominio (como clientTransferProhibited, que impide transferencias no autorizadas) y los contactos designados para el reporte de abusos.

A pesar de su utilidad histórica, la disponibilidad de datos personales en consultas WHOIS ha cambiado drásticamente debido a regulaciones de privacidad como el Reglamento General de Protección de Datos (RGPD) en Europa y la ley CCPA en California. En la actualidad, los servicios de privacidad ocultan habitualmente el nombre, la dirección y el correo del propietario tras leyendas de privacidad. No obstante, el análisis de datos WHOIS sigue siendo valioso para los auditores al examinar las fechas del dominio (para calcular la antigüedad de la infraestructura o planificar ataques de ingeniería social cerca de los periodos de renovación) y los registros históricos a través de plataformas como Whoxy, los cuales pueden revelar propietarios pasados o migraciones de infraestructura que indiquen compromisos previos.

Como hito fundamental en la gobernanza de Internet, la ICANN suspendió oficialmente el protocolo tradicional WHOIS el 28 de enero de 2025 para dominios genéricos de primer nivel (gTLD), sustituyéndolo obligatoriamente por el Protocolo de Acceso a Datos de Registro (RDAP). RDAP constituye el sucesor moderno de WHOIS al funcionar sobre HTTPS en lugar de texto plano, devolver respuestas en formato JSON estructurado y legible por máquina, ofrecer soporte nativo para internacionalización e integrar controles de acceso diferenciados para cumplir con las normativas de protección de datos. Para consultar registros RDAP desde la línea de comandos, se utilizan herramientas como curl combinadas con procesadores JSON como jq, o clientes especializados como OpenRDAP, obteniendo datos estructurados directamente de las entidades autorizadas.

### 1.3 Consultas de Registros DNS con nslookup y dig
El Sistema de Nombres de Dominio (DNS) es una base de datos distribuida que traduce nombres de dominio legibles por humanos en direcciones IP numéricas. Las consultas DNS representan una técnica de reconocimiento completamente pasiva, ya que las peticiones se dirigen a solucionadores recursivos públicos o abiertos (como Cloudflare 1.1.1.1 o Google 8.8.8.8) en lugar de consultar directamente los servidores del objetivo. El uso de solucionadores públicos cifrados mediante DNS sobre HTTPS (DoH) o DNS sobre TLS (DoT) evita además que el proveedor de servicios de Internet (ISP) registre la actividad de búsqueda del auditor.

Entre las herramientas de consulta destacan nslookup y dig. Aunque nslookup es una utilidad clásica presente por compatibilidad en sistemas Windows y entornos heredados, la herramienta estándar y recomendada en auditorías profesionales es dig (Domain Information Groper). Dig proporciona una salida limpia, muestra el tiempo de vida en caché (TTL) de cada registro y permite redactar scripts de automatización con mayor fiabilidad. Los tipos de registros DNS más relevantes durante el reconocimiento incluyen las entradas A (direcciones IPv4), AAAA (direcciones IPv6), CNAME (alias que apuntan un dominio a otro nombre canónico), MX (servidores de correo con sus valores de prioridad donde números menores indican mayor preferencia), SOA (Inicio de Autoridad con el correo del administrador y número de serie de la zona) y TXT (registros de texto arbitrario empleados masivamente para autenticación de correo con SPF, DKIM y DMARC, así como para la verificación de propiedad de dominios).

### 1.4 Descubrimiento de Subdominios y la Plataforma DNSDumpster
Las consultas DNS tradicionales mediante dig o nslookup solo responden sobre nombres de host que el auditor ya conoce. Sin embargo, las organizaciones suelen mantener una amplia variedad de subdominios que no están anunciados públicamente, como entornos de desarrollo, portales internos, APIs heredadas o instalaciones olvidadas de gestores de contenido (TI en la sombra). Estos subdominios representan una superficie de ataque crítica, ya que suelen presentar configuraciones defensivas más débiles que el sitio web principal.

DNSDumpster es una plataforma pública y gratuita de OSINT que permite descubrir subdominios de forma totalmente pasiva. En lugar de utilizar ataques de fuerza bruta que enviarían miles de peticiones al objetivo, DNSDumpster recopila y agrega datos de fuentes públicas, incluyendo cachés de motores de búsqueda, bases de datos de transferencias de zona públicas y registros de certificados SSL/TLS. La plataforma no solo enumera los subdominios encontrados con sus respectivas direcciones IP y ubicaciones geográficas, sino que también genera mapas visuales que ilustran las relaciones entre los dominios, los servidores de correo MX y la infraestructura de red subyacente.

### 1.5 Registros de Transparencia de Certificados (Certificate Transparency - CT Logs)
La técnica más potente y completa para el descubrimiento pasivo de subdominios en la actualidad consiste en la consulta de los Registros de Transparencia de Certificados (CT Logs). La Transparencia de Certificados es un marco público e idóneo de auditoría implementado de forma obligatoria para las Autoridades de Certificación (CA). Cada vez que una CA emite un certificado SSL/TLS para un dominio o subdominio, el certificado debe registrarse obligatoriamente en un libro mayor público e inalterable.

Cada certificado digital contiene un campo denominado Nombre Alternativo del Sujeto (SAN - Subject Alternative Name), el cual lista explícitamente todos los subdominios e infraestructura cubierta por dicha firma criptográfica. A través de plataformas web como crt.sh, un auditor puede buscar mediante comodines (por ejemplo, %.dominio.com) y obtener un listado histórico de cada certificado emitido para la organización. Esta técnica funciona en tiempo real, no requiere enviar ningún paquete a la red objetivo y suele revelar de diez a cien veces más subdominios que las búsquedas DNS convencionales, exponiendo con frecuencia infraestructura interna o en pruebas.

### 1.6 Censo de Dispositivos Conectados con Shodan.io y Censys
Shodan.io es un motor de búsqueda especializado que escanea continuamente la totalidad del direccionamiento IPv4 e IPv6 público, recopilando e indexando las respuestas y banners expuestos en los puertos abiertos de cualquier dispositivo conectado. A diferencia de motores de búsqueda tradicionales como Google, que indexan contenido web HTML, Shodan indexa infraestructura: servidores web, bases de datos, routers, dispositivos de IoT, cámaras de seguridad y sistemas de control industrial (ICS/SCADA).

Durante la fase de reconocimiento pasivo, consultar Shodan permite obtener una radiografía completa de los activos de una organización sin tocar su red. Al ingresar una dirección IP, un rango o un nombre de dominio, Shodan devuelve el número de sistema autónomo (ASN), el proveedor de alojamiento o nube (como AWS, Azure o Cloudflare), la ubicación geográfica, la lista de puertos abiertos con sus respectivos banners de versión y etiquetas automáticas de vulnerabilidad (como la presencia de exploits conocidos asociados al software detectado). Shodan admite filtros de búsqueda avanzados como hostname:dominio.com, org:"Nombre Empresa", port:443 o http.component:"wordpress". Como alternativa o complemento, la plataforma Censys.io ofrece capacidades similares de análisis de infraestructura y certificados digitales para cotejar hallazgos.

---

## 2. Reconocimiento Activo (Active Reconnaissance)

### 2.1 Introducción y Consideraciones de Riesgo
El reconocimiento activo marca la transición hacia la interacción directa con los sistemas, aplicaciones e infraestructura del objetivo. A diferencia de las técnicas pasivas, el reconocimiento activo exige transmitir paquetes de red, establecer conexiones TCP/UDP y sondear servicios en escucha. La consecuencia directa de esta interacción es la generación inevitable de huellas digitales en forma de registros de acceso en servidores web, eventos en cortafuegos, bloqueos en Firewalls de Aplicación Web (WAF), alertas en Sistemas de Detección y Prevención de Intrusiones (IDS/IPS) y eventos en plataformas SIEM o soluciones de detección en el endpoint (EDR).

Debido a su naturaleza detectable y al riesgo de causar interrupciones en servicios de producción, el reconocimiento activo exige estrictamente disponer de una autorización legal por escrito (como un contrato de prueba de penetración con un alcance claramente delimitado o los términos de un programa de Bug Bounty). El sondeo no autorizado de redes es ilegal en la mayoría de las jurisdicciones. En el contexto defensivo, las organizaciones monitorizan sus perímetros para identificar patrones de escaneo temprano, mientras que desde la perspectiva del equipo de ataque (Red Team), el objetivo principal radica en camuflar las sondas dentro del tráfico legítimo de la red, emulando el comportamiento de usuarios reales mediante el ajuste de velocidad, el uso de agentes de usuario realistas y la distribución de peticiones.

### 2.2 Reconocimiento mediante Navegador Web y Herramientas de Desarrollador
El navegador web es una de las herramientas de reconocimiento activo más efectivas y menos sospechosas, ya que su tráfico se confunde de manera natural con las miles de peticiones de usuarios legítimos. Los navegadores se conectan por defecto al puerto TCP 80 para tráfico HTTP sin cifrar y al puerto TCP 443 para tráfico HTTPS cifrado con TLS. En aplicaciones modernas, es común encontrar el uso de HTTP/3 ejecutado sobre el protocolo QUIC, el cual combina TCP y TLS sobre el puerto UDP 443 para acelerar la conexión. Es posible sondear servicios en puertos no estándar especificando el puerto directamente en la URL del navegador.

Las Herramientas para Desarrolladores (DevTools), accesibles mediante la combinación de teclas Ctrl + Mayús + I en la mayoría de navegadores, ofrecen un conjunto de pestañas clave para la extracción de información:
La pestaña Red (Network) registra todas las peticiones y respuestas en tiempo real, exponiendo encabezados HTTP críticos como Server, X-Powered-By y Content-Security-Policy, además de códigos de estado y cookies.
La pestaña Consola (Console) permite ejecutar código JavaScript en el contexto del DOM e interactuar con la aplicación.
La pestaña Fuentes (Sources o Debugger) permite inspeccionar los archivos HTML, CSS y JavaScript cargados. El análisis minucioso del código JavaScript cliente es una de las prácticas más fructíferas, ya que los desarrolladores suelen dejar accidentalmente puntos finales de API no documentados, rutas de directorios internos, claves de API secundarias y comentarios con información sensible.
La pestaña Aplicación (Application o Storage) permite auditar las cookies de sesión, el almacenamiento local (LocalStorage) y de sesión (SessionStorage), donde pueden residir tokens JWT o credenciales.
La pestaña Seguridad (Security) expone los detalles del certificado digital TLS y sus nombres alternativos (SAN).

### 2.3 Extensiones del Navegador para Identificación de Tecnologías
El uso de extensiones especializadas transforma el navegador en una plataforma de reconocimiento avanzada. FoxyProxy facilita el conmutado rápido de tráfico entre diferentes proxies de interceptación como Burp Suite, OWASP ZAP o túneles SOCKS5. User-Agent Switcher and Manager permite alterar la cadena de agente de usuario para emular navegadores móviles u otros sistemas operativos, lo que ayuda a descubrir puntos finales específicos para dispositivos móviles o comportamientos condicionales en el servidor. Wappalyzer e identificadores similares como BuiltWith, WhatRuns o Library Detector analizan pasivamente las respuestas de la página para identificar automáticamente el gestor de contenidos (CMS), la versión del servidor web, los marcos de trabajo de JavaScript (como React o Angular), las bibliotecas cliente y las herramientas de analítica utilizadas por el objetivo.

### 2.4 Comprobación de Host con Ping e ICMP
El comando ping utiliza el Protocolo de Mensajes de Control de Internet (ICMP) para verificar si un host remoto está encendido y accesible a través de la red. Funciona enviando un paquete ICMP Echo Request (Tipo 8) y esperando recibir un paquete ICMP Echo Reply (Tipo 0). En sistemas Linux y macOS se utiliza el parámetro -c para definir la cantidad de paquetes a enviar, mientras que en Windows se emplea la opción -n. Se pueden forzar versiones de IP específicas mediante los parámetros -4 y -6.

Además de confirmar la conectividad y medir la latencia de red, el análisis del valor Time To Live (TTL) devuelto en la respuesta ICMP proporciona una pista fundamental para la identificación del sistema operativo (OS Fingerprinting). El valor TTL representa el número máximo de saltos por enrutadores que un paquete puede atravesar antes de ser descartado. Los sistemas operativos establecen valores TTL iniciales característicos: Linux y Unix utilizan típicamente un TTL inicial de 64, mientras que Microsoft Windows utiliza un valor inicial de 128. Dado que cada enrutador en la ruta decrementa el valor TTL en una unidad, recibir una respuesta con un TTL de 58 sugiere un sistema Linux situado a seis saltos de distancia.

Si el comando ping no recibe respuesta y muestra un 100% de pérdida de paquetes, no significa necesariamente que el host esté apagado. Las razones más comunes para la falta de respuesta incluyen el bloqueo de peticiones ICMP por cortafuegos locales (el Firewall de Windows bloquea ping por defecto), la filtración de ICMP por parte de proveedores de nube (AWS, Azure, GCP), la presencia de WAFs o CDNs, o la existencia de reglas de red que descartan paquetes ICMP salientes o entrantes.

### 2.5 Trazado de Rutas de Red con Traceroute y MTR
El comando traceroute (tracert en Windows) permite mapear la topología de la red identificando las direcciones IP de los enrutadores intermedios (saltos) situados entre el equipo del auditor y el destino. Debido a que no existe un comando directo para solicitar la ruta completa a una red, traceroute explota de forma ingeniosa el campo TTL de los paquetes IP. La herramienta envía una secuencia de paquetes incrementando el TTL comenzando en 1. Cuando el primer enrutador recibe el paquete con TTL=1, decrementa el valor a 0, descarta el paquete y devuelve al emisor un mensaje ICMP Time Exceeded (Tipo 11, Código 0). Incrementando el TTL a 2, se fuerza al segundo enrutador a responder, y así sucesivamente hasta alcanzar el destino final.

En sistemas Linux, traceroute envía datagramas UDP por defecto a puertos altos no estándar. Para eludir cortafuegos que bloqueen UDP, se puede forzar el uso de sondas TCP mediante el parámetro traceroute -T o sondas ICMP mediante traceroute -I. Cuando un enrutador está configurado para no responder a paquetes de tiempo excedido, la salida muestra asteriscos (*). Es fundamental comprender que las rutas de red no son estáticas; el uso de enrutamiento dinámico (BGP, OSPF), balanceadores de carga y redes CDN (como Cloudflare) provoca que los paquetes sigan rutas distintas incluso en ejecuciones consecutivas del comando. Para una monitorización dinámica en tiempo real se utiliza la herramienta mtr (My Traceroute), la cual combina la funcionalidad de traceroute y ping en una sola interfaz interactiva.

### 2.6 Captura de Banners e Interacción con Telnet y Netcat
El protocolo TELNET, diseñado originalmente en 1969 para la administración remota en el puerto TCP 23, transmite absolutamente todos los datos en texto claro sin cifrado. Por esta razón, ha sido reemplazado por SSH. Sin embargo, el cliente telnet de la línea de comandos sigue siendo una herramienta de reconocimiento muy útil para realizar la técnica conocida como captura de banners (Banner Grabbing). Dado que telnet establece conexiones TCP crudas, es posible conectarse a cualquier puerto TCP en escucha e interactuar manualmente con el servicio.

Al conectarse al puerto TCP 80 de un servidor web mediante el comando telnet IP 80 y enviar una petición HTTP básica (como GET / HTTP/1.1 seguido del encabezado Host: objetivo y dos pulsaciones de Enter), el servidor devolverá su respuesta HTTP acompañada de sus encabezados. El encabezado Server expone directamente el nombre y la versión exacta del software instalado (por ejemplo, Server: nginx/1.18.0). Esta información permite buscar vulnerabilidades conocidas en bases de datos públicas como CVE o Exploit-DB. La misma técnica se aplica a servidores FTP (puerto 21) o SMTP (puerto 25). Para servicios cifrados modernos con TLS (como HTTPS en el puerto 443 o SMTPS en el 465), telnet no puede procesar el cifrado, por lo que se utilizan herramientas como curl --head, openssl s_client -connect IP:443 o ncat --ssl.

Por su parte, Netcat (nc) se conoce como la navaja suiza de las redes. Además de actuar como cliente para capturar banners, Netcat puede funcionar como servidor en escucha utilizando la sintaxis nc -vnlp PUERTO. El parámetro -l activa el modo escucha, -p especifica el puerto, -n evita resoluciones DNS para acelerar el proceso y -v activa la salida detallada. Esta capacidad de escucha es fundamental para verificar la conectividad de red, realizar transferencias de archivos sencillas o recibir conexiones de shell inverso durante una auditoría.

---

## 3. Protocolos y Servidores I (Protocols and Servers)

### 3.1 Acceso Remoto Inseguro con Telnet
Telnet es un protocolo de capa de aplicación diseñado para proporcionar acceso a una interfaz de línea de comandos (CLI) en un equipo remoto. Tras establecer la conexión TCP en el puerto 23, el servidor solicita un nombre de usuario y una contraseña. Una vez validada la autenticación, se otorga una consola interactiva en la máquina remota.

La debilidad crítica de Telnet radica en la ausencia total de cifrado. Todos los datos intercambiados —incluyendo el nombre de usuario, la contraseña introducida carácter por carácter y los comandos ejecutados con sus respuestas— se transmiten en texto plano por la red. Cualquier atacante o analizador de tráfico situado en el mismo segmento de red local, en un enrutador intermedio o ejecutando un ataque Man-in-the-Middle (MITM) puede capturar las credenciales sin esfuerzo. En la actualidad, encontrar un servidor Telnet abierto es un indicador claro de la presencia de un sistema heredado, un dispositivo IoT con recursos limitados o una severa mala configuración de seguridad.

### 3.2 Protocolo de Transferencia de Hipertexto (HTTP)
El protocolo HTTP es la columna vertebral de la World Wide Web, diseñado para la transferencia de archivos de hipertexto, código HTML, imágenes y servicios web. Opera bajo una arquitectura cliente-servidor donde el cliente (navegador web) envía solicitudes y el servidor responde con los recursos solicitados o códigos de estado.

#### Comparativa entre HTTP y HTTPS
HTTP transmite todas las peticiones y respuestas en texto claro a través del puerto TCP 80. HTTPS (HTTP Secure) envuelve el tráfico HTTP dentro de un canal cifrado mediante TLS en el puerto TCP 443, garantizando la confidencialidad de las credenciales, galletas de sesión y datos transmitidos. Aunque la gran mayoría del tráfico web actual utiliza HTTPS, la estructura interna de los mensajes HTTP (métodos, encabezados y cuerpo) es exactamente idéntica en ambos protocolos.

#### Peticiones Manuales e Inspección de Encabezados
Al conectar manualmente a un servidor web en el puerto 80 mediante Telnet o Netcat, el auditor debe construir la petición HTTP/1.1 de forma textual. La estructura exige especificar el método HTTP (GET, POST, PUT, DELETE), la ruta solicitada, la versión del protocolo y la cabecera obligatoria Host, finalizando la solicitud con una línea en blanco (dos saltos de línea):

```http
GET /index.html HTTP/1.1
Host: objetivo.com

```

El servidor responde devolviendo la línea de estado (por ejemplo, HTTP/1.1 200 OK), los encabezados de respuesta y el cuerpo del mensaje. Los encabezados de respuesta revelan detalles valiosos para la fase de reconocimiento, tales como Server: Apache/2.4.58 (Ubuntu) o X-Powered-By: PHP/8.2.10, permitiendo acotar la versión exacta del entorno de ejecución.

#### Servidores Web y Evolución del Protocolo
Los servidores web más desplegados en la industria son Nginx (destacado por su arquitectura orientada a eventos e impulsado para conexiones concurrentes masivas), Apache HTTP Server (ampliamente configurable mediante módulos y archivos .htaccess), e Internet Information Services (IIS de Microsoft, común en entornos corporativos Windows). Las versiones del protocolo han evolucionado desde HTTP/1.1 (basado en texto y conexiones persistentes) hacia HTTP/2 (que introduce formato binario y multiplexación de peticiones sobre una sola conexión TCP) y HTTP/3 (que reemplaza TCP por el protocolo QUIC sobre UDP, optimizando el rendimiento en redes móviles).

### 3.3 Protocolo de Transferencia de Archivos (FTP)
El protocolo FTP, definido originalmente en los inicios de las redes informáticas, está diseñado para la transferencia eficiente de archivos entre sistemas. Opera mediante dos canales TCP independientes: el canal de control (puerto TCP 21), por donde se transmiten los comandos de autenticación y navegación en texto plano, y el canal de datos, utilizado para la transferencia efectiva de los archivos.

#### Modos de Funcionamiento: Activo vs. Pasivo
La arquitectura de doble canal de FTP requiere definir cómo se establece la conexión de datos:
En el Modo Activo, el cliente se conecta desde un puerto aleatorio al puerto 21 del servidor para enviar comandos. Cuando se solicita enviar o recibir un archivo, el cliente abre un puerto local e indica al servidor su dirección y puerto mediante el comando PORT. Posteriormente, el servidor inicia la conexión de datos desde su puerto TCP 20 hacia el puerto especificado por el cliente. Este modo suele fallar en redes modernas porque los cortafuegos y routers NAT del cliente bloquean la conexión entrante iniciada por el servidor.
En el Modo Pasivo (solicitado mediante el comando PASV), el cliente inicia ambas conexiones. Tras recibir el comando PASV, el servidor abre un puerto efímero aleatorio (superior al 1024) e informa al cliente de su dirección e IP. El cliente se conecta a dicho puerto para canalizar la transferencia. Este modo es el estándar en clientes modernos al atravesar cortafuegos sin inconvenientes.

#### Autenticación y FTP Anónimo
Un servidor FTP exige autenticarse mediante los comandos USER usuario y PASS contraseña. Si las credenciales son válidas, el servidor permite ejecutar comandos como LIST (listar archivos), RETR (descargar archivo), STOR (subir archivo) y QUIT. El comando TYPE A cambia la transferencia al modo ASCII (para archivos de texto), mientras que TYPE I establece el modo binario (para ejecutables, imágenes o archivos comprimidos).

Muchos servidores FTP están configurados para permitir el acceso anónimo (Anonymous FTP), aceptando el usuario anonymous o ftp con cualquier cadena o correo ficticio como contraseña. Durante una auditoría, verificar la presencia de FTP anónimo es un paso obligado, ya que estos repositorios suelen contener accidentalmente copias de seguridad de bases de datos, archivos de configuración con credenciales o directorios con permisos de escritura donde un atacante podría subir archivos maliciosos. Debido a la falta de cifrado en FTP tradicional, la industria ha migrado hacia SFTP (SSH File Transfer Protocol, ejecutado sobre SSH en el puerto 22) y FTPS (FTP Secure, que añade cifrado TLS sobre el puerto 990 o mediante STARTTLS en el puerto 21).

### 3.4 Protocolo Simple de Transferencia de Correo (SMTP)
El correo electrónico en Internet depende de una arquitectura de componentes interconectados:
MUA (Mail User Agent): El cliente de correo utilizado por el usuario (Thunderbird, Outlook, interfaz webmail).
MSA (Mail Submission Agent): Servidor que recibe los correos emitidos por el MUA y verifica errores de formato antes de entregarlos al MTA.
MTA (Mail Transfer Agent): Servidor encargado de enrutar y transferir los correos entre diferentes dominios a través de Internet (por ejemplo, Postfix, Sendmail, Exchange).
MDA (Mail Delivery Agent): Componente que almacena los mensajes recibidos en el buzón local del usuario para su posterior lectura.

#### Puertos SMTP y Cifrado
El Protocolo SMTP regula la comunicación de envío y transferencia entre servidores y clientes. Sus puertos operativos son:
Puerto TCP 25: Puerto estándar para la comunicación servidor a servidor (MTA a MTA). Transmite en texto plano por defecto, permitiendo negociar cifrado opcional mediante la instrucción STARTTLS.
Puerto TCP 587: Puerto designado para el envío de correos desde clientes MUA hacia el servidor MSA. Requiere autenticación obligatoria y exige cifrado mediante STARTTLS.
Puerto TCP 465: Puerto histórico utilizado para SMTPS (SMTP sobre TLS implícito), donde el cifrado TLS se inicia inmediatamente al conectar.

#### Envío Manual y Suplantación de Identidad (Email Spoofing)
La interacción manual con un servidor SMTP en el puerto 25 revela la sencillez de su funcionamiento. Tras establecer la conexión, el cliente inicia el saludo mediante HELO dominio o EHLO dominio (para SMTP extendido). Posteriormente, se definen las direcciones del emisor y receptor, y se introduce el contenido del mensaje:

```smtp
HELO atacante.com
MAIL FROM: administrador@banco.com
RCPT TO: victima@empresa.com
DATA
Subject: Aviso Importante
Por favor actualice sus credenciales.
.
```

El mensaje finaliza introduciendo un único punto . en una línea independiente. El aspecto más crítico desde la perspectiva de la seguridad es que la especificación original de SMTP no realiza ninguna verificación para comprobar si el emisor especificado en MAIL FROM realmente posee o controla dicha cuenta de correo. Esta carencia fundamental es la causa raíz que permite la suplantación de identidad (Email Spoofing) en ataques de phishing. Para mitigar esta vulnerabilidad, las organizaciones deben desplegar mecanismos de autenticación defensiva en sus registros DNS públicos: SPF (Sender Policy Framework), DKIM (DomainKeys Identified Mail) y DMARC (Domain-based Message Authentication, Reporting, and Conformance).

### 3.5 Protocolo de Oficina Postal 3 (POP3)
El protocolo POP3, definido en el RFC 1939, opera en el puerto TCP 110 (o puerto TCP 995 para su versión cifrada POP3S) y está diseñado para recuperar y descargar los mensajes almacenados en el servidor MDA hacia el cliente local.

#### Comportamiento "Descargar y Eliminar"
El modelo operativo clásico de POP3 se basa en la descarga y eliminación. Cuando el cliente de correo se conecta al servidor POP3, se autentica mediante los comandos USER usuario y PASS contraseña, consulta los mensajes con STAT (devuelve el número total de correos y el tamaño ocupado) y LIST, descarga el contenido del mensaje con RETR numero_mensaje, y marca el correo para su borrado en el servidor mediante DELE numero_mensaje. La sesión concluye con el comando QUIT, momento en el cual el servidor elimina definitivamente los correos marcados.

Este comportamiento presenta serias limitaciones en la actualidad. Dado que los correos se almacenan únicamente en el equipo local que realizó la descarga, no existe posibilidad de sincronización entre múltiples dispositivos (teléfonos móviles, portátiles, tablets). Si un usuario lee un correo en su ordenador, dicho mensaje no estará disponible en su teléfono móvil. Por esta razón, POP3 se reserva actualmente para escenarios donde el almacenamiento en el servidor es extremadamente reducido o se requiere archivar correo localmente sin conexión.

### 3.6 Protocolo de Acceso a Mensajes de Internet (IMAP)
El protocolo IMAP opera en el puerto TCP 143 (o puerto TCP 993 para IMAPS cifrado con TLS) y fue diseñado para superar las limitaciones de POP3, convirtiéndose en el estándar universal de lectura de correo electrónico.

#### Sincronización Multidispositivo y Comandos
A diferencia de POP3, IMAP mantiene todos los correos electrónicos y carpetas almacenados centralmente en el servidor. El cliente IMAP se limita a sincronizar el estado del buzón. Si un usuario lee, organiza en carpetas o elimina un correo desde su dispositivo móvil, el servidor IMAP actualiza el estado y refleja los mismos cambios instantáneamente en cualquier otro dispositivo conectado.

Las sesiones IMAP requieren que cada comando enviado por el cliente vaya precedido por una etiqueta o prefijo aleatorio (como c1, c2) para rastrear las respuestas asíncronas del servidor. Tras autenticarse con c1 LOGIN usuario contraseña, se pueden listar las carpetas del buzón mediante c2 LIST "" "*", seleccionar la bandeja de entrada con c3 EXAMINE INBOX o c3 SELECT INBOX, y recuperar encabezados o cuerpos de mensajes específicos mediante c4 FETCH.

#### Implicaciones de Seguridad
Dado que IMAP tradicional envía credenciales en texto plano, la captura de tráfico en la red expone inmediatamente el usuario y la clave. Un buzón IMAP comprometido representa una amenaza masiva para la organización: el atacante obtiene acceso no solo a las credenciales, sino a todo el historial de conversaciones de la víctima, posibilitando el robo de información confidencial, la interceptación de enlaces de restablecimiento de contraseñas de otras plataformas corporativas y la ejecución de ataques de Compromiso del Correo Corporativo (BEC - Business Email Compromise).

---

## 4. Protocolos y Servidores II (Protocols and Servers 2)

### 4.1 Panorama Moderno de Ataques y la Tríada CIA frente al Modelo DAD
La evaluación de la seguridad de cualquier infraestructura requiere analizar los activos desde la perspectiva de los tres pilares de la seguridad de la información, conocidos como la Tríada CIA:
Confidencialidad: Garantizar que la información sea accesible exclusivamente para las partes autorizadas.
Integridad: Asegurar que los datos transmitidos no hayan sido alterados, manipulados o corrompidos en tránsito.
Disponibilidad: Garantizar que los servicios y recursos estén accesibles para los usuarios autorizados cuando los requieran.

Los ataques informáticos están diseñados para romper estos pilares, provocando los tres efectos del Modelo DAD:
Divulgación (Disclosure): Pérdida de confidencialidad provocada por la interceptación de datos cifrados o de texto claro.
Alteración (Alteration): Pérdida de integridad causada por la modificación no autorizada de mensajes en tránsito.
Destrucción (Destruction) o Denegación: Pérdida de disponibilidad provocada por ataques de Denegación de Servicio (DoS/DDoS) o borrado de datos.

### 4.2 Ataques de Olfateo de Red (Sniffing) y Mitigaciones
Un ataque de olfateo (Sniffing) consiste en el uso de herramientas de captura e inspección de paquetes de red para interceptar el tráfico que circula por la interfaz de red. Cuando un protocolo transmite información en texto claro (como HTTP, FTP, Telnet, POP3 o IMAP), un analizador de tráfico puede extraer directamente nombres de usuario, contraseñas, galletas de sesión y el contenido completo de las comunicaciones.

#### Escenarios Relevantes en la Actualidad
Aunque la adopción masiva del cifrado TLS ha reducido el impacto del olfateo en Internet pública, esta técnica continúa siendo altamente efectiva en múltiples escenarios de auditoría:
Redes corporativas internas donde el tráfico entre servidores y clientes no implementa cifrado de extremo a extremo.
Entornos con sistemas heredados, controladores industriales (ICS) o dispositivos IoT que no soportan TLS.
Servidores mal configurados que ofrecen cifrado pero no lo exigen obligatoriamente.
Ataques en redes inalámbricas Wi-Fi no cifradas o mal protegidas.
Fases posteriores a un ataque Man-in-the-Middle (MITM) exitoso donde se ha degradado el cifrado.

#### Herramientas de Captura y Ejemplo Práctico con Tcpdump
Las herramientas de captura de tráfico más utilizadas son Tcpdump (potente interfaz de línea de comandos preinstalada en entornos Linux), Wireshark (interfaz gráfica de análisis profundo con miles de disectores de protocolos) y Tshark (versión CLI de Wireshark ideal para automatización).

Para capturar credenciales de un protocolo inseguro como POP3 en el puerto 110 utilizando Tcpdump, se ejecuta el siguiente comando con privilegios de superusuario:

```bash
sudo tcpdump -i any port 110 -A
```

El parámetro -i any especifica escuchar en todas las interfaces de red, port 110 filtra únicamente el tráfico dirigido o procedente del servidor POP3, y -A fuerza la visualización del contenido de los paquetes en formato ASCII legible. Cuando el usuario se autentique, Tcpdump mostrará explícitamente en la terminal las cadenas USER frank y PASS clave123 enviadas en paquetes separados.

#### Estrategias de Mitigación Defensiva
La defensa definitiva contra el olfateo de red es la implementación de cifrado en la capa de aplicación o transporte mediante TLS y SSH. Las medidas defensivas complementarias incluyen:
Segmentación de red mediante VLANs e infraestructura aislada para limitar el alcance del tráfico de difusión.
Autenticación basada en puerto IEEE 802.1X para impedir que dispositivos no autorizados se conecten a la red física.
Despliegue de una Arquitectura de Confianza Cero (Zero Trust), la cual asume que la red interna es intrínsecamente hostil y exige cifrado y autenticación en todas las comunicaciones internas.
Monitorización activa de tablas ARP para detectar técnicas de redirección de tráfico.

### 4.3 Ataques de Hombre en el Medio (MITM) y Defensas Modernas
Un ataque Man-in-the-Middle (MITM) ocurre cuando un atacante (E) logra posicionarse de forma transparente en la vía de comunicación entre la víctima (A) y el destino legítimo (B). La víctima cree estar comunicándose directamente con el servidor, pero en realidad transmite todo su tráfico al atacante, quien puede inspeccionar el contenido (violando la confidencialidad) o modificar los datos al vuelo antes de reenviarlos al destino (violando la integridad).

#### Técnicas de Posicionamiento y Herramientas
Para forzar a que el tráfico pase a través del equipo del atacante se utilizan diversas técnicas de envenenamiento y redirección a nivel de red local:
Envenenamiento ARP (ARP Spoofing): El atacante emite respuestas ARP falsificadas a la red local asociando su dirección MAC con la IP de la puerta de enlace predeterminada, obligando a las víctimas a enviar su tráfico a través de su máquina.
Suplantación DNS (DNS Spoofing): Respuestas DNS manipuladas que redirigen a los usuarios a direcciones IP bajo el control del atacante.
Puntos de Acceso Falsos (Rogue APs): Puntos de acceso Wi-Fi maliciosos configurados con nombres idénticos a redes legítimas.
Explotación de LLMNR y NBT-NS: Uso de la herramienta Responder en entornos Windows para responder a consultas de resolución de nombres locales no resueltas y capturar hashes de autenticación NTLMv2.
Herramientas avanzadas como Bettercap o Mitmproxy permiten automatizar el envenenamiento ARP, la resolución DNS maliciosa y la manipulación de peticiones HTTP/HTTPS.

#### Ataques contra Tráfico Cifrado y Mecanismos de Protección
Cuando el tráfico utiliza HTTPS, los atacantes intentan eludir el cifrado mediante técnicas como el Desnudado SSL (SSL Stripping), donde herramientas como Bettercap interceptan las peticiones e intercambian de forma transparente las URLs de HTTPS a HTTP no cifrado hacia el navegador de la víctima.

Para contrarrestar estos ataques, la infraestructura web moderna despliega protecciones avanzadas:
HSTS (HTTP Strict Transport Security): Cabecera HTTP que ordena al navegador conectarse de forma exclusiva mediante HTTPS durante un periodo determinado. La inclusión en listas de precarga HSTS (HSTS Preload) garantiza que el navegador nunca intente una conexión HTTP inicial.
Transparencia de Certificados (CT): Libros mayores públicos obligatorios que registran todos los certificados emitidos por Autoridades de Certificación (CA), impidiendo la emisión oculta de certificados fraudulentos.
Fijación de Certificados (Certificate Pinning): Técnica utilizada en aplicaciones móviles que valida que el certificado recibido coincida exactamente con la clave pública incrustada en la aplicación.

### 4.4 Seguridad de la Capa de Transporte (TLS / SSL)
El protocolo Transport Layer Security (TLS), y su antecesor obsoleto Secure Sockets Layer (SSL), proporciona la capa fundamental de seguridad criptográfica para proteger la confidencialidad e integridad del tráfico en Internet.

#### Evolución Histórica de los Protocolos
SSL 2.0 y SSL 3.0 (diseñados por Netscape en los años 90) se encuentran completamente obsoletos y prohibidos debido a fallos estructurales graves (ataques POODLE).
TLS 1.0 y TLS 1.1 quedaron obsoletos oficialmente en 2021 debido a vulnerabilidades ante ataques como BEAST y CRIME; los navegadores modernos rechazan estas conexiones.
TLS 1.2 (publicado en 2008) continúa siendo ampliamente utilizado y se considera seguro cuando se configura desactivando cifrados débiles.
TLS 1.3 (estándar actual definido en 2018) representa una mejora radical al reducir la negociación a un solo viaje de ida y vuelta (1-RTT), eliminar algoritmos obsoletos y exigir el uso obligatorio de Secreto Directo (Forward Secrecy), garantizando que el compromiso futuro de la clave privada del servidor no permita descifrar el tráfico capturado en el pasado.

#### Cifrado Implícito frente a STARTTLS
Existen dos formas de implementar TLS en los protocolos de red:
TLS Implícito: Utiliza un puerto dedicado exclusivo donde la negociación cifrada comienza inmediatamente al establecer la conexión TCP (por ejemplo, HTTPS en el puerto 443, POP3S en el 995 o IMAPS en el 993). Es el modelo más seguro.
STARTTLS: Se conecta inicialmente al puerto estándar en texto claro (por ejemplo, SMTP en el puerto 25 o 587) y envía la orden STARTTLS para actualizar la conexión a un canal cifrado. Puede ser vulnerable a ataques de degradación (Downgrade Attacks) si un atacante MITM elimina la orden de actualización.

#### La Negociación TLS (TLS Handshake)
Para establecer un canal cifrado en TLS 1.2, el cliente envía un mensaje ClientHello con las versiones y cifrados soportados. El servidor responde con ServerHello, selecciona los parámetros y adjunta su certificado digital. Tras intercambiar claves e identificar la clave simétrica de sesión mediante algoritmos como Diffie-Hellman, ambas partes envían el mensaje Finished y comienzan a transmitir datos cifrados de la capa de aplicación.

#### Evaluación de Configuraciones TLS
Para auditar la seguridad de la configuración TLS de un servidor web se utilizan herramientas como testssl.sh (comando CLI exhaustivo ideal para redes internas), Sslyze, la plataforma web SSL Labs y el script de Nmap ssl-enum-ciphers. Los fallos más comunes incluyen la habilitación de TLS 1.0/1.1, el soporte de cifrados RC4 o CBC y el uso de certificados caducados o automandados.

### 4.5 Administracion Remota Segura con Secure Shell (SSH)
El protocolo SSH opera en el puerto TCP 22 y fue creado para reemplazar por completo a Telnet y a las utilidades rsh/rlogin, ofreciendo un canal de administración remota cifrado con garantías de autenticación, confidencialidad e integridad.

#### Métodos de Autenticación
SSH admite múltiples mecanismos para autenticar usuarios:
Autenticación por Contraseña: El usuario envía su clave a través del canal cifrado SSH. Es vulnerable a ataques de fuerza bruta.
Autenticación por Clave Pública: Método altamente recomendado. El auditor genera un par de claves criptográficas compuestas por una clave privada (guardada de forma confidencial en la máquina local) y una clave pública (depositada en el archivo ~/.ssh/authorized_keys del servidor remoto). La clave recomendada actualmente es Ed25519 por su alta velocidad y seguridad superior frente a algoritmos RSA tradicionales.
Autenticación por Certificados SSH: Empleada en entornos corporativos masivos donde una Autoridad de Certificación SSH firma las claves de los usuarios con fechas de caducidad.

#### Verificación de Claves de Host y Transferencia Segura
Cuando un cliente se conecta por primera vez a un servidor SSH, se muestra la huella digital (fingerprint) de la clave pública del host. Tras aceptarla, esta clave se almacena en el archivo local ~/.ssh/known_hosts. Si en conexiones posteriores la clave del host cambia de forma imprevista, SSH interrumpe la conexión e emite una advertencia de seguridad crítica para alertar de un posible ataque Man-in-the-Middle o una reinstalación del servidor.

Para la transferencia segura de archivos sobre SSH se dispone de:
SFTP (SSH File Transfer Protocol): Protocolo interactivo estándar recomendado ejecutado sobre el puerto 22.
SCP (Secure Copy Protocol): Utilidad tradicional en desuso debido a limitaciones de seguridad en su protocolo subyacente.
Rsync sobre SSH: La mejor opción para sincronizar grandes volúmenes de datos al transmitir únicamente los bloques modificados.

#### Endurecimiento de SSH (Hardening)
La configuración del servicio SSH reside en el archivo /etc/ssh/sshd_config. Entre las directivas de seguridad fundamentales destacan:
Desactivar la autenticación por contraseña mediante PasswordAuthentication no para exigir el uso exclusivo de claves públicas.
Desactivar el acceso directo a la cuenta de superusuario especificando PermitRootLogin no.
Restringir el acceso a usuarios específicos mediante AllowUsers o AllowGroups.
Implementar herramientas de monitoreo activo como Fail2ban para bloquear automáticamente direcciones IP tras múltiples intentos fallidos de autenticación.

### 4.6 Ataques a Contraseñas y la Herramienta THC Hydra
La autenticación basada en contraseñas representa el factor tradicional de seguridad "algo que sabes". A pesar de las recomendaciones defensivas, las contraseñas débiles, predecibles o reutilizadas continúan siendo una de las causas principales de compromiso de redes corporativas.

#### Modalidades de Ataques a Contraseñas
Los ataques contra mecanismos de autenticación se dividen en diversas categorías estratégicas:
Ataques de Diccionario: Comprobación sistemática de palabras contenidas en una lista de palabras (Wordlist) como RockYou.txt o colecciones de SecLists.
Ataques de Fuerza Bruta Pura: Generación matemática de todas las combinaciones posibles de caracteres.
Credential Stuffing: Automatización que toma pares de usuario y contraseña filtrados en brechas de datos masivas y los prueba contra cientos de servicios web diferentes, explotando la reutilización de claves por parte de los usuarios.
Password Spraying (Rociado de Contraseñas): Técnica sigilosa que prueba una o dos contraseñas extremadamente comunes (como Verano2024! o Welcome1) contra miles de cuentas de usuario diferentes. Al no acumular intentos fallidos en una sola cuenta, esta técnica elude las políticas de bloqueo de cuentas.

#### Uso Práctico de THC Hydra
THC Hydra es la herramienta estándar y de mayor rendimiento para ejecutar ataques de diccionario y fuerza bruta en línea contra servicios de red que requieran autenticación.

La sintaxis básica de Hydra requiere definir el nombre de usuario o lista de usuarios, la contraseña o lista de contraseñas, la dirección IP del objetivo y el servicio a atacar:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.1.50 ssh -t 4 -V -f
```

Los parámetros más utilizados en Hydra se desglosan a continuación:
-l usuario: Especifica un único nombre de usuario objetivo.
-L archivo_usuarios.txt: Especifica un archivo con una lista de nombres de usuario.
-p contraseña: Pruebas con una única contraseña específica.
-P wordlist.txt: Especifica la ruta hacia el diccionario de contraseñas a probar.
-t hilos: Define el número de conexiones paralelas simultáneas (por ejemplo, -t 4 para SSH o -t 16 para FTP).
-V o -vV: Muestra el progreso de cada intento de autenticación en pantalla.
-f: Ordena a Hydra detener la ejecución inmediatamente al encontrar la primera combinación de credenciales válida.
-s puerto: Especifica un puerto no estándar para el servicio objetivo.

Hydra soporta docenas de protocolos, incluyendo ssh, ftp, pop3, imap, smtp, http-get, http-post-form (para atacar formularios de inicio de sesión web analizando las cadenas de fallo) y smb.

#### Mecanismos de Mitigación Defensiva
Para proteger los servicios ante ataques de credenciales en línea, las organizaciones deben implementar una estrategia de defensa en profundidad basada en las directrices de estándares como NIST SP 800-63B:
Fomentar la longitud sobre la complejidad y bloquear activamente el uso de contraseñas presentes en bases de datos de brechas conocidas.
Implementar políticas de bloqueo temporal de cuenta o retraso exponencial (Rate Limiting / Throttling) tras fallos consecutivos.
Desplegar Autenticación Multifactor (MFA) obligatoria utilizando aplicaciones de autenticación o claves físicas FIDO2/WebAuthn.
Avanzar hacia la autenticación sin contraseña (Passkeys) para eliminar por completo los vectores de ataque basados en credenciales estáticas.
