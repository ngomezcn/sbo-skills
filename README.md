# sbo-skills

Marketplace de plugins de Claude Code para trabajar con SAP Business One (por ahora Service Layer).

Este repo es solo el producto instalable. La fabrica (fuentes, scripts, skills de construccion, evidencias) vive en el repo exterior, que incluye este como submodulo en `sbo-skills/`.

## Plugins

Cada sistema es un unico plugin, que se instala completo y tiene tres partes:

- `service-layer`
  - **setup** (skill `setup`): conversacion con Claude para elegir la version de B1, la version de OData, los entornos (dev, uat, prod), una pregunta cada vez. La URL, la company, el usuario y el password los escribe el desarrollador en un archivo del repo, no en el chat. Despues prueba el login. Mas adelante se puede anadir otro entorno con `/service-layer:add-environment`.
  - **use** (skill `use`): lee y escribe contra el Service Layer real. Las escrituras son en seco hasta que se repite la llamada con `--execute`.
  - **docs** (skill `docs`): documentacion unica para todas las versiones de B1. Exige el Setup hecho (lee `.sbo-skills/service-layer/config.md`).

Requisitos: Claude Code y Node 20 o superior. Por ahora solo se da soporte a Claude Code. El plugin guarda sus datos locales en `.sbo-skills/service-layer/` del repo del desarrollador (el Setup lo anade a `.gitignore`).

## Instalacion

    /plugin marketplace add ngomezcn/sbo-skills
    /plugin install service-layer@sbo-skills

Despues, abre Claude Code en tu repo y pidele que configure Service Layer (skill `setup`).

## No se edita a mano

`plugins/service-layer/dist/` y `skills/` se generan desde la fabrica (`npm run publish-plugin` en `factory/plugins-src/service-layer`). Para cambiarlos, se edita la fabrica y se vuelve a generar; `.claude-plugin/plugin.json` y este README si se editan aqui.
