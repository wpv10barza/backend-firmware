# T-030 — Especificaciones verificadas de la implementación consolidada

## 1. Fuentes
- `wpv10barza/esp32-firmware-backend-`
- `wpv10barza/asistente-3c`
- Destino: `wpv10barza/backend-firmware`

## 2. Firmware verificado
En `firmware/platformio.ini`:
- `panel_4848s040`
- `espressif32@6.8.1`
- `esp32-s3-devkitm-1`
- Arduino
- Flash 16 MB
- PSRAM OPI
- Arduino-GFX 1.5.9

En `firmware/src/main.cpp`:
- 480 × 480
- `Arduino_RGB_Display`
- `st7701_type8_init_operations`
- backlight GPIO 38
- LCD CS GPIO 39
- LCD clock GPIO 48
- LCD MOSI GPIO 47
- touch SDA GPIO 19
- touch SCL GPIO 45
- touch I2C address `0x5D`
- I2S BCLK GPIO 1
- I2S LRCLK GPIO 2
- I2S DATA GPIO 40
- `WebServer web(80)`

Estados verificados:
`Booting`, `Offline`, `Ready`, `Busy`, `Pending`,
`Applied`, `Rejected`, `Error`.

## 3. Backend verificado
Stack:
- Node.js
- TypeScript
- Express
- React
- Vite
- tsx
- `@google/genai`
- `@react-oauth/google`

IA:
- modelo por defecto `gemini-2.5-flash`
- `temperature: 0`
- salida JSON
- `responseSchema`

## 4. Validación
`FIELD_RULES` limita las columnas modificables a:
`B, C, H, I, J, K, L, M, N, O`.

Se validan tipos, unidades de tiempo, encabezados, catálogos y textos largos.

## 5. API
Endpoints:
- GET `/api/device/v1/health`
- POST `/api/device/v1/commands`
- GET `/api/device/v1/commands/pending`
- GET `/api/device/v1/commands/:id`
- POST `/api/device/v1/commands/:id/result`

Contrato:
- protocolo 1.0
- `X-3C-Device-Token`
- `request_id`
- confirmación humana requerida

Autorización observada:
- protegidos: POST `commands` y GET `commands/:id`
- no se observa `authorizeDeviceRequest()` en `pending` ni `result`
- health no requiere autorización

## 6. Ciclo de vida
Comandos:
`pending_confirmation → applied/rejected`

- TTL: 15 minutos
- límite: 50 comandos
- deduplicación por dispositivo + request_id

Propuestas:
`proposed → approved/rejected`

- TTL: 5 minutos
- bloqueo temporal de fila
- almacenamiento en memoria

## 7. Precisión documental
No deben declararse como demostrados exclusivamente por los archivos
inspeccionados: GT911, Xtensa LX7/doble núcleo, ST7701S como variante exacta,
datos de datasheet, métricas experimentales o persistencia que no aparezca
explícitamente en el código.

## 8. Seguridad
No se copia `firebase-applet-config.json` con valores reales.
Tampoco se versionan `.env`, `local_config.h` ni artefactos binarios.
