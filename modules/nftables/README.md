# Tabla de Contenido


- [Firewall](#firewall)
  - [Configuración Básica](#configuración-básica)
  - [Protegiendo Nuestro Servidor](#protegiendo-nuestro-servidor)
  - [Protegiendo SSH contra Ataques de Fuerza Bruta](#protegiendo-ssh-contra-ataques-de-fuerza-bruta)
    - [Probando nuestra defensa](#probando-nuestra-defensa)
  - [Controlando el Tráfico Saliente](#controlando-el-tráfico-saliente)
  - [ICMP](#icmp)


# Firewall


## Configuración Básica

- Para proteger nuestro servidor vamos a configurar nuestro firewall utilizando `nftables`, para esto empezamos configurando las mismas reglas que hasta ahora teniamos con `iptables` pero con la sintaxis de `nftables` en el archivo `/etc/nftables.conf`:
  - [nftables.conf](./configs/nftables.conf)
    
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
        
        - Con `typeof` especificamos que vamos a estar utilizando la ip de origen de los paquetes
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

Parte de la configuración implementada en este módulo surge del análisis práctico de distintos protocolos de red y de posibles escenarios de abuso.

Para comprender el razonamiento detrás de ciertas reglas, mitigaciones y decisiones de hardening, se recomienda revisar la documentación técnica y laboratorios asociados:

- [`ICMP`](../../docs/icmp/README.md) — Análisis del protocolo, recreación de abusos y diseño de mitigaciones mediante `nftables`.