<h1>
  <img src="https://cdn-images.tryhackme.com/modules/owasp-top-10-2025-1779873028221.svg" width="70px" align="absmiddle">
  <span> Metasploit y Explotación</span>
</h1>

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/owasptopten2025one-1785241451372.png" width="65x  px" align="absmiddle">
  <span> OWASP Top 10 2025: IAAA Failures</span>
</h2>

Para controlar la presencia y las acciones de los usuarios, los sistemas utilizan el modelo **IAAA**, el cual funciona como una escalera de cuatro niveles donde no es posible saltarse ningún peldaño.
* **Identidad**: representa la cuenta única (como un correo electrónico o nombre de usuario) que identifica a una persona o servicio.
* **Autenticación**: es la comprobación o prueba de esa identidad, realizada mediante contraseñas, códigos OTP o claves.
* **Autorización**: determina los permisos específicos y las acciones que esa identidad tiene permitido realizar.
* **Responsabilidad (** **Accountability** **)**: consiste en registrar y alertar sobre quién hizo qué, cuándo lo hizo y desde dónde.

Cuando este modelo falla en una aplicación, surgen tres problemas graves. El primero es el **control de acceso roto**, que ocurre cuando el servidor no comprueba correctamente si un usuario tiene permiso para ver o modificar un recurso[4]. Un ejemplo común es el fallo IDOR, donde una persona simplemente altera un número en la dirección web (por ejemplo, cambiando `?id=7` por `?id=6`) y logra consultar o modificar los datos privados de otro usuario porque la aplicación confió en la petición del cliente sin verificarla en el servidor.

El segundo problema son los **fallos de autenticación**, que suceden cuando el sistema no puede validar la identidad de forma confiable, permitiendo a los atacantes adivinar contraseñas débiles por falta de límites de intentos o crear cuentas ambiguas para ingresar en perfiles administrativos ajenos.

El tercer problema son los **fallos de registro y alerta**, los cuales se presentan cuando la aplicación no guarda registros de los eventos de seguridad; si ocurre un ataque, los defensores quedan a ciegas y no pueden investigar lo sucedido ni determinar responsabilidades porque faltan los datos históricos.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/owasptopten2025one-1785241451372.png" width="65x  px" align="absmiddle">
  <span> Defectos de Diseño en la Aplicación</span>
</h2>

Incluso cuando el código individual no tiene errores sintácticos, la seguridad puede colapsar si la arquitectura o la configuración del sistema se diseñaron con fallos desde el origen.

En primer lugar se encuentran las **configuraciones erróneas de seguridad**, las cuales se producen al desplegar aplicaciones o servidores con valores predeterminados inseguros, permisos demasiado abiertos o servicios innecesarios expuestos a Internet. Esto quedó demostrado en 2017 cuando Uber expuso un depósito de almacenamiento en la nube AWS S3 de respaldo de acceso público, lo que permitió que usuarios externos descargaran información confidencial de conductores y pasajeros directamente sin necesidad de credenciales.

En segundo lugar están los **fallos en la cadena de suministro de software**, los cuales se originan porque las aplicaciones modernas se construyen utilizando bibliotecas, componentes y modelos de inteligencia artificial de terceros. Si uno de estos componentes externos está comprometido o no se verifica adecuadamente, todo el sistema se vuelve vulnerable; un ejemplo relevante fue el ataque a SolarWinds Orion en 2021, donde los atacantes inyectaron código malicioso dentro de una actualización legítima y distribuida oficialmente.

En tercer lugar se ubica el **diseño inseguro**, que ocurre cuando la lógica de la aplicación contiene fallos estructurales por no haber realizado un modelado de amenazas durante su creación. Un ejemplo claro fue la aplicación Clubhouse, cuya API de *backend* permitía consultar datos de usuarios y conversaciones privadas directamente porque sus desarrolladores asumieron que el público solo interactuaría a través de la aplicación móvil y omitieron la autenticación en las consultas directas. Además, en la era de la inteligencia artificial, el diseño inseguro se manifiesta en riesgos como la inyección de indicaciones (*prompt injection*), donde el texto ingresado por el usuario se mezcla con las instrucciones del sistema, o en la confianza ciega en los resultados generados por modelos de IA sin supervisión humana.

<br>

---

<br>

<h2>
  <img src="https://cdn-images.tryhackme.com/room-icons/owasptopten2025one-1785241451372.png" width="65x  px" align="absmiddle">
  <span> Manejo Inseguro de la Información</span>
</h2>

La última dimensión analiza cómo la aplicación procesa, protege y valida los datos que maneja en su interior.

El primer pilar son los **fallos criptográficos**, los cuales ocurren cuando la información confidencial no se cifra o se protege mediante algoritmos débiles u obsoletos como MD5 o SHA-1. Un error frecuente en esta área es intentar "diseñar un algoritmo de cifrado propio" en lugar de utilizar bibliotecas estándar de la industria, o almacenar contraseñas en texto claro en lugar de procesarlas con funciones de *hash* robustas como bcrypt, scrypt o Argon2.

El segundo pilar es la **inyección**, uno de los fallos más clásicos y peligrosos en el desarrollo web. Sucede cuando la aplicación toma datos ingresados por el usuario sin filtrarlos ni desinfectarlos y los utiliza directamente para construir comandos o consultas procesadas por un intérprete, como una base de datos SQL, el shell del sistema operativo o un modelo de IA. Para prevenir la inyección, se debe tratar siempre la entrada del usuario como no confiable, utilizando consultas parametrizadas o declaraciones preparadas en lugar de concatenar texto directamente en las instrucciones.

El tercer pilar corresponde a los **fallos de integridad en el software o los datos**, que suceden cuando el sistema confía en código, archivos de configuración o datos recibidos sin verificar si han sido alterados. Para solucionar este riesgo, las aplicaciones deben definir límites de confianza estrictos y aplicar comprobaciones criptográficas, tales como sumas de comprobación en los paquetes de actualización y controles de verificación dentro de los procesos de integración y despliegue continuo (CI/CD)
