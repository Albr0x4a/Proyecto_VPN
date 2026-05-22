Proyecto VPN
======

Este proyecto comenzó como una VPN basada en WireGuard para uso personal, pero evolucionó hacia un laboratorio práctico de redes, sistemas Linux y observabilidad de tráfico.
La idea principal no es únicamente desplegar servicios, sino utilizarlos como entorno para estudiar y experimentar.

Tabla de Contenido
===========

- [Estructura del Repositorio](#estructura-del-repositorio)
- [Tecnologias Implementadas](#tecnologias-implementadas)
- [Compatibilidad](#compatibilidad)
- [Módulos](#módulos)

Estructura del Repositorio
=====

La organización del proyecto sigue un modelo modular para facilitar la separación entre configuración, análisis y documentación técnica.

- `/modules`: Contiene la configuración y documentación específica de cada componente del laboratorio, incluyendo archivos de configuración, despliegue y pruebas relacionadas.
- `/docs`: Directorio destinado a notas, pruebas, observaciones e investigaciones que van surgiendo durante el desarrollo del laboratorio y el estudio de distintos protocolos y comportamientos de red.

Tecnologias Implementadas
=====

El proyecto integra las siguientes tecnologías de red:

- [Wireguard](https://www.wireguard.com/)
- [Unbound](https://unbound.docs.nlnetlabs.nl)
- [Nftables](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)

Compatibilidad
======

El laboratorio fue desarrollado y probado principalmente sobre Arch Linux, aunque la mayoría de configuraciones y conceptos utilizados pueden adaptarse fácilmente a otras distribuciones Linux modernas.

Módulos
=======

1. [Wireguard](/modules/wireguard/README.md)
2. [Unbound](/modules/unbound/README.md)
3. [Nftables](/modules/nftables/README.md)
