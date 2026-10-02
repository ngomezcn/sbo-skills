# sbo-skills

Marketplace de plugins de Claude Code para trabajar con SAP Business One (por ahora Service Layer).

Este repo es solo el producto instalable. La fabrica (fuentes, scripts, skills de construccion, evidencias) vive en el repo exterior, que incluye este como submodulo en `sbo-skills/`.

## Plugins

Cada sistema es un unico plugin, que se instala completo y tiene tres partes:

- `service-layer`
  - **setup** (skill `setup`): el desarrollador ejecuta un script en su propio terminal que le pregunta la version de B1, la version de OData y las credenciales de dev, uat o prod, y prueba el login. Las credenciales no pasan por la IA.
  - **use** (skill `use`): lee y escribe contra el Service Layer real. Las escrituras son en seco hasta que se repite la llamada con `--execute`.
  - **docs** (skill `docs`): documentacion unica para todas las versiones de B1. Exige el Setup hecho (lee `.sbo-skills/service-layer/config.md`).

Requisitos: Node 20 o superior. El plugin guarda sus datos locales en `.sbo-skills/service-layer/` del repo del desarrollador (el Setup lo anade a `.gitignore`).

## Instalacion

    /plugin marketplace add <owner>/sbo-skills
    /plugin install service-layer@sbo-skills

Despues, pide a Claude que configure Service Layer (skill `setup`).

## No se edita a mano

`plugins/service-layer/dist/` y `skills/` se generan desde la fabrica (`npm run publish-plugin` en `factory/plugins-src/service-layer`). Para cambiarlos, se edita la fabrica y se vuelve a generar; `.claude-plugin/plugin.json` y este README si se editan aqui.
