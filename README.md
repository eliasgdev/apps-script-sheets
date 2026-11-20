# Apps Script + Google Sheets

Skill para **Claude Code** para construir APIs y apps pequeñas con **Google Apps Script** usando **Google Sheets como base de datos**: web apps con `doGet`/`doPost`, apps de campo (PWA) que leen y escriben hojas, automatizaciones y reportes.

## Origen

Estos patrones salieron de herramientas internas y apps de campo hechas para proyectos propios y de clientes, alojadas en repositorios privados. En todas, el cliente ya trabajaba en Google Sheets y necesitaba una app sencilla encima. Recogen lo que hizo falta para que funcionaran de verdad con varios usuarios: candados al escribir, envíos sin duplicados cuando la app trabaja sin conexión, columnas leídas por nombre y despliegues que no cambian la URL.

## Contenido

```
apps-script-sheets/
├── SKILL.md                       # cuándo usar este enfoque, 13 reglas, lista antes de entregar
└── references/
    ├── patrones.md                # appsscript.json, token, doGet/doPost, lectura por encabezados,
    │                              # LockService + idempotencia, inyección de fórmulas, caché, archivos, lotes
    └── frontend-y-pruebas.md      # fetch sin preflight, PWA sin conexión, pruebas en Node, clasp
```

## Instalación

```bash
git clone https://github.com/eliasgdev/apps-script-sheets.git
mkdir -p ~/.claude/skills
cp -r apps-script-sheets/apps-script-sheets ~/.claude/skills/
```

## Licencia

MIT
