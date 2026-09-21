# Consolidación T-030

Se consolidan en una raíz limpia:

- `wpv10barza/esp32-firmware-backend-`
- `wpv10barza/asistente-3c`

Estructura:
- `firmware/`: código, configuración y pruebas del ESP32-S3-4848S040.
- `backend/`: aplicación Asistente 3C, API, HMI y pruebas.

No se importan historiales Git de los repositorios fuente.

No se copia:
- `firebase-applet-config.json` con valores reales;
- archivos `.env`;
- `firmware/include/local_config.h`;
- artefactos `*.bin`, `*.elf`, `*.map`;
- workflows que dependían de clonar el otro repositorio.

El README de la raíz es una versión consolidada y avanzada para la implementación
actual y para la trazabilidad técnica del T-030.
