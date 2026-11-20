---
name: apps-script-sheets
description: Construir APIs y apps pequeñas con Google Apps Script usando Google Sheets como base de datos (web apps con doGet/doPost, PWAs o apps de campo que leen y escriben hojas, formularios, reportes y automatizaciones). Úsala cuando se escriba o revise código .gs, appsscript.json, un despliegue de web app de Apps Script, llamadas fetch a script.google.com, o cuando pidan "usar una hoja de cálculo como backend", "API con Google Sheets", "Apps Script", "guardar desde la app en el Excel de Drive" o "clasp".
---

# Apps Script + Google Sheets como backend

Sirve para herramientas internas, apps de campo y prototipos con pocos usuarios: cero servidor, cero costo, y los datos quedan en una hoja que el cliente ya sabe usar. **No** sirve para miles de usuarios, datos sensibles a gran escala ni transacciones complejas; en ese caso hay que migrar a una base real.

Detalle en `references/patrones.md` (código) y `references/frontend-y-pruebas.md` (llamadas desde la web, offline, pruebas).

## Reglas

1. **La hoja es un contrato**: leer columnas **por nombre de encabezado**, nunca por posición fija. Crear las columnas que falten de forma idempotente (`asegurarColumnas_`) en vez de asumir que existen.
2. **Leer y escribir en bloque**: un `getValues()` / `setValues()` por rango, no celda por celda dentro de un ciclo. Cada llamada a `SpreadsheetApp` es lenta.
3. **`LockService` en toda escritura**: dos usuarios guardando a la vez pisan filas. Tomar `getScriptLock()`, `waitLock`, escribir, `SpreadsheetApp.flush()`, liberar en `finally`.
4. **Idempotencia**: cada envío desde el cliente lleva un identificador único (`ref`); si ya está registrado, devolver éxito sin duplicar. Imprescindible si la app reintenta envíos sin conexión.
5. **Identificar filas por una llave estable** (ID o código del negocio), nunca por número de fila: las filas se ordenan, se filtran y se borran.
6. **Secretos y configuración en `PropertiesService`** (propiedades del script), nunca en el código. Una función `instalar()` los crea la primera vez.
7. **Control de acceso explícito**: una web app desplegada como "Cualquier persona" es pública. Exigir un token largo y aleatorio (comparado en el servidor) o desplegar con acceso restringido a la cuenta o al dominio. Considerar el token como contraseña: se puede rotar.
8. **Validar todo lo que entra**: tipos, largos, valores permitidos. Nunca escribir en la hoja un texto que empiece por `=`, `+`, `-` o `@` sin neutralizarlo (inyección de fórmulas).
9. **Respuesta JSON uniforme**: `{ ok: true, datos }` o `{ ok: false, error }`, siempre con `ContentService` y tipo JSON. Atrapar los errores y no devolver trazas internas.
10. **Respetar las cuotas**: máximo ~6 minutos por ejecución, ~30 ejecuciones simultáneas, límites diarios de correos y `UrlFetch`. Cachear lecturas pesadas con `CacheService` y procesar lotes grandes con disparadores por tiempo.
11. **Zona horaria y formato**: fijar `timeZone` en `appsscript.json`; fechas con `Utilities.formatDate`. Cuidado con decimales con coma y números guardados como texto: normalizarlos al leer.
12. **Desplegar sin cambiar la URL**: en *Implementar → Gestionar implementaciones → editar → Nueva versión*. Crear una implementación nueva cambia la URL y rompe a los clientes instalados.
13. **Permisos mínimos**: declarar `oauthScopes` en `appsscript.json` con lo justo (por ejemplo `spreadsheets` en vez de `drive` si no se tocan archivos).

## Lista antes de entregar

- [ ] Petición sin token o con token incorrecto → `{ ok: false }` y nada escrito.
- [ ] Dos guardados simultáneos sobre la misma fila → ninguno se pierde.
- [ ] El mismo `ref` enviado dos veces → una sola fila de historial.
- [ ] Columnas reordenadas en la hoja → sigue funcionando.
- [ ] Texto con `=HYPERLINK(...)` → se guarda como texto, no como fórmula.
- [ ] Nueva versión publicada sobre la **misma** implementación.
- [ ] Instrucciones de instalación para el dueño de la hoja (pegar código, correr `instalar()`, autorizar, implementar).
