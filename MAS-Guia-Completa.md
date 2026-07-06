# MAS (Microsoft Activation Scripts) — Guía y análisis

> Activador open source (GPLv3) de **Windows y Office** del grupo *massgravel*.
> Son scripts `.cmd` de texto, auditables, que activan productos de Microsoft sin clave comprada.
> Versión analizada: **3.11** · Fecha del análisis: **2026-06-11**.

---

## ⚡ Lo esencial (TL;DR)

- **No es malware.** Código auditado: sin descargas de binarios externos, sin robo de datos, sin desactivar el antivirus, sin persistencia oculta (salvo la tarea de renovación de KMS, que es visible y eliminable).
- **Sus riesgos no son técnicos sino legales y operativos:** es **activación no autorizada** (incumple los términos de Microsoft) y provoca **falsos positivos de antivirus garantizados**.
- **No hay que compilar nada:** los `.cmd` se ejecutan directos; el C# interno se compila solo en memoria.
- **Qué método usar:** Windows 10/11 → **HWID** · Office → **Ohook** · Server/LTSC/ESU/casos raros → **TSforge** · **Online KMS** → evitar (contacta servidores de terceros).
- **El mayor peligro real es ejecutar una copia FALSIFICADA.** Usa siempre una copia verificada (como esta, ya auditada), nunca comandos pegados de webs o vídeos de terceros.

---

## Tabla resumen de métodos

| Método | Para qué | Permanencia | Red | Dónde "vive" la activación |
|--------|----------|-------------|-----|-----------------------------|
| **HWID** | Windows 10 / 11 | Permanente | 1 conexión a **Microsoft** | Servidores de Microsoft (ligada al hardware) |
| **Ohook** | Office (incl. Microsoft 365 Apps) | Permanente | **Offline** | DLL parcheado en disco |
| **TSforge** | Windows / Office / **ESU** / Server / LTSC | Permanente | **Offline** | Almacén SPP **local** (esa instalación) |
| **Online KMS** | Windows / Office por volumen | 180 días (vitalicio con tarea) | **Terceros, recurrente** | Tarea programada de renovación |

---

## Cuándo usar cada método

### HWID — la mejor opción para Windows 10/11
- Registra una **licencia digital real en los servidores de Microsoft**, ligada al **hash del hardware** (no a una cuenta).
- **Funciona con cuenta local** — no requiere cuenta Microsoft.
- **Sobrevive a un formateo:** misma edición + mismo hardware → se **reactiva sola** al conectar a internet, sin clave ni reejecutar el script.
- "Configúralo y olvídate" mientras no cambies el hardware (un cambio de placa base puede romperlo).

### Ohook — la mejor opción para Office
- Parchea localmente el DLL de licencias de Office → **100% offline**.
- Funciona con Office 2013–2024 y **Microsoft 365 Apps**, y es **muy resistente a las actualizaciones** Click-to-Run.
- Una **suscripción M365 activa con sesión iniciada** tiene prioridad y lo "pisa"; cuando esa suscripción expira, **Ohook vuelve a mandar**.
- Iniciar sesión con una cuenta **sin** suscripción **no lo desactiva**.

### TSforge — el comodín universal
- Activa lo que los demás no cubren: **Windows Server, LTSC, ESU**, Windows antiguos, casos límite, o cuando HWID/Ohook fallan. También sirve para Windows y Office normales.
- **100% offline** y sin tarea de renovación.
- Forja la licencia en el **almacén SPP local** → vive solo en **esa instalación**: si formateas, hay que **volver a ejecutarlo**.
- **ESU = Extended Security Updates:** parches de seguridad de pago para sistemas ya en fin de soporte (Win 7/8.1/10, Server, embebidos/POS). TSforge activa ese complemento para seguir recibiendo actualizaciones.

### Online KMS — evitar salvo necesidad
- Úsalo **solo** si no hay otra opción (algún producto por volumen) y aceptas el contacto con terceros.
- Conecta el equipo a **servidores KMS públicos de operadores desconocidos**.
- Activación **temporal (180 días)** + **tarea programada** que renueva contactándolos de forma recurrente.

---

## Cómo ejecutarlo — paso a paso

### ¿Hay que compilar algo? No.
- Los ficheros `.cmd` se ejecutan **directamente**; no hay que compilar ni instalar herramientas.
- El código C# interno (TSforge y el parcheo de Ohook) lo **compila el propio script en memoria** al vuelo, usando el compilador de .NET Framework que **ya viene en Windows**. Es automático y transparente.
- El único binario "compilable" (`BIN\LibTSforge.dll`) es una comodidad **opcional para desarrolladores**; un usuario normal **nunca** lo necesita.

### Opción A — Ejecutar la copia local auditada (recomendada)
1. Abre la carpeta del repositorio → `MAS\All-In-One-Version-KL\`.
2. **Click derecho en `MAS_AIO.cmd` → "Ejecutar como administrador".**
   - Si SmartScreen/Defender lo bloquea: *Más información → Ejecutar de todas formas*, o añade una exclusión temporal para esa carpeta.
   - *Alternativa por función:* usa `MAS\Separate-Files-Version\Activators\` y ejecuta el `.cmd` del método concreto (p. ej. `HWID_Activation.cmd`, `Ohook_Activation_AIO.cmd`, `TSforge_Activation.cmd`).
3. Se abre un **menú de consola**. Escribe el número de la opción **verde** (las recomendadas) y pulsa Enter.
   - Windows → **HWID** · Office → **Ohook** · Resto/ESU/Server/LTSC → **TSforge**.
4. Espera el **mensaje de confirmación** (p. ej. *"permanently activated with a digital license"*).
5. **Verifica:** opción de menú *"Check Activation Status"*, o en consola `slmgr /xpr` (Windows); en Office, abre una app y mira el estado de la cuenta.

### Opción B — One-liner de PowerShell (menos recomendable para entornos controlados)
```powershell
irm https://get.activated.win | iex
```
Descarga y ejecuta el script desde la web en el momento. Cómodo, pero **implica confiar en la red/DNS y descargar al vuelo**. Para un equipo de pruebas aislado o si quieres control total, usa la **Opción A**.

### Para deshacer / reparar
Ejecuta `Troubleshoot.cmd` (o la opción *Troubleshoot* del menú): resetea licencias, elimina la tarea de renovación de KMS y revierte los cambios.

---

## Cómo funciona internamente (resumen)

Cada `.cmd` es un **políglota**: empieza como batch (cmd) y, cuando necesita lógica avanzada, lanza **PowerShell**, que a su vez **compila C# embebido en memoria**. No deja ningún `.exe`/`.dll` "de fuera": los binarios se generan o parchean en el momento desde el propio texto.

- **HWID:** instala la clave genérica (GVLK), genera `GenuineTicket.xml` y lo registra vía el servicio **ClipSVC** contra los servidores de Microsoft → licencia digital ligada al hardware.
- **Ohook:** coloca en la carpeta de Office una copia **parcheada** del DLL de licencias (`OSPPC.DLL`/`sppc`), recalculando el checksum del PE para que sea válido. Office consulta su licencia a través del DLL parcheado y se cree activado. Todo local.
- **TSforge:** compila la librería C# **`LibTSforge`** en memoria y la usa para construir un **almacén de licencias falso** en el SPP de Windows: cifra los datos con **AES** y firma la clave con **RSA** usando claves de producción de Microsoft incluidas en el código, de modo que el SPP lo acepta.
- **Online KMS:** instala la GVLK, apunta el host KMS (puerto 1688) a **servidores KMS públicos** y crea la tarea `Activation-Renewal` (como SYSTEM) que reactiva cada ~180 días.

> **Sobre la ofuscación:** verás cadenas partidas (`s%nil%cht%nil%asks`, `kms.03%-%k.org`, `m{}assgrave`). **No ocultan funcionalidad**: solo evitan que el antivirus reconozca cadenas y marque el fichero. El intento está documentado en la cabecera del propio script.

---

## Riesgos y puntos a tener en cuenta

### Generales (verificados)
- ✅ **No es malware:** sin descargas de binarios externos (solo contacta endpoints **oficiales de Microsoft**; servidores KMS de terceros únicamente si eliges ese método), sin exfiltración de datos, sin desactivar Defender (solo avisa), sin backdoor ni creación de usuarios.
- ⚠️ **Legal:** es **activación no autorizada** de productos Microsoft (incumple sus términos de licencia). Es el riesgo principal.
- ⚠️ **Antivirus:** falsos positivos **garantizados** (HackTool/Riskware/PUA). Defender puede poner ficheros en cuarentena **a mitad del proceso** y dejar la activación incompleta.
- ⚠️ **Requiere admin** y hace cambios reales de sistema (registro, almacén SPP, DLLs, tareas). Reversibles con `Troubleshoot`.
- ⚠️ **Copia falsificada = mayor peligro real.** Usa siempre código verificado/auditado, nunca comandos pegados de fuentes no fiables.

### Por método
- **HWID:** necesita **una conexión a Microsoft**. Un **cambio grande de hardware** (placa base) puede romper la activación → enlazar a una cuenta Microsoft *antes* ayuda a recuperarla.
- **Ohook:** una **suscripción M365 activa con sesión** lo anula mientras esté vigente; puede mostrar un banner de inicio de sesión puntual.
- **TSforge:** la activación es **local a la instalación** → **no sobrevive a un formateo** (reejecutar tras reinstalar). Una reparación/refresco de licencia de Office puede borrar la de Office.
- **Online KMS:** único método con **contacto recurrente a terceros** → descártalo si quieres minimizar conexiones externas. Deja una tarea programada como SYSTEM (eliminable desde el script).

### Recomendación para un equipo de pruebas con mínimas conexiones externas
- **Offline total:** **Ohook** (Office) y **TSforge** (Windows/Office).
- **Una sola conexión, solo a Microsoft:** **HWID**.
- **Descartar:** **Online KMS** (terceros + renovación recurrente).
- No es apto para equipos de empresa ni entornos con cumplimiento normativo.
