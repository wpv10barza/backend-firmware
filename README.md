# T-030 — Backend + Firmware ESP32-S3-4848S040

Repositorio consolidado de la implementación actual del sistema T-030.

Este repositorio reúne en una sola base limpia los dos subsistemas que
actualmente forman la solución:

- **Firmware:** interfaz embebida ESP32-S3-4848S040.
- **Backend:** Asistente 3C con Node.js, TypeScript, Express, React, Vite y
  Google GenAI.

La consolidación conserva el código fuente, contratos, pruebas y documentación
técnica de las versiones actuales, pero elimina la dependencia de clonar dos
repositorios separados para construir y revisar el sistema.

## 1. Estructura del repositorio

```text
backend-firmware/
├── firmware/                 # ESP32-S3-4848S040 + PlatformIO
│   ├── include/
│   ├── src/
│   ├── test/
│   ├── tests/
│   ├── e2e/
│   ├── contract/
│   ├── scripts/
│   ├── site/
│   └── platformio.ini
│
├── backend/                  # Asistente 3C + API + HMI web
│   ├── server/
│   ├── src/
│   ├── test/
│   ├── scripts/
│   ├── docs/
│   ├── contract/
│   ├── package.json
│   ├── package-lock.json
│   ├── server.ts
│   └── vite.config.ts
│
├── docs/
│   └── ESPECIFICACIONES_T030_VERIFICADAS.md
│
├── .github/
│   └── workflows/
│       └── ci.yml
│
├── .gitignore
└── README.md
```

## 2. Arquitectura funcional

```text
┌──────────────────────────────────────────────────────────┐
│                 ESP32-S3-4848S040                        │
│                                                          │
│  Pantalla 480x480 ── Touch I2C ── Editor ── Estados     │
│                       │                                  │
│                       └──── Wi-Fi / HTTP ──────────┐    │
└────────────────────────────────────────────────────┼────┘
                                                     │
                                                     ▼
┌──────────────────────────────────────────────────────────┐
│                    BACKEND / 3C                          │
│                                                          │
│  Express + TypeScript                                    │
│       │                                                  │
│       ├── Device API                                     │
│       ├── Cola temporal de comandos                      │
│       ├── Google GenAI                                   │
│       ├── Reglas deterministas / FIELD_RULES             │
│       └── AnalyticalReviewStore / HITL                   │
│                                                          │
└───────────────────────────┬──────────────────────────────┘
                            │
                            ▼
                 ┌───────────────────────┐
                 │   HMI web autenticada │
                 │                       │
                 │ Vista previa          │
                 │ Confirmación humana   │
                 │ Rechazo               │
                 └──────────┬────────────┘
                            │
                            ▼
                 ┌───────────────────────┐
                 │    Google Sheets      │
                 │   Fuente maestra      │
                 └───────────────────────┘
```

El firmware **no escribe directamente** en Google Sheets. Su función es
capturar la interacción, transmitir la orden, consultar el estado y presentar
el resultado.

El backend interpreta y valida la orden, pero la persistencia efectiva de la
versión vigente se realiza desde la interfaz web autenticada después de la
revisión humana.

## 3. Firmware

### Plataforma verificada

El proyecto utiliza:

- entorno PlatformIO: `panel_4848s040`;
- `espressif32@6.8.1`;
- placa configurada: `esp32-s3-devkitm-1`;
- framework Arduino;
- Flash configurada en 16 MB;
- PSRAM OPI;
- Arduino-GFX 1.5.9.

### Interfaz

En el código actual se verifican:

- resolución lógica 480 × 480;
- `Arduino_RGB_Display`;
- secuencia de inicialización `st7701_type8_init_operations`;
- retroiluminación en GPIO 38;
- bus I2C de touch con SDA GPIO 19 y SCL GPIO 45;
- dirección I2C `0x5D`;
- I2S TX con GPIO 1, 2 y 40;
- servidor web local `WebServer web(80)`.

El firmware implementa estados de panel:

`Booting`, `Offline`, `Ready`, `Busy`, `Pending`,
`Applied`, `Rejected`, `Error`.

También incluye editor de comandos, cursor, desplazamiento de ventana de texto,
teclado virtual y gestión de la cola de comandos.

### Precisión de hardware

La fuente de código confirma la configuración utilizada por el firmware, pero
determinadas características físicas deben sustentarse con documentación del
fabricante. Por ello, el repositorio no presenta como demostrados únicamente
por el código datos como "GT911", "Xtensa LX7", "dual-core" o la variante
comercial exacta del controlador si no aparecen literalmente en la fuente
inspeccionada.

## 4. Backend Asistente 3C

El backend actual utiliza:

- Node.js;
- TypeScript;
- Express;
- React;
- Vite;
- tsx;
- `@google/genai`;
- `@react-oauth/google`.

El modelo configurado por defecto es:

`gemini-2.5-flash`

La generación utiliza:

- `temperature: 0`;
- `responseMimeType: application/json`;
- `responseSchema`.

La inteligencia artificial no ejecuta directamente la escritura: genera
operaciones estructuradas que posteriormente son validadas.

## 5. Validación determinista

`server.ts` define `FIELD_RULES` para controlar los campos que pueden ser
actualizados.

Las columnas permitidas son:

`B, C, H, I, J, K, L, M, N, O`.

Las columnas de identidad/búsqueda y las columnas fuera de la lista blanca no
pueden ser modificadas por este flujo.

La validación comprueba, entre otros:

- correspondencia de encabezados;
- enteros positivos para frecuencia;
- unidades de tiempo canónicas;
- textos largos sin puntos suspensivos ni "etc.";
- pertenencia a catálogos existentes;
- rechazo de campos no permitidos.

La regla de prompt y la validación determinista mantienen el principio:

> No inventar columnas, encabezados, IDs ni valores.

## 6. API del dispositivo

La versión actual expone:

| Método | Endpoint | Función |
|---|---|---|
| GET | `/api/device/v1/health` | Salud del servicio |
| POST | `/api/device/v1/commands` | Recepción de comando |
| GET | `/api/device/v1/commands/pending` | Consulta de pendiente |
| GET | `/api/device/v1/commands/:id` | Consulta de estado |
| POST | `/api/device/v1/commands/:id/result` | Reporte de resultado |

El contrato utiliza:

- `protocol_version = 1.0`;
- `requires_human_confirmation = true`;
- `X-3C-Device-Token`;
- `request_id` para idempotencia;
- `device_id`;
- texto de comando.

### Cobertura de autorización

La implementación actual aplica `authorizeDeviceRequest()` en:

- POST `/commands`;
- GET `/commands/:id`.

La versión inspeccionada no muestra esa llamada en:

- GET `/commands/pending`;
- POST `/commands/:id/result`.

Por ello, no se documenta una protección uniforme de todos los endpoints.

## 7. Ciclo de vida de comandos

`server/deviceCommands.ts` implementa:

```text
pending_confirmation
        │
        ├──────────► applied
        │
        └──────────► rejected
```

Parámetros verificados:

- TTL: 15 minutos;
- límite temporal de almacenamiento: 50 comandos;
- UUID para identificadores;
- deduplicación por `device_id + request_id`;
- eliminación de comandos expirados.

## 8. Control analítico y HITL

`server/reviewControl.ts` implementa:

```text
proposed
   ├──► approved
   └──► rejected
```

Parámetros verificados:

- TTL de propuesta: 5 minutos;
- bloqueo temporal de fila;
- lista blanca de columnas;
- liberación de fila al aprobar, rechazar o expirar;
- rechazo de una propuesta concurrente sobre la misma fila.

El almacenamiento de este componente es **en memoria del proceso**; un reinicio
del backend elimina las propuestas y bloqueos activos.

## 9. Contrato

El contrato del dispositivo se encuentra en:

```text
firmware/contract/device-command-v1.json
backend/contract/device-command-v1.json
```

Ambas copias corresponden al contrato vigente de la implementación fuente.

## 10. Configuración segura

No se incorpora al repositorio consolidado ningún archivo que contenga valores
reales de credenciales del entorno de ejecución.

El backend utiliza:

```text
backend/.env.example
```

con variables como:

```env
GEMINI_API_KEY=
GEMINI_MODEL=gemini-2.5-flash
SPREADSHEET_ID=
SHEET_NAME=Data
HEADER_ROW=4
PORT=3000
ESP32_API_TOKEN=
ALLOW_INSECURE_DEVICE_API=false
```

La configuración privada del firmware utiliza:

```text
firmware/include/local_config.h
```

y este archivo está excluido por `.gitignore`.

## 11. Ejecución del backend

```bash
cd backend
npm ci
cp -n .env.example .env
# editar .env
npm run lint
npm test
npm run build
npm run dev
```

El servidor se configura para escuchar en `0.0.0.0`.

## 12. Compilación del firmware

Con PlatformIO instalado:

```bash
cd firmware
pio run --environment panel_4848s040
```

Pruebas nativas incluidas:

```bash
cd firmware

g++ -std=c++17 -Wall -Wextra -Werror -Iinclude \
  tests/test_command_buffer_native.cpp \
  -o /tmp/command_buffer_native

/tmp/command_buffer_native

g++ -std=c++17 -Wall -Wextra -Werror -Iinclude \
  test/command_text_viewport_test.cpp \
  -o /tmp/command_text_viewport_test

/tmp/command_text_viewport_test

g++ -std=c++17 -Wall -Wextra -Werror -Iinclude \
  test/virtual_keyboard_hit_test.cpp \
  -o /tmp/virtual_keyboard_hit_test

/tmp/virtual_keyboard_hit_test

python test/command_buffer_regression.py
```

## 13. Integración WSL / red local

La arquitectura vigente contempla:

```text
ESP32
  │
  │ Wi-Fi / HTTP
  ▼
Windows / WSL
  │
  └── backend :3000
```

El ESP32 debe utilizar una dirección del host accesible desde su red. No debe
configurarse `127.0.0.1` como destino del backend cuando el cliente es el
dispositivo físico.

La modalidad de red de WSL2 y las reglas del firewall deben permitir la
conectividad desde el panel hacia el puerto configurado del backend.

## 14. Seguridad

El sistema mantiene separados:

1. captura física;
2. transporte;
3. interpretación IA;
4. validación determinista;
5. propuesta;
6. confirmación humana;
7. persistencia.

Las claves y tokens no deben introducirse en Git.

La conexión ESP32 ↔ backend utiliza HTTP sobre la LAN confiable en la versión
vigente. El acceso a servicios de Google se realiza mediante los mecanismos de
autenticación definidos por la interfaz web.

## 15. Pruebas y CI

El repositorio consolidado incluye pruebas del firmware y del backend y una
acción de CI unificada.

El pipeline valida como mínimo:

- contrato JSON;
- lint del backend;
- pruebas del backend;
- build del backend;
- pruebas nativas del firmware;
- compilación PlatformIO del ESP32-S3-4848S040;
- validación básica del sitio estático.

Las pruebas que requieren credenciales reales de Google deben ejecutarse
mediante secretos configurados en GitHub Actions y no mediante archivos
versionados.

## 16. T-030 — correspondencia con el Capítulo III

| Capítulo III | Evidencia en este repositorio |
|---|---|
| 3.2 Hardware | `firmware/platformio.ini`, `firmware/src/main.cpp` |
| 3.2.1–3.2.5 | `firmware/include`, `src`, documentación de hardware |
| 3.3 Software y algoritmos | `backend/package.json`, `backend/server.ts` |
| 3.3.1 Rendimiento | métricas únicamente cuando exista evidencia experimental |
| 3.3.2 Contrato | `deviceApi.ts` + `contract/device-command-v1.json` |
| 3.3.3 Ciclo de vida | `deviceCommands.ts` |
| 3.3.4 HITL | `reviewControl.ts` + interfaz |
| 3.5.1 Prototipo ESP32 | `firmware/src/main.cpp` |
| 3.5.2 Software híbrido | `firmware` + `backend` |
| 3.5.3 Datos/persistencia | backend + HMI autenticada + Google Sheets |
| 3.6 Inferencia/validación | `backend/server.ts` |
| 3.7 Integración/emulación/pruebas | `firmware/tests`, `firmware/e2e`, `backend/test`, CI |

No se incorpora una sección 3.8 en la organización de la tesis.

## 17. Evidencia y criterio documental

Las afirmaciones técnicas deben clasificarse como:

- **Verificado:** aparece explícitamente en código/configuración inspeccionada.
- **Inferido:** conclusión técnica que requiere fuente adicional.
- **Fuente externa necesaria:** requiere datasheet, esquemático, manual,
  experimento u otra evidencia documental.

Las especificaciones verificadas detalladas se mantienen en:

`docs/ESPECIFICACIONES_T030_VERIFICADAS.md`.

## 18. Repositorios fuente consolidados

La base del repositorio limpio procede de:

- `wpv10barza/esp32-firmware-backend-`
- `wpv10barza/asistente-3c`

El nuevo repositorio es:

`wpv10barza/backend-firmware`

La consolidación no conserva los historiales Git independientes como
submódulos. El objetivo es disponer de una única raíz reproducible para
desarrollo, auditoría técnica y referencia de la implementación T-030.
