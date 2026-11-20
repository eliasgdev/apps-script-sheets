# Llamadas desde la web, modo sin conexión y pruebas

## fetch a la web app

- La URL es `https://script.google.com/macros/s/<ID>/exec` (la `/dev` solo funciona para el dueño).
- Apps Script responde con una **redirección** a `script.googleusercontent.com`: `fetch` la sigue sola.
- Para POST desde el navegador, usar `Content-Type: text/plain` para evitar la petición *preflight* de CORS, que Apps Script no responde:

```js
async function llamar(accion, datos) {
  const res = await fetch(URL_API, {
    method: 'POST',
    headers: { 'Content-Type': 'text/plain;charset=utf-8' },
    body: JSON.stringify({ token: TOKEN, accion, datos }),
  });
  const json = await res.json();
  if (!json.ok) throw new Error(json.error);
  return json.datos;
}
```

- No poner el token en el código del sitio público: pedirlo una vez al usuario y guardarlo en el dispositivo (`localStorage` o IndexedDB).

## Apps de campo sin conexión (PWA)

- **Bandeja de salida** en IndexedDB: cada guardado se encola con su `ref` (UUID del cliente) y se envía cuando hay red. El servidor ignora los `ref` repetidos.
- Mostrar al usuario cuántos envíos están pendientes y cuáles fallaron (con el mensaje del servidor).
- Cachear el listado para consultarlo sin red, con la fecha de la última actualización visible.
- Service worker: subir la versión de la caché (`const CACHE = 'app-v12'`) cada vez que cambian los archivos web, o los usuarios se quedan con la versión vieja.
- Fotos: comprimir en el cliente (canvas, máximo ~1600 px, JPEG 0.7) antes de encolarlas.

## Pruebas sin Google

Apps Script no tiene un entorno local oficial, pero el código V8 es JavaScript normal. Se puede probar con Node simulando los servicios:

```js
// prueba.js — node prueba.js
const vm = require('vm');
const fs = require('fs');

function hojaFalsa(filas) {
  return {
    getLastRow: () => filas.length,
    getLastColumn: () => filas[0].length,
    getDataRange: () => ({ getValues: () => filas.map(f => [...f]) }),
    getRange: (f, c, nf = 1, nc = 1) => ({
      getValues: () => filas.slice(f - 1, f - 1 + nf).map(r => r.slice(c - 1, c - 1 + nc)),
      setValue: v => { filas[f - 1][c - 1] = v; },
      setValues: vs => vs.forEach((r, i) => r.forEach((v, j) => { filas[f - 1 + i][c - 1 + j] = v; })),
      setFontWeight() { return this; },
      createTextFinder: t => ({ matchEntireCell() { return this; },
        findNext: () => filas.slice(f - 1).some(r => String(r[c - 1]) === t) || null }),
    }),
    appendRow: r => filas.push(r),
  };
}

const contexto = {
  SpreadsheetApp: { openById: () => ({ getSheetByName: n => hojas[n] }), flush() {} },
  LockService: { getScriptLock: () => ({ waitLock() {}, releaseLock() {} }) },
  PropertiesService: { getScriptProperties: () => ({ getProperty: () => 'secreto' }) },
  CacheService: { getScriptCache: () => ({ get: () => null, put() {}, remove() {} }) },
  ContentService: { MimeType: { JSON: 'json' }, createTextOutput: t => ({ setMimeType: () => t }) },
  Utilities: { getUuid: () => 'uuid' },
};
const hojas = {
  Datos: hojaFalsa([['ID', 'Estado'], ['A1', 'Pendiente']]),
  Historial: hojaFalsa([['ref', 'id', 'estado', 'fecha']]),
};
vm.createContext(contexto);
vm.runInContext(fs.readFileSync('Codigo.gs', 'utf8'), contexto);

// Casos: guardar, repetir el mismo ref, token malo, fórmula.
```

Probar al menos: token incorrecto, `ref` repetido, estado no permitido, columnas en otro orden y texto que empieza por `=`.

## clasp (opcional)

`npm i -g @google/clasp`, `clasp login`, `clasp clone <scriptId>` permite versionar el código en git y subirlo con `clasp push`. Agregar `.clasprc.json` al `.gitignore` (contiene credenciales).
