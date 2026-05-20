# Proyecto_VPN
Infraestructura modular para el despliegue de una VPN segura y servicios de red endurecidos (Hardening). Enfoque integral en la privacidad, el blindaje de firewall y la resolución DNS validante.
# VPN

# Habilitar IPv4 Forwarding en Arch

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
    
    - Navegamos a `/etc/sysctl.d/`  y creamos el siguiente archivo:
        
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

# Generar Claves

- Podemos generar un par de claves de la siguiente manera:
    
    ```powershell
    [albr@albr-arch ~]$ (umask 0077; wg genkey > peer.key) # Clave Privada
    [albr@albr-arch ~]$ wg pubkey < peer.key > peer.pub # Clave Pública
    [albr@albr-arch ~]$ ls
    Downloads  peer.key  peer.pub
    ```
    
- Se puede hacer todo de una vez con el siguiente comando:
    
    ```powershell
    [albr@albr-arch albr]$ wg genkey | (umask 0077 && tee peer.key) | wg pubkey > peer.pub
    [albr@albr-arch albr]$ ls
    Downloads  peer.key  peer.pub
    ```
    
- Ahora creamos 2 pares de claves, una para el cliente ( o para cada cliente) y otra para el servidor:
    
    ```powershell
    [albr@albr-arch keys]$ wg genkey | (umask 0077 && tee client.key) | wg pubkey > client.pub
    [albr@albr-arch keys]$ wg genkey | (umask 0077 && tee server.key) | wg pubkey > server.pub
    [albr@albr-arch keys]$ ls
    client.key  client.pub  server.key  server.pub
    ```
    

# Servidor

# Archivo de Configuración

- Podemos crear el archivo de configuración del servidor `wg0.conf` en el directorio `/etc/wireguard`
- Aquí un ejemplo básico de este archivo:
    
    ```powershell
    [root@albr-arch wireguard]# pwd
    /etc/wireguard
    [root@albr-arch wireguard]# cat wg0.conf
    
    [Interface]
    Address = 10.10.10.1/24
    ListenPort = 51820
    PrivateKey = <Server PivateKey>
    
    PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
    PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens33 -j MASQUERADE
    
    [Peer]
    # Client 1
    PublicKey = <Client PublicKey>
    AllowedIPs = 10.10.10.2/32
    ```
    
    - Este archivo crea la configuración basica para nuestra interfaz y nuestro cliente (o clientes).
    - También establece reglas de firewall básicas para cuando se levanta la interfaz y las elimina cuando se termina
        - Estas reglas se van a sustiuir más adelante, pero por ahora es necesaria para que funcione.
    - Se debe cambiar `ens33` por el nombre de su interfaz con acceso a internet.

# Levantar la Interfaz

## wg-quick

- Podemos utilizar la utilidad `wg-quick` para levantar y matar la interfaz:
    
    ```bash
    [albr@albr-arch ~]$ ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000                                                            [0/1]
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host noprefixroute
           valid_lft forever preferred_lft forever
    2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 00:0c:29:e8:2d:1d brd ff:ff:ff:ff:ff:ff
        altname enp2s1
        altname enx000c29e82d1d
        inet 192.168.14.136/24 brd 192.168.14.255 scope global dynamic noprefixroute ens33
           valid_lft 1797sec preferred_lft 1572sec
        inet6 fe80::9722:da02:e854:550a/64 scope link
           valid_lft forever preferred_lft forever
    [albr@albr-arch ~]$ sudo wg-quick up wg0
    [#] ip link add dev wg0 type wireguard
    [#] wg setconf wg0 /dev/fd/63
    [#] ip -4 address add 10.10.10.1/24 dev wg0
    [#] ip link set mtu 1420 up dev wg0
    [#] iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT
    [albr@albr-arch ~]$ ip a
    
    -----------------------------------CUT---------------------------------
    
    3: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
        link/none
        inet 10.10.10.1/24 scope global wg0
           valid_lft forever preferred_lft forever
    ```
    
    ```bash
    [albr@albr-arch ~]$ wg-quick down wg0
    [#] ip link delete dev wg0
    [#] iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT
    [albr@albr-arch ~]$ ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host noprefixroute
           valid_lft forever preferred_lft forever
    2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 00:0c:29:e8:2d:1d brd ff:ff:ff:ff:ff:ff
        altname enp2s1
        altname enx000c29e82d1d
        inet 192.168.14.136/24 brd 192.168.14.255 scope global dynamic noprefixroute ens33
           valid_lft 1631sec preferred_lft 1406sec
        inet6 fe80::9722:da02:e854:550a/64 scope link
           valid_lft forever preferred_lft forever
    ```
    

## systemd

- Podemos hacer esto también mediante `systemd` utilizando la siguiente unidad:
    
    ```bash
    [albr@albr-arch system]$ systemctl list-unit-files --type=service "wg-quick*"
    UNIT FILE         STATE    PRESET
    wg-quick@.service disabled disabled
    
    1 unit files listed.
    ```
    
- Podemos activar y desactivar la interfaz de la siguiente forma:
    
    ```bash
    [albr@albr-arch system]$ ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host noprefixroute
           valid_lft forever preferred_lft forever
    2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 00:0c:29:e8:2d:1d brd ff:ff:ff:ff:ff:ff
        altname enp2s1
        altname enx000c29e82d1d
        inet 192.168.14.136/24 brd 192.168.14.255 scope global dynamic noprefixroute ens33
           valid_lft 1155sec preferred_lft 930sec
        inet6 fe80::9722:da02:e854:550a/64 scope link
           valid_lft forever preferred_lft forever
           
    [albr@albr-arch system]$ sudo systemctl start wg-quick@wg0
    [albr@albr-arch system]$ ip a
    
    ------------------------------CUT-------------------------------
    4: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
        link/none
        inet 10.10.10.1/24 scope global wg0
           valid_lft forever preferred_lft forever
    ```
    
    ```bash
    [albr@albr-arch system]$ sudo systemctl stop wg-quick@wg0
    [albr@albr-arch system]$ ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host noprefixroute
           valid_lft forever preferred_lft forever
    2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 00:0c:29:e8:2d:1d brd ff:ff:ff:ff:ff:ff
        altname enp2s1
        altname enx000c29e82d1d
        inet 192.168.14.136/24 brd 192.168.14.255 scope global dynamic noprefixroute ens33
           valid_lft 1097sec preferred_lft 872sec
        inet6 fe80::9722:da02:e854:550a/64 scope link
           valid_lft forever preferred_lft forever
    ```
    
- Podemos hacer que se habilite de forma predeterminada al iniciar el sistema con el siguiente comando:
    
    ```bash
    [albr@albr-arch system]$ sudo systemctl enable wg-quick@wg0
    Created symlink '/etc/systemd/system/multi-user.target.wants/wg-quick@wg0.service' → '/usr/lib/systemd/system/wg-quick@.service'.
    ```
    
    - De esta forma siempre que se inicie el sistema se levantará la interfaz de forma automatica

# Cliente

- Debemos crear un archivo de configuración para cada cliente:
    
    ```bash
    [albr@albr-arch ~]$ cat client_wg0.conf
    
    [Interface]
    PrivateKey = <Client Private Key>
    Address = 10.10.10.2/24
    DNS = <Server IP>
    
    [Peer]
    PublicKey = <Server Public Key>
    AllowedIPs = 0.0.0.0/0, ::/0
    Endpoint = <Server IP>:51820 
    persistentKeepalive = 25
    ```
    
- Con esta configuración ya deberiamos tener lo más básico funcionando.

# Comprobar Funcionamiento

- Para comprobar que todo funciona correactamente tenemos el siguiente escenario de prueba:
    - Dos redes: `192.168.1.0/24` (con sálida a internet) y `192.168.129.0/24` ( sin sálida).
    - 3 Hosts:
        - Windows: Con IP `192.168.1.43`
        - Arch Linux (VPN Server): Con dos adaptadores de red: `192.168.1.94` y `192.168.129.135`
            
            ```bash
            [root@albr-arch albr]# ip a
            1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
                link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
                inet 127.0.0.1/8 scope host lo
                   valid_lft forever preferred_lft forever
                inet6 ::1/128 scope host noprefixroute
                   valid_lft forever preferred_lft forever
            2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
                link/ether 00:0c:29:e8:2d:1d brd ff:ff:ff:ff:ff:ff
                altname enp2s1
                altname enx000c29e82d1d
                inet 192.168.129.135/24 brd 192.168.129.255 scope global dynamic noprefixroute ens33
                   valid_lft 1111sec preferred_lft 886sec
                inet6 fe80::9722:da02:e854:550a/64 scope link
                   valid_lft forever preferred_lft forever
            3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
                link/ether 00:0c:29:e8:2d:27 brd ff:ff:ff:ff:ff:ff
                altname enp2s5
                altname enx000c29e82d27
                inet 192.168.1.94/24 brd 192.168.1.255 scope global dynamic noprefixroute ens37
                   valid_lft 79410sec preferred_lft 68610sec
                inet6 fe80::83d0:5b96:ca57:c78e/64 scope link
                   valid_lft forever preferred_lft forever
            ```
            
        - Ubuntu Server: Con IP `192.168.129.136`
            
            ```bash
            root@Ubuntu-Server:~# ip a
            1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
                link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
                inet 127.0.0.1/8 scope host lo
                   valid_lft forever preferred_lft forever
                inet6 ::1/128 scope host noprefixroute
                   valid_lft forever preferred_lft forever
            2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
                link/ether 00:0c:29:7c:33:3f brd ff:ff:ff:ff:ff:ff
                altname enp2s1
                altname enx000c297c333f
                inet 192.168.129.136/24 metric 100 brd 192.168.129.255 scope global dynamic ens33
                   valid_lft 1652sec preferred_lft 1652sec
                inet6 fe80::20c:29ff:fe7c:333f/64 scope link proto kernel_ll
                   valid_lft forever preferred_lft forever
            ```
            
    - Al conectarnos al tunel deberiamos poder acceder a internet desde ubuntu y tambien llegar a la máquina Windows.
        - Primero sin configurar nada intentamos hacer ping a la ip de la máquina `Windows` y a la DNS de Google:
            
            ```bash
            root@Ubuntu-Server:~# ping -c 1 192.168.1.43
            ping: connect: Network is unreachable
            root@Ubuntu-Server:~# ping -c 1 8.8.8.8
            ping: connect: Network is unreachable
            ```
            
        - Como era de esperar no funciona, asi que levantamos la vpn en el servidor y luego conectamos el cliente:
            
            ```bash
            [root@arch ~]# systemctl start wg-quick@wg0
            [root@arch ~]# ip a
            
            ------------------------CUT---------------------------
            
            6: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
                link/none
                inet 10.10.10.1/32 scope global wg0
                   valid_lft forever preferred_lft forever
            ```
            
            ```bash
            root@Ubuntu-Server:~# systemctl start wg-quick@wg0
            root@Ubuntu-Server:~# ip a
            
            -------------------------CUT------------------------
            
            9: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
                link/none
                inet 10.10.10.2/24 scope global wg0
                   valid_lft forever preferred_lft forever
            ```
            
        - Ahora realizamos las mismas pruebas anteriores:
            
            ```bash
            root@Ubuntu-Server:~# ping -c1 192.168.1.43
            PING 192.168.1.43 (192.168.1.43) 56(84) bytes of data.
            64 bytes from 192.168.1.43: icmp_seq=1 ttl=127 time=2.34 ms
            
            --- 192.168.1.43 ping statistics ---
            1 packets transmitted, 1 received, 0% packet loss, time 0ms
            rtt min/avg/max/mdev = 2.342/2.342/2.342/0.000 ms
            root@Ubuntu-Server:~# ping -c1 8.8.8.8
            PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
            64 bytes from 8.8.8.8: icmp_seq=1 ttl=119 time=5.58 ms
            
            --- 8.8.8.8 ping statistics ---
            1 packets transmitted, 1 received, 0% packet loss, time 0ms
            rtt min/avg/max/mdev = 5.583/5.583/5.583/0.000 ms
            ```
            
        - Esta vez si funciona, lo que nos confirma que el tunel está funcionando correctamente, para ver si se está realizando nat en el servidor VPN, levantamos de forma rápida un servidor http con python el la máquina Windows:
            
            ```powershell
            PS C:\Users\alber> python3 -m http.server 80
            Serving HTTP on :: port 80 (http://[::]:80/) 
            ```
            
        - Ahora enviamos una solicitud http desde `Ubuntu`  si todo está bien deberiamos ver la ip del `Servidor VPN` (`192.168.1.94`):
            
            ```bash
            root@Ubuntu-Server:~# curl 192.168.1.43 &>/dev/null
            ```
            
            ```powershell
            PS C:\Users\alber> python3 -m http.server 80
            Serving HTTP on :: port 80 (http://[::]:80/) ...
            ::ffff:192.168.1.94 - - [14/Mar/2026 14:16:45] "GET / HTTP/1.1" 200 -
            ```
            
            - Se muestra la IP del Servidor VPN lo que nos confirma que se está utilizando NAT.
                - Para este ejemplo tuve que cambiar la interfaz de las reglas de firewall a `ens37`

# DNS

## DNS Público

- Aunque el tunel funciona, si intentamos enviar un ping a `google.es` vemos que falla la resolución DNS:
    
    ```bash
    root@Ubuntu-Server:~# ping google.es
    ping: google.es: Temporary failure in name resolution
    ```
    
- Para solucionar esto podemos agregar en la configuración del cliente la dirección IP de un servidor VPN válido, por ejemplo la dns de google: `8.8.8.8`
    
    ```bash
    root@Ubuntu-Server:/etc/wireguard# cat wg0.conf
    [Interface]
    PrivateKey = <Private Key>
    Address = 10.10.10.2/24
    DNS = 8.8.8.8
    
    [Peer]
    PublicKey = <Public Key>
    AllowedIPs = 0.0.0.0/0, ::/0
    Endpoint = 192.168.129.135:51820
    persistentKeepalive = 25
    ```
    
    - Si reiniciamos la conexión vpn en el cliente pasa lo siguiente:
        
        ```bash
        root@Ubuntu-Server:/etc/wireguard# wg-quick up wg0
        stat: cannot read table of mounted file systems: Permission denied
        stat: cannot read table of mounted file systems: Permission denied
        /usr/bin/wg-quick: line 47: ((: ( &  & 0007) == 0: syntax error: operand expected (error token is "&  & 0007) == 0")
        [#] ip link add wg0 type wireguard
        [#] wg setconf wg0 /dev/fd/63
        [#] ip -4 address add 10.10.10.2/24 dev wg0
        [#] ip link set mtu 1420 up dev wg0
        [#] resolvconf -a wg0 -m 0 -x
        /usr/bin/wg-quick: line 32: resolvconf: command not found
        [#] ip link delete dev wg0
        ```
        
    - Para solucionar esto entre varias opciones podemos usar `systemd-resolved`
        - En caso de obtener este error con arch, podemos solucionarlo instalando el siguiente paquete:
            
            ```bash
            [root@albr-arch albr]# pacman -Ss systemd-resolvconf
            core/systemd-resolvconf 259.5-1 [installed]
                systemd resolvconf replacement (for use with systemd-resolved)
            ```
            
    - Para esto primero verificamos que `systemd-resolved` exista en el sistema:
        
        ```bash
        root@Ubuntu-Server:/home/albr# systemctl list-unit-files systemd-resolved.service
        UNIT FILE                STATE    PRESET
        systemd-resolved.service disabled enabled
        
        1 unit files listed.
        ```
        
    - Luego lo configuramos para que se ejecute al iniciar el sistema:
        
        ```bash
        root@Ubuntu-Server:/home/albr# systemctl enable --now systemd-resolved
        Created symlink '/etc/systemd/system/dbus-org.freedesktop.resolve1.service' → '/usr/lib/systemd/system/systemd-resolved.service'.
        Created symlink '/etc/systemd/system/sysinit.target.wants/systemd-resolved.service' → '/usr/lib/systemd/system/systemd-resolved.service'.
        ```
        
    - Por último creamos el siguiente enlace simbólico al archivo `/etc/resolv.conf`
        
        ```bash
        root@Ubuntu-Server:/home/albr# ln -sf ../run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
        root@Ubuntu-Server:/home/albr# ls -l /etc/resolv.conf
        lrwxrwxrwx 1 root root 39 Mar 14 15:42 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
        ```
        
- Ahora podemos levantar la interfaz correctamente:
    
    ```bash
    root@Ubuntu-Server:/home/albr# systemctl start wg-quick@wg0
    root@Ubuntu-Server:/home/albr# ip a
    
    ---------------------------CUT------------------------------
    
    4: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
        link/none
        inet 10.10.10.2/24 scope global wg0
           valid_lft forever preferred_lft forever
    ```
    
- Si intentamos hacer ping a `google.es` lo logramos sin problemas:
    
    ```bash
    root@Ubuntu-Server:/home/albr# ping -c1 google.es
    PING google.es (142.251.140.227) 56(84) bytes of data.
    64 bytes from dia01s03-in-f3.1e100.net (142.251.140.227): icmp_seq=1 ttl=118 time=8.66 ms
    
    --- google.es ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms
    rtt min/avg/max/mdev = 8.663/8.663/8.663/0.000 ms
    ```
    

## Servidor DNS

- Aunque con la solución anterior funcionaria, en mi caso implementaré un servidor DNS recursivo que opere directamente en el VPN, ya que esto me permitirá más control sobre las consultas y mayor privacidad.
- Para esto vamos a utilizar [Unbound](https://unbound.docs.nlnetlabs.nl/en/latest/), por lo que primeramente instalamos el paquete:
    
    ```bash
    [root@albr-arch albr]# pacman -S unbound
    resolving dependencies...
    --------------------CUT--------------------
    
    [root@albr-arch albr]# pacman -Ss ^unbound$
    extra/unbound 1.24.2-2 [installed]
        Validating, recursive, and caching DNS resolver
    ```
    
- Para comprobar que se instalo correctamente, podemos iniciar el servicio e intentar de resolver la ip de `google.es`:
    
    ```bash
    [root@albr-arch albr]# systemctl start unbound.service
    [root@albr-arch albr]# dig google.es @127.0.0.1
    
    ; <<>> DiG 9.20.20 <<>> google.es @127.0.0.1
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58894
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
    
    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 1232
    ;; QUESTION SECTION:
    ;google.es.                     IN      A
    
    ;; ANSWER SECTION:
    google.es.              300     IN      A       142.250.185.3
    
    ;; Query time: 536 msec
    ;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
    ;; WHEN: Mon Mar 16 13:05:58 CET 2026
    ;; MSG SIZE  rcvd: 54
    ```
    
    - Al parecer está funcionando correctamente.
    - Para que `Unbound` se inicie automaticamente al iniciar la máquina hacemos lo siguiente:
        
        ```bash
        [root@albr-arch albr]# systemctl enable unbound
        Created symlink '/etc/systemd/system/multi-user.target.wants/unbound.service' → '/usr/lib/systemd/system/unbound.service'.
        ```
        

### Configurando Unbound de forma local

- Podemos configurar nuestra máquina para que utilice directamente `unbound` simplemente escribiendo directamente el archivo `/etc/resolv.conf`

```bash
[root@albr-arch albr]# cat /etc/resolv.conf

nameserver 127.0.0.1
```

- Luego comprobamos que se esté utilizando:
    
    ```bash
    [root@albr-arch albr]# dig google.es
    
    ; <<>> DiG 9.20.20 <<>> google.es
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49264
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
    
    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 1232
    ;; QUESTION SECTION:
    ;google.es.                     IN      A
    
    ;; ANSWER SECTION:
    google.es.              300     IN      A       142.250.184.3
    
    ;; Query time: 145 msec
    ;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
    ;; WHEN: Wed Mar 18 15:05:32 CET 2026
    ;; MSG SIZE  rcvd: 54
    ```
    
    - Como podemos ver se está utilizando nuestro responder que se encuentra en escucha en `127.0.0.1:53`

### Configurando Unbound para el resto de la red

- Actualmente no podemos utilizar `Unbound` desde otra máquina de la red:
    
    ```bash
    root@Ubuntu-Server:/etc/wireguard# dig google.es @10.10.10.1
    ;; communications error to 10.10.10.1#53: connection refused
    ;; communications error to 10.10.10.1#53: connection refused
    ;; communications error to 10.10.10.1#53: connection refused
    
    ; <<>> DiG 9.20.11-1ubuntu2.1-Ubuntu <<>> google.es @10.10.10.1
    ;; global options: +cmd
    ;; no servers could be reached
    ```
    
- Esto pasa porque actualmente `Unbound` solamente está escuchando en `127.0.0.1:53`, para permitir que escuche en otras interfaces podemos editar su archivo de configuración `/etc/unbound/unbound.conf`
    - Este archivo de configuración viene de forma predeterminada con todas las configuraciones posibles, lo que lo hace bastante extense:
        
        ```bash
        [root@albr-arch unbound]# wc -l unbound.conf
        1424 unbound.conf
        ```
        
    - Ya que no necesitamos toda esta configuración, e iremos agregando poco a poco lo que necesitemos, vamos a sustituir todo el contenido de este archivo con la siguiente configuración, siguiendo la recomendación de la documentación oficial: [https://unbound.docs.nlnetlabs.nl/en/latest/use-cases/home-resolver.html#setting-up-for-the-rest-of-the-network](https://unbound.docs.nlnetlabs.nl/en/latest/use-cases/home-resolver.html#setting-up-for-the-rest-of-the-network)
        
        ```bash
        server:
            # location of the trust anchor file that enables DNSSEC
            auto-trust-anchor-file: "/var/lib/unbound/root.key"
            # send minimal amount of information to upstream servers to enhance privacy
            qname-minimisation: yes
            # the interface that is used to connect to the network (this will listen to all interfaces)
            interface: 0.0.0.0
            # interface: ::0
            # addresses from the IP range that are allowed to connect to the resolver
            access-control: 192.168.0.0/16 allow
            # access-control: 2001:DB8/64 allow
        
        remote-control:
            # allows controling unbound using "unbound-control"
            control-enable: yes
        ```
        
    - Para que funcione podemos adaptar esta configuración a nuestro caso:
        
        ```bash
        [root@albr-arch unbound]# cat unbound.conf
        server:
            # location of the trust anchor file that enables DNSSEC
            auto-trust-anchor-file: "/etc/unbound/trusted-key.key"
            # send minimal amount of information to upstream servers to enhance privacy
            qname-minimisation: yes
            # the interface that is used to connect to the network (this will listen to all interfaces)
            interface: 0.0.0.0
            # interface: ::0
            # addresses from the IP range that are allowed to connect to the resolver
            access-control: 10.10.10.0/24 allow
            access-control: 127.0.0.0/8 allow
            # access-control: 2001:DB8/64 allow
        
        remote-control:
            # allows controling unbound using "unbound-control"
            control-enable: no
        ```
        
        - En esta configuración mediante `interface` estamos haciendo que `Unbound` escuche en todas las interfaces
        - También con `access-control` hacemos que solo se permitan las solicitudes que provienen de la red de la vpn (`10.10.10.0/24`) y el localhost
        - Establecí `control-enable` en `no` ya que no voy a utilizar por ahora esta utilidad.
        - También actualizamos la ubicación de la key para ralizar la validación DNSSEC, esta key es una copia de `/etc/trusted-key.key`, como se explica en [https://wiki.archlinux.org/title/Unbound#DNSSEC_validation](https://wiki.archlinux.org/title/Unbound#DNSSEC_validation)
            - Para validar que DNSSEC este funcionando correctamente podemos utilizar la utilidad `drill`
                - Primero comprobamos si el `resolver` no devuelve una respuesta para un dominio con una firma no válida:
                    
                    ```bash
                    [root@albr-arch unbound]# drill -D badsig.test.dnscheck.tools
                    ;; ->>HEADER<<- opcode: QUERY, rcode: SERVFAIL, id: 3078
                    ;; flags: qr rd ra ; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0
                    ;; QUESTION SECTION:
                    ;; badsig.test.dnscheck.tools.  IN      A
                    
                    ;; ANSWER SECTION:
                    
                    ;; AUTHORITY SECTION:
                    
                    ;; ADDITIONAL SECTION:
                    
                    ;; Query time: 305 msec
                    ;; EDNS: version 0; flags: do ; udp: 1232
                    ;; SERVER: 127.0.0.1
                    ;; WHEN: Mon Mar 16 15:58:32 2026
                    ;; MSG SIZE  rcvd: 55
                    ```
                    
                    - Obtenemos `rcode: SERVFAIL` lo que significa que se está validando.
                - Ahora consultamos un dominio con una firma válida y la respuesta debe ser contraria:
                    
                    ```bash
                    [root@albr-arch unbound]# drill -D test.dnscheck.tools
                    ;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 2004
                    ;; flags: qr rd ra ad ; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0
                    ;; QUESTION SECTION:
                    ;; test.dnscheck.tools. IN      A
                    
                    ;; ANSWER SECTION:
                    test.dnscheck.tools.    1       IN      A       116.203.95.251
                    test.dnscheck.tools.    1       IN      RRSIG   A 13 3 1 20260316155947 20260316135946 5069 test.dnscheck.tools. w58W/LXouw1X9Pv/vsO6JsvF6aBNJgds3VbjVw9QsOjc/9vgre+7n6m1iNRdSbKvVhNTJLuilAwGySJbcdq8wg==
                    
                    ;; AUTHORITY SECTION:
                    
                    ;; ADDITIONAL SECTION:
                    
                    ;; Query time: 37 msec
                    ;; EDNS: version 0; flags: do ; udp: 1232
                    ;; SERVER: 127.0.0.1
                    ;; WHEN: Mon Mar 16 15:59:46 2026
                    ;; MSG SIZE  rcvd: 179
                    ```
                    
    - Ahora podemos desde nuestro cliente, hacer consultas dns utilizando `Unbound`:
        
        ```bash
        root@Ubuntu-Server:/etc/wireguard# hostname
        Ubuntu-Server
        root@Ubuntu-Server:/etc/wireguard# dig google.es @10.10.10.1
        
        ; <<>> DiG 9.20.11-1ubuntu2.1-Ubuntu <<>> google.es @10.10.10.1
        ;; global options: +cmd
        ;; Got answer:
        ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 36057
        ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
        
        ;; OPT PSEUDOSECTION:
        ; EDNS: version: 0, flags:; udp: 1232
        ;; QUESTION SECTION:
        ;google.es.                     IN      A
        
        ;; ANSWER SECTION:
        google.es.              37      IN      A       142.250.184.3
        
        ;; Query time: 0 msec
        ;; SERVER: 10.10.10.1#53(10.10.10.1) (UDP)
        ;; WHEN: Mon Mar 16 13:41:44 UTC 2026
        ;; MSG SIZE  rcvd: 54
        ```
        
    - Para que esto se configure de forma automatica podemos editar nuestro archivo `wg0.conf` en el cliente para que incluya la dirección DNS:
        
        ```bash
        root@Ubuntu-Server:/etc/wireguard# cat client_wg0.conf
        
        [Interface]
        PrivateKey = <Client Private Key>
        Address = 10.10.10.2/24
        DNS = 10.10.10.1
        
        [Peer]
        PublicKey = <Server Public Key>
        AllowedIPs = 0.0.0.0/0, ::/0
        Endpoint = 192.168.129.135:51820 
        persistentKeepalive = 25
        ```
        
        - De esta manera al conectarnos a la vpn con esta configuración utilzaremos por defecto `Unbound`:
            
            ```bash
            root@Ubuntu-Server:/etc/wireguard# nslookup google.es
            ;; Got SERVFAIL reply from 127.0.0.53
            Server:         127.0.0.53
            Address:        127.0.0.53#53
            
            ** server can't find google.es: SERVFAIL
            
            root@Ubuntu-Server:/etc/wireguard# systemctl restart wg-quick@wg0
            root@Ubuntu-Server:/etc/wireguard# nslookup google.es
            Server:         127.0.0.53
            Address:        127.0.0.53#53
            
            Non-authoritative answer:
            Name:   google.es
            Address: 142.250.184.3
            Name:   google.es
            Address: 2a00:1450:4003:808::2003
            ```
            

### Caché NSEC Agresiva

- Esta opción es muy interesante, ya que nos permite consultar de forma rápida y sin salir a internet mediante el registro `NSEC` si un dominio existe o no.
- `NSEC` es un registro el cual al solicitar un dominio dentro de una zona, nos responde con el proximo dominio válido en orden lexicográfico dentro de esa zona, lo que permite consultar de forma rápida si un dominio existe sin necesidad de realizar la consulta a los servidores.
    - Por ejemplo: Si realizamos una consulta a `admin.ejemplo.com` el registro `NSEC` nos responde con el siguiente dominio en orden lexicográfico, por ejemplo `ver.ejemplo.com` , de esta forma sabemos que el siguiente dominio válido es `ver.ejemplo.com`
    - Gracias a esto, si realizamos una consulta a  `dev.ejemplo.com` como `d` se encuentra entre `a` y `v`, de forma rápida y sin necesidad de consultar a los servidores podemos saber que este dominio no existe en esa zona.
    - Sin embargo si realizamos una consulta a `web.ejemplo.com` como `w` no se encuentra dentro de  `a` y `v`, la consulta se realizará.
- Gracias a esto, mejoramos la privacidad al no tener que enviar todas las consultas, mejoramos los tiempos de respuesta y el rendimiento, y disminuimos el tráfico en la red.
- Esto podemos activarlo agregando lo siguiente en el archivo de configuración de `Unbound`:
    
    ```bash
    aggressive-nsec: yes
    ```
    
    ```bash
    [root@albr-arch albr]# cat /etc/unbound/unbound.conf
    server:
        # location of the trust anchor file that enables DNSSEC
        auto-trust-anchor-file: "/etc/unbound/trusted-key.key"
        # send minimal amount of information to upstream servers to enhance privacy
        qname-minimisation: yes
        # the interface that is used to connect to the network (this will listen to all interfaces)
        interface: 0.0.0.0
    
        # addresses from the IP range that are allowed to connect to the resolver
        access-control: 10.10.10.0/24 allow
        access-control: 127.0.0.0/8 allow
    
        aggressive-nsec: yes
    
    remote-control:
        # allows controling unbound using "unbound-control"
        control-enable: no
    ```
    

### Root Hints

- Aunque `unbound` viene con `root hints` predeterminadas, es buena práctica mantener estas actualizadas, para eso podemos indicarlas en el archivo de configuración de `unobund` de la siguiente forma:
    
    ```bash
    root-hints: root.hints
    ```
    
- Estas podemos descargarlas con el siguiente comando y almacenarlas en el directorio de configuración de `unbound`:
    
    ```bash
    curl -s --output /etc/unbound/root.hints https://www.internic.net/domain/named.cache
    ```
    
    ```bash
    [root@albr-arch unbound]# curl -s --output /etc/unbound/root.hints https://www.internic.net/domain/named.cache
    [root@albr-arch unbound]# ls
    dev  root.hints  run  trusted-key.key  unbound.conf
    ```
    
- Para asegurarnos que nuestras `root hints` se encuentren actualizadas debemos actualizar este archivo cada cierto tiempo, esto lo podemos automatizar mediante un `timer` de `systemd` , como se muestra en: [https://wiki.archlinux.org/title/Unbound#Roothints_systemd_timer](https://wiki.archlinux.org/title/Unbound#Roothints_systemd_timer)
    - Primero creamos el servicio `roothints.service` en el directorio `/etc/systemd/system/` con el siguiente contenido:
        
        ```bash
        [root@albr-arch system]# realpath roothints.service
        /etc/systemd/system/roothints.service
        [root@albr-arch system]# cat roothints.service
        [Unit]
        Description=Update root hints for unbound
        After=network.target
        
        [Service]
        ExecStart=/usr/bin/curl -o /etc/unbound/root.hints https://www.internic.net/domain/named.cache
        ```
        
    - Luego en la misma ubicación creamos el `timer`, con el siguiente contenido:
        
        ```bash
        [root@albr-arch system]# realpath roothints.timer
        /etc/systemd/system/roothints.timer
        [root@albr-arch system]# cat roothints.timer
        [Unit]
        Description=Run root.hints monthly
        
        [Timer]
        OnCalendar=monthly
        Persistent=true
        
        [Install]
        WantedBy=timers.target
        ```
        
    - Por último habilitamos el `timer`:
        
        ```bash
        [root@albr-arch system]# systemctl enable --now roothints.timer
        Created symlink '/etc/systemd/system/timers.target.wants/roothints.timer' → '/etc/systemd/system/roothints.timer'.
        
        [root@albr-arch system]# systemctl status roothints.timer
        ● roothints.timer - Run root.hints monthly
             Loaded: loaded (/etc/systemd/system/roothints.timer; enabled; preset: disabled)
             Active: active (waiting) since Wed 2026-03-18 14:32:41 CET; 1s ago
         Invocation: e665b30a589a46e8b95729d01a8ba0ef
            Trigger: Wed 2026-04-01 00:00:00 CEST; 1 week 6 days left
           Triggers: ● roothints.service
        
        Mar 18 14:32:41 albr-arch systemd[1]: Started Run root.hints monthly.
        ```
        
    - De esta forma tendremos nuestras `root hints` actualizadas de forma mensual.

### Otras Configuraciones

- Para evitar identificación de versión e identidad es recomendable utilizar las siguientes opciones:
    
    ```bash
    hide-identity: yes
    hide-version: yes
    ```
    
    - Sin estas opciones si realizamos un analisis de versión con nmap al puerto 53 vemos lo siguiente:
        
        ```bash
        root@Ubuntu-Server:/home/albr# nmap -p53 10.10.10.1 -sCV
        Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-18 15:49 UTC
        Nmap scan report for 10.10.10.1
        Host is up (0.00089s latency).
        
        PORT   STATE SERVICE VERSION
        53/tcp open  domain  Unbound 1.24.2
        | dns-nsid:
        |   id.server: albr-arch
        |_  bind.version: unbound 1.24.2
        
        Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
        Nmap done: 1 IP address (1 host up) scanned in 14.27 seconds
        ```
        
    - Como podemos ver nmap está recopilando esta información.
    - Si añadimos las opciones anteriores a la configuración de `unbound` y reiniciamos el servicio, para luego repetir el escaner vemos lo siguiente:
        
        ```bash
        [root@albr-arch albr]# vim /etc/unbound/unbound.conf
        [root@albr-arch albr]# cat /etc/unbound/unbound.conf
        server:
        		# username
            username: unbound
            # location of the trust anchor file that enables DNSSEC
            auto-trust-anchor-file: "/etc/unbound/trusted-key.key"
            # send minimal amount of information to upstream servers to enhance privacy
            qname-minimisation: yes
            # the interface that is used to connect to the network (this will listen to all interfaces)
            interface: 0.0.0.0
        
            # addresses from the IP range that are allowed to connect to the resolver
            access-control: 10.10.10.0/24 allow
            access-control: 127.0.0.0/8 allow
            
            # aggressive nsec caching
            aggressive-nsec: yes
        
            # root hints
            root-hints: root.hints
        
            # hide identity and version
            hide-identity: yes
            hide-version: yes
        
        remote-control:
            # allows controling unbound using "unbound-control"
            control-enable: no
        
        [root@albr-arch albr]# systemctl restart unbound
        ```
        
        ```bash
        root@Ubuntu-Server:/home/albr# nmap -p53 10.10.10.1 -sCV
        Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-18 15:56 UTC
        Nmap scan report for 10.10.10.1
        Host is up (0.0012s latency).
        
        PORT   STATE SERVICE VERSION
        53/tcp open  domain  Unbound
        
        Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
        Nmap done: 1 IP address (1 host up) scanned in 14.27 seconds
        ```
        
        - Como podemos observar nmap no fue capaz de recolectar toda la información.

# Firewall

## Configuración Básica

- Para proteger nuestro servidor vamos a configurar nuestro firewall utilzando `nftables`, para esto empezamos configurando las mismas reglas que hasta ahora teniamos con `iptables` pero con la sintaxis de `nftables` :
    
    ```bash
    iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
    ```
    
    - Primero podemos crear una tabla, que trabaje sobre `IPv4` e `IPv6` , esto lo hacemos especificando la familia `inet`
        
        ```bash
        #!/usr/bin/nft -f
        
        flush ruleset
        
        table inet filter {}
        ```
        
        - `flush ruleset` limpia las configuraciones previas de nuestro firewall.
    - Dentro de esta tabla podemos crear una cadena, la cual trabajara en el `hook` `forward` (afecta al trafico que no va dirigido a la máquina, sino que se dirige hacia otra interfaz)
        - Dentro de esta cadena podemos definir varias reglas:
            - Establecer una politica predeterminada de `drop`, haciendo que todo paquete que no sea afectado por ninguna regla se descarte.
            - Aceptar todos los paquetes que pertenezcan a conexiones ya establecidas.
            - Aceptar los paquetes que provienen de la interfaz `wg0` y salgan por la interfaz con acceso a internet, `ens37` en mi caso.
            
            ```bash
            #!/usr/bin/nft -f
            
            flush ruleset
            
            table inet filter {
                    chain forward {
                            type filter hook forward priority 0;
                            policy drop;
            
                            ct state established,related accept
            
                            iifname wg0 oifname ens37 accept
                    }
            }
            ```
            
    - Para que los paquetes que atraviesen el servidor utilicen `snat` y de esta forma el servidor final sepa enviarlo de vuelta, debemos hacer `masquerade`:
        - Para esto creamos una cadena que trabaje en el `hook` `postrouting`
        - Esta cadena va a realizar masquerade a todo paquete que provenga de la interfaz `wg0` y se dirija a `ens37`:
        
        ```bash
        chain nat {
                        type nat hook postrouting priority srcnat;
                        policy accept;
        
                        iifname wg0 oifname ens37 masquerade
                }
        ```
        
- De esta forma nuestro archivo quedaria así:
    
    ```bash
    #!/usr/bin/nft -f
    
    flush ruleset
    
    table inet filter {
            chain forward {
                    type filter hook forward priority 0;
                    policy drop;
    
                    ct state established,related accept
    
                    iifname wg0 oifname ens37 accept
            }
    
            chain nat {
                    type nat hook postrouting priority srcnat;
                    policy accept;
    
                    iifname wg0 oifname ens37 masquerade
            }
    }
    ```
    
    - Ya que se repiten en las configuraciones las interfaces, podemos definir variables, para de esta forma cambiarlas de forma rápida:
        
        ```bash
        #!/usr/bin/nft -f
        
        flush ruleset
        
        define WG_IF = wg0
        define EXT_IF = ens37
        
        table inet filter {
                chain forward {
                        type filter hook forward priority 0;
                        policy drop;
        
                        ct state established,related accept
        
                        iifname $WG_IF oifname $EXT_IF accept
                }
        
                chain nat {
                        type nat hook postrouting priority srcnat;
                        policy accept;
        
                        iifname $WG_IF oifname $EXT_IF masquerade
                }
        }
        ```
        
- Para que se cargue al iniciar el sistema podemos utilzar la siguiente unidad de `systemd`:
    
    ```bash
    [root@albr-arch nftables]# systemctl list-unit-files nft*
    UNIT FILE        STATE    PRESET
    nftables.service disabled disabled
    
    1 unit files listed.
    ```
    
    ```bash
    [root@albr-arch nftables]# nft list ruleset
    [root@albr-arch nftables]# systemctl enable --now nftables
    Created symlink '/etc/systemd/system/multi-user.target.wants/nftables.service' → '/usr/lib/systemd/system/nftables.service'.
    [root@albr-arch nftables]# nft list ruleset
    table inet filter {
            chain forward {
                    type filter hook forward priority filter; policy drop;
                    ct state established,related accept
                    iifname "wg0" oifname "ens37" accept
            }
    
            chain nat {
                    type nat hook postrouting priority srcnat; policy accept;
                    iifname "wg0" oifname "ens37" masquerade
            }
    }
    ```
    
    - También debemos eliminar las lineas `PostUp` y `PostDown` del archivo de configuración de wireguard en el servidor para que quede de la siguiente forma:
        
        ```bash
        [root@albr-arch nftables]# cat /etc/wireguard/wg0.conf
        [Interface]
        Address = 10.10.10.1/32
        ListenPort = 51820
        PrivateKey = <Private Key>
        
        [Peer]
        # Client 1
        PublicKey = <Public Key>
        AllowedIPs = 10.10.10.2/32
        ```
        

## Protegiendo Nuestro Servidor

- Para mantener nuestro servidor protegido es recomendable filtrar los paquetes que llegan a nuestra máquina, para hacer esto podemos crear una cadena dentro de la tabla que trabaja en la familia `inet`, esta cadena va a trabajar en el `hook` `input` (paquetes dirigidos a la máquina) y como politica general va a descartar todos los paquetes que no coincidan con ninguna regla:
    
    ```bash
    [root@albr-arch albr]# cat /etc/nftables.conf
    #!/usr/bin/nft -f
    
    flush ruleset
    
    define WG_IF = wg0
    define EXT_IF = ens37
    
    table inet filter {
            
    ----------------CUT----------------------------
    
            chain input {
                    type filter hook input priority 0;
                    policy drop
            }
    }
    
    ----------------CUT----------------------------
    ```
    
- Como primeras reglas vamos a permitir todo el tráfico que pertenezca a una conexión establecida y a descartar el cual pertenezca a conexiones invalidas:
    - También permitimos todo el tráfico proveniente del localhost.
    
    ```bash
    chain input {
                    type filter hook input priority 0;
                    policy drop
                    
                    ct state established,related accept
                    ct state invalid drop
                    
                    iifname lo accept
            }
    ```
    
- A continuación vamos a permitir el tráfico desde el exterior solamente a los puertos necesarios, en este caso el de `wireguard` (`51820`):
    
    ```bash
    chain input {
                    type filter hook input priority 0;
                    policy drop
                    
                    ct state established,related accept
                    ct state invalid drop
                    
                    iifname lo accept
                    
                    ct state new udp dport 51820 accept
            }
    ```
    
    - La regla aplica especificamente a nuevas conexiones udp con destino al puerto `51820`.
- Támbien podemos permitir el tráfico hacia nuestro `resolver` solamente si proviene de la vpn
    
    ```bash
    chain input {
                    type filter hook input priority 0;
                    policy drop
                    
                    ct state established,related accept
                    ct state invalid drop
                    
                    iifname lo accept
                    
                    ct state new udp dport 51820 accept
                    
                    iifname $WG_IF ct state new tcp dport 53 accept
                    iifname $WG_IF ct state new udp dport 53 accept
            }
    ```
    
    - Las reglas especifican que nuevos paquetes provenientes de `wg0` tanto `tcp` como `udp` con destino al puerto `53` serán aceptados.
- De igual forma es recomendable permitir conexiones `ssh` solamente desde el tunel, de esta forma no tenemos el puerto `22` tan expuesto:
    
    ```bash
    chain input {
                    type filter hook input priority 0;
                    policy drop
                    
                    ct state established,related accept
                    ct state invalid drop
                    
                    iifname lo accept
                    
                    ct state new udp dport 51820 accept
                    
                    iifname $WG_IF ct state new tcp dport 53 accept
                    iifname $WG_IF ct state new udp dport 53 accept
                    
                    iifname $WG_IF ct state new tcp dport 22 accept
            }
    ```
    

## Protegiendo SSH contra Ataques de Fuerza Bruta

- Aunque al permitir conexiones ssh solamente desde el tunel disminuimos bastante los posibles ataques, es buena práctica establecer controles para evitar ataques de fuerza bruta.
- Para esto vamos a utilizar `timers` y `sets dinámicos`, primero vamos a establecer toda la lógica en una cadena diferente para tener todo un poco más organizado, para esto vamos a cambiar nuestra regla a ssh para que la envie a otra cadena y vamos a crear la nueva cadena:
    
    ```bash
    chain ssh_guard {
    
            }
    chain input {
                    type filter hook input priority 0;
                    policy drop
    
                    ct state established,related accept
                    ct state invalid drop
    
                    iifname lo accept
    
                    ct state new udp dport 51820 accept
    
                    iifname $WG_IF ct state new tcp dport 53 accept
                    iifname $WG_IF ct state new udp dport 53 accept
    
                    iifname $WG_IF ct state new tcp dport 22 jump ssh_guard
            }
    ```
    
- Ahora podemos empezar a establecer la lógica en la nueva cadena, la idea es que al entrar un primer paquete, este se añada a un primer set, el cual lo mantiene por unos pocos segundos, si entra otro paquete a ssh, con la misma ip antes de que se termine el tiempo, este se añadirá a un segundo set y si llega un tercero con la misma ip antes de que acabe el tiempo del segundo set, este se añadirá a un tercer set el cual bloqueará la ip por 12h.
    - Para implementar esto primero vamos a crear los 3 sets dentro de la tabla:
        
        ```bash
        ------------------------CUT----------------------
        table inet filter {
        
                set ssh_step1 {
                        typeof ip saddr
                        flags timeout
                }
        
                set ssh_step2 {
                        typeof ip saddr
                        flags timeout
                }
        
                set ip_blacklist {
                        typeof ip saddr
                        flags timeout
                }
         ------------------------CUT----------------------
        ```
        
        - Con `typeof` especificamos que vamos a estar utilzando la ip de origen de los paquetes
        - Con la flag `timeout` habilitamos que las entradas tengan un timeout.
    - Ahora que tenemos nuestros sets, podemos empezar a elaborar la lógica en nuestra cadena `ssh_guard`:
        - Primero al llegar un paquete a la chain `ssh_guard` se va a añadir al set `ssh_set1` por 10 segundos y se va a aceptar:
            
            ```bash
            chain ssh_guard {
                            add @ssh_step1 { ip saddr timeout 10s } accept
                    }
            ```
            
        - Si llega otro paquete y su ip de origen se encuentra en el primer set, se va a añadir por 10 segundo al set `ssh_step2` y se va a aceptar:
            
            ```bash
            chain ssh_guard {
                            ip saddr @ssh_step1 add @ssh_step2 { ip saddr timeout 10s } accept
                            add @ssh_step1 { ip saddr timeout 10s } accept
                    }
            ```
            
        - Si llega un tercer paquete y su ip de origen se encuentra en el segundo set este se añadirá al set `ip_blacklist` y se descartará
            
            ```bash
            chain ssh_guard {
                            ip saddr @ssh_step2 add @ip_blacklist { ip saddr timeout 12h } drop
                            ip saddr @ssh_step1 add @ssh_step2 { ip saddr timeout 10s } accept
                            add @ssh_step1 { ip saddr timeout 10s } accept
                    }
            ```
            
        - Para que la ip quede bloqueada debemos en la cadena `input` y `forward`para todo paquete dirigido a nuestro sistema, comprobar si la ip de origen se encuentra en la blacklist y si es el caso descartarlo:
            
            ```bash
            chain forward {
                            type filter hook forward priority 0;
                            policy drop;
            
                            ip saddr @ip_blacklist drop
            
                            ct state established,related accept
            
            chain input {
                            type filter hook input priority 0;
                            policy drop
            
                            ip saddr @ip_blacklist drop
            
                            ct state established,related accept
                            ct state invalid drop
            
                            iifname lo accept
            
                            ct state new udp dport 51820 accept
            
                            iifname $WG_IF ct state new tcp dport 53 accept
                            iifname $WG_IF ct state new udp dport 53 accept
            
                            iifname $WG_IF ct state new tcp dport 22 jump ssh_guard
                    }
            ```
            
        - Nuestra configuración quedaria de la siguiente forma:
            
            ```bash
            #!/usr/bin/nft -f
            
            flush ruleset
            
            define WG_IF = wg0
            define EXT_IF = ens37
            
            table inet filter {
            
                    set ssh_step1 {
                            typeof ip saddr
                            flags timeout
                    }
            
                    set ssh_step2 {
                            typeof ip saddr
                            flags timeout
                    }
            
                    set ip_blacklist {
                            typeof ip saddr
                            flags timeout
                    }
                    
                    chain ssh_guard {
                            ip saddr @ssh_step2 add @ip_blacklist { ip saddr timeout 12h } drop
                            ip saddr @ssh_step1 add @ssh_step2 { ip saddr timeout 10s } accept
                            add @ssh_step1 { ip saddr timeout 10s } accept
                    }
            
                    chain forward {
                            type filter hook forward priority 0;
                            policy drop;
            
                            ip saddr @ip_blacklist drop
            
                            ct state established,related accept
            
                            iifname $WG_IF oifname $EXT_IF accept
                    }
            
                    chain input {
                            type filter hook input priority 0;
                            policy drop
            
                            ip saddr @ip_blacklist drop
            
                            ct state established,related accept
                            ct state invalid drop
            
                            iifname lo accept
            
                            ct state new udp dport 51820 accept
            
                            iifname $WG_IF ct state new tcp dport 53 accept
                            iifname $WG_IF ct state new udp dport 53 accept
            
                            iifname $WG_IF ct state new tcp dport 22 jump ssh_guard
                    }
            
                    chain nat {
                            type nat hook postrouting priority srcnat;
                            policy accept;
            
                            iifname $WG_IF oifname $EXT_IF masquerade
                    }
            }
            ```
            

### Probando nuestra defensa

- Para ver si funciona nuestra defensa podemos añadir `counters` a las reglas dentro de la chain `ssh_guard`
    
    ```bash
    chain ssh_guard {
                    ip saddr @ssh_step2 counter add @ip_blacklist { ip saddr timeout 12h } drop
                    ip saddr @ssh_step1 counter add @ssh_step2 { ip saddr timeout 10s } accept
                    counter add @ssh_step1 { ip saddr timeout 10s } accept
            }
    ```
    
- Ahora si desde el cliente `Ubuntu Server` enviamos una conexión al puerto 22 vemos lo siguiente:
    
    ```bash
    root@Ubuntu-Server:/home/albr# nc 10.10.10.1 22
    SSH-2.0-OpenSSH_10.2
    
    Invalid SSH identification string.
    ```
    
    ```bash
    [root@albr-arch albr]# nft list ruleset
    table inet filter {
            set ssh_step1 {
                    typeof ip saddr
                    size 65535      # count 1
                    flags dynamic,timeout
                    elements = { 10.10.10.2 timeout 10s expires 6s899ms }
            }
    
    ----------------------CUT----------------------------
    
            chain ssh_guard {
                    ip saddr @ssh_step2 counter packets 0 bytes 0 add @ip_blacklist { ip saddr timeout 12h } drop
                    ip saddr @ssh_step1 counter packets 0 bytes 0 add @ssh_step2 { ip saddr timeout 10s } accept
                    counter packets 1 bytes 60 add @ssh_step1 { ip saddr timeout 10s } accept
            }
    }
    
    ----------------------CUT----------------------------
    ```
    
    - Como podemos ver se registro un paquete en el primer set por 10s.
- Ahora podemos enviar 2 paquetes seguidos a ver que pasa:
    
    ```bash
    root@Ubuntu-Server:/home/albr# nc 10.10.10.1 22
    SSH-2.0-OpenSSH_10.2
    
    Invalid SSH identification string.
    
    root@Ubuntu-Server:/home/albr# nc 10.10.10.1 22
    SSH-2.0-OpenSSH_10.2
    
    Invalid SSH identification string.
    ```
    
    ```bash
    [root@albr-arch albr]# nft list ruleset
    table inet filter {
            set ssh_step1 {
                    typeof ip saddr
                    size 65535      # count 1
                    flags dynamic,timeout
                    elements = { 10.10.10.2 timeout 10s expires 6s959ms }
            }
    
            set ssh_step2 {
                    typeof ip saddr
                    size 65535      # count 1
                    flags dynamic,timeout
                    elements = { 10.10.10.2 timeout 10s expires 8s14ms }
            }
    
    ----------------------CUT----------------------------
    
            chain ssh_guard {
                    ip saddr @ssh_step2 counter packets 0 bytes 0 add @ip_blacklist { ip saddr timeout 12h } drop
                    ip saddr @ssh_step1 counter packets 1 bytes 60 add @ssh_step2 { ip saddr timeout 10s } accept
                    counter packets 1 bytes 60 add @ssh_step1 { ip saddr timeout 10s } accept
            }
    }
    
    ----------------------CUT----------------------------
    ```
    
    - Como era de esperar al detectar que la ip de origen se encontraba en el primer set, el paquete se aceptó y se añadio al segundo set
- Ahora vamos a probar si al intentar 3 conexiones seguidas, se bloquea la ip y perdemos acceso a internet  ( ya que el cliente envia todo su tráfico a travez de la VPN) :
    
    ```bash
    root@Ubuntu-Server:/home/albr# ping -c 1 google.es
    PING google.es (142.250.184.3) 56(84) bytes of data.
    64 bytes from mad41s10-in-f3.1e100.net (142.250.184.3): icmp_seq=1 ttl=117 time=7.71 ms
    
    --- google.es ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms
    rtt min/avg/max/mdev = 7.711/7.711/7.711/0.000 ms
    root@Ubuntu-Server:/home/albr# nc 10.10.10.1 22
    SSH-2.0-OpenSSH_10.2
    
    Invalid SSH identification string.
    
    root@Ubuntu-Server:/home/albr# nc 10.10.10.1 22
    SSH-2.0-OpenSSH_10.2
    
    Invalid SSH identification string.
    
    root@Ubuntu-Server:/home/albr# nc 10.10.10.1 22
    ^C
    root@Ubuntu-Server:/home/albr# ping -c 1 google.es
    PING google.es (142.250.184.3) 56(84) bytes of data.
    
    --- google.es ping statistics ---
    1 packets transmitted, 0 received, 100% packet loss, time 0ms
    ```
    
    - Como podemos observar nuestra máquina podia hacer ping a [`google.es`](http://google.es), luego se intentó conectar 3 veces a ssh, las 2 primeras se aceptaron y la 3 se denegó, al intentar volver a hacer ping a `google.es` no pudimos.
    - Esto nos muestra que la ip se bloqueó correctamente, si utilizamos `nft list ruleset` podemos ver el flujo y la ip bloqueada en el set:
        
        ```bash
        [root@albr-arch albr]# nft list ruleset
        table inet filter {
        
        ----------------------CUT----------------------------
        
                set ip_blacklist {
                        typeof ip saddr
                        size 65535      # count 1
                        flags dynamic,timeout
                        elements = { 10.10.10.2 timeout 12h expires 11h59m39s594ms }
                }
        
        ----------------------CUT----------------------------
        
                chain ssh_guard {
                        ip saddr @ssh_step2 counter packets 1 bytes 60 add @ip_blacklist { ip saddr timeout 12h } drop
                        ip saddr @ssh_step1 counter packets 1 bytes 60 add @ssh_step2 { ip saddr timeout 10s } accept
                        counter packets 1 bytes 60 add @ssh_step1 { ip saddr timeout 10s } accept
                }
        }
        
        ----------------------CUT----------------------------
        ```
        
        - Como podemos se detecto 1 paquete en cada regla de la cadena `ssh_guard` y la ip se encuentra bloqueada correctamente en la blacklist.
- Por último podemos reiniciar `nftables` para que se elimine la ip de `Ubuntu` de la blacklist y probamos realizar un ataque de fuerza bruta con `hydra`:
    
    ```bash
    root@Ubuntu-Server:/home/albr# hydra -l test -P wordlist.txt  ssh://10.10.10.1
    Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).
    
    Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-20 13:05:17
    [WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
    [DATA] max 16 tasks per 1 server, overall 16 tasks, 190 login tries (l:1/p:190), ~12 tries per task
    [DATA] attacking ssh://10.10.10.1:22/
    [ERROR] all children were disabled due too many connection errors
    ```
    
    ```bash
    [root@albr-arch albr]# nft list ruleset
    table inet filter {
    
    ----------------------CUT----------------------------
    
            set ip_blacklist {
                    typeof ip saddr
                    size 65535      # count 1
                    flags dynamic,timeout
                    elements = { 10.10.10.2 timeout 12h expires 11h59m46s266ms }
            }
    
    ----------------------CUT----------------------------
    ```
    
    - Como podemos ver la ip se bloqueó exitosamente.

## Controlando el Tráfico Saliente

- Con nuestra configuración actual permitimos todo el tráfico saliente de nuestro servidor, para permitir solamente lo necesario vamos a crear dentro de la tabla `filter` una cadena `output` la cual va a trabajar en el `hook` `output` .
    
    ```bash
    table inet filter {
    
    ------------------------CUT----------------------
    
            chain output {
                    type filter hook output priority 0;
    				}
    }
    ```
    
- Esta cadena va a tener una politica predeterminada de `drop`:
    
    ```bash
    chain output {
                    type filter hook output priority 0;
                    policy drop;
    				}
    ```
    
- A continuación vamos a agregar la siguientes reglas:
    - Permitimos todo el tráfico que pertenezca a una conexión establecida:
        
        ```bash
        chain output {
                        type filter hook output priority 0;
                        policy drop;
                        
                        ct state established,related accept
        				}
        ```
        
    - Permitimos el tráfico hacia el localhost:
        
        ```bash
        chain output {
                        type filter hook output priority 0;
                        policy drop;
                        
                        ct state established,related accept
                        
                        oifname lo accept
        				}
        ```
        
    - Para tareas como actualizaciones con `Pacman` vamos a permitir el trafico `http/https`
        
        ```bash
        chain output {
                        type filter hook output priority 0;
                        policy drop;
                        
                        ct state established,related accept
                        
                        oifname lo accept
                        
                        ct state new tcp dport { 80, 443 } accept
        				}
        ```
        
    - Por último vamos a permitir el tráfico DNS y NTP:
        
        ```bash
        chain output {
                        type filter hook output priority 0;
                        policy drop;
                        
                        ct state established,related accept
                        
                        oifname lo accept
                        
                        ct state new tcp dport { 53, 80, 443 } accept
                        ct state new udp dport { 53, 123 } accept
        				}
        ```
        

## ICMP