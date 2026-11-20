# Patrones de código (Apps Script V8)

## appsscript.json

```json
{
  "timeZone": "America/Bogota",
  "runtimeVersion": "V8",
  "exceptionLogging": "STACKDRIVER",
  "oauthScopes": ["https://www.googleapis.com/auth/spreadsheets"],
  "webapp": { "executeAs": "USER_DEPLOYING", "access": "ANYONE_ANONYMOUS" }
}
```

`USER_DEPLOYING` + `ANYONE_ANONYMOUS` = la app actúa con los permisos del dueño y cualquiera con la URL puede llamarla → **el token es obligatorio**.

## Instalación y token

```js
const HOJA_ID = 'ID_DE_LA_HOJA'; // o guardarlo también en propiedades

function instalar() {
  const props = PropertiesService.getScriptProperties();
  if (!props.getProperty('TOKEN')) {
    props.setProperty('TOKEN', Utilities.getUuid() + Utilities.getUuid());
  }
  Logger.log('Token de acceso: ' + props.getProperty('TOKEN'));
}

function cambiarToken() {
  PropertiesService.getScriptProperties().deleteProperty('TOKEN');
  instalar();
}

function validarToken_(token) {
  const esperado = PropertiesService.getScriptProperties().getProperty('TOKEN');
  if (!esperado) throw new Error('Falta ejecutar instalar().');
  if (typeof token !== 'string' || token !== esperado) throw new Error('Acceso denegado.');
}
```

## Entrada y respuesta

```js
function doGet(e) {
  return responder_(() => {
    validarToken_(e.parameter.token);
    switch (e.parameter.accion) {
      case 'listar': return listar_();
      default: throw new Error('Acción desconocida.');
    }
  });
}

function doPost(e) {
  return responder_(() => {
    const cuerpo = JSON.parse(e.postData && e.postData.contents || '{}');
    validarToken_(cuerpo.token);
    switch (cuerpo.accion) {
      case 'guardar': return guardar_(cuerpo.datos);
      default: throw new Error('Acción desconocida.');
    }
  });
}

function responder_(fn) {
  let salida;
  try { salida = { ok: true, datos: fn() }; }
  catch (err) { salida = { ok: false, error: String(err && err.message || err) }; }
  return ContentService.createTextOutput(JSON.stringify(salida))
    .setMimeType(ContentService.MimeType.JSON);
}
```

## Leer una tabla por encabezados

```js
function leerTabla_(hoja) {
  const valores = hoja.getDataRange().getValues();
  const encabezados = valores.shift().map(v => String(v).trim());
  return valores.map(fila => aObjeto_(encabezados, fila));
}

function aObjeto_(encabezados, fila) {
  const o = {};
  encabezados.forEach((h, i) => { if (h) o[h] = fila[i]; });
  return o;
}
```

Usar `getDisplayValues()` cuando importa lo que el usuario ve (formatos, ceros a la izquierda) y `getValues()` cuando se necesitan números y fechas reales.

## Asegurar columnas (idempotente)

```js
function asegurarColumnas_(hoja, requeridas) {
  const ancho = Math.max(hoja.getLastColumn(), 1);
  const actuales = hoja.getRange(1, 1, 1, ancho).getValues()[0].map(v => String(v).trim());
  const faltantes = requeridas.filter(c => !actuales.includes(c));
  if (faltantes.length) {
    hoja.getRange(1, actuales.filter(Boolean).length + 1, 1, faltantes.length)
      .setValues([faltantes]).setFontWeight('bold');
  }
}
```

## Escribir con candado e idempotencia

```js
function guardar_(datos) {
  validar_(datos);
  const candado = LockService.getScriptLock();
  candado.waitLock(20000);
  try {
    const libro = SpreadsheetApp.openById(HOJA_ID);
    const historial = libro.getSheetByName('Historial');
    if (refYaRegistrada_(historial, datos.ref)) return { duplicado: true };

    const hoja = libro.getSheetByName('Datos');
    const encabezados = hoja.getRange(1, 1, 1, hoja.getLastColumn()).getValues()[0];
    const fila = buscarFila_(hoja, encabezados, datos.id);
    if (!fila) throw new Error('Registro no encontrado.');

    const col = encabezados.indexOf('Estado') + 1;
    hoja.getRange(fila, col).setValue(textoSeguro_(datos.estado));
    historial.appendRow([datos.ref, datos.id, textoSeguro_(datos.estado), new Date()]);
    SpreadsheetApp.flush();
    return { guardado: true };
  } finally {
    candado.releaseLock();
  }
}

function refYaRegistrada_(hoja, ref) {
  if (hoja.getLastRow() < 2) return false;
  return hoja.getRange(2, 1, hoja.getLastRow() - 1, 1)
    .createTextFinder(String(ref)).matchEntireCell(true).findNext() !== null;
}

function buscarFila_(hoja, encabezados, id) {
  const col = encabezados.indexOf('ID');
  const valores = hoja.getRange(2, col + 1, Math.max(hoja.getLastRow() - 1, 1), 1).getValues();
  const i = valores.findIndex(v => String(v[0]) === String(id));
  return i === -1 ? null : i + 2;
}

// Evita que un texto se interprete como fórmula.
function textoSeguro_(v) {
  const s = String(v == null ? '' : v).slice(0, 2000);
  return /^[=+\-@]/.test(s) ? "'" + s : s;
}

function validar_(d) {
  if (!d || typeof d !== 'object') throw new Error('Datos inválidos.');
  if (!/^[\w-]{8,64}$/.test(String(d.ref || ''))) throw new Error('Referencia inválida.');
  if (!d.id) throw new Error('Falta el ID.');
  const permitidos = ['Pendiente', 'Visitado', 'Descartado'];
  if (!permitidos.includes(d.estado)) throw new Error('Estado no permitido.');
}
```

## Caché de lecturas pesadas

```js
function listar_() {
  const cache = CacheService.getScriptCache();
  const guardado = cache.get('listado');
  if (guardado) return JSON.parse(guardado);
  const datos = leerTabla_(SpreadsheetApp.openById(HOJA_ID).getSheetByName('Datos'));
  const texto = JSON.stringify(datos);
  if (texto.length < 100000) cache.put('listado', texto, 300); // límite ~100 KB por clave
  return datos;
}
```

Invalidar (`cache.remove('listado')`) después de cada escritura que cambie ese listado.

## Archivos (fotos, evidencias)

- Recibir en base64 desde el cliente, validar tipo y tamaño, crear con `Utilities.newBlob(bytes, mime, nombre)` en una carpeta fija (ID en propiedades).
- Guardar en la hoja el enlace o el ID del archivo, no el contenido.
- No compartir la carpeta públicamente salvo que sea necesario.

## Procesos largos

Para recorrer miles de filas: procesar por lotes, guardar el avance en `PropertiesService` y programar la continuación con `ScriptApp.newTrigger('continuar').timeBased().after(60 * 1000).create()`. Borrar el disparador al terminar.
