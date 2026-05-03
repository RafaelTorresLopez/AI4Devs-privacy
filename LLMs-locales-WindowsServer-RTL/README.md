# Instalación de un LLM local en Windows Server

**Objetivo:** instalar una infraestructura local de IA generativa en **Windows Server** para una primera versión piloto de baja carga, reduciendo la transferencia de datos a terceros y facilitando controles relacionados con **GDPR/RGPD** y **EU AI Act**.

> **Aviso importante:** una instalación local no garantiza por sí sola el cumplimiento legal. Ayuda a reducir riesgos, pero el cumplimiento requiere también medidas jurídicas, organizativas y documentales: base jurídica, registro de tratamientos, control de accesos, formación, políticas internas, evaluación de riesgos, revisión humana y documentación del sistema.

---

## Índice

1. [Arquitectura propuesta](#1-arquitectura-propuesta)
2. [Supuestos del piloto](#2-supuestos-del-piloto)
3. [Requisitos mínimos recomendados](#3-requisitos-mínimos-recomendados)
4. [Crear estructura de carpetas](#4-crear-estructura-de-carpetas)
5. [Descargar e instalar Ollama standalone](#5-descargar-e-instalar-ollama-standalone)
6. [Descargar NSSM](#6-descargar-nssm)
7. [Crear script de arranque de Ollama](#7-crear-script-de-arranque-de-ollama)
8. [Registrar Ollama como servicio Windows](#8-registrar-ollama-como-servicio-windows)
9. [Descargar el modelo local](#9-descargar-el-modelo-local)
10. [Instalar Python 3.11](#10-instalar-python-311)
11. [Instalar Open WebUI en entorno virtual](#11-instalar-open-webui-en-entorno-virtual)
12. [Crear script de arranque de Open WebUI](#12-crear-script-de-arranque-de-open-webui)
13. [Registrar Open WebUI como servicio Windows](#13-registrar-open-webui-como-servicio-windows)
14. [Desactivar altas abiertas tras crear el administrador](#14-desactivar-altas-abiertas-tras-crear-el-administrador)
15. [Configurar Open WebUI desde la interfaz](#15-configurar-open-webui-desde-la-interfaz)
16. [Exposición opcional a red interna](#16-exposición-opcional-a-red-interna)
17. [Bloquear salidas a Internet tras descargar el modelo](#17-bloquear-salidas-a-internet-tras-descargar-el-modelo)
18. [Pruebas funcionales](#18-pruebas-funcionales)
19. [Controles GDPR/RGPD recomendados](#19-controles-gdprrgpd-recomendados)
20. [Controles EU AI Act recomendados](#20-controles-eu-ai-act-recomendados)
21. [Política interna mínima de uso](#21-política-interna-mínima-de-uso)
22. [Backup manual](#22-backup-manual)
23. [Parada, arranque y reinicio](#23-parada-arranque-y-reinicio)
24. [Actualización controlada](#24-actualización-controlada)
25. [Checklist final de validación](#25-checklist-final-de-validación)
26. [Mejoras para una segunda fase](#26-mejoras-para-una-segunda-fase)
27. [Fuentes oficiales de referencia](#27-fuentes-oficiales-de-referencia)

---

## 1. Arquitectura propuesta

### Stack recomendado para piloto

| Capa | Herramienta | Motivo |
|---|---|---|
| Motor de inferencia LLM | **Ollama para Windows / CLI standalone** | Ejecuta modelos localmente y expone API local en `http://localhost:11434`. |
| Modelo inicial | **Mistral NeMo 12B Instruct** vía Ollama | Modelo abierto, 12B, buen rendimiento multilingüe, contexto amplio y licencia Apache 2.0. |
| Interfaz web | **Open WebUI** | Plataforma self-hosted, puede operar offline y soporta Ollama. |
| Ejecución persistente | **NSSM** como wrapper de servicios Windows | Permite ejecutar procesos como servicios Windows. |
| Base de datos piloto | SQLite interna de Open WebUI | Suficiente para pruebas de baja carga. |
| Seguridad | Firewall Windows, autenticación Open WebUI, bloqueo de APIs cloud | Reduce exposición de datos y controla accesos. |
| Auditoría | Logs de Open WebUI en modo metadata | Trazabilidad sin registrar el contenido completo de prompts. |

### Diagrama lógico

```text
Usuarios internos
      |
      | HTTP interno / localhost
      v
Open WebUI
      |
      | http://127.0.0.1:11434
      v
Ollama
      |
      v
Modelo local: mistral-nemo:12b
      |
      v
Disco local: C:\AIStack\models
```

---

## 2. Supuestos del piloto

Este documento asume:

1. Windows Server 2019, 2022 o 2025.
2. Uso interno, baja carga, pocos usuarios.
3. Sin exposición directa a Internet.
4. Un único servidor.
5. La prueba inicial puede ejecutarse por CPU, aunque se recomienda GPU NVIDIA si se quiere una experiencia fluida.
6. Se usará el puerto:
   - `11434` para Ollama, solo local.
   - `8080` para Open WebUI.
7. No se activará web search, OpenAI, Gemini, Mistral API cloud ni otros conectores externos.

---

## 3. Requisitos mínimos recomendados

Para una prueba básica:

- CPU x86_64 moderna.
- RAM mínima: **16 GB**.
- RAM recomendada: **32 GB**.
- Disco libre: **50 GB mínimo**, mejor **100 GB**.
- GPU opcional:
  - NVIDIA con drivers actualizados.
  - VRAM recomendada: 12 GB o más.
- Cuenta con permisos de administrador local durante la instalación.

---

## 4. Crear estructura de carpetas

Abrir **PowerShell como Administrador** y ejecutar:

```powershell
$Base = "C:\AIStack"

New-Item -ItemType Directory -Force -Path `
"$Base", `
"$Base\downloads", `
"$Base\ollama", `
"$Base\models", `
"$Base\openwebui", `
"$Base\openwebui\data", `
"$Base\services", `
"$Base\logs", `
"$Base\logs\ollama", `
"$Base\logs\openwebui", `
"$Base\config", `
"$Base\backup"
```

---

## 5. Descargar e instalar Ollama standalone

Aunque Ollama ofrece instalador Windows, para servidor es preferible usar la versión **standalone CLI**, porque facilita ejecutarlo como servicio.

Ejecuta:

```powershell
$Base = "C:\AIStack"

Invoke-WebRequest `
  -Uri "https://github.com/ollama/ollama/releases/latest/download/ollama-windows-amd64.zip" `
  -OutFile "$Base\downloads\ollama-windows-amd64.zip"

Expand-Archive `
  -Path "$Base\downloads\ollama-windows-amd64.zip" `
  -DestinationPath "$Base\ollama" `
  -Force

Get-ChildItem "$Base\ollama" -Recurse -Filter "ollama.exe"
```

Verifica que existe:

```powershell
C:\AIStack\ollama\ollama.exe --version
```

---

## 6. Descargar NSSM

NSSM se usará para registrar Ollama y Open WebUI como servicios Windows.

```powershell
$Base = "C:\AIStack"

Invoke-WebRequest `
  -Uri "https://nssm.cc/ci/nssm-2.24-101-g897c7ad.zip" `
  -OutFile "$Base\downloads\nssm.zip"

Expand-Archive `
  -Path "$Base\downloads\nssm.zip" `
  -DestinationPath "$Base\downloads\nssm" `
  -Force

Copy-Item `
  -Path "$Base\downloads\nssm\nssm-2.24-101-g897c7ad\win64\nssm.exe" `
  -Destination "$Base\services\nssm.exe" `
  -Force
```

Verifica:

```powershell
C:\AIStack\services\nssm.exe version
```

---

## 7. Crear script de arranque de Ollama

Crea el archivo:

```powershell
notepad C:\AIStack\config\run-ollama.cmd
```

Contenido:

```bat
@echo off
set "OLLAMA_MODELS=C:\AIStack\models"
set "OLLAMA_HOST=127.0.0.1:11434"
set "OLLAMA_ORIGINS=http://localhost:8080,http://127.0.0.1:8080"
C:\AIStack\ollama\ollama.exe serve
```

Explicación:

- `OLLAMA_MODELS` fuerza que los modelos se almacenen en `C:\AIStack\models`.
- `OLLAMA_HOST=127.0.0.1:11434` evita que Ollama quede expuesto en la red.
- Open WebUI accederá a Ollama localmente.

---

## 8. Registrar Ollama como servicio Windows

Ejecuta:

```powershell
$Base = "C:\AIStack"
$NSSM = "$Base\services\nssm.exe"

& $NSSM install OllamaLocal C:\Windows\System32\cmd.exe "/c C:\AIStack\config\run-ollama.cmd"
& $NSSM set OllamaLocal AppDirectory "C:\AIStack\config"
& $NSSM set OllamaLocal AppStdout "C:\AIStack\logs\ollama\stdout.log"
& $NSSM set OllamaLocal AppStderr "C:\AIStack\logs\ollama\stderr.log"
& $NSSM set OllamaLocal AppRotateFiles 1
& $NSSM set OllamaLocal AppRotateBytes 10485760
& $NSSM set OllamaLocal Start SERVICE_AUTO_START
& $NSSM set OllamaLocal AppExit Default Restart
& $NSSM start OllamaLocal
```

Verifica:

```powershell
Get-Service OllamaLocal
Invoke-RestMethod http://127.0.0.1:11434/api/tags
```

---

## 9. Descargar el modelo local

Para el piloto se propone **Mistral NeMo 12B**.

Ejecuta:

```powershell
C:\AIStack\ollama\ollama.exe pull mistral-nemo:12b
```

Prueba inferencia:

```powershell
$Body = @{
  model = "mistral-nemo:12b"
  prompt = "Responde solo con la palabra OK."
  stream = $false
} | ConvertTo-Json

Invoke-RestMethod `
  -Method Post `
  -Uri "http://127.0.0.1:11434/api/generate" `
  -Body $Body `
  -ContentType "application/json"
```

Deberías recibir una respuesta con `OK` o similar.

---

## 10. Instalar Python 3.11

Open WebUI puede instalarse por Python/pip. Para evitar problemas de compatibilidad, se recomienda Python 3.11.

Descarga e instala Python 3.11.9:

```powershell
$Base = "C:\AIStack"

Invoke-WebRequest `
  -Uri "https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe" `
  -OutFile "$Base\downloads\python-3.11.9-amd64.exe"

Start-Process `
  -FilePath "$Base\downloads\python-3.11.9-amd64.exe" `
  -ArgumentList "/quiet InstallAllUsers=1 PrependPath=1 Include_launcher=1" `
  -Wait
```

Verifica:

```powershell
py -3.11 --version
```

---

## 11. Instalar Open WebUI en entorno virtual

```powershell
$Base = "C:\AIStack"

py -3.11 -m venv "$Base\openwebui\.venv"

& "$Base\openwebui\.venv\Scripts\python.exe" -m pip install --upgrade pip

& "$Base\openwebui\.venv\Scripts\pip.exe" install open-webui
```

Verifica:

```powershell
& "C:\AIStack\openwebui\.venv\Scripts\open-webui.exe" --help
```

Open WebUI se arranca con:

```bash
open-webui serve
```

---

## 12. Crear script de arranque de Open WebUI

Crea el archivo:

```powershell
notepad C:\AIStack\config\run-openwebui.cmd
```

Contenido inicial:

```bat
@echo off

set "DATA_DIR=C:\AIStack\openwebui\data"

set "OLLAMA_BASE_URL=http://127.0.0.1:11434"
set "ENABLE_OLLAMA_API=True"

set "ENABLE_OPENAI_API=False"
set "ENABLE_OPENAI_API_PASSTHROUGH=False"

set "ENABLE_WEB_SEARCH=False"
set "ENABLE_IMAGE_GENERATION=False"
set "ENABLE_COMMUNITY_SHARING=False"
set "ENABLE_MESSAGE_RATING=False"
set "ENABLE_EVALUATION_ARENA_MODELS=False"

set "OFFLINE_MODE=True"
set "HF_HUB_OFFLINE=1"

set "WEBUI_AUTH=True"
set "ENABLE_SIGNUP=True"
set "ENABLE_PASSWORD_VALIDATION=True"

set "ENABLE_AUDIT_LOGS_FILE=True"
set "AUDIT_LOG_LEVEL=METADATA"
set "AUDIT_LOGS_FILE_PATH=C:\AIStack\logs\openwebui\audit.log"
set "AUDIT_LOG_FILE_ROTATION_SIZE=50MB"

set "GLOBAL_LOG_LEVEL=INFO"
set "LOG_FORMAT=json"

set "CORS_ALLOW_ORIGIN=http://localhost:8080;http://127.0.0.1:8080"

C:\AIStack\openwebui\.venv\Scripts\open-webui.exe serve --host 127.0.0.1 --port 8080
```

Notas de seguridad:

- `ENABLE_OPENAI_API=False`: desactiva proveedor OpenAI.
- `ENABLE_WEB_SEARCH=False`: evita búsquedas web.
- `OFFLINE_MODE=True`: evita llamadas no necesarias, aunque no sustituye al bloqueo por firewall.
- `ENABLE_COMMUNITY_SHARING=False`: desactiva funciones de compartir con comunidad.
- `AUDIT_LOG_LEVEL=METADATA`: registra metadatos, no cuerpos completos de prompts/respuestas.

---

## 13. Registrar Open WebUI como servicio Windows

```powershell
$Base = "C:\AIStack"
$NSSM = "$Base\services\nssm.exe"

& $NSSM install OpenWebUILocal C:\Windows\System32\cmd.exe "/c C:\AIStack\config\run-openwebui.cmd"
& $NSSM set OpenWebUILocal AppDirectory "C:\AIStack\config"
& $NSSM set OpenWebUILocal AppStdout "C:\AIStack\logs\openwebui\stdout.log"
& $NSSM set OpenWebUILocal AppStderr "C:\AIStack\logs\openwebui\stderr.log"
& $NSSM set OpenWebUILocal AppRotateFiles 1
& $NSSM set OpenWebUILocal AppRotateBytes 10485760
& $NSSM set OpenWebUILocal Start SERVICE_AUTO_START
& $NSSM set OpenWebUILocal AppExit Default Restart
& $NSSM start OpenWebUILocal
```

Verifica:

```powershell
Get-Service OpenWebUILocal
```

Abre en el propio servidor:

```text
http://127.0.0.1:8080
```

Crea la primera cuenta administradora.

---

## 14. Desactivar altas abiertas tras crear el administrador

Después de crear el primer usuario administrador, modifica:

```powershell
(Get-Content "C:\AIStack\config\run-openwebui.cmd") `
  -replace 'set "ENABLE_SIGNUP=True"', 'set "ENABLE_SIGNUP=False"' `
  | Set-Content "C:\AIStack\config\run-openwebui.cmd"

C:\AIStack\services\nssm.exe restart OpenWebUILocal
```

Verifica que ya no permite registro libre.

---

## 15. Configurar Open WebUI desde la interfaz

Entrar como administrador en:

```text
http://127.0.0.1:8080
```

Acciones:

1. Ir a **Admin Panel**.
2. Revisar **Connections**.
3. Confirmar que solo aparece Ollama:

   ```text
   http://127.0.0.1:11434
   ```

4. Desactivar cualquier proveedor externo.
5. En usuarios:
   - Crear usuarios manualmente.
   - No permitir autorregistro.
   - Asignar roles mínimos.
6. En modelos:
   - Habilitar solo:

     ```text
     mistral-nemo:12b
     ```

7. En funciones/herramientas:
   - No activar web search.
   - No activar conectores externos.
   - No activar herramientas experimentales.

---

## 16. Exposición opcional a red interna

Para máxima seguridad en piloto, usa solo acceso por RDP al servidor.

Si necesitas que otros usuarios de la LAN entren desde su navegador, cambia en `run-openwebui.cmd`:

```bat
C:\AIStack\openwebui\.venv\Scripts\open-webui.exe serve --host 0.0.0.0 --port 8080
```

Y ajusta CORS, sustituyendo `<IP_SERVIDOR>`:

```bat
set "CORS_ALLOW_ORIGIN=http://localhost:8080;http://127.0.0.1:8080;http://<IP_SERVIDOR>:8080"
```

Reinicia:

```powershell
C:\AIStack\services\nssm.exe restart OpenWebUILocal
```

Abre firewall solo para la subred autorizada. Ejemplo para `192.168.1.0/24`:

```powershell
New-NetFirewallRule `
  -DisplayName "AIStack OpenWebUI LAN 8080" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 8080 `
  -RemoteAddress 192.168.1.0/24 `
  -Action Allow
```

No abras el puerto `11434` de Ollama. Debe quedar solo en `127.0.0.1`.

---

## 17. Bloquear salidas a Internet tras descargar el modelo

Cuando ya esté instalado Open WebUI, Ollama y el modelo, bloquea salidas de los procesos para evitar fugas accidentales.

```powershell
New-NetFirewallRule `
  -DisplayName "Block Ollama Outbound" `
  -Direction Outbound `
  -Program "C:\AIStack\ollama\ollama.exe" `
  -Action Block

New-NetFirewallRule `
  -DisplayName "Block OpenWebUI Python Outbound" `
  -Direction Outbound `
  -Program "C:\AIStack\openwebui\.venv\Scripts\python.exe" `
  -Action Block

New-NetFirewallRule `
  -DisplayName "Block OpenWebUI Executable Outbound" `
  -Direction Outbound `
  -Program "C:\AIStack\openwebui\.venv\Scripts\open-webui.exe" `
  -Action Block
```

Cuando necesites actualizar Open WebUI o descargar nuevos modelos, deshabilita temporalmente esas reglas y vuelve a activarlas.

---

## 18. Pruebas funcionales

### 18.1. Probar Ollama

```powershell
$Body = @{
  model = "mistral-nemo:12b"
  prompt = "Resume en una frase qué es el RGPD."
  stream = $false
} | ConvertTo-Json

Invoke-RestMethod `
  -Method Post `
  -Uri "http://127.0.0.1:11434/api/generate" `
  -Body $Body `
  -ContentType "application/json"
```

### 18.2. Probar Open WebUI

Abrir:

```text
http://127.0.0.1:8080
```

Enviar este prompt:

```text
Responde únicamente: instalación local verificada.
```

### 18.3. Verificar servicios

```powershell
Get-Service OllamaLocal
Get-Service OpenWebUILocal
```

### 18.4. Verificar logs

```powershell
Get-Content C:\AIStack\logs\ollama\stdout.log -Tail 50
Get-Content C:\AIStack\logs\openwebui\stdout.log -Tail 50
Get-Content C:\AIStack\logs\openwebui\audit.log -Tail 50
```

---

## 19. Controles GDPR/RGPD recomendados

El RGPD exige principios como licitud, lealtad, transparencia, limitación de finalidad, minimización, exactitud, limitación del plazo, integridad/confidencialidad y responsabilidad proactiva.

También exige medidas técnicas y organizativas adecuadas al riesgo, incluyendo confidencialidad, integridad, disponibilidad y resiliencia cuando proceda.

### 19.1. Medidas técnicas

1. **Sin proveedores cloud**
   - OpenAI API desactivada.
   - Web search desactivado.
   - Community sharing desactivado.
   - Outbound bloqueado por firewall.

2. **Datos locales**
   - Modelos en:

     ```text
     C:\AIStack\models
     ```

   - Datos de Open WebUI en:

     ```text
     C:\AIStack\openwebui\data
     ```

3. **Control de acceso**
   - Autenticación obligatoria.
   - Registro abierto desactivado.
   - Usuarios creados por administrador.
   - Roles mínimos.

4. **Logs con minimización**
   - Auditoría en `METADATA`.
   - No registrar prompts y respuestas completas salvo necesidad justificada.

5. **Cifrado en reposo**
   - Activar BitLocker en el volumen del servidor.
   - Restringir permisos NTFS a administradores y cuenta de servicio.

6. **Backups**
   - Copiar periódicamente:

     ```text
     C:\AIStack\openwebui\data
     C:\AIStack\models
     C:\AIStack\config
     ```

   - Cifrar backups.

### 19.2. Medidas organizativas

1. Crear entrada en el **registro de actividades de tratamiento**.
2. Definir finalidad:

   ```text
   Asistente interno de productividad y consulta documental, sin toma de decisiones automatizadas con efectos jurídicos.
   ```

3. Prohibir inicialmente:
   - Datos de salud.
   - Datos de menores.
   - Datos biométricos.
   - Datos judiciales.
   - Decisiones de RRHH, crédito, scoring, despidos, selección automatizada o evaluación de personas.
4. Publicar política interna de uso.
5. Formar a usuarios.
6. Definir plazo de retención de conversaciones.
7. Establecer procedimiento de borrado/exportación de datos de usuario.
8. Evaluar si hace falta DPIA/EIPD si se tratan datos sensibles o se amplía el alcance.

---

## 20. Controles EU AI Act recomendados

El AI Act entró en vigor el 1 de agosto de 2024 y su aplicación es gradual.

Para este piloto:

### 20.1. Clasificación inicial

Uso previsto:

```text
Asistente conversacional interno de propósito general para apoyo a tareas de redacción, resumen y consulta.
```

Con este alcance, debería tratarse como sistema de riesgo limitado o mínimo, siempre que **no** se use para casos de alto riesgo como empleo, educación evaluativa, crédito, servicios esenciales, justicia, migración, biometría o aplicación de la ley.

### 20.2. Transparencia

Añadir en la pantalla de bienvenida o política interna:

```text
Este sistema es un asistente de inteligencia artificial generativa. Sus respuestas pueden contener errores. No debe utilizarse como única fuente para decisiones legales, laborales, médicas, financieras o con efectos sobre personas. El usuario es responsable de validar la información antes de usarla.
```

### 20.3. AI literacy

El artículo 4 del AI Act exige que proveedores y deployers adopten medidas para asegurar un nivel suficiente de alfabetización en IA del personal que opera o usa sistemas de IA.

Acciones mínimas:

1. Formación inicial de 1 hora.
2. Guía de uso permitido/prohibido.
3. Ejemplos de prompts seguros.
4. Protocolo de revisión humana.
5. Registro de usuarios formados.

### 20.4. Documentación mínima del sistema

Crear un documento interno con:

1. Nombre del sistema:

   ```text
   AIStack Local LLM Pilot
   ```

2. Modelo:

   ```text
   mistral-nemo:12b
   ```

3. Finalidad.
4. Usuarios autorizados.
5. Datos permitidos/prohibidos.
6. Medidas de seguridad.
7. Logs y retención.
8. Limitaciones conocidas.
9. Responsable técnico.
10. Responsable funcional.
11. Fecha de puesta en marcha.
12. Fecha de revisión.

---

## 21. Política interna mínima de uso

Puedes copiar esta política:

```text
Política de uso del asistente LLM local

1. El asistente se utilizará exclusivamente para tareas internas de apoyo: redacción, resumen, clasificación no decisoria, generación de ideas y ayuda técnica.

2. No se introducirán datos personales sensibles salvo autorización expresa del responsable de tratamiento.

3. No se utilizará el sistema para tomar decisiones automatizadas sobre personas.

4. No se utilizará para selección de personal, evaluación laboral, scoring, crédito, decisiones jurídicas, médicas o equivalentes.

5. Toda respuesta generada por IA deberá ser revisada por una persona antes de su uso externo.

6. El usuario deberá identificar como generado o asistido por IA cualquier contenido publicado externamente cuando pueda inducir a error.

7. Queda prohibido intentar extraer credenciales, manipular el sistema, evadir controles o introducir información no autorizada.

8. Las conversaciones podrán quedar registradas con fines de seguridad, auditoría y mejora interna conforme a la política de retención aplicable.
```

---

## 22. Backup manual

Crear backup:

```powershell
$Date = Get-Date -Format "yyyyMMdd-HHmm"
$Backup = "C:\AIStack\backup\AIStack-$Date"

New-Item -ItemType Directory -Force -Path $Backup

Copy-Item "C:\AIStack\config" "$Backup\config" -Recurse -Force
Copy-Item "C:\AIStack\openwebui\data" "$Backup\openwebui-data" -Recurse -Force
Copy-Item "C:\AIStack\models" "$Backup\models" -Recurse -Force

Compress-Archive -Path "$Backup\*" -DestinationPath "$Backup.zip" -Force
```

---

## 23. Parada, arranque y reinicio

Parar:

```powershell
C:\AIStack\services\nssm.exe stop OpenWebUILocal
C:\AIStack\services\nssm.exe stop OllamaLocal
```

Arrancar:

```powershell
C:\AIStack\services\nssm.exe start OllamaLocal
C:\AIStack\services\nssm.exe start OpenWebUILocal
```

Reiniciar:

```powershell
C:\AIStack\services\nssm.exe restart OllamaLocal
C:\AIStack\services\nssm.exe restart OpenWebUILocal
```

---

## 24. Actualización controlada

### 24.1. Actualizar Open WebUI

```powershell
C:\AIStack\services\nssm.exe stop OpenWebUILocal

& "C:\AIStack\openwebui\.venv\Scripts\pip.exe" install -U open-webui

C:\AIStack\services\nssm.exe start OpenWebUILocal
```

### 24.2. Actualizar Ollama

1. Descargar de nuevo:

   ```text
   https://github.com/ollama/ollama/releases/latest/download/ollama-windows-amd64.zip
   ```

2. Parar servicio:

   ```powershell
   C:\AIStack\services\nssm.exe stop OllamaLocal
   ```

3. Sustituir contenido de:

   ```text
   C:\AIStack\ollama
   ```

4. Arrancar servicio:

   ```powershell
   C:\AIStack\services\nssm.exe start OllamaLocal
   ```

5. Validar:

   ```powershell
   C:\AIStack\ollama\ollama.exe --version
   ```

---

## 25. Checklist final de validación

| Control | Estado |
|---|---|
| Ollama escucha solo en `127.0.0.1:11434` | ☐ |
| Open WebUI funciona en `127.0.0.1:8080` o LAN controlada | ☐ |
| Modelo `mistral-nemo:12b` descargado | ☐ |
| OpenAI API desactivada | ☐ |
| Web search desactivado | ☐ |
| Community sharing desactivado | ☐ |
| Registro de usuarios abierto desactivado | ☐ |
| Primer usuario admin creado | ☐ |
| Firewall inbound limitado | ☐ |
| Firewall outbound bloqueado tras instalación | ☐ |
| Logs configurados en metadata | ☐ |
| Política interna publicada | ☐ |
| Usuarios formados en AI literacy | ☐ |
| Registro de tratamiento creado | ☐ |
| Backup probado | ☐ |
| BitLocker activado | ☐ |

---

## 26. Mejoras para una segunda fase

Para pasar de piloto a producción:

1. Sustituir SQLite por PostgreSQL.
2. Integrar SSO con Microsoft Entra ID.
3. Añadir HTTPS con certificado interno.
4. Ejecutar servicios con cuenta dedicada o gMSA, no LocalSystem.
5. Añadir proxy inverso.
6. Centralizar logs en SIEM.
7. Definir retención automática de conversaciones.
8. Evaluar RAG local con vector database corporativa.
9. Hacer DPIA/EIPD si se tratan datos personales relevantes.
10. Hacer evaluación formal de clasificación AI Act por caso de uso.

---

## 27. Fuentes oficiales de referencia

- Ollama para Windows: <https://docs.ollama.com/windows>
- Descargas de Ollama: <https://github.com/ollama/ollama/releases>
- Open WebUI - documentación: <https://docs.openwebui.com/>
- Open WebUI - configuración por variables de entorno: <https://docs.openwebui.com/reference/env-configuration/>
- Open WebUI - instalación rápida: <https://docs.openwebui.com/getting-started/quick-start/>
- Open WebUI en PyPI: <https://pypi.org/project/open-webui/>
- NSSM: <https://nssm.cc/>
- Python 3.11.9: <https://www.python.org/downloads/release/python-3119/>
- Mistral NeMo: <https://mistral.ai/news/mistral-nemo>
- GDPR Artículo 5: <https://gdpr-info.eu/art-5-gdpr/>
- GDPR Artículo 30: <https://gdpr-info.eu/art-30-gdpr/>
- GDPR Artículo 32: <https://gdpr-info.eu/art-32-gdpr/>
- EU AI Act - Comisión Europea: <https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai>
- AI literacy - Comisión Europea: <https://digital-strategy.ec.europa.eu/en/faqs/ai-literacy-questions-answers>

---

## Nota final

Esta guía está pensada para una **primera instalación piloto**. Antes de usarla en producción o con datos personales relevantes, conviene realizar una revisión formal de seguridad, privacidad, clasificación AI Act, retención de datos y gobierno del sistema.
