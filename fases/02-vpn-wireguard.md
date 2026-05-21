**Tabla de Contenido**
-----
-----
- [Generar Claves](#generar-claves)
- [Servidor](#servidor)
  - [Archivo de Configuración](#archivo-de-configuración)
  - [Levantar la Interfaz](#levantar-la-interfaz)
    - [wg-quick](#wg-quick)
    - [systemd](#systemd)
- [Cliente](#cliente)
- [Comprobar Funcionamiento](#comprobar-funcionamiento)

------

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

## Archivo de Configuración

- Podemos crear el archivo de configuración del servidor `wg0.conf` en el directorio `/etc/wireguard`
- Aquí un ejemplo básico de este archivo:
  - [wg0.conf](/configs/02-vpn-wireguard/wg0_basico.conf)
    
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

## Levantar la Interfaz

### wg-quick

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
    

### systemd

- Podemos automatizar este paso mediante `systemd` utilizando la siguiente unidad:
    
    ```bash
    [albr@albr-arch system]$ systemctl list-unit-files --type=service "wg-quick*"
    UNIT FILE         STATE    PRESET
    wg-quick@.service disabled disabled
    
    1 unit files listed.
    ```
    
- Activamos y desactivamos la interfaz de la siguiente forma:
    
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
    
- Con el siguiente comandos hacemos que se habilite de forma predeterminada al iniciar el sistema:
    
    ```bash
    [albr@albr-arch system]$ sudo systemctl enable wg-quick@wg0
    Created symlink '/etc/systemd/system/multi-user.target.wants/wg-quick@wg0.service' → '/usr/lib/systemd/system/wg-quick@.service'.
    ```
    
    - De esta forma siempre que se inicie el sistema se levantará la interfaz de forma automatica

# Cliente

- Debemos crear un archivo de configuración para cada cliente:
  - [client_wg0.conf](/configs/02-vpn-wireguard/client_wg0.conf)
    
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
