# sap-b1-marketplace

Marketplace de plugins de Claude Code para trabajar con SAP Business One (por ahora Service Layer).

Este repo es solo el producto instalable. La fabrica (fuentes, scripts, skills de construccion, evidencias) vive en el repo exterior, que incluye este como submodulo en `marketplace/`.

## Plugins

- `setup-<sistema>`: prepara `.sbo-b1/<sistema>/`.
- `docs-<sistema>`: documentacion unica para todas las versiones de B1.
- `use-<sistema>`: opera contra el sistema real.

## Instalacion

    /plugin marketplace add <owner>/sap-b1-marketplace
    /plugin install use-service-layer@sap-b1-marketplace
