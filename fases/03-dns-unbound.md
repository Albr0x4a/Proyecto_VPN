**Tabla de Contenido**
--------
--------
- [DNS](#dns)
  - [DNS Público](#dns-público)
  - [Servidor DNS](#servidor-dns)
    - [Configurando Unbound de forma local](#configurando-unbound-de-forma-local)
    - [Configurando Unbound para el resto de la red](#configurando-unbound-para-el-resto-de-la-red)
    - [Caché NSEC Agresiva](#caché-nsec-agresiva)
    - [Root Hints](#root-hints)
    - [Otras Configuraciones](#otras-configuraciones)

-------

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
    - Este archivo de configuración viene de forma predeterminada con todas las configuraciones posibles, lo que lo hace bastante extenso:
        
        ```bash
        [root@albr-arch unbound]# wc -l unbound.conf
        1424 unbound.conf
        ```
        
    - Ya que no necesitamos toda esta configuración, e iremos agregando poco a poco lo que necesitemos, vamos a sustituir todo el contenido de este archivo con la siguiente configuración, siguiendo la recomendación de la documentación oficial: [https://unbound.docs.nlnetlabs.nl/en/latest/use-cases/home-resolver.html#setting-up-for-the-rest-of-the-network](https://unbound.docs.nlnetlabs.nl/en/latest/use-cases/home-resolver.html#setting-up-for-the-rest-of-the-network)
      - [unbound.conf](/configs/03-dns-unbound/unbound.conf)
        
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
        
        - De esta manera al conectarnos a la vpn con esta configuración utilizaremos por defecto `Unbound`:
            
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
        - [roothints.service](/systemd/roothints.service)
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
        - [roothints.timer](/systemd/roothints.timer)
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
