# 🥒 Prueba 2 — Gestión de Riesgos en Redes Corporativas

### Auditoría Técnica, Evaluación de Vulnerabilidades y Plan de Mitigación — Máquina "Pickle Rick"

![Asignatura](https://img.shields.io/badge/Asignatura-Gesti%C3%B3n%20de%20Riesgos%20en%20Redes%20Corporativas-005073?style=for-the-badge&logo=cisco&logoColor=white)
![Estado](https://img.shields.io/badge/Evaluaci%C3%B3n-100%25%20Completada-success?style=for-the-badge)

---

## 👤 Datos Académicos

| Campo | Detalle |
|:------|:--------|
| **Institución** | DUOC UC |
| **Asignatura** | Gestión de Riesgos en Redes Corporativas |
| **Docente** | Jaime Andrés Gómez González |
| **Evaluación** | Prueba Práctica N° 2 (Auditoría Técnica y Evaluación de Riesgos) |
| **Entorno de Prueba** | Máquina "Pickle Rick" — TryHackMe (Servidor Linux Ubuntu) |
| **Integrante** | Lisandra González |
| **Fecha de Entrega** | Noviembre 2026 |

---

## 🎯 Resumen Ejecutivo y Alcance

El presente informe documenta la auditoría de seguridad, evaluación de vulnerabilidades y análisis de riesgos realizada sobre un servidor web corporativo (máquina **"Pickle Rick"**), en el contexto de la **Prueba 2** de la asignatura.

Siguiendo el estándar **NIST SP 800-30** (*Guide for Conducting Risk Assessments*) y la metodología **PTES** (*Penetration Testing Execution Standard*), el trabajo abarcó:

1. **Identificar la superficie de ataque expuesta** mediante escaneo de puertos y servicios.
2. **Enumerar la aplicación web** y detectar la divulgación de credenciales en el código fuente y en archivos de configuración.
3. **Evaluar una consola de comandos web sin sanitización** que permite la ejecución remota de comandos (RCE).
4. **Verificar una configuración permisiva de privilegios (`sudoers`)** que posibilita el compromiso total del sistema.
5. **Formular una matriz formal de riesgos** y proponer controles de mitigación alineados con **ISO/IEC 27001**.

El objetivo del escenario consistió en recuperar los **tres ingredientes secretos**, que en términos de auditoría representan la extracción de información confidencial en tres niveles de privilegio distintos (aplicación, usuario y superusuario).

> **Resultado:** la auditoría tuvo éxito en los tres niveles, evidenciando un servidor con vulnerabilidades críticas encadenables que permiten pasar de un visitante anónimo a control total (`root`) del sistema.

---

## 🗺️ Topología y Flujo Metodológico

```
Fase 1: Reconocimiento          ->  Nmap (22/SSH, 80/HTTP)
Fase 2: Enumeración Web         ->  Código fuente + robots.txt (usuario y contraseña expuestos)
Fase 3: Acceso Inicial          ->  login.php  (R1ckRul3s : Wubbalubbadubdub)
Fase 4: RCE                     ->  Command Panel -> www-data -> 1er ingrediente
Fase 5: Enumeración Local       ->  /home/rick -> 2do ingrediente
Fase 6: Escalamiento (root)     ->  sudo NOPASSWD: ALL -> /root -> 3er ingrediente
```

---

## 🌐 Inventario de Activos y Servicios

| Puerto | Protocolo | Estado | Servicio | Software / Versión | Riesgo Inherente |
|:------:|:---------:|:------:|:--------:|:------------------:|:----------------:|
| **22** | TCP | `open` | SSH | OpenSSH 8.2p1 (Ubuntu) | 🟡 Medio (servicio administrativo expuesto) |
| **80** | TCP | `open` | HTTP | Apache httpd 2.4.41 (Ubuntu) | 🔴 Crítico (aloja portal con RCE y credenciales expuestas) |

> **Nota:** el vector de intrusión **no fue el servicio SSH**, sino la aplicación web del puerto 80. El sitio opera sobre **HTTP sin cifrado**, por lo que toda credencial transita en texto claro y es susceptible de intercepción.

---

## 🔍 Desarrollo Paso a Paso de la Auditoría

### Paso 1: Reconocimiento y Escaneo Perimetral

**Comando ejecutado:**

```bash
nmap -sV -T4 10.65.170.111
```

**Evidencia de salida:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

![Figura 1 - Escaneo Nmap](img/01-nmap.png)
*Figura 1. Escaneo Nmap: se detectan los puertos 22/SSH y 80/HTTP abiertos.*

---

### Paso 2: Enumeración de Superficie Web

**a) Análisis del código fuente (Ctrl + U):**

```html
<!--
  Note to self, remember username!
  Username: R1ckRul3s
-->
```

![Figura 2 - Usuario expuesto](img/02-source-username.png)
*Figura 2. Comentario del desarrollador en el HTML que filtra el usuario `R1ckRul3s`.*

**b) Revisión del archivo `robots.txt`:**

```
Wubbalubbadubdub
```

![Figura 3 - robots.txt](img/03-robots.png)
*Figura 3. El archivo `robots.txt` se usó indebidamente para almacenar una cadena usada como contraseña.*

**Credenciales reconstruidas:**
- **Usuario:** `R1ckRul3s`
- **Contraseña:** `Wubbalubbadubdub`

---

### Paso 3: Evaluación de Credenciales y Acceso Inicial

```
URL:        http://10.65.170.111/login.php
Usuario:    R1ckRul3s
Contraseña: Wubbalubbadubdub
```

Autenticación exitosa y redirección a `portal.php`. Solo la sección **Commands** habilita un panel interactivo funcional.

![Figura 4 - Command Panel](img/04-portal.png)
*Figura 4. Acceso exitoso al panel autenticado (`portal.php`), sección Command Panel.*

---

### Paso 4: Ejecución Remota de Comandos y Primer Ingrediente

El panel ejecuta instrucciones directamente en el sistema operativo sin sanitización: vulnerabilidad de **Ejecución Remota de Comandos (RCE)**.

```bash
ls -la
```

![Figura 5 - Listado del directorio web](img/05-ls-webroot.png)
*Figura 5. Listado del directorio web: se identifica `Sup3rS3cretPickl3Ingred.txt`.*

> **Control superficial:** el panel bloquea el comando `cat` (devuelve "Command disabled"). Es un filtro por texto, evadible con `less`, `head`, `tail` o `grep`.

```bash
grep -i "." Sup3rS3cretPickl3Ingred.txt
```

```
mr. meeseek hair
```

![Figura 6 - Primer ingrediente](img/06-first-ingredient.png)
*Figura 6. Lectura del primer ingrediente con `grep` (el filtro de `cat` se evade usando otro binario).*

🥒 **Primer ingrediente obtenido:** `mr. meeseek hair`

---

### Paso 5: Enumeración Local y Segundo Ingrediente

```bash
ls -la /home/rick
```

![Figura 7 - /home/rick](img/07-home-rick.png)
*Figura 7. En `/home/rick` se encuentra el archivo `second ingredients` (el nombre contiene un espacio).*

```bash
grep -i "." "/home/rick/second ingredients"
```

```
1 jerry tear
```

![Figura 8 - Segundo ingrediente](img/08-second-ingredient.png)
*Figura 8. Lectura del segundo ingrediente en `/home/rick/second ingredients`.*

🥒 **Segundo ingrediente obtenido:** `1 jerry tear`

---

### Paso 6: Escalamiento de Privilegios y Tercer Ingrediente (Root)

```bash
sudo -l
```

```
User www-data may run the following commands on this host:
    (ALL) NOPASSWD: ALL
```

![Figura 9 - sudo -l](img/09-sudo-l.png)
*Figura 9. `sudo -l` confirma la directiva crítica `(ALL) NOPASSWD: ALL` para la cuenta `www-data`.*

> ⚠️ **Fallo crítico en el Principio de Menor Privilegio:** la directiva `(ALL) NOPASSWD: ALL` autoriza a `www-data` a ejecutar cualquier comando como `root` sin contraseña. Como esa cuenta ya estaba comprometida vía RCE, se obtiene control total del servidor.

```bash
sudo less /root/3rd.txt
```

```
3rd ingredients: fleeb juice
```

![Figura 10 - Tercer ingrediente](img/10-third-ingredient.png)
*Figura 10. Lectura del tercer ingrediente con privilegios de root: control total del servidor.*

🥒 **Tercer ingrediente obtenido:** `fleeb juice`

---

## 📝 Cuestionario Oficial y Respuestas

| N° | Pregunta Evaluada | Respuesta Correcta | Justificación Técnica |
|:--:|:------------------|:------------------:|:----------------------|
| 1 | ¿Cuál es el primer ingrediente? | **`mr. meeseek hair`** | Archivo `Sup3rS3cretPickl3Ingred.txt` en la raíz web, leído vía RCE. |
| 2 | ¿Cuál es el segundo ingrediente? | **`1 jerry tear`** | Archivo `second ingredients` ubicado en `/home/rick`. |
| 3 | ¿Cuál es el último y definitivo? | **`fleeb juice`** | Archivo `3rd.txt` en `/root`, accesible por escalamiento vía `sudo`. |

---

## 📊 Matriz de Riesgos (NIST SP 800-30)

| ID | Activo Afectado | Vulnerabilidad | Amenaza | Prob. | Imp. | Nivel de Riesgo | Tratamiento |
|:--:|:----------------|:---------------|:--------|:-----:|:----:|:---------------:|:------------|
| R-01 | Aplicación Web | Panel de comandos sin sanitización (RCE) | Ejecución arbitraria y toma de control | 5 | 5 | 🔴 Crítico (25) | Eliminar la consola web; usar APIs con listas blancas, nunca una shell. |
| R-02 | Sistema Operativo | `sudo (ALL) NOPASSWD: ALL` sobre `www-data` | Escalamiento a superusuario (`root`) | 5 | 5 | 🔴 Crítico (25) | Corregir `/etc/sudoers` con mínimo privilegio; jamás `NOPASSWD: ALL` a cuentas de servicio. |
| R-03 | Código / Config | Credenciales en comentario HTML y `robots.txt` | Acceso no autorizado con credenciales legítimas | 5 | 4 | 🔴 Alto (20) | Retirar credenciales del código y archivos públicos; gestión de secretos. |
| R-04 | Autenticación Web | Transporte en texto claro (HTTP sin TLS) | Intercepción de credenciales (*sniffing*) | 4 | 4 | 🟠 Alto (16) | Forzar HTTPS/TLS y redirección 301 de HTTP a HTTPS. |
| R-05 | Control de Aplicación | Lista negra (bloqueo de `cat`) evadible | Falsa sensación de seguridad; bypass trivial | 5 | 2 | 🟡 Medio (10) | Validar por diseño (lista blanca); eliminar la ejecución de comandos. |

*Metodología: Nivel de Riesgo = Probabilidad × Impacto (escala 1–5), según NIST SP 800-30.*

---

## 🛠️ Plan de Tratamiento y Mitigación (ISO/IEC 27001)

**1. Eliminación de la RCE (R-01):** retirar el panel de comandos de producción; si se requiere, usar funciones acotadas con listas blancas, nunca invocando una shell (`system()`, `exec()`, `shell_exec()`); desplegar un WAF (ModSecurity + OWASP CRS).

**2. Reestructuración de Sudo (R-02):** eliminar la directiva `www-data ALL=(ALL) NOPASSWD: ALL` de `/etc/sudoers`; aplicar mínimo privilegio; de requerirse, autorizar solo un binario con ruta absoluta y argumentos específicos.

**3. Gestión de Secretos (R-03):** eliminar comentarios con credenciales antes del despliegue (SAST); no usar `robots.txt` como almacén de datos sensibles; centralizar credenciales en un gestor de secretos (Vault) y rotarlas.

**4. Cifrado del Transporte (R-04):** instalar certificado TLS, forzar HTTPS en Apache y habilitar HSTS.

**5. Validación por Diseño (R-05):** reemplazar listas negras por validación positiva (lista blanca). El bloqueo de `cat` resultó inútil frente a `less` o `grep`.

---

## 🏁 Conclusiones

1. **La información filtrada es el primer eslabón de la cadena de ataque:** un comentario en el código fuente y una cadena en `robots.txt` entregaron credenciales válidas sin fuerza bruta.
2. **Impacto en cadena (Kill Chain):** una funcionalidad web mal diseñada permitió el acceso inicial como `www-data`, que escaló a control absoluto por un error en `sudoers`.
3. **Los controles superficiales generan falsa seguridad:** el filtrado de `cat` no impidió la lectura de los archivos.
4. **Importancia del mínimo privilegio:** conceder `NOPASSWD: ALL` a una cuenta de servicio anula las barreras de contención del sistema.
5. **Valor de la gestión proactiva de riesgos:** aplicar matrices de riesgo y auditorías periódicas permite corregir fallos antes de que se conviertan en incidentes reales.

---

**DUOC UC — Escuela de Informática y Telecomunicaciones**
*Gestión de Riesgos en Redes Corporativas*
**Autora: Lisandra González**
