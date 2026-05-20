# Proyecto_VPN

Este repositorio contiene la documentación técnica y los archivos de configuración para el despliegue de una infraestructura de red privada segura.

## Estructura del Repositorio

La organización del código sigue un modelo modular para facilitar la auditoría y el despliegue por componentes:

- `/fases`: Contiene la documentación técnica detallada de cada etapa del proyecto, incluyendo metodologías de configuración, registros de logs y pruebas de validación.
- `/configs`: Directorio de archivos de configuración para los servicios implementados
- `/systemd`: Unidades de servicio y temporizadores para la automatización de tareas.

## Tecnologias Implementadas

El proyecto integra las siguientes tecnologías de red:

- [Wireguard](https://www.wireguard.com/)
- [Unbound](https://unbound.docs.nlnetlabs.nl)
- [Nftables](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)

## Compatibilidad

Aunque este proyecto nació y se probó a fondo en Arch Linux, no estás limitado a esta distribución. La arquitectura y la lógica de red que he implementado son totalmente flexibles: puedes llevar estas configuraciones a Ubuntu, Debian, Red Hat o cualquier otro sistema Linux moderno sin complicaciones.
Solo necesitas asegurarte de que tu sistema tenga soporte para el kernel y las herramientas estándar de WireGuard, Unbound y nftables. Los archivos de configuración son fácilmente adaptables, por lo que puedes usarlos como base independientemente de la distribución que prefieras.

## Fases de Construcción

1. [Fase 01: Configuración de Red](/fases/01-preparacion.md): Preparación del stack de red y reenvío de paquetes.
2. [Fase 02: Implementación del Túnel VPN con WireGuard](/fases/02-vpn-wireguard.md): Creación del túnel cifrado y la gestión segura de identidades criptográficas.
3. [Fase 03: Resolución DNS Recursiva y Privacidad con Unbound](/fases/03-dns-unbound.md): Despliegue de un resolver propio para evitar fugas de información y asegurar la autenticidad de las respuestas.