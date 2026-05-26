# ICMP

ICMP (Internet Control Message Protocol) es un protocolo auxiliar de IP utilizado para señalización de errores, diagnóstico y control del comportamiento de red. Aunque suele asociarse únicamente con `ping`, ICMP participa en mecanismos fundamentales como `traceroute`, `Path MTU Discovery (PMTU)` y la notificación de errores de red, afectando indirectamente al rendimiento, estabilidad y seguridad de las comunicaciones.

Desde una perspectiva de ciberseguridad, ICMP también puede utilizarse para tareas de reconocimiento, enumeración, evasión, túneles de datos o ataques de denegación de servicio. Debido a ello, una configuración insegura o demasiado permisiva del firewall puede facilitar abuso del protocolo o exponer información innecesaria sobre la red.

El objetivo de esta sección es analizar ICMP desde un enfoque práctico orientado a **pentesting, hardening y defensa**, recreando técnicas ofensivas, observando el comportamiento real del protocolo y diseñando configuraciones de `nftables` para mitigar abuso sin romper funcionalidades legítimas de red.

## Tabla de Contenido

- [Objetivos](#objetivos)
- [Documentación recomendada](#documentación-recomendada)
- [Laboratorios](#laboratorios)

## Objetivos

- Comprender el funcionamiento interno de ICMP.
- Analizar mensajes, tipos y códigos relevantes del protocolo.
- Recrear técnicas ofensivas y abusos comunes de ICMP.
- Observar el comportamiento real del tráfico ICMP en laboratorio.
- Diseñar configuraciones defensivas de `nftables` para limitar abuso del protocolo.
- Evaluar tradeoffs entre seguridad, operatividad y diagnóstico de red.

## Documentación recomendada

- [RFC 792](https://datatracker.ietf.org/doc/html/rfc792): Especificación del protocolo ICMP.
- [Recommendations for filtering ICMP messages - draft-ietf-opsec-icmp-filtering-04](https://datatracker.ietf.org/doc/html/draft-ietf-opsec-icmp-filtering-04): Qué ICMP filtrar, limitar y por qué bloquearlo mal rompe redes.
- [RFC 1191](https://datatracker.ietf.org/doc/html/rfc1191): Funcionamiento de Path MTU Discovery (PMTU).
- [RFC 1812 - Section 4.3](https://datatracker.ietf.org/doc/html/rfc1812#section-4.3): Comportamiento esperado de routers IPv4 respecto al procesamiento de ICMP.
- [IANA ICMP Parameters](https://www.iana.org/assignments/icmp-parameters/icmp-parameters.xhtml#icmp-parameters-types): Tipos y códigos ICMP.
- [ICMP Usage in Scanning - Ofir Arkin](https://www.cs.dartmouth.edu/~sergey/netreads/ICMP_Scanning_v3.0.pdf): Paper ofensivo sobre abuso de ICMP.

## Laboratorios

- [Echo Discovery (Ping Sweep)](./labs/echo_discovery.md): Enumeración de hosts mediante `echo-request`, análisis del tráfico ICMP y mitigación con `nftables`.
