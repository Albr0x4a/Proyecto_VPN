Tabla de Contenido
===========

- [Habilitar IPv4 Forwarding](#habilitar-ipv4-forwarding)


Habilitar IPv4 Forwarding
=======

- Primero listamos nuestra configuración con respecto al reenvio de puertos:
    
    ```bash
    [albr@albr-arch ~]$ sudo sysctl -a | grep "forward"
    net.ipv4.conf.all.bc_forwarding = 0
    net.ipv4.conf.all.forwarding = 0
    net.ipv4.conf.all.mc_forwarding = 0
    net.ipv4.conf.default.bc_forwarding = 0
    net.ipv4.conf.default.forwarding = 0
    net.ipv4.conf.default.mc_forwarding = 0
    net.ipv4.conf.ens33.bc_forwarding = 0
    net.ipv4.conf.ens33.forwarding = 0
    net.ipv4.conf.ens33.mc_forwarding = 0
    net.ipv4.conf.lo.bc_forwarding = 0
    net.ipv4.conf.lo.forwarding = 0
    net.ipv4.conf.lo.mc_forwarding = 0
    net.ipv4.ip_forward = 0
    net.ipv4.ip_forward_update_priority = 1
    net.ipv4.ip_forward_use_pmtu = 0
    net.ipv6.conf.all.force_forwarding = 0
    net.ipv6.conf.all.forwarding = 0
    net.ipv6.conf.all.mc_forwarding = 0
    net.ipv6.conf.default.force_forwarding = 0
    net.ipv6.conf.default.forwarding = 0
    net.ipv6.conf.default.mc_forwarding = 0
    net.ipv6.conf.ens33.force_forwarding = 0
    net.ipv6.conf.ens33.forwarding = 0
    net.ipv6.conf.ens33.mc_forwarding = 0
    net.ipv6.conf.lo.force_forwarding = 0
    net.ipv6.conf.lo.forwarding = 0
    net.ipv6.conf.lo.mc_forwarding = 0
    ```
    
- Como podemos ver todo se encuentra en 0, para cambiar esto debemos agregar lo siguiente a nuestros archivos de configuración de sysctl:
    
    ```bash
    net.ipv4.ip_forward = 1
    net.ipv4.conf.all.forwarding = 1
    net.ipv6.conf.all.forwarding = 1
    ```
    
    - Navegamos a `/etc/sysctl.d/`  y creamos el siguiente [archivo](/configs/01-preparacion/30-ipv4-forwarding.conf):
        
        ```bash
        [albr@albr-arch sysctl.d]$ cat 30-ipv4-forwarding.conf
        # Enable IPv4 Forwarding
        
        net.ipv4.ip_forward = 1
        net.ipv4.conf.all.forwarding = 1
        net.ipv6.conf.all.forwarding = 1
        ```
        
    - Confirmamos que se esté cargando la configuración
        
        ```bash
        [albr@albr-arch sysctl.d]$ systemd-analyze cat-config sysctl.d
        
        -------------CUT----------------------
        
        # /etc/sysctl.d/30-ipv4-forwarding.conf
        # Enable IPv4 Forwarding
        
        net.ipv4.ip_forward = 1
        net.ipv4.conf.all.forwarding = 1
        net.ipv6.conf.all.forwarding = 1
        
        --------------CUT--------------------
        ```
        
    - Reiniciamos y verificamos nuestra configuración actual:
        
        ```bash
        [albr@albr-arch ~]$ sudo sysctl -a | grep "forward"
        net.ipv4.conf.all.bc_forwarding = 0
        net.ipv4.conf.all.forwarding = 1
        net.ipv4.conf.all.mc_forwarding = 0
        net.ipv4.conf.default.bc_forwarding = 0
        net.ipv4.conf.default.forwarding = 1
        net.ipv4.conf.default.mc_forwarding = 0
        net.ipv4.conf.ens33.bc_forwarding = 0
        net.ipv4.conf.ens33.forwarding = 1
        net.ipv4.conf.ens33.mc_forwarding = 0
        net.ipv4.conf.lo.bc_forwarding = 0
        net.ipv4.conf.lo.forwarding = 1
        net.ipv4.conf.lo.mc_forwarding = 0
        net.ipv4.ip_forward = 1
        net.ipv4.ip_forward_update_priority = 1
        net.ipv4.ip_forward_use_pmtu = 0
        net.ipv6.conf.all.force_forwarding = 0
        net.ipv6.conf.all.forwarding = 1
        net.ipv6.conf.all.mc_forwarding = 0
        net.ipv6.conf.default.force_forwarding = 0
        net.ipv6.conf.default.forwarding = 1
        net.ipv6.conf.default.mc_forwarding = 0
        net.ipv6.conf.ens33.force_forwarding = 0
        net.ipv6.conf.ens33.forwarding = 1
        net.ipv6.conf.ens33.mc_forwarding = 0
        net.ipv6.conf.lo.force_forwarding = 0
        net.ipv6.conf.lo.forwarding = 1
        net.ipv6.conf.lo.mc_forwarding = 0
        ```
        
        - Ahora se encuentra activado.