# Registro de Configuración CLI — Router R-LOG

Este documento registra, en orden cronológico, la secuencia de configuración ejecutada sobre el router R-LOG. Se presenta primero el estado base de la red y las configuraciones inseguras utilizadas para evidenciar las vulnerabilidades del caso, seguidas de las configuraciones de mitigación aplicadas sobre los mismos elementos hasta alcanzar el estado final seguro.

## 1. Configuración básica inicial

Se asigna el nombre del equipo, se activa la interfaz física troncal y se crean las subinterfaces correspondientes a las VLAN 10, 30 y 99, cada una con su dirección de puerta de enlace.

```
Router>enable
Router#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#hostname R-LOG
R-LOG(config)#interface g0/1
R-LOG(config-if)#no shutdown
R-LOG(config-if)#exit
R-LOG(config)#interface g0/1.10
R-LOG(config-subif)#encapsulation dot1Q 10
R-LOG(config-subif)#ip address 192.168.10.1 255.255.255.0
R-LOG(config-subif)#exit
R-LOG(config)#interface g0/1.30
R-LOG(config-subif)#encapsulation dot1Q 30
R-LOG(config-subif)#ip address 192.168.30.1 255.255.255.0
R-LOG(config-subif)#exit
R-LOG(config)#interface g0/1.99
R-LOG(config-subif)#encapsulation dot1Q 99
R-LOG(config-subif)#ip address 192.168.99.1 255.255.255.0
R-LOG(config-subif)#exit
%LINK-5-CHANGED: Interface GigabitEthernet0/1, changed state to up
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to up
%LINK-3-UPDOWN: Interface GigabitEthernet0/1.10, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1.10, changed state to up
%LINK-3-UPDOWN: Interface GigabitEthernet0/1.30, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1.30, changed state to up
%LINK-3-UPDOWN: Interface GigabitEthernet0/1.99, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1.99, changed state to up
```

## 2. Configuración insegura: acceso remoto mediante Telnet (RSK-NET-01)

Con el propósito de evidenciar la vulnerabilidad asociada al uso de protocolos de administración sin cifrado, se configura una cuenta local y se habilita Telnet como método de acceso a las líneas VTY.

```
R-LOG(config)#username admin privilege 15 password cisco
R-LOG(config)#line vty 0 4
R-LOG(config-line)#transport input telnet
R-LOG(config-line)#login local
R-LOG(config-line)#exit
```

## 3. Mitigación de RSK-NET-01: habilitación de SSHv2

Se genera el par de claves RSA, se restringe el acceso remoto exclusivamente a SSH versión 2 y se reemplaza la cuenta anterior por una cuenta con contraseña cifrada.

```
R-LOG(config)#ip domain-name logistica.local
R-LOG(config)#crypto key generate rsa
The name for the keys will be: R-LOG.logistica.local
Choose the size of the key modulus in the range of 360 to 4096 for your
  General Purpose Keys. Choosing a key modulus greater than 512 may take
  a few minutes.

How many bits in the modulus [512]: 2048
% Generating 2048 bit RSA keys, keys will be non-exportable...[OK]

R-LOG(config)#ip ssh version 2
*Mar 1 0:41:41.121: %SSH-5-ENABLED: SSH 1.99 has been enabled
R-LOG(config)#username operador secret Cl4v3.Fuerte
R-LOG(config)#line vty 0 4
R-LOG(config-line)#transport input ssh
R-LOG(config-line)#login local
R-LOG(config-line)#exec-timeout 5 0
R-LOG(config-line)#exit
R-LOG(config)#
R-LOG#
%SYS-5-CONFIG_I: Configured from console by console
```

## 4. Mitigación de RSK-NET-02: protección del modo privilegiado

**Estado previo (sin mitigar):** la configuración base del equipo no incluye ninguna línea `enable secret` ni `enable password`. En ese estado, el comando `enable` desde el modo usuario no solicita ninguna credencial, permitiendo el acceso directo al modo privilegiado —y con ello a la configuración completa del equipo— sin autenticación alguna. No se requiere ningún comando para llegar a este estado: corresponde a la configuración por defecto del router.

**Mitigación aplicada:** se configura una contraseña cifrada mediante `enable secret` y se habilita el cifrado del resto de contraseñas presentes en la configuración.

```
R-LOG>enable
R-LOG#config t
Enter configuration commands, one per line.  End with CNTL/Z.
R-LOG(config)#enable secret Adm1n.2026
R-LOG(config)#service password-encryption
R-LOG(config)#end
R-LOG#
%SYS-5-CONFIG_I: Configured from console by console
```

Verificación posterior, tras reiniciar la sesión de consola:

```
R-LOG>enable
Password:
R-LOG#show running-config | include enable secret
enable secret 5 $1$mERr$jgoaTcGYDb6lVwydGwCi5.
```

## 5. Mitigación de RSK-NET-03: restricción de origen en líneas VTY

**Estado previo (sin mitigar):** las líneas VTY configuradas en la sección anterior (`transport input ssh`, `login local`) no tenían ningún filtro de origen asociado. En ese estado, cualquier equipo de la red, sin importar la VLAN en la que se encuentre, puede intentar establecer una sesión SSH contra el router y llegar hasta el prompt de autenticación. No se requiere ningún comando adicional para llegar a este estado: es el resultado directo de no aplicar ninguna lista de acceso sobre las líneas VTY.

**Mitigación aplicada:** se crea una lista de acceso estándar que autoriza únicamente el segmento de administración (VLAN 99) y se aplica sobre las líneas VTY como filtro de entrada.

```
R-LOG#config t
Enter configuration commands, one per line.  End with CNTL/Z.
R-LOG(config)#access-list 10 permit 192.168.99.0 0.0.0.255
R-LOG(config)#line vty 0 4
R-LOG(config-line)#access-class 10 in
R-LOG(config-line)#exit
R-LOG(config)#do show access-lists
Standard IP access list 10
    10 permit 192.168.99.0 0.0.0.255
```

## 6. Mitigación de RSK-NET-06: registro centralizado de eventos (Syslog) y sincronización NTP

**Estado previo (sin mitigar):** sin un destino de logging configurado, los eventos generados por el router (cambios de estado de interfaz, intentos de acceso, etc.) sólo se almacenan en el búfer local de memoria volátil del equipo, con la hora contada desde el último arranque y sin ninguna copia externa. Ante un reinicio o la manipulación del propio equipo, ese historial se pierde por completo. Este es igualmente el estado por defecto del equipo, sin comandos adicionales de por medio.

**Mitigación aplicada:** se configura el envío de eventos del router hacia el servidor SRV-SIEM, con marcas de tiempo sincronizadas mediante NTP contra el mismo servidor.

```
R-LOG(config)#service timestamps log datetime msec
R-LOG(config)#clock timezone CLT -4
R-LOG(config)#ntp server 192.168.30.10
R-LOG(config)#logging host 192.168.30.10
R-LOG(config)#logging trap informational
         ^
% Invalid input detected at '^' marker.

R-LOG(config)#logging on
```

El nivel de detalle (`trap`) no fue especificado de forma explícita, ya que la versión de IOS utilizada no admitió el parámetro `informational` mediante ese comando (se verificó con `logging trap ?` que únicamente se aceptaba la palabra clave `debugging`). Se optó por omitir la línea, dado que el nivel `informational` corresponde al valor por defecto del sistema, lo cual se confirmó en la verificación siguiente:

```
R-LOG(config)#do show logging
Trap logging: level informational, 11 message lines logged
    Logging to 192.168.30.10  (udp port 514,  audit disabled,
         authentication disabled, encryption disabled, link up),
         0 message lines logged,
         0 message lines rate-limited,
         0 message lines dropped-by-MD,
         xml disabled, sequence number disabled
         filtering disabled
```

## 7. Detección de intentos de autenticación fallidos

Dado que el router no registra por defecto los intentos de inicio de sesión, se habilita explícitamente el registro de eventos de autenticación, tanto exitosos como fallidos, sobre las líneas de acceso remoto.

```
R-LOG(config)#login on-failure log
R-LOG(config)#login on-success log
```

Verificación mediante un intento de acceso SSH con credenciales incorrectas desde PC-ADMIN:

```
*Sep 23, 23:50:40.5050: SEC_LOGIN-5-LOGIN_FAILED: Login failed [user: operador] [Source: 192.168.99.20] [localport: 22] [Reason: Login Authentication Failed] at 23:50:40 UTC Wed Sep 23 2026
*Sep 23, 23:50:41.5050: SEC_LOGIN-5-LOGIN_FAILED: Login failed [user: operador] [Source: 192.168.99.20] [localport: 22] [Reason: Login Authentication Failed] at 23:50:41 UTC Wed Sep 23 2026
*Sep 23, 23:50:42.5050: SEC_LOGIN-5-LOGIN_FAILED: Login failed [user: operador] [Source: 192.168.99.20] [localport: 22] [Reason: Login Authentication Failed] at 23:50:42 UTC Wed Sep 23 2026
```

Confirmación de que los eventos fueron enviados al servidor centralizado:

```
R-LOG(config)#do show logging
    Trap logging: level informational, 14 message lines logged
        Logging to 192.168.30.10  (udp port 514,  audit disabled,
             authentication disabled, encryption disabled, link up),
             3 message lines logged,
             0 message lines rate-limited,
             0 message lines dropped-by-MD,
             xml disabled, sequence number disabled
             filtering disabled
```

## 8. Estado final del equipo

Al término de la secuencia anterior, el router R-LOG queda con enrutamiento entre VLAN operativo, acceso administrativo remoto exclusivamente por SSHv2, modo privilegiado protegido con contraseña cifrada, acceso administrativo restringido al segmento 192.168.99.0/24, y envío de eventos de seguridad (incluyendo intentos de autenticación fallidos) hacia el servidor SIEM.
