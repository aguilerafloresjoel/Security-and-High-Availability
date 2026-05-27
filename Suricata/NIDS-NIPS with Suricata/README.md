# INSTALACIÓN Y CONFIGURACIÓN DE UN NIDS/NIPS CON SURICATA

----

En este proyecto vamos a montar un sistema con 3 máquinas con SO Linux, Ubuntu 24.04; una máquina “Atacante” con interfaz gráfica. Una máquina servidor que hará de “Víctima”, a quien le intentarán atacar, y por último, una tercera máquina servidor que hará de “Espía” con el software de Suricata. Ésta última con dicho software, hará de controladora del tráfico de red y lo almacenará. Bajo un escenario real, estas máquinas estarían bajo una arquitectura distinta, pero como se trata de una simulación, la M.Espía se comunicará por un lado con la M.Atacante a través de una interfaz de red (P1) y por otro lado con la M.Víctima por una segunda interfaz de red (P2). Es decir, crearemos dos LAN, haciendo que la máquina intermediaria (la que tendrá suricata) permita pasar el tráfico de la M.Atacante a la de la M.Víctima y viceversa, redirigiendo el tráfico con nftables.


![Captura1](./image/1.png)

----

## ESQUEMA DE MÁQUINAS Y SU CONFIGURACIÓN DE RED E INTERFACES

----

A modo claro y esquemático, tendremos 3 máquinas y estas serán sus interfaces de red y sus configuraciones del netplan:

### Máquina atacante (M.Atacante)

| Interfaz | Nombre    | Descripción                                                                                                             |
| -------- | --------- | ----------------------------------------------------------------------------------------------------------------------- |
| enp1s0   | Default   | La predeterminada la utilizaremos para darle salida a internet por si queremos instalar cualquier herramienta.          |
| enp2s0   | Wireguard | La VPN la utilizaremos para acceder con facilidad a la máquina sin hacer ninguna modificación en la red.                |
| enp3s0   | Personal1 | Esta interfaz la añadiremos como extra porque es la que, para poder hacer el ejercicio, conectará con la máquina espía. |


> Configuración del netplan:

```bash
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: true
    enp2s0: 
     dhcp4: true
    enp3s0:
      dhcp4: false
      addresses: [192.168.30.2/24]
      routes:
        - to: 192.168.50.0/24
          via: 192.168.30.1
````
----

### Máquina víctima (M.Víctima)

| Interfaz | Nombre    | Descripción                                                                                                             |
| -------- | --------- | ----------------------------------------------------------------------------------------------------------------------- |
| enp1s0   | Default   | La predeterminada la utilizaremos para darle salida a internet por si queremos instalar cualquier herramienta.          |
| enp2s0   | Wireguard | La VPN la utilizaremos para acceder con facilidad a la máquina sin hacer ninguna modificación en la red.                |
| enp3s0   | Personal2 | Esta interfaz la añadiremos como extra porque es la que, para poder hacer el ejercicio, conectará con la máquina espía. |


> Configuración del netplan:

```bash
network:
  version: 2
  ethernets:
    enp1s0: 
      dhcp4: true
    enp2s0: 
      dhcp4: true
    enp3s0:
      dhcp4: false
      addresses: [192.168.50.2/24]
      routes:
        - to: 192.168.30.0/24
          via: 192.168.50.1
```
----

### Máquina Espía (M.Espía)

| Interfaz | Nombre    | Descripción                                                                                                    |
| -------- | --------- | -------------------------------------------------------------------------------------------------------------- |
| enp1s0   | Default   | La predeterminada la utilizaremos para darle salida a internet por si queremos instalar cualquier herramienta. |
| enp2s0   | Wireguard | La VPN la utilizaremos para acceder con facilidad a la máquina sin hacer ninguna modificación en la red.       |
| enp3s0   | Personal1 | La añadiremos para tener conexión con la máquina atacante.                                                     |
| enp4s0   | Personal2 | La añadiremos para tener conexión con la máquina víctima.                                                      |


> Configuración del netplan:

```bash
network:
  version: 2
  ethernets:
    enp1s0: 
      dhcp4: true
    enp2s0: 
      dhcp4: true
    enp3s0:
      dhcp4: false      addresses: [192.168.30.1/24]
    enp4s0:
      dhcp4: false
      addresses: [192.168.50.1/24]
```
----

## REDIRECCIONAMIENTO DE PUERTOS

----

Realizado lo anterior, nos faltará completar el enrutamiento entre las máquinas para que se redireccione el tráfico por los puertos que queremos. Es decir, para que la M.Espía, que va a ser la que haga de intermediaria entre las otras dos, permita el tráfico, si no se trata de un ataque, vamos a tener que habilitar el redireccionamiento de puertos.

Y esto lo vamos a poder hacer editando el fichero **/etc/sysctl.conf** al descomentar la línea **net.ipv4.ip_forward=1**.
Una vez descomentada la línea, aplicaremos los cambios con **sudo sysctl -p**.

----

## INSTALACIÓN Y ACTIVACIÓN DE NFTABLES

----

Para instalar nftables podemos usar el comando **sudo apt install nftables y**, aunque como estamos usando la versión de Ubuntu 24.04 seguramente venga instalado de serie. 
No obstante, debemos revisar que esté activado (que seguramente no) con **systemctl status nftables**.
Y si no está activado, deberemos habilitarlo con **sudo systemctl enable nftables**

```bash
isard@espiasuricata:~$ systemctl status nftables
○ nftables.service - nftables
 	Loaded: loaded (/usr/lib/systemd/system/nftables.service; disabled; preset: enabled)
 	Active: inactive (dead)
   	Docs: man:nft(8)
         	http://wiki.nftables.org
isard@espiasuricata:~$ systemctl enable nftables
Created symlink /etc/systemd/system/sysinit.target.wants/nftables.service → /usr/lib/systemd/system/nftables.service.
isard@espiasuricata:~$ sudo systemctl status nftables
[sudo] password for isard:
● nftables.service - nftables
 	Loaded: loaded (/usr/lib/systemd/system/nftables.service; enabled; preset:>
 	Active: active (exited) since Wed 2025-12-10 23:27:18 UTC; 27min ago
   	Docs: man:nft(8)
         	http://wiki.nftables.org
	Process: 346 ExecStart=/usr/sbin/nft -f /etc/nftables.conf (code=exited, st>
   Main PID: 346 (code=exited, status=0/SUCCESS)
    	CPU: 24ms
Dec 10 23:27:18 espiasuricata systemd[1]: Finished nftables.service - nftables.
lines 1-10/10 (END)
```

Ahora que tenemos nftables activado en nuestra M.Espía, vamos a establecer las reglas básicas de filtrado, editando en el archivo de configuración principal: **/etc/nftables.conf**
Así aparece por defecto el archivo:

```bash
#!/usr/sbin/nft -f
flush ruleset
table inet filter {
    	chain input {
            	type filter hook input priority filter;
    	}
    	chain forward {
            	type filter hook forward priority filter;
    	}
    	chain output {
            	type filter hook output priority filter;
    	}
}
```
Y así es como debe de quedar:

```bash
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
	chain input {
    	type filter hook input priority 0;
    	policy accept;
	}

	chain forward {
    	type filter hook forward priority 0;
    	policy accept;
	}

	chain output {
    	type filter hook output priority 0;
    	policy accept;
	}
}
```

Estas reglas lo que van a hacer es permitir que el tráfico circule sin restricciones, de modo que el enrutamiento que ya existía gracias al netplan y al ip_forward, siga funcionando. 

> [!NOTE]
> A partir de aquí se pueden añadir más reglas, más o menos restrictivas u otras redirecciones en caso de que sean necesarias.

Ahora aplicamos estos dos comandos para aplicar las nuevas reglas, y si todo está bien, debería de aparecernos por terminal nuestra tabla.

```bash
isard@espiasuricata:~$ sudo nft -f /etc/nftables.conf
sudo nft list ruleset
table inet filter {
    chain input {
   	 type filter hook input priority filter; policy accept;
    }

    chain forward {
   	 type filter hook forward priority filter; policy accept;
    }

    chain output {
   	 type filter hook output priority filter; policy accept;
    }
}
```

Y para comprobar que todo está bien, vamos a volver a comprobar que los pings entre las máquinas atacante y víctima siguen pudiéndose efectuar.

- Desde la M.Atacante a la M.Víctima:

```bash
isard@atacante:~$ ping 192.168.50.2
PING 192.168.50.2 (192.168.50.2) 56(84) bytes of data.
64 bytes from 192.168.50.2: icmp_seq=1 ttl=63 time=6.21 ms
64 bytes from 192.168.50.2: icmp_seq=2 ttl=63 time=0.771 ms
^C
--- 192.168.50.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.771/3.492/6.213/2.721 ms
isard@atacante:~$
```

- Desde la M.Víctima a la M.Atacante:

```bash
isard@vctima:~$ ping 192.168.30.2
PING 192.168.30.2 (192.168.30.2) 56(84) bytes of data.
64 bytes from 192.168.30.2: icmp_seq=1 ttl=63 time=1.35 ms
64 bytes from 192.168.30.2: icmp_seq=2 ttl=63 time=0.848 ms
^C
--- 192.168.30.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.848/1.098/1.348/0.250 ms
isard@vctima:~$
```
----

## INSTALACIÓN DE SURICATA EN M.ESPÍA

----

Por ahora no vamos a aplicar ninguna otra regla en nftables. Ahora el siguiente paso será instalar en nuestra M.Espía el software de Suricata.
Como siempre, lo primero será actualizar nuestros repositorios:

```bash
isard@espiasuricata:~$ sudo apt update
[sudo] password for isard:
Hit:1 http://de.archive.ubuntu.com/ubuntu noble InRelease
Hit:2 http://de.archive.ubuntu.com/ubuntu noble-updates InRelease
Hit:3 http://de.archive.ubuntu.com/ubuntu noble-backports InRelease
Hit:4 http://security.ubuntu.com/ubuntu noble-security InRelease
Reading package lists... Done
```

Una vez actualizados, vamos a proceder a instalar el software con:

```bash
isard@espiasuricata:~$ sudo apt install suricata suricata-update
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following package was automatically installed and is no longer required:
  python3-debian
Use 'sudo apt autoremove' to remove it.
…
```

A continuación nos iremos al archivo de configuración principal para añadir las redes de ambas máquinas en la línea que pone **vars**.

```bash
isard@espiasuricata:~$ sudo nano /etc/suricata/suricata.yaml
vars:
  # more specific is better for alert accuracy and performance
  address-groups:
	HOME_NET: "[192.168.30.0/24,192.168.50.0/24]"
  #HOME_NET: "[192.168.0.0/16]"
	#HOME_NET: "[10.0.0.0/8]"
	#HOME_NET: "[172.16.0.0/12]"
	#HOME_NET: "any"
```
De esta manera estamos definiendo las redes que vamos a vigilar.

Acabado lo anterior, también en el mismo archivo vamos a activar los logs con formato eve json para luego usar con el software de Loki y Grafana. Por ello, buscaremos la línea **outputs:** y dejaremos las líneas predeterminadas de la siguiente manera:

```bash
outputs:
  # a line based alerts log similar to Snort's fast.log
  - fast:
  	enabled: yes
  	filename: fast.log
  	append: yes
  	#filetype: regular # 'regular', 'unix_stream' or 'unix_dgram'

  # Extensible Event Format (nicknamed EVE) event log in JSON format
  - eve-log:
  	enabled: yes
  	filetype: regular #regular|syslog|unix_dgram|unix_stream|redis
  	filename: /var/log/suricata/eve.json
```

Guardamos y pasamos a comprobar que no hay errores de sintaxis antes de arrancar el servicio.

```bash
isard@espiasuricata:~$ sudo suricata -T -c /etc/suricata/suricata.yaml
[sudo] password for isard:
Error: conf-yaml-loader: Failed to parse configuration file at line 19: did not find expected key
```

> [!WARNING]
> En efecto, teníamos un error de sintaxis porque nos habíamos dejado el cierre en la línea 19, colocando **”**.

Corregimos y volvemos a aplicar el comando de revisión de sintaxis:

```bash
isard@espiasuricata:~$ sudo nano /etc/suricata/suricata.yaml
isard@espiasuricata:~$ sudo suricata -T -c /etc/suricata/suricata.yaml
Info: conf-yaml-loader: Configuration node 'HOME_NET' redefined.
Info: conf-yaml-loader: Configuration node 'cluster-id' redefined.
Info: conf-yaml-loader: Configuration node 'cluster-type' redefined.
Info: conf-yaml-loader: Configuration node 'defrag' redefined.
i: suricata: This is Suricata version 7.0.3 RELEASE running in SYSTEM mode
W: detect: No rule files match the pattern /var/lib/suricata/rules/suricata.rules
isard@espiasuricata:~$
```

Si todo está correcto pasaremos a reiniciar el servicio y a verificar su estado:

```bash
isard@espiasuricata:~$ sudo systemctl restart suricata
```
```bash
isard@espiasuricata:~$ sudo systemctl status suricata
[sudo] password for isard:
● suricata.service - Suricata IDS/IDP daemon
 	Loaded: loaded (/usr/lib/systemd/system/suricata.service; enabled; preset:>
 	Active: active (running) since Sun 2025-12-14 21:06:39 UTC; 45s ago
…
```

----

## CONFIGURACIÓN DE SURICATA CON NFTABLES

----

Una vez el servicio ya está en funcionamiento, pasamos a la parte en que vamos a hacer del software un IPS real, que establecerá las normas que nosotros le indicaremos y que le transmitirá a **nftables** para que ejecute el bloqueo. Es por eso que vamos a integrar Suricata con nftables para que todo el tráfico de la M.Atacante hacia la M.Víctima pase por Suricata y que cuando éste detecte un ataque, el paquete sea frenado por nftables. 

Aquí es donde entrará en juego **NFQUEUE**, que es un mecanismo que se utiliza con nftables y que permite delegar la decisión sobre qué hacer o no hacer con determinados paquetes.
Lo primero que tendremos que hacer es configurar Suricata para dejarlo en **modo inline** desde el archivo de configuración **/etc/suricata/suricata.yaml** añadiendo después de las líneas pertinentes lo siguiente:

```bash
isard@espiasuricata:~$ sudo nano /etc/suricata/suricata.yaml
engine:
  mode: ips
```

> [!NOTE]
> Aunque durante los tests el software indica “IDS mode by default”, al operar con NFQUEUE funciona en modo IPS, ya que los paquetes son aceptados o bloqueados en línea antes de continuar el flujo.

Y más abajo en las isguientes líneas...

```bash
nfqueue:
  mode: accept
  repeat-mark: 1
  route-queue: yes
```

De esta forma nfqueue recibirá los paquetes desde nftables.
Así que el siguiente paso será el de configurar nftables añadiendo lo siguiente en el archivo, para enviar el tráfico a Suricata:

```bash
isard@espiasuricata:~$ sudo nano /etc/nftables.conf
#!/usr/sbin/nft -f
flush ruleset
table inet filter {
	chain input {
    	type filter hook input priority 0;
    	policy accept;
	}

	chain forward {
    	type filter hook forward priority 0;
    	policy accept;

    	# M.Atacante ----> M.Víctima
    	ip saddr 192.168.30.0/24 ip daddr 192.168.50.0/24 counter queue num 0

    	# M.Víctima ----> M.Atacante
    	ip saddr 192.168.50.0/24 ip daddr 192.168.30.0/24 counter queue num 0
	}
}
```

Esto envía todo el tráfico entre ambas máquinas a la cola 0 (prioritaria) del kernel, donde Suricata lo examinará antes de decidir si se podrá enviar o no.
Debemos aplicar las reglas que hemos añadido en el archvio con: 

```bash
isard@espiasuricata:~$ sudo nft -f /etc/nftables.conf
sudo nft list ruleset
table inet filter {
    chain input {
   	 type filter hook input priority filter; policy accept;
    }
    chain forward {
   	 type filter hook forward priority filter; policy accept;
   	  ip saddr 192.168.30.0/24 ip daddr 192.168.50.0/24 counter packets 0 bytes 0 queue to 0
   	 ip saddr 192.168.50.0/24 ip daddr 192.168.30.0/24 counter packets 0 bytes 0 queue to 0
    }
    chain output {
   	 type filter hook output priority filter; policy accept;
    }
}
```

De nuevo, volveremos a editar el archivo **/etc/default/suricata** para cambiar el **RUN=no** por **RUN=yes**.

```bash
# Default config for Suricata

# set to yes to start the server in the init.d script
RUN=yes
# Configuration file to load
SURCONF=/etc/suricata/suricata.yaml
# Listen mode: pcap, nfqueue or af-packet
# depending on this value, only one of the two following options
# will be used (af-packet uses neither).
# Please note that IPS mode is only available when using nfqueue
LISTENMODE=nfqueue
# Interface to listen on (for pcap mode)
IFACE=eth0
# Queue number to listen on (for nfqueue mode)
NFQUEUE=0
```

Acto seguido deberemos editar el siguiente archivo para añadir un par de líneas:

```bash
isard@espiasuricata:~$ sudo systemctl edit suricata
[Service]
ExecStart=
ExecStart=/usr/bin/suricata -D -q 0 -c /etc/suricata/suricata.yaml --pidfile /run/suricata.pid
```

Esto fuerza a Suricata a escuchar la cola de NFQUEUE 0, lo que es necesario para hacer funcionar el software en modo IPS tal y como le hemos indicado antes.
Al acabar reiniciamos y comprobamos estado:

```bash
isard@espiasuricata:~$ sudo systemctl daemon-reload
sudo systemctl restart suricata
sudo systemctl status suricata
● suricata.service - Suricata IDS/IDP daemon
 	Loaded: loaded (/usr/lib/systemd/system/suricata.service; enabled; preset:>
	Drop-In: /etc/systemd/system/suricata.service.d
         	└─override.conf
 	Active: active (running) since Mon 2026-01-12 23:02:40 UTC; 38ms ago
   	Docs: man:suricata(8)
         	man:suricatasc(8)
…
```

Y tan solo para comprobar que está funcionando, vamos a inventarnos una regla por ejemplo de bloqueo, que al detectar cualquier ping, lo bloquee y genere un nuevo registro en los logs.
Para ello vamos a añadir un archivo en el directorio de configuración de normas **/etc/suricata/rules** con el nombre de **local.rules**. Y será aquí donde estableceremos nuestra normas.

```bash
isard@espiasuricata:~$ sudo nano /etc/suricata/rules/local.rules
Y vamos a añadirle la norma de prueba:
drop icmp any any -> any any (msg:"ICMP BLOQUEADO POR SURICATA"; sid:1000001; rev:1;)
```

Una vez creada la norma, en el directorio que hemos creado, deberemos volver al archivo de configuración **/etc/suricata/suricata.yaml** donde antes encontrábamos esto:

```bash
## Configure Suricata to load Suricata-Update managed rules.
##
default-rule-path: /var/lib/suricata/rules
rule-files:
  - suricata.rules
```

Antes apuntaba al directorio con las reglas oficiales de Suricata, pero por simplificación en nuestro entorno de pruebas, vamos a poner únicamente la dirección que apunta a nuestro directorio con la regla añadida:

```bash
## Configure Suricata to load Suricata-Update managed rules.
##
default-rule-path: /etc/suricata/rules
rule-files:
  - local.rules
```

Ahora validamos con:

```bash
isard@espiasuricata:~$ sudo suricata -T -c /etc/suricata/suricata.yaml -v
[sudo] password for isard:
Info: conf-yaml-loader: Configuration node 'HOME_NET' redefined.
Info: conf-yaml-loader: Configuration node 'cluster-id' redefined.
Info: conf-yaml-loader: Configuration node 'cluster-type' redefined.
Info: conf-yaml-loader: Configuration node 'defrag' redefined.
Notice: suricata: This is Suricata version 7.0.3 RELEASE running in SYSTEM mode
Info: cpu: CPUs/cores online: 3
Info: suricata: Running suricata under test mode
Info: suricata: Setting engine mode to IDS mode by default
Info: exception-policy: master exception-policy set to: auto
Info: logopenfile: fast output device (regular) initialized: fast.log
Info: logopenfile: eve-log output device (regular) initialized: /var/log/suricata/eve.json
Info: logopenfile: stats output device (regular) initialized: stats.log
Info: detect: 1 rule files processed. 1 rules successfully loaded, 0 rules failed, 0
Info: threshold-config: Threshold config parsed: 0 rule(s) found
Info: detect: 1 signatures processed. 1 are IP-only rules, 0 are inspecting packet payload, 0 inspect application layer, 0 are decoder event only
Notice: suricata: Configuration provided was successfully loaded. Exiting.
```

Reiniciaremos el servicio de nuevo y verificaremos su estado:

```bash
isard@espiasuricata:~$ sudo systemctl restart suricata
sudo systemctl status suricata
● suricata.service - Suricata IDS/IDP daemon
 	Loaded: loaded (/usr/lib/systemd/system/suricata.service; enabled; preset: enabled)
 	Active: active (running) since Sun 2025-12-14 22:45:16 UTC; 27ms ago
   	Docs: man:suricata(8)
…
```

## COMPROBACIÓN DE FUNCIONAMIENTO DE SURIATA

----

Y ahora sí que sí, probaremos si la regla se cumple haciendo un ping desde la M.Atacante a la M.Víctima.

```bash
isard@atacante:~$ ping 192.168.50.2
PING 192.168.50.2 (192.168.50.2) 56(84) bytes of data.
^C
--- 192.168.50.2 ping statistics ---
73 packets transmitted, 0 received, 100% packet loss, time 73704ms
```

Y en efecto, el ping que antes funcionaba, ahora no funciona porque Suricata lo ha bloqueado con la nueva norma de prueba que le hemos añadido en el fichero anterior. 
Es más, como hemos añadido que deje un registros en los logs, podemos comprobarlo de la siguente manera:

```bash
isard@espiasuricata:~$ sudo tail -f /var/log/suricata/fast.log
==> /var/log/suricata/fast.log <==
01/12/2026-19:14:15.256563  [Drop] [**] [1:1000001:1] ICMP BLOQUEADO POR SURICATA [**] [Classification: (null)] [Priority: 3] {ICMP} 192.168.30.2:8 -> 192.168.50.2:0
01/12/2026-22:06:18.758449  [Drop] [**] [1:1000001:1] ICMP BLOQUEADO POR SURICATA [**] [Classification: (null)] [Priority: 3] {ICMP} 192.168.30.2:8 -> 192.168.50.2:0
01/12/2026-23:06:46.305971  [Drop] [**] [1:1000001:1] ICMP BLOQUEADO POR SURICATA [**] [Classification: (null)] [Priority: 3] {ICMP} 192.168.30.2:8 -> 192.168.50.2:0
==> /var/log/suricata/fast.log.1 <==
==> /var/log/suricata/fast.log.1-2026011111.backup <==
```

----

## MÁS REGLAS REFLEJADAS EN LOCAL.RULES

----

A continuación vamos a añadir algunas reglas más para seguir haciendo pruebas en nuestro escenario de simulación:

- Regla que bloquea intentos de escaneo TCP:
  
```bash
drop tcp any any -> any any (flags:S; msg:"TCP SYN BLOQUEADO (posible escaneo)"; sid:1000002; rev:1;)
```

- Regla que bloquea los intentos de acceso por SSH:

```bash
drop tcp any any -> any 22 (msg:"INTENTO SSH BLOQUEADO"; sid:1000003; rev:1;)
```

- Regla que detecta el tráfico HTTP, pero no lo bloquea:

```bash
alert http any any -> any any (msg:"TRÁFICO HTTP DETECTADO"; sid:1000004; rev:1;)
```

De tal forma que añadiéndolas en el archivo **local.rules** y dejándolo limpio y ordenado quedaría así:

```bash
isard@espiasuricata:~$ sudo nano /etc/suricata/rules/local.rules
[sudo] password for isard:
# Bloqueo ICMP
drop icmp any any -> any any (msg:"ICMP BLOQUEADO POR SURICATA"; sid:1000001; rev:1;)

# Bloqueo escaneo TCP SYN
drop tcp any any -> any any (flags:S; msg:"TCP SYN BLOQUEADO (posible escaneo)"; sid:1000002; rev:1;)

# Bloqueo conexiones SSH
drop tcp any any -> any 22 (msg:"INTENTO SSH BLOQUEADO"; sid:1000003; rev:1;)

# Detección HTTP (solo alerta, no bloqueo)
alert http any any -> any any (msg:"TRÁFICO HTTP DETECTADO"; sid:1000004; rev:1;)
```

Modificado el documento, lo guardamos y aplicamos los siguientes comandos:

```bash
isard@espiasuricata:~$ sudo suricata -T -c /etc/suricata/suricata.yaml
sudo systemctl restart suricata
Info: conf-yaml-loader: Configuration node 'HOME_NET' redefined.
Info: conf-yaml-loader: Configuration node 'cluster-id' redefined.
Info: conf-yaml-loader: Configuration node 'cluster-type' redefined.
Info: conf-yaml-loader: Configuration node 'defrag' redefined.
i: suricata: This is Suricata version 7.0.3 RELEASE running in SYSTEM mode
i: suricata: Configuration provided was successfully loaded. Exiting.
```

----

## COMPROBACIONES DE FUNCIONAMIENTO DE REGLAS

----

Si ahora probamos regla por regla para verificar que actúan correctamente y se cumplen:

- Bloqueo escaneo TCP SYN:
  
1. Desde la M.atacante:

```bash
isard@atacante:~$ sudo nmap -sS 192.168.50.2
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-24 14:01 CET
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
Nmap done: 1 IP address (0 hosts up) scanned in 3.18 seconds
```

2. Desde la M.Espía revisando los logs:

```bash
isard@espiasuricata:~$ sudo tail -f /var/log/suricata/fast.log
01/24/2026-13:01:34.036634  [Drop] [**] [1:1000002:1] TCP SYN BLOQUEADO (posible escaneo) [**] [Classification: (null)] [Priority: 3] {TCP} 192.168.30.2:60737 -> 192.168.50.2:443
01/24/2026-13:01:36.038344  [Drop] [**] [1:1000002:1] TCP SYN BLOQUEADO (posible escaneo) [**] [Classification: (null)] [Priority: 3] {TCP} 192.168.30.2:60739 -> 192.168.50.2:443
```

- Bloqueo conexiones SSH:

1. Desde la M.atacante:
   
```bash
isard@atacante:~$ ssh atacante@192.168.50.2
...
```

Y la conexión se queda colgada porque efectivamente se produce el bloqueo.

2. Desde la M.Espía revisando los logs:

```bash
isard@espiasuricata:~$ sudo tail -f /var/log/suricata/fast.log
01/24/2026-13:08:33.587140  [Drop] [**] [1:1000003:1] INTENTO SSH BLOQUEADO [**] [Classification: (null)] [Priority: 3] {TCP} 192.168.30.2:49074 -> 192.168.50.2:22
```

- Detección HTTP (solo alerta, no bloqueo):

1. Desde la M.atacante:

```bash
isard@atacante:~$ curl http://192.168.50.2
...
```

2. Desde la M.Espía revisando los logs:

```bash
isard@espiasuricata:~$ sudo tail -f /var/log/suricata/fast.log
01/24/2026-13:10:54.871938  [Drop] [**] [1:1000002:1] TCP SYN BLOQUEADO (posible escaneo) [**] [Classification: (null)] [Priority: 3] {TCP} 192.168.30.2:47458 -> 192.168.50.2:80
01/24/2026-13:11:03.063885  [Drop] [**] [1:1000002:1] TCP SYN BLOQUEADO (posible escaneo) [**] [Classification: (null)] [Priority: 3] {TCP} 192.168.30.2:47458 -> 192.168.50.2:80
01/24/2026-13:11:19.452978  [Drop] [**] [1:1000002:1] TCP SYN BLOQUEADO (posible escaneo) [**] [Classification: (null)] [Priority: 3] {TCP} 192.168.30.2:47458 -> 192.168.50.2:80
01/24/2026-13:11:51.706905  [Drop] [**] [1:1000002:1] TCP SYN BLOQUEADO (posible escaneo) [**] [Classification: (null)] [Priority: 3] {TCP} 192.168.30.2:47458 -> 192.168.50.2:80
```

Pero claro, este sistema de lectura no es que sea muy visual que digamos… Por ello vamos a establecer un sistema que lea esos datos, los almacene y se muestren a través de un gráfico. Y esto será posible gracias al software complementario entre sí de Alloy, Loki y Grafana. Alloy es quien se encargará de leer la recogida de datos de eve.json y enviárselos a Loki, una base de datos que guardará esos datos de manera ordenada. Y por último, Loki le enviará los datos a Grafana, que será quien los muestre de una manera más presentable al ojo de una persona, a través de gráficos que podremos personalizar a nuestro gusto.
Por lo que el siguiente paso será el de hacer la instalación de estos softwares y hacer que funcionen entre sí.
Además, como se trata de una práctica de clase en un entorno de pruebas que no va a tener mucho tráfico, en lugar de un entorno laboral, más serio y mucho mayor en cuanto a volumen de tráfico y trabajo, vamos a instalarlo todo en la M.Suricata en lugar de repartir Loki y Grafana en otras máquinas distintas.

----

# INSTALACIÓN Y CONFIGURACIÓN DE SOFTWARE COMPLEMENTARIO A SURICATA

----

## GRAFANA

----

Grafana es el software que utilizaremos para mostrar los datos a través de gráficos, de manera que podremos observar los datos de una manera mucho más agradable y visual que no a través de líneas en una terminal.

Como los repositorios de Grafana no vienen por defecto con Ubuntu, lo primero que haremos será añadir los repositorios oficiales en la M.Espía, para que se descargue con la versión más actual.

```bash
isard@espiasuricata:~$ sudo apt update
sudo apt install -y apt-transport-https software-properties-common wget
[sudo] password for isard:
Get:1 http://security.ubuntu.com/ubuntu noble-security InRelease [126 kB]
Hit:2 http://de.archive.ubuntu.com/ubuntu noble InRelease
Get:3 http://de.archive.ubuntu.com/ubuntu noble-updates InRelease [126 kB]
Get:4 http://de.archive.ubuntu.com/ubuntu noble-backports InRelease [126 kB]
Get:5 http://security.ubuntu.com/ubuntu noble-security/main amd64 Components [21
…
```
```bash
isard@espiasuricata:~$ wget -q -O - https://apt.grafana.com/gpg.key | sudo tee /etc/apt/keyrings/grafana.asc
-----BEGIN PGP PUBLIC KEY BLOCK-----

mQGNBGTnhmkBDADUE+SzjRRyitIm1siGxiHlIlnn6KO4C4GfEuV+PNzqxvwYO+1r
mcKlGDU0ugo8ohXruAOC77Kwc4keVGNU89BeHvrYbIftz/yxEneuPsCbGnbDMIyC
isard@espiasuricata:~$ echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" \ | sudo tee /etc/apt/sources.list.d/grafana.list
deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main  
```
```bash
isard@espiasuricata:~$ sudo apt update
Get:1 https://apt.grafana.com stable InRelease [7661 B]
Hit:2 http://de.archive.ubuntu.com/ubuntu noble InRelease                 	 
Hit:3 http://security.ubuntu.com/ubuntu noble-security InRelease
Hit:4 http://de.archive.ubuntu.com/ubuntu noble-updates InRelease
```

Ahora que ya tenemos los repositorios pertinentes, instalaremos el servicio de Grafana.

```bash
isard@espiasuricata:~$ sudo apt install -y grafana
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
```

Arrancamos el servicio y lo habilitamos para que se inicie automáticamente con cada inicio de máquina y no tengamos que iniciarlo manualmente.

```bash
isard@espiasuricata:~$ sudo systemctl enable grafana-server
sudo systemctl start grafana-server
Synchronizing state of grafana-server.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable grafana-server
Created symlink /etc/systemd/system/multi-user.target.wants/grafana-server.service → /usr/lib/systemd/system/grafana-server.service.
```

Comprobamos que esté corriendo:

```bash
isard@espiasuricata:~$ sudo systemctl status grafana-server
● grafana-server.service - Grafana instance
 	Loaded: loaded (/usr/lib/systemd/system/grafana-server.service; enabled; p>
 	Active: active (running) since Sat 2026-01-24 16:50:51 UTC; 21s ago
   	Docs: http://docs.grafana.org
   Main PID: 2014 (grafana)
```

Ahora desde la M.Atacante, que es de donde veremos los resultados, porque es la máquina que tiene entorno gráfico y con la que nos conectamos mediante SSH a las demás... Accederemos al navegador con la IP de la M.Espía (que tiene Suricata) y el puerto 3000 **http://192.168.50.1:3000** que es el que usa por defecto Grafana, para entrar a Grafana porque funciona como un servidor web.

![Captura1](./image/2.png)

Por defecto las credenciales del usuario y la contraseña son **admin**, pero nada más acceder nos hará modificarla (en nuestro caso por “joel”).

![Captura1](./image/3.png)

![Captura1](./image/4.png)

----

## LOKI

----

Ahora que ya tenemos el software de Grafana, vamos a descargar Loki, que será el encargado de almacenar los logs que visualizaremos en Grafana. Es decir, Loki actuará como base de datos para almacenar los logs y Grafana será nuestro reproductor de gráficos para ver esos logs de manera gráfica.


Dejaremos el archivo .zip en el directorio temporal **/tmp** porque una vez lo descomprimamos no lo necesitaremos.

```bash
isard@espiasuricata:~$ cd /tmp
wget https://github.com/grafana/loki/releases/latest/download/loki-linux-amd64.zip
--2026-01-24 16:56:46--  https://github.com/grafana/loki/releases/latest/download/loki-linux-amd64.zip
Resolving github.com (github.com)... 140.82.121.3
Connecting to github.com (github.com)|140.82.121.3|:443... connected.
HTTP request sent, awaiting response... 302 Found
```

Lo descomprimiremos e instalaremos...

```bash
isard@espiasuricata:/tmp$ sudo apt install -y unzip
unzip loki-linux-amd64.zip
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following package was automatically installed and is no longer required:
```

Movemos el paquete a un directorio de usuario y le damos permiso de ejecución para poder iniciarlo.

```bash
isard@espiasuricata:/tmp$ sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki
isard@espiasuricata:/tmp$
```

Ahora creamos dos directorios necesarios.

- El de /etc... para almacenar los archivos de configuración de Loki, siguiendo la estructura de Linux con sus archivos de configuración.
- El de /var/lib... para datos internos del servicio, como los datos generados por Loki, etc.

```bash
isard@espiasuricata:~$ sudo mkdir -p /etc/loki
sudo mkdir -p /var/lib/loki
```

Y hacemos la configuración básica del archivo de configuración principal de Loki. Con dicha configuración le vamos a decir que almacene los logs en el sistema de ficheros **/var/lib/loki** y que lo replique a una sola máquina, que es con la que estamos trabajando.

```bash
sudo nano /etc/loki/loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  allow_structured_metadata: true
```

Creamos el servicio **systemd** para Loki, para que también se inicie de manera automática con el inicio de la máquina.

```bash
sudo nano /etc/systemd/system/loki.service
[Unit]
Description=Loki Log Aggregation System
After=network.target

[Service]
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yaml
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

Ahora arrancamos el servicio y como siempre comprobamos su estado.

```bash
isard@espiasuricata:~$ sudo systemctl daemon-reload
sudo systemctl enable loki
sudo systemctl start loki
Created symlink /etc/systemd/system/multi-user.target.wants/loki.service → /etc/systemd/system/loki.service.
```
```bash
isard@espiasuricata:~$ sudo systemctl status loki
● loki.service - Loki Log Aggregation System
 	Loaded: loaded (/etc/systemd/system/loki.service; enabled; p>
 	Active: active (running) since Thu 2026-01-29 18:18:59 UTC; >
   Main PID: 7445 (loki)
  	Tasks: 9 (limit: 8225)
 	Memory: 123.3M (peak: 124.1M)
    	CPU: 626ms
 	CGroup: /system.slice/loki.service
         	└─7445 /usr/local/bin/loki -config.file=/etc/loki/lo>

Jan 29 18:19:01 espiasuricata loki[7445]: level=info ts=2026-01-2>
Jan 29 18:19:01 espiasuricata loki[7445]: level=info ts=2026-01-2>
```

Por último vamos a hacer una comprobación con su endpoint para conocer si está preparado para recibir datos. /ready nos permite comprobar si Loki está preparado para recibir datos. 

```bash
isard@espiasuricata:~$ curl http://192.168.50.1:3100/ready
Ingester not ready: waiting for 15s after being ready
isard@espiasuricata:~$ curl http://192.168.50.1:3100/ready
ready
```

----

## CONEXIÓN ENTRE GRAFANA Y LOKI

----

Ahora vamos a decirle a Grafana dónde está Loki y dónde debe consultar esos logs, ya que Grafana actuará como cliente y se conectará a Loki en el puerto 3100. Entraremos a la página principal de Grafana y en el panel izquierdo nos dirigiremos a **Connections** - **Data sources** - **Add data source**.

![Captura1](./image/5.png)

En el buscador buscaremos por **Loki** y lo añadiremos.

![Captura1](./image/6.png)

Se añade la IP de la M.Suricata.

![Captura1](./image/7.png)

Y ahora guardamos y salimos.

![Captura1](./image/8.png)

----

## ALLOY 

----

A continuación vamos a proceder con la instalación de Alloy en nuestra M.Espía. Alloy será nuestro lector, el encargado de leer el archivo de logs de Suricata en formato **eve.json** y enviarselo a Loki. Es decir, sin Alloy, Loki no recibiría ningún dato.

Estos son los comandos para realizarla de manera exitosa (gracias Chus).

```bash
isard@espiasuricata:~$ sudo apt install gpg
[sudo] password for isard:
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
```
```bash
isard@espiasuricata:~$ wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /usr/share/keyrings/grafana.gpg > /dev/null
```
```bash
isard@espiasuricata:~$ echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
deb [signed-by=/usr/share/keyrings/grafana.gpg] https://apt.grafana.com stable main
```
```bash
isard@espiasuricata:~$ sudo apt update
sudo apt install alloy
Get:1 https://apt.grafana.com stable InRelease [7660 B]
Hit:2 http://de.archive.ubuntu.com/ubuntu noble InRelease                  	 
Get:3 http://de.archive.ubuntu.com/ubuntu noble-updates InRelease [126 kB] 	 
Get:4 http://de.archive.ubuntu.com/ubuntu noble-backports InRelease [126 kB]
```

Ahora está instalado, pero no funcional, porque primero debemos configurarlo para que envíe los logs de Suricata a Loki. Es por ello que deberemos ir, como es de costumbre, al archivo de configuración principal del servicio **/etc/alloy/alloy.conf**.

Una vez dentro deberemos eliminar lo que hay por defecto y dejar la configuración siguiente: 

```bash
// Sample config for Alloy.
//
// For a full configuration reference, see https://grafana.com/docs/alloy
logging {
  level = "info"
}

loki.write "local" {
  endpoint {
	url = "http://127.0.0.1:3100/loki/api/v1/push"
  }
}

loki.source.file "suricata_logs" {
  targets = [
	{
  	__path__ = "/var/log/suricata/eve.json",
  	job  	= "suricata",
	},
  ]

  forward_to = [loki.write.local.receiver]
}
```

Vamos a explicar un poco como funciona…
- En la línea **level =** hay tres niveles: info (normal), debug (con muchos detalles) y warn (solo envía advertencias).
- Cuando vamos al **endpoint** y ponemos la URL del destino, es para indicarle a dónde queremos que envíe los logs. En este caso ponemos **localhost** porque Loki, que es a quien queremos enviarle, está en la misma máquina que Alloy.
- En la línea que pone **__path__** es donde le estamos indicando el archivo que queremos que vigile en todo momento, para que detecte los nuevos cambios al producirse ataques.
- En la que pone **job = suricata** es donde se indexan los logs y por el nombre que podremos filtrar en Grafana.

Con esta configuración básica nos debería ser suficiente para llevar a cabo la práctica en nuestro entorno de prueba.

Pero antes vamos a terminar de explicar de una manera clara y "esquemática" el camino que seguirán los logs.

1. Suricata detecta el tráfico y escribe lo que pasa en eve.json.
2. Alloy monitoriza ese archivo y envía los eventos a Loki.
3. Loki almacena los logs.
4. Grafana consulta a Loki y los representa con el tipo de gráfico que escojamos.

Guardada la configuración y entendido el funcionamiento general, iniciaremos el servicio y comprobaremos que está leyendo el archivo.

```bash
isard@espiasuricata:~$ sudo systemctl start alloy
```
```bash
isard@espiasuricata:~$ sudo systemctl status alloy
● alloy.service - Vendor-agnostic OpenTelemetry Collector distribution with programmable pipelines
 	Loaded: loaded (/usr/lib/systemd/system/alloy.service; enabled; preset: enabled)
 	Active: active (running) since Thu 2026-02-12 11:22:26 UTC; 19ms ago
sudo journalctl -u alloy -f
```

```bash
isard@espiasuricata:~$ sudo ls -ld /var/log/suricata
sudo ls -l /var/log/suricata/eve.json
id alloy
drwxr-xr-x 2 root root 4096 Feb 12 09:01 /var/log/suricata
-rw-r--r-- 1 root root 13058519 Feb 12 12:40 /var/log/suricata/eve.json
uid=999(alloy) gid=988(alloy) groups=988(alloy),4(adm),999(systemd-journal)
```

Como el archivo eve.json pertenece al usuario root, Alloy no tiene permisos para leerlo por defecto. Por ello utilizamos **setfacl** para dar permisos  de lectura al usuario alloy sin modificar los permisos generales del sistema.

```bash
isard@espiasuricata:~$ sudo apt install -y acl
sudo setfacl -m u:alloy:rx /var/log/suricata
sudo setfacl -m u:alloy:r /var/log/suricata/eve.json
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following package was automatically installed and is no longer required:
  python3-debian
Use 'sudo apt autoremove' to remove it.
sum by (alert_signature) (
  count_over_time({job="suricata"} | json | event_type="alert" [1m])
)
```

Esta última consulta que nos aparece con la función **sum** es la que cuenta el número de alertas generadas por Suricata a cada minuto y las agrupa por el tipo de firma (alert_signature). 

Y llegados a este punto, con toda la configuración que hemos hecho, hemos conseguido centralizar y visualizar las alertas de Suricata en tiempo real. De esta forma no solo detectamos ataques sino que lo monitorizamos y visualizamos gráficamente clasificados por el tipo de ataque que está teniendo nuestro sistema.

## DEMOSTRACIÓN GRÁFICA DE LOGS DE SURICATA EN GRAFANA

En primer lugar, desde la M.Atacante vamos a realizar una serie de ataques (aquellos para los que pusimos reglas a mitad de actividad) a la M.Víctima ahora que ya está todo montado y funcionando, para que los logs de Suricata que recoja Alloy, se los envíe a Loki y los veamos reflejados en Grafana.

Una vez repetidos los ataques, vamos a ir al servicio web de Grafana desde la M.Atacante (recordemos que es la única que tiene entorno gráfico). Una vez en el menú principal de la página, iremos a la lista de la izquierda, donde pone **Explore**. Iremos a **Queries** y en los espacios a rellenar, pondremos **job** y **suricata**.

Y justo debajo, **{job="suricata"} |= ''**.

Podemos filtrar en la cabecera, a partir de cuanto tiempo queremos ver los ataques (en nuestro caso "Last 1 hour".

![Captura1](./image/14.png)

Y por último le damos a **Run query**.

Nos aparecerá este gráfico de los logs por hora en los que se han producido los ataques.

![Captura1](./image/9.png)

Si bajamos un poco más, podremos ver lo mismo en formato código.

![Captura1](./image/10.png)

Ahora si vamos de nuevo a la cabecera, y le damos a **Add** y **Add to dashboard**, podremos añadirlo a un gráfico. 

![Captura1](./image/13.png)

Por defecto, nos viene en una tabla en formato código, pero si vamos a **Edit**.

![Captura1](./image/15.png)

Podemos escoger entre distintos tipos de gráficos, en función de lo que busquemos o lo que nos resulte más fácil para visualizar.

![Captura1](./image/16.png)

Por ejemplo, este formato no se adecua a una fácil visualización de los logs separados por el tipo de ataque, a pesar de la leyenda de abajo.

![Captura1](./image/11.png)

Por lo que buscamos otro formato como este otro, en el que sí que podemos visualizar todo de forma más sencilla.

![Captura1](./image/12.png)

Y llegados a este punto, ya podremos decir que tenemos el sistema de monitorizaje montado y listo para funcionar, adaptar a las necesidades y mejorar con más normas.

