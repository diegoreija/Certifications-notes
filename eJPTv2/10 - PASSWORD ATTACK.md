<h1>
  <img src="" width="70px" align="absmiddle">
  <span> NMAP</span>
</h1>

---

> Este módulo entra en el libro de jugadas del atacante para robar y descifrar credenciales, abriendo con phishing, el clásico de la ingeniería social que todavía toma desprevenidos a los usuarios cautelosos. A partir de ahí, ejecutará ataques en línea con Hydra, creará listas de palabras personalizadas adaptadas a un objetivo y las usará para descifrar los hashes de contraseña capturados sin conexión. Un desafío de contraseña en vivo cierra el módulo, poniendo cada técnica en una cadena de ataque de credenciales realista.

---

## 🚀 Matriz de Consulta Rápida (Cheat Sheet de Credenciales y Phishing)

| Módulo / Herramienta | Vector o Comando Clave | Parámetros y Configuración Principal | Propósito Operativo |
| :--- | :--- | :--- | :--- |
| **SET (Social Engineering Toolkit)** | `setoolkit` | Option 1 (Social Engineering) $
ightarrow$ Option 2 (Website) $
ightarrow$ Option 3 (Credential Harvester) | Clonación de páginas o importación de HTML para captura de credenciales. |
| **Rainloop & Email Spoofing** | Portal Web Mailer | Alias de remitente: `support@tryaccounting.thm` $
ightarrow$ Target: `bob@tryaccounting.thm` | Envío de correos de phishing con suplantación de identidad en cabeceras. |
| **Hydra (SSH)** | `hydra -l <user> -P <passlist> <IP> -t 4 ssh` | `-l` usuario único, `-P` diccionario, `-t` hilos paralelos | Forzado bruto de credenciales en servicios de acceso remoto SSH. |
| **Hydra (Formulario Web POST)** | `hydra -L <users> -P <passes> <IP> http-post-form "<path>:<body_post>:<fail_str>"` | Marcadores `^USER^` y `^PASS^`, delimitador `:` y condición de fallo `F=` o éxito `S=` | Forzado bruto de formularios de autenticación HTTP POST en aplicaciones web. |
| **CeWL** | `cewl -d 2 -m 3 --lowercase --with-numbers -e -w words.txt <URL>` | `-d` profundidad de araña, `-m` longitud mínima, `-e` extracción de emails | Arañado web para extracción de palabras clave y correos electrónicos corporativos. |
| **Crunch** | `crunch <min> <max> <charset> -t <pattern> -o <output>` | Patrón `-t` con `%%` para dígitos, `@@` para minúsculas, `,` para mayúsculas | Generación combinatoria de listas de palabras basadas en patrones específicos. |
| **ffuf (Enumeración Web)** | `ffuf -u http://<IP>/FUZZ -w <wordlist> -e .php,.html,/ -mc 200,301` | Marcador `FUZZ`, extensiones `-e`, filtro de código de respuesta `-mc` | Descubrimiento de directorios y archivos ocultos en servidores web. |
| **John the Ripper** | `john --format=<format> --wordlist=<list> <hashfile>` | Formato explícito `--format=raw-md5`, muestra de resultados `--show` | Descifrado de hashes mediante CPU con ataque de diccionario o reglas. |
| **Hashcat (Diccionario / Reglas)** | `hashcat -m <mode> -a 0 <hashfile> <wordlist> -r <rulefile>` | Modos: `0` (MD5), `1000` (NTLM), `1400` (SHA-256), `3200` (bcrypt). Regla: `-r best64.rule` | Descifrado de hashes acelerado por GPU mediante diccionarios y mutaciones. |
| **Hashcat (Ataque de Máscara)** | `hashcat -m <mode> -a 3 <hashfile> ?u?l?l?l?d?d?d?d` | Modo `-a 3`, comodines: `?l` minúscula, `?u` mayúscula, `?d` dígito, `?s` símbolo | Fuerza bruta estructurada mediante patrones de caracteres definidos. |

---

<br>

<h2>
  <img src="" width="60px" align="absmiddle">
  <span> Fundamentos de Phishing e Ingeniería Social (Phishing Basics)</span>
</h2>

### 1.1 Definición y Vectores de Ataque
El phishing representa una de las formas de ciberataque más prevalentes y efectivas en el panorama de la seguridad informática, fundamentándose en la manipulación psicológica y la ingeniería social para engañar a los individuos e inducirlos a revelar información confidencial o ejecutar código malicioso en sus sistemas. A diferencia de los vectores de ataque puramente técnicos que explotan vulnerabilidades de software o fallos de configuración, el phishing se dirige explícitamente a las debilidades del factor humano, diseñando narrativas creíbles y aplicando tácticas de presión emocional para que las víctimas comprometan voluntariamente su propia seguridad.

Los canales de distribución del phishing abarcan diversos medios de comunicación cotidiana. El correo electrónico sigue siendo el canal principal debido a su ubicuidad en entornos corporativos, aunque los ataques se extienden con frecuencia a la mensajería SMS (modalidad conocida como smishing), las llamadas telefónicas de voz (denominadas vishing) y la creación de páginas web fraudulentas diseñadas para suplantar interfaces legítimas. Los objetivos fundamentales que persiguen los atacantes mediante estas campañas incluyen la obtención de beneficios financieros directos, el acceso no autorizado a credenciales de inicio de sesión de redes internas, el robo de datos personales o regulados y la infección de dispositivos mediante la entrega e instalación de malware.

En el ámbito de las pruebas de penetración y la consultoría de seguridad ética, la simulación de campañas de phishing resulta imprescindible para evaluar la exposición de una organización ante amenazas de ingeniería social. Mediante el diseño de escenarios controlados que emulan los métodos de atacantes reales pero sin causar daño operativo, los auditores de seguridad pueden medir cuantitativamente la vulnerabilidad del personal interno, identificar departamentos de alto riesgo y establecer métricas concretas sobre la concienciación de los empleados.

<br>

### 1.2 Categorías y Tipos de Phishing
Los ataques de phishing se clasifican en tres categorías principales en función del grado de personalización, el objetivo perseguido y la amplitud del público al que van dirigidos.

El phishing genérico o masivo constituye la forma más tradicional de esta amenaza. Su metodología se basa en la estrategia de lanzar una red amplia, enviando miles de mensajes idénticos a listas masivas de destinatarios. Los pretextos utilizados abordan temas de interés común o rutinario, tales como avisos de facturas pendientes, alertas de caducidad de contraseñas bancarias o notificaciones de paquetes postales. Al carecer de personalización específica, estos correos contienen salulos genéricos o detalles ligeramente desalineados, pero los atacantes compensan la baja tasa de éxito con el elevado volumen de envíos para lograr victorias rápidas a gran escala.

El spear phishing representa un salto cualitativo hacia el ataque dirigido y altamente personalizado. En lugar de emitir mensajes masivos, el atacante investiga minuciosamente a un individuo o a un grupo reducidos dentro de una organización específica, como el departamento de finanzas, recursos humanos o administración de sistemas. El mensaje se redacta utilizando jerga interna, nombres de proyectos reales, referencias a colegas o logotipos corporativos auténticos recopilados previamente mediante reconocimiento. El objetivo del spear phishing es conseguir que la víctima haga clic en un enlace específico, descargue un archivo adjunto modificado o proporcione credenciales corporativas para permitir al atacante establecer un punto de apoyo inicial dentro de la red interna.

La caza de ballenas o whaling es una variante especializada del spear phishing enfocada exclusivamente en altos ejecutivos y tomadores de decisiones de nivel C, tales como directores ejecutivos, directores financieros o miembros de la junta directiva. La distinción fundamental entre el spear phishing convencional y el whaling no radica en las técnicas técnicas empleadas, sino en el perfil del objetivo y el impacto del apalancamiento obtenido. Los ejecutivos poseen autoridad para autorizar transferencias bancarias de gran volumen, anular controles de seguridad internos o acceder a información estratégica y datos altamente regulados, lo que convierte a estos ataques en operaciones de inteligencia altamente sofisticadas y lucrativas.

<br>

### 1.3 Principios de Ingeniería Social y Psicología Humana
Las campañas de phishing exitosas no dependen del azar, sino de la aplicación rigurosa de principios clave de la psicología humana y la ingeniería social que fuerzan a la mente a responder de manera impulsiva, omitiendo el pensamiento crítico o las comprobaciones habituales de seguridad.

La escasez explota la aversión a la pérdida y el miedo a perderse una oportunidad (concepto conocido como FOMO). Al presentar una oferta o recurso como limitado en cantidad o disponibilidad, la víctima siente la necesidad de actuar inmediatamente antes de que la oportunidad desaparezca. Los redactores de phishing emplean frases como plazas limitadas, última oportunidad disponible o acceso exclusivo para desencadenar esta reacción.

La urgencia añade un componente de presión temporal que prioriza la velocidad de respuesta sobre la verificación cuidadosa. Al imponer una fecha límite estricta o amenazar con consecuencias incómodas inmediatas, tales como la suspensión de la cuenta en doce horas o el bloqueo permanente de accesos, el cerebro de la víctima reduce la atención hacia los detalles y ejecuta la acción solicitada para evitar el perjuicio anunciado.

La autoridad aprovecha la tendencia natural de las personas a cumplir las instrucciones emitidas por figuras de poder, expertos o departamentos oficiales. Los atacantes suplantan identidades de directivos de la empresa, administradores del departamento de TI o entidades gubernamentales, incorporando logotipos oficiales, firmas formales y vocabulario corporativo para que la solicitud parezca una orden legítima e inobjetable.

El miedo utiliza la alarma y la amenaza explícita para provocar una reacción defensiva inmediata. Las notificaciones sobre accesos no autorizados a la cuenta, violaciones de seguridad detectadas o posibles acciones legales generan un nivel de ansiedad que anula el escepticismo habitual, empujando al usuario a hacer clic en un enlace de verificación para solucionar el problema reportado.

La curiosidad atrae la atención de la víctima prometiendo información intrigante o relevante que genera un vacío de conocimiento que el cerebro desea cerrar. Asuntos de correo cortos y ambiguos sobre evaluaciones de desempeño, listas de despidos o detalles confidenciales de la empresa inducen al usuario a abrir el mensaje o el archivo adjunto.

La confianza se apoya en la familiaridad hacia marcas consolidadas, plataformas de uso diario o compañeros de trabajo. Al replicar las plantillas de servicios como Microsoft 365, Google Workspace o herramientas internas de la empresa, el mensaje se percibe como seguro por defecto, reduciendo las defensas del usuario ante una solicitud aparentemente rutinaria.

<br>

### 1.4 Manipulación Técnica de URLs y Dominios
En el plano técnico, los atacantes emplean diversas técnicas de manipulación de enlaces y registros de dominio para engañar visualmente al usuario y conseguir que interactúe con infraestructura maliciosa.

El enmascaramiento de URLs consiste en disfrazar la dirección de destino real situando un hipervínculo que muestra un texto o una URL legítima mientras que la etiqueta HTML apunta hacia un servidor malicioso controlado por el atacante. Por ejemplo, el texto visible del enlace puede mostrar la dirección oficial de la empresa mientras que el atributo de navegación dirige al usuario hacia un portal de recolección de credenciales.

Los ataques homógrafos explotan las similitudes visuales entre caracteres de diferentes juegos de texto o alfabetos internacionales (como el cirílico o el griego) que resultan prácticamente indistinguibles a simple vista de los caracteres alfanuméricos latinos. De igual forma, la sustitución de letras por dígitos numéricos visualmente idénticos permite registrar dominios engañosos que pasan desapercibidos durante una lectura rápida.

El typosquatting se basa en el registro deliberado de nombres de dominio que contienen errores tipográficos comunes cometidos por los usuarios al escribir rápidamente en la barra de direcciones del navegador. Al omitir una letra, duplicar un carácter o alterar el orden de las vocales en el nombre de una marca conocida, los atacantes capturan el tráfico de usuarios despistados o alojan páginas de phishing asociadas a campañas de correo electrónico.

El uso de servicios acortadores de URL permite ocultar el dominio final de destino detrás de una cadena aleatoria generada por un proveedor de acortamiento. Esta técnica complica la inspección manual por parte de la víctima y puede superar ciertos filtros básicos de seguridad que no analizan la cadena de redirecciones HTTP completa.

<br>

### 1.5 Suplantación de Correo Electrónico (Email Spoofing)
La suplantación de correo electrónico representa la técnica central para falsificar la identidad del remitente en un mensaje. La vulnerabilidad fundamental radica en el diseño original del protocolo SMTP (Simple Mail Transfer Protocol), el cual fue desarrollado sin mecanismos nativos de autenticación para verificar la veracidad de la dirección indicada en el campo de origen.

La suplantación directa de cabeceras permite modificar los campos del protocolo SMTP para especificar cualquier dirección de correo electrónico deseada en el parámetro de remitente. Si el servidor de correo del dominio suplantado no implementa políticas estrictas de autenticación, la solicitud se procesa y se entrega en la bandeja de entrada de la víctima mostrando la dirección corporativa falsificada.

La suplantación del nombre para mostrar (Display Name Spoofing) es una variante altamente eficaz que aprovecha el comportamiento de las aplicaciones de correo electrónico móviles y clientes de escritorio. En lugar de alterar la dirección de correo real de la cuenta de envío, el atacante modifica únicamente el nombre descriptivo asociado al remitente para que muestre el término Soporte de TI o Dirección General. Debido a que la mayoría de las interfaces móviles muestran exclusivamente el nombre para mostrar en la vista previa, la víctima asume la legitimidad del mensaje sin inspeccionar la dirección real oculta en las cabeceras.

Para combatir estas técnicas, las organizaciones despliegan tres mecanismos defensivos complementarios. El marco SPF (Sender Policy Framework) permite a un dominio publicar en sus registros DNS una lista oficial de las direcciones IP autorizadas para enviar correos en su nombre. El mecanismo DKIM (DomainKeys Identified Mail) añade una firma criptográfica digital a cada correo saliente, permitiendo al servidor receptor verificar que el mensaje no ha sido alterado en tránsito. Finalmente, la política DMARC (Domain-based Message Authentication, Reporting, and Conformance) utiliza SPF y DKIM para determinar qué acciones debe tomar el servidor receptor (como rechazar el correo o enviarlo a spam) cuando un mensaje no supera las validaciones de autenticación.

<br>

### 1.6 Recolección de Credenciales y Mecanismos de Entrega
La recolección de credenciales mediante clonación de aplicaciones web es uno de los objetivos operacionales más comunes en las campañas de phishing. El atacante replica minuciosamente la interfaz gráfica, los logotipos, las tipografías y la estructura HTML del portal de inicio de sesión de la organización objetivo, alojando esta copia en un dominio bajo su control. Cuando la víctima introduce su usuario y contraseña en el formulario falso, la solicitud POST no se dirige al servidor de autenticación legítimo, sino a un script de captura gestionado por el atacante que almacena las credenciales en una base de datos o archivo de registro. Inmediatamente después, el script redirige al usuario hacia el sitio web auténtico, simulando un fallo de conexión rutinario para evitar levantar sospechas.

En cuanto a los mecanismos de entrega de payloads ejecutables, el uso de documentos de Microsoft Office que contienen macros VBA sigue siendo una táctica habitual. El atacante adjunta un archivo de texto o hoja de cálculo habilitada para macros y aplica un pretexto de ingeniería social dentro del propio documento, indicando a la víctima que debe hacer clic en el botón Habilitar contenido para descifrar o visualizar la información. Una vez que el usuario habilita las macros, el código VBA empaquetado se ejecuta en segundo plano, invocando comandos del sistema operativo para descargar y ejecutar código malicioso o establecer una conexión saliente de tipo baliza hacia la infraestructura del auditor.

Las herramientas de software especializadas facilitan la gestión integral de estas campañas. La plataforma GoPhish proporciona una interfaz web completa para diseñar plantillas de correo con editores gráficos, clonar páginas de aterrizaje, gestionar servidores SMTP, programar envíos escalonados y visualizar paneles analíticos con las métricas de la campaña en tiempo real. La herramienta EvilNginx actúa como un proxy inverso man-in-the-middle capaz de eludir la autenticación multifactor (MFA), capturando no solo el nombre de usuario y la contraseña, sino también las cookies de sesión y los tokens de autenticación transmitidos entre la víctima y el servicio legítimo. Por su parte, la suite SET (Social Engineering Toolkit) integra módulos automatizados para clonar sitios web, generar vectores de ataque por correo electrónico y configurar recolectores de credenciales.

<br>

### 1.7 Ciclo de Vida de una Campaña de Phishing
Una campaña de phishing profesional desarrollada en el marco de una auditoría de seguridad requiere seguir una metodología rigurosa estructurada en cinco fases secuenciales.

La fase de planificación y definición del alcance establece los límites operativos de la simulación mediante un acuerdo formal con el cliente. En esta etapa se definen los grupos de usuarios autorizados, las técnicas permitidas, la ventana temporal de envío, los canales de comunicación y las métricas específicas a evaluar, diferenciando entre la tasa de apertura de correos, la tasa de clics en enlaces y la tasa de envío de credenciales. Asimismo, se aprueban las reglas de compromiso, un canal de comunicación de emergencia y un mecanismo de detención inmediata (kill switch) para pausar la campaña ante cualquier imprevisto.

La fase de reconocimiento se enfoca en la recopilación ética de información pública mediante técnicas OSINT sobre la organización objetivo. La inspección del sitio web corporativo, ofertas de empleo, publicaciones en redes sociales profesionales y registros públicos permite identificar la jerga interna, las plataformas tecnológicas utilizadas y las direcciones de correo electrónico de los empleados sin vulnerar la privacidad personal.

La fase de desarrollo de escenarios y payloads transforma la información recopilada en pretextos realistas e inofensivos. Se crean plantillas de correo que imitan comunicaciones internas auténticas y se desarrollan páginas de aterrizaje o adjuntos inocuos que registran interacciones sin desplegar software malicioso ni almacenar credenciales reales en texto plano.

La fase de ejecución implica el envío controlado de los correos electrónicos, ya sea en una sola emisión masiva o en oleadas escalonadas para evitar saturar los controles de seguridad. Durante esta etapa se monitorizan en tiempo real las métricas de interacción y las alertas emitidas por los equipos de seguridad internos del cliente.

La fase de informe y cierre analiza cuantitativa y cualitativamente los resultados obtenidos. Las métricas crudas se traducen en recomendaciones prácticas de seguridad, identificando las debilidades organizativas y proscribiendo la inclusión de nombres individuales en el documento final. El informe propone medidas correctoras concretas, tales como la implementación de autenticación multifactor resistente al phishing, la configuración de registros SPF, DKIM y DMARC, y el despliegue de programas de capacitación continua para el personal.

<br>

### 1.8 Práctica Operativa con SET y Rainloop
En el desarrollo práctico de un escenario de simulación de phishing utilizando la AttackBox, la ejecución comienza estableciendo una sesión SSH hacia la máquina de trabajo. Para configurar el recolector de credenciales mediante el Social Engineering Toolkit (SET), se invoca la herramienta desde la terminal utilizando el alias del sistema y se selecciona sucesivamente la opción uno correspondiente a ataques de ingeniería social, la opción dos para vectores de ataque web y la opción tres para el método de recolector de credenciales.

Al seleccionar la opción de importación personalizada, la herramienta solicita la dirección IP de retorno para el procesamiento de las solicitudes POST y la ruta absoluta del archivo HTML que servirá como interfaz de captura. Tras especificar la URL del servicio objetivo, el servidor web interno de SET inicia su escucha en el puerto 80, quedando preparado para interceptar y registrar cualquier credencial enviada a través del formulario.

Para el envío del correo de phishing utilizando suplantación de identidad, se accede al cliente web de correo Rainloop alojado en el puerto 8080 del entorno de laboratorio. Mediante el uso de alias de correo preconfigurados en la plataforma, el auditor selecciona la dirección corporativa interna de soporte como remitente del mensaje dirigido a la cuenta del objetivo financiero. Al redactar un asunto persuasivo sobre la caducidad inminente de la contraseña corporativa e incluir el enlace que redirige hacia la IP del servidor de SET, el mensaje evade los filtros básicos de seguridad por correo al simular una comunicación interna legítima. Cuando el usuario abre el enlace e introduce sus datos en la página clonada, las credenciales son capturadas en texto plano y mostradas de forma inmediata en la consola de la terminal de SET.

<br>

---

<br>

<h2>
  <img src="" width="60px" align="absmiddle">
  <span> Forzado Bruto en Línea de Servicios (Hydra)</span>
</h2>

### 2.1 Introducción a Hydra
Hydra es la herramienta estándar de la industria para la ejecución de ataques de descifrado de contraseñas en línea mediante la técnica de fuerza bruta y diccionario sobre servicios de autenticación de red. A diferencia de las herramientas de descifrado fuera de línea que procesan hashes almacenados localmente a gran velocidad, Hydra interactúa directamente con los protocolos de red expuestos por un servidor remoto, simulando intentos de inicio de sesión continuos para determinar las credenciales correctas.

La necesidad de emplear herramientas automatizadas como Hydra surge de la imposibilidad de adivinar manualmente combinaciones de usuario y contraseña a través de interfaces de red de forma eficiente. Cuando una aplicación o servicio utiliza credenciales débiles, contraseñas habituales extraídas de filtraciones de datos o claves configuradas por defecto en dispositivos de red e infraestructuras web, Hydra permite iterar diccionarios masivos sobre los formularios de autenticación a gran velocidad.

La herramienta destaca por su versatilidad y soporte multiplataforma, estando disponible de forma nativa en distribuciones orientadas a ciberseguridad como Kali Linux y gestionándose mediante gestores de paquetes tradicionales en sistemas Debian, Ubuntu o Fedora.

<br>

### 2.2 Protocolos Soportados y Riesgos de Credenciales
La capacidad operativa de Hydra abarca una extensa lista de protocolos de red y servicios de aplicaciones web. Entre los protocolos compatibles más comunes se encuentran los servicios de acceso remoto e infraestructura como SSH, FTP, Telnet, RDP, VNC, Rexec y Rlogin; servicios de bases de datos como MySQL, PostgreSQL, MS-SQL, Oracle y MongoDB; protocolos de mensajería y correo electrónico como IMAP, POP3 y SMTP; así como protocolos web HTTP y HTTPS en sus modalidades GET, POST y formularios estructurados.

La efectividad de los ataques con Hydra pone de manifiesto la importancia crítica de abandonar el uso de contraseñas predeterminadas o sencillas. Muchos dispositivos de red, cámaras de videovigilancia CCTV, paneles de administración web y marcos de desarrollo se despliegan inicialmente con combinaciones universales del tipo admin y password. Si estas credenciales no se modifican inmediatamente tras la instalación, cualquier escaneo automatizado conectado a una lista de palabras básica conseguirá acceso no autorizado en cuestión de segundos.

<br>

### 2.3 Parámetros y Sintaxis de Comandos
La ejecución de Hydra se adapta a las particularidades de cada protocolo mediante una serie de parámetros de línea de comandos que definen el comportamiento de la herramienta durante la fase de ataque.

El parámetro `-l` se utiliza para especificar un único nombre de usuario conocido sobre el cual se desea probar la lista de contraseñas. Si se requiere probar múltiples nombres de usuario simultáneamente, se emplea el parámetro `-L` indicando la ruta hacia un archivo de texto que contenga un usuario por línea.

El parámetro `-p` permite probar una única contraseña fija sobre una lista de usuarios, técnica conocida como rociado de contraseñas o password spraying. Para la modalidad tradicional de diccionario, se utiliza el parámetro `-P` seguido de la ruta hacia el archivo que almacena la lista de contraseñas candidatas.

El parámetro `-t` define el número de hilos o tareas paralelas que Hydra ejecutará simultáneamente. Incrementar el número de hilos acelera el proceso de ataque, aunque debe ajustarse con precaución para evitar saturar el ancho de banda de la red o provocar una denegación de servicio en el servidor de destino que bloquee las conexiones salientes.

El parámetro `-s` se utiliza cuando el servicio de destino se encuentra escuchando en un puerto de red no predeterminado, permitiendo especificar explícitamente el número de puerto de conexión. Por su parte, el parámetro `-V` o `-v` activa la salida detallada en pantalla, mostrando cada intento de inicio de sesión realizado en tiempo real.

<br>

### 2.4 Forzado Bruto sobre Servicios de Red (SSH)
Para ilustrar la ejecución de Hydra sobre un servicio de infraestructura como SSH, se configura un comando que toma un nombre de usuario conocido, una lista de palabras de contraseñas y la dirección IP del objetivo.

En la sintaxis del comando para SSH, el auditor especifica el nombre de usuario root mediante el parámetro `-l`, la lista de contraseñas contenida en un archivo de texto mediante el parámetro `-P`, el número de hilos paralelos fijado en cuatro con el parámetro `-t 4`, la dirección IP de la máquina de destino y el nombre del protocolo `ssh` al final de la instrucción.

Al ejecutar la orden, Hydra establece conexiones SSH sucesivas hacia la IP especificada, enviando las credenciales del archivo una a una a través de los cuatro hilos de ejecución. En el momento en que el servidor SSH acepta una combinación de usuario y contraseña, la herramienta detiene el proceso para ese par y muestra la credencial válida descubierta en la consola.

<br>

### 2.5 Forzado Bruto sobre Formularios Web HTTP POST
El ataque contra formularios de autenticación en aplicaciones web requiere una configuración más detallada, ya que la herramienta debe replicar la estructura exacta de la solicitud HTTP transmitida por el navegador.

Antes de construir el comando de Hydra, el auditor debe inspeccionar la solicitud HTTP emitida por la aplicación web utilizando las herramientas de desarrollo del navegador en la pestaña de red. Esta inspección permite identificar el método de envío utilizado (habitualmente POST), la URL exacta del script que procesa el formulario, los nombres de los campos de texto correspondientes al usuario y la contraseña, y la cadena de texto específica que el servidor devuelve cuando la autenticación falla.

El módulo `http-post-form` de Hydra toma como argumento una cadena dividida en tres secciones bien definidas separadas por el carácter de dos puntos. La primera sección especifica la ruta de la página de inicio de sesión. La segunda sección contiene el cuerpo de la solicitud POST con los nombres de los campos del formulario, utilizando las variables especiales de Hydra `^USER^` y `^PASS^` en los lugares donde se inyectarán dinámicamente los valores del diccionario. La tercera sección define la condición de fallo, indicando una cadena de texto presente en la respuesta HTML cuando el inicio de sesión es incorrecto, precedida por el prefijo `F=`. Alternativamente, si se prefiere definir una condición de éxito, se especifica la cadena presente al iniciar sesión precedida por el prefijo `S=`.

Un ejemplo concreto de comando para forzar un formulario web POST incluye el parámetro `-l` con el usuario objetivo, el parámetro `-P` con el archivo de contraseñas, la dirección IP del servidor web, el módulo `http-post-form` y la cadena de especificación del formulario. Durante la ejecución, Hydra sustituye las variables de usuario y clave en cada petición HTTP POST, analizando la respuesta devuelta por el servidor web hasta identificar la combinación que no genera el mensaje de fallo configurado.

<br>

---

<br>

<h2>
  <img src="" width="60px" align="absmiddle">
  <span> Introducción y Generación de Listas de Palabras (Wordlists)</span>
</h2>

### 3.1 Definición y Usos de las Listas de Palabras
Una lista de palabras o wordlist es un archivo de texto plano estructurado de manera simple, donde cada línea contiene una palabra, frase, contraseña, nombre de usuario, parámetro o ruta de archivo potencial. En la disciplina de la ciberseguridad y las pruebas de penetración, estas listas constituyen la materia prima para automatizar procesos de prueba y adivinación que resultarían inviables de forma manual.

La utilidad de las listas de palabras se extiende a múltiples etapas de una evaluación de seguridad. En el ámbito del descifrado de contraseñas fuera de línea, herramientas como John the Ripper y Hashcat procesan archivos de hashes comparándolos contra millones de palabras por segundo. En el forzado bruto de servicios de autenticación de red, herramientas como Hydra y Medusa introducen pares de credenciales en formularios y servicios remotos. En la fase de descubrimiento de contenido web, escáneres de directorios como Gobuster, ffuf y DirBuster leen listas de nombres de archivos y carpetas estándar para localizar recursos ocultos, archivos de respaldo o paneles administrativos en servidores web. Asimismo, en la fase de reconocimiento de infraestructura, herramientas de enumeración DNS utilizan listas de palabras para descubrir subdominios no enlazados públicamente.

<br>

### 3.2 Fuentes Comunes y Repositorios Especializados
Las listas de palabras utilizadas en auditorías de seguridad se dividen entre repositorios prefabricados de uso universal y listas personalizadas adaptadas a objetivos específicos.

Entre las listas prefabricadas más destacadas se encuentra `rockyou.txt`, un archivo distribuido de forma nativa en sistemas Kali Linux que contiene más de catorce millones de contraseñas reales obtenidas durante la filtración de datos de la plataforma RockYou en el año 2009. Debido a que refleja hábitos de elección de contraseñas de usuarios reales, sigue siendo la lista de primer paso obligatoria para ataques de diccionario.

Otro recurso imprescindible es el repositorio `SecLists`, una vasta colección organizada modularmente que abarca listas de nombres de usuario, contraseñas habituales, estructuras de directorios web, rutas de API, nombres de subdominios DNS y payloads de ataque para diversas vulnerabilidades. SecLists incluye archivos específicos diseñados para escenarios concretos, como listas de archivos comunes en servidores web, listas cortas de contraseñas de alto rendimiento y diccionarios adaptados a plataformas tecnológicas específicas.

<br>

### 3.3 Listas Personalizadas y Generación Combinatoria con Crunch
A pesar de la utilidad de las listas prefabricadas universales, las listas de palabras personalizadas adaptadas al contexto de la organización objetivo ofrecen tasas de éxito considerablemente superiores. Un diccionario personalizado incorpora términos específicos de la empresa, nombres de productos internos, tecnologías de la infraestructura, nombres de ejecutivos, ubicaciones geográficas y referencias culturales locales, reduciendo el ruido operativo y concentrando los intentos de adivinación en patrones altamente plausibles.

Cuando se conoce la estructura fija o el patrón de una contraseña pero se requiere agotar todas las combinaciones posibles dentro de un conjunto de caracteres, se emplean generadores combinatorios automatizados como Crunch. Esta herramienta permite especificar la longitud mínima y máxima de las cadenas a generar, así como el juego de caracteres permitido, produciendo un archivo de texto que agota el espacio de búsqueda definido.

El uso de Crunch requiere definir la longitud inicial, la longitud final, el conjunto de caracteres y, opcionalmente, la ruta del archivo de salida mediante el parámetro `-o`. La herramienta permite definir patrones estructurados mediante el parámetro `-t`, utilizando marcadores de posición especiales como el símbolo de porcentaje para representar dígitos numéricos, la arroba para letras minúsculas, la coma para letras mayúsculas y el acento circunflejo para símbolos especiales. Aunque los generadores combinatorios son altamente efectivos para espacios restringidos, deben emplearse con criterio debido a que la generación de cadenas de longitud elevada produce archivos de tamaño masivo que pueden agotar el almacenamiento en disco.

<br>

### 3.4 Recopilación de Información mediante OSINT (Cosecha de Datos)
La construcción de un diccionario personalizado comienza con la fase de recopilación de información pública sobre el objetivo a través de fuentes OSINT (Open Source Intelligence).

Las redes sociales profesionales representan una fuente primaria de información. La inspección de perfiles de empleados y ofertas de empleo permite identificar los nombres reales del personal, las estructuras de correo electrónico corporativo y las tecnologías específicas empleadas en la infraestructura interna. La presencia de términos de marcos de desarrollo o herramientas en las ofertas de trabajo proporciona pistas concretas para agregar nombres de directorios o parámetros a las listas de descubrimiento.

El sitio web corporativo de la organización, los comunicados de prensa y las cuentas oficiales en redes sociales revelan nombres de proyectos, marcas registradas, ubicaciones de oficinas y eslóganes corporativos que los usuarios suelen emplear como raíz para sus contraseñas o nombres de carpetas en servidores web.

Las búsquedas WHOIS sobre los dominios registrados y la consulta de registros de transparencia de certificados en plataformas públicas permiten descubrir subdominios históricos, nombres de servidores DNS y direcciones de correo electrónico de contacto del personal técnico de la entidad.

<br>

### 3.5 Herramientas de Extracción Técnica (CeWL y Procesamiento de Documentos)
Para automatizar la extracción de palabras clave directamente desde la infraestructura web del objetivo, se utilizan herramientas de arañado y procesamiento de archivos.

La herramienta CeWL es un script desarrollado en Ruby que realiza un rastreo sobre una URL especificada, extrayendo todas las palabras únicas presentes en las páginas web navegadas. CeWL permite configurar la profundidad del rastreo mediante el parámetro `-d`, la longitud mínima de las palabras a extraer mediante el parámetro `-m`, e incluye opciones para convertir todos los términos a minúsculas, procesar números integrados en las palabras y extraer direcciones de correo electrónico encontradas mediante la bandera `-e`, guardando los resultados en archivos de texto independientes.

Además del rastreo web, la descarga recursiva de archivos de documentación pública alojados en el servidor (como manuales o informes en formato PDF) permite extraer cadenas de texto legibles y direcciones de correo de contacto presentes en los metadatos o pies de página de los documentos.

Mediante el uso de comandos del sistema operativo como `grep` con expresiones regulares avanzadas, es posible filtrar los correos electrónicos extraídos, eliminar duplicados con `sort -u` y aislar la parte local del usuario eliminando el dominio. Posteriormente, utilizando la herramienta `awk`, se pueden generar automáticamente permutaciones habituales de nombres de usuario basadas en el nombre y apellido de los empleados, tales como el formato de nombre punto apellido, inicial del nombre seguida del apellido o nombre seguido de la inicial del apellido.

<br>

### 3.6 Limpieza, Normalización y Uso Práctico con ffuf
Las listas de palabras recién extraídas de fuentes web o documentos contienen ruido operacional, duplicados, caracteres especiales no válidos, inconsistencias de mayúsculas y cadenas excesivamente cortas o largas. El proceso de limpieza y normalización transforma estas listas desordenadas en diccionarios optimizados de alto rendimiento.

La normalización implica combinar los diferentes archivos de palabras extraídos en un único archivo consolidado, ordenar alfabéticamente las líneas y eliminar todas las entradas duplicadas utilizando el comando `sort -u`. A continuación, se emplean herramientas de transformación de texto como `tr` para convertir todos los caracteres a minúsculas y eliminar los retornos de carro procedentes de archivos creados en entornos Windows. Finalmente, se aplican expresiones regulares con `grep` para filtrar cadenas que no comiencen por caracteres alfanuméricos o que no alcancen una longitud mínima de seguridad.

Una vez generadas las listas limpias, se procede a su utilización en herramientas de enumeración como ffuf para descubrir directorios y archivos ocultos en el servidor web. El comando de ffuf utiliza el parámetro `-w` para especificar la lista de palabras limpia, el parámetro `-u` indicando la URL del objetivo con el marcador de posición `FUZZ`, el parámetro `-e` para probar extensiones de archivo habituales como `.php` o `.html`, y el parámetro `-mc` para filtrar únicamente aquellas respuestas que devuelvan códigos de estado HTTP satisfactorios o de redirección. Tras identificar rutas de acceso ocultas o paneles de administración, el auditor utiliza las listas de nombres de usuario y contraseñas limpias desarrolladas previamente para ejecutar el forzado bruto del formulario mediante Hydra.

<br>

---

<br>

<h2>
  <img src="" width="60px" align="absmiddle">
  <span> Descifrado de Contraseñas y Hashes (Password Cracking)</span>
</h2>

### 4.1 Almacenamiento Seguro de Contraseñas y Funciones Hash
El almacenamiento de contraseñas en texto plano dentro de una base de datos representa un fallo de seguridad catastrófico. Ante cualquier acceso no autorizado o filtración de la base de datos, todas las cuentas de los usuarios quedan expuestas de forma inmediata sin necesidad de procesamiento técnico adicional. La solución estándar en la ingeniería de software consiste en almacenar un resumen criptográfico o hash de la contraseña en lugar de la clave original.

Cuando un usuario intenta autenticarse en el sistema, la aplicación calcula el hash de la contraseña introducida en el formulario y lo compara contra la cadena de hash almacenada en la base de datos. Si ambos valores coinciden exactamente, se concede el acceso al sistema, garantizando que la contraseña original en texto plano no necesita ser guardada en ningún componente de la infraestructura.

Para que una función hash sea criptográficamente adecuada para el almacenamiento de contraseñas, debe cumplir con cuatro propiedades fundamentales.

La unidireccionalidad establece que debe ser computacionalmente imposible revertir un hash generado para recuperar la cadena de entrada original. La única forma de determinar qué texto produjo un hash específico es mediante un proceso prospectivo: aplicar la función hash sobre candidatos potenciales y comparar los resultados.

El determinismo garantiza que la misma entrada exacta siempre producirá la misma salida idéntica independientemente del sistema o del momento en que se calcule la función.

La longitud fija de salida determina que la cadena resultante poseerá siempre el mismo tamaño en bits y caracteres hexadecimales, sin importar si la entrada es una palabra corta de cuatro letras o un documento de miles de caracteres.

La resistencia a colisiones establece que debe ser impracticable encontrar dos entradas diferentes que generen exactamente la misma cadena de salida hash. Cuando un algoritmo pierde esta propiedad, se considera roto y obsoleto para uso en seguridad.

<br>

### 4.2 Algoritmos de Hashing Comunes y Salación (Salting)
Los algoritmos de hashing han evolucionado significativamente en respuesta al incremento en la capacidad de cómputo de los equipos modernos.

Algoritmos antiguos como MD5 (salida de 128 bits / 32 caracteres hexadecimales) y SHA-1 (salida de 160 bits / 40 caracteres hexadecimales) fueron diseñados originalmente para verificaciones rápidas de integridad de archivos y firmas digitales. Debido a su elevada velocidad de procesamiento y a la existencia de colisiones conocidas, se consideran completamente obsoletos y vulnerables. Algoritmos como SHA-256 ofrecen mayor resistencia, pero al ser funciones de cálculo rápido, una unidad de procesamiento gráfico (GPU) moderna puede calcular miles de millones de intentos por segundo.

En sistemas Windows tradicionales, el algoritmo NTLM (basado en MD4) sigue utilizándose para la autenticación local y de dominio, produciendo cadenas de 32 caracteres hexadecimales altamente vulnerables al descifrado por diccionario.

Por el contrario, algoritmos modernos como bcrypt y Argon2 fueron diseñados específicamente para el almacenamiento seguro de contraseñas. bcrypt genera cadenas de aproximadamente 60 caracteres identificables por prefijos como `$2a$`, `$2b$` o `$2y$`, e incorpora un factor de costo configurable que ralentiza deliberadamente el cálculo del hash, obligando a las herramientas de ataque a gastar un tiempo considerable por cada candidato probado.

La salación o salting resuelve el problema de las contraseñas idénticas y las tablas precomputadas. Si dos usuarios eligen la misma contraseña en una aplicación que no utiliza sal, sus hashes almacenados serán idénticos, permitiendo a un atacante utilizar tablas arcoíris (rainbow tables) para buscar coincidencias instantáneas. Una sal es una cadena aleatoria única generada para cada usuario que se combina con su contraseña antes de calcular el hash. Debido a que cada usuario posee una sal diferente almacenada en la base de datos junto al hash, dos contraseñas idénticas producen hashes completamente distintos, volviendo inútiles las tablas precomputadas.

<br>

### 4.3 Identificación de Tipos de Hash
Antes de iniciar cualquier ataque de descifrado fuera de línea, es imprescindible identificar con precisión el algoritmo que produjo la cadena hash. Especificar el modo o formato incorrecto en las herramientas de descifrado provocará que el ataque falle por completo sin obtener resultados.

La identificación se realiza analizando la longitud en caracteres, el juego de texto y los prefijos característicos de la cadena. Un hash hexadecimal de 32 caracteres sin prefijo corresponde habitualmente a MD5 o NTLM. En este caso, el contexto de extracción resuelve la ambigüedad: un hash extraído de la base de datos SAM de Windows o de Active Directory es inequívocamente NTLM, mientras que uno obtenido de una aplicación web suele ser MD5. Un hash hexadecimal de 40 caracteres corresponde a SHA-1, mientras que uno de 64 caracteres pertenece a SHA-256. Los hashes de bcrypt se identifican de forma inmediata por su longitud de 60 caracteres y sus prefijos `$2a$`, `$2b$` o `$2y$`, seguidos del número de costo de procesamiento.

Para automatizar la identificación, se emplea la herramienta `hashid` desde la línea de comandos, la cual analiza la estructura de la cadena e imprime una lista de algoritmos compatibles. De igual forma, la herramienta Hashcat incluye el parámetro `--identify`, permitiendo analizar un archivo de hashes y devolver directamente los números de modo correspondientes listos para ser copiados en los comandos de ataque.

<br>

### 4.4 Formatos de John the Ripper y Modos de Hashcat
Tanto John the Ripper como Hashcat requieren la traducción del algoritmo identificado a su nomenclatura interna de comandos.

En John the Ripper, el formato se especifica mediante el parámetro `--format=`. Algunos de los formatos más comunes incluyen `raw-md5` para hashes MD5 simples, `raw-sha1` para SHA-1, `raw-sha256` para SHA-256, `nt` para hashes NTLM de Windows y `bcrypt` para funciones bcrypt.

En Hashcat, el algoritmo se define mediante el parámetro de modo `-m` seguido de un número entero fijo. Los modos principales corresponden a `-m 0` para MD5, `-m 100` para SHA-1, `-m 1000` para NTLM, `-m 1400` para SHA-256, `-m 1700` para SHA-512 y `-m 3200` para bcrypt.

<br>

### 4.5 Estrategias de Ataque: Diccionario, Fuerza Bruta, Reglas y Máscaras
El éxito del descifrado fuera de línea depende de seleccionar la estrategia de ataque adecuada en función de la información disponible y la capacidad de cómputo.

El ataque de diccionario consiste en probar una lista preconstruida de contraseñas candidatas contra el archivo de hashes, una por una. Es el método más rápido y siempre debe constituir la primera línea de ataque. Se emplean diccionarios como `rockyou.txt` o colecciones masivas actualizadas como `RockYou2024`, que compila miles de millones de claves reales extraídas de filtraciones históricas.

El ataque de fuerza bruta pura genera todas las combinaciones posibles de caracteres dentro de una longitud determinada. Aunque garantiza recuperar la contraseña si se dispone de tiempo ilimitado, resulta impracticable para longitudes superiores a seis o siete caracteres debido al crecimiento exponencial del espacio de búsqueda.

El ataque basado en reglas combina la rapidez del diccionario con mutaciones dinámicas. A partir de una lista de palabras base, la herramienta aplica reglas de transformación que simulan los hábitos de modificación de los usuarios, tales como poner en mayúscula la primera letra, añadir dígitos al final, incluir caracteres especiales o realizar sustituciones leet (como cambiar la letra a por el símbolo arroba). En Hashcat, las reglas se aplican mediante el parámetro `-r` especificando archivos como `best64.rule` o `rockyou-30000.rule`.

El ataque de máscara es una fuerza bruta estructurada en la que se define el patrón o plantilla fija de la contraseña mediante comodines de conjuntos de caracteres. La sintaxis de máscaras utiliza los marcadores `?l` para letras minúsculas, `?u` para letras mayúsculas, `?d` para dígitos numéricos, `?s` para símbolos especiales y `?a` para todos los caracteres imprimibles. Definir un patrón conocido reduce drásticamente el número de candidatos a probar frente a una fuerza bruta no restringida.

<br>

### 4.6 Ejecución Práctica con John the Ripper y Hashcat
Para ejecutar un ataque de diccionario básico con John the Ripper, el auditor guarda las cadenas de hashes en un archivo de texto y ejecuta el comando especificando el parámetro `--format=` con el tipo de hash adecuado, el parámetro `--wordlist=` con la ruta del diccionario y la ruta del archivo de hashes al final. Para consultar las contraseñas descifradas previamente en el archivo histórico de resultados (potfile), se utiliza la orden con el parámetro `--show` indicando el mismo formato.

En el caso de Hashcat, la herramienta aprovecha la aceleración por hardware mediante GPU para procesar miles de millones de combinaciones por segundo. Para ejecutar un ataque de diccionario básico, se utiliza el parámetro de modo `-m` con el número del algoritmo, el parámetro de tipo de ataque `-a 0` para modo diccionario, la ruta del archivo de hashes y la ruta del archivo de la lista de palabras.

Si se desea aplicar un ataque basado en reglas en Hashcat, se añade el parámetro `-r` especificando la ruta de la regla deseada. Para configurar un ataque de máscara estructurado, se cambia el modo de ataque a `-a 3` mediante el parámetro correspondiente y se sustituye la lista de palabras por la cadena de patrones de máscara deseada.

Durante ejecuciones extensas, Hashcat permite gestionar el nombre de la sesión de trabajo mediante el parámetro `--session=` e interrumpir o reanudar el progreso en cualquier momento utilizando el parámetro `--restore`, garantizando la continuidad de la prueba sin perder el cómputo procesado.
