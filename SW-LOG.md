# Registro de Configuración CLI — Switch SW-LOG

Este documento registra, en orden cronológico, la secuencia de configuración ejecutada sobre el switch SW-LOG: configuración base de VLAN y puertos, y las medidas de mitigación aplicadas posteriormente sobre el mismo equipo (control de acceso por puerto, desactivación de puertos sin uso y registro centralizado de eventos).

## 1. Configuración básica inicial

Se asigna el nombre del equipo, se crean las VLAN de la red (usuarios, monitoreo, administración y la VLAN de descarte para puertos sin uso), se configura el enlace troncal hacia el router y los puertos de acceso de cada segmento, y se habilita la interfaz de administración del switch.

```
Switch>enable
Switch#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#hostname SW-LOG
SW-LOG(config)#vlan 10
SW-LOG(config-vlan)#name USUARIOS
SW-LOG(config-vlan)#vlan 30
SW-LOG(config-vlan)#name MONITOREO
SW-LOG(config-vlan)#vlan 99
SW-LOG(config-vlan)#name ADMINISTRACION
SW-LOG(config-vlan)#vlan 999
SW-LOG(config-vlan)#name SIN_USO
SW-LOG(config-vlan)#exit
SW-LOG(config)#interface g0/1
SW-LOG(config-if)#switchport mode trunk
SW-LOG(config-if)#switchport trunk allowed vlan 10,30,99
SW-LOG(config-if)#exit
SW-LOG(config)#interface fa0/1
SW-LOG(config-if)#switchport mode access
SW-LOG(config-if)#switchport access vlan 10
SW-LOG(config-if)#interface fa0/10
SW-LOG(config-if)#switchport mode access
SW-LOG(config-if)#switchport access vlan 30
SW-LOG(config-if)#interface fa0/20
SW-LOG(config-if)#switchport mode access
SW-LOG(config-if)#switchport access vlan 99
SW-LOG(config-if)#exit
SW-LOG(config)#interface vlan 99
SW-LOG(config-if)#ip address 192.168.99.2 255.255.255.0
SW-LOG(config-if)#no shutdown
SW-LOG(config-if)#exit
SW-LOG(config)#ip default-gateway 192.168.99.1
%LINK-5-CHANGED: Interface Vlan99, changed state to up
%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan99, changed state to up
```

## 2. Mitigación de RSK-NET-04: Port-Security en el puerto Fa0/1

**Estado previo (sin mitigar):** el puerto Fa0/1, configurado únicamente como `switchport mode access` en la sección anterior, acepta sin restricción cualquier dirección MAC que se conecte físicamente a él. En ese estado, basta con desconectar el equipo autorizado (PC-OPER) y conectar cualquier otro dispositivo para obtener acceso a la VLAN 10, sin que el switch registre ni bloquee el cambio. No se requiere ningún comando adicional para llegar a este estado: es el resultado de un puerto de acceso sin control de capa 2.

**Mitigación aplicada:** se habilita el control de direcciones MAC en el puerto de acceso del operador, limitando su uso a una sola dirección aprendida de forma automática.

```
SW-LOG(config)#interface fa0/1
SW-LOG(config-if)#switchport mode access
SW-LOG(config-if)#switchport port-security
SW-LOG(config-if)#switchport port-security maximum 1
SW-LOG(config-if)#switchport port-security mac-address sticky
SW-LOG(config-if)#switchport port-security violation restrict
SW-LOG(config-if)#exit
```

Verificación inmediatamente posterior a la configuración, antes de que el puerto registrara tráfico:

```
SW-LOG(config)#do show port-security interface fa0/1
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Restrict
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 1
Total MAC Addresses        : 0
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 0
Last Source Address:Vlan   : 0000.0000.0000:0
Security Violation Count   : 0
```

Verificación posterior, luego de generar tráfico legítimo desde el equipo conectado al puerto (PC-OPER), lo que permitió al switch aprender la dirección MAC de forma dinámica (sticky):

```
SW-LOG(config)#do show port-security interface fa0/1
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Restrict
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 1
Total MAC Addresses        : 1
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 1
Last Source Address:Vlan   : 0060.3E3E.A87D:10
Security Violation Count   : 0
```

## 3. Mitigación de RSK-NET-05: desactivación de puertos sin uso

**Estado previo (sin mitigar):** los puertos que no forman parte de la topología activa permanecen, por defecto, en la VLAN 1, en modo dinámico y sin ninguna restricción de acceso. Cualquier persona con acceso físico al switch puede conectar un equipo a uno de estos puertos y obtener conectividad hacia el resto de la VLAN 1, sin pasar por ningún control adicional. No se requiere ningún comando para llegar a este estado: es la configuración de fábrica de un puerto no utilizado.

**Mitigación aplicada:** se seleccionan en conjunto todos los puertos que no forman parte de la topología activa, se asignan a la VLAN de descarte (999, sin enrutamiento hacia el resto de la red) y se desactivan administrativamente.

```
SW-LOG(config)#interface range fa0/2-9, fa0/11-19, fa0/21-24
SW-LOG(config-if-range)#switchport mode access
SW-LOG(config-if-range)#switchport access vlan 999
SW-LOG(config-if-range)#shutdown
SW-LOG(config-if-range)#exit
```

Verificación del estado final de VLAN y puertos:

```
SW-LOG(config)#do show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gig0/2
10   USUARIOS                         active    Fa0/1
30   MONITOREO                        active    Fa0/10
99   ADMINISTRACION                   active    Fa0/20
999  SIN_USO                          active    Fa0/2, Fa0/3, Fa0/4, Fa0/5
                                                 Fa0/6, Fa0/7, Fa0/8, Fa0/9
                                                 Fa0/11, Fa0/12, Fa0/13, Fa0/14
                                                 Fa0/15, Fa0/16, Fa0/17, Fa0/18
                                                 Fa0/19, Fa0/21, Fa0/22, Fa0/23
                                                 Fa0/24
1002 fddi-default                     active
1003 token-ring-default               active
1004 fddinet-default                  active
1005 trnet-default                    active
```

## 4. Mitigación de RSK-NET-06: registro centralizado de eventos (Syslog) y sincronización NTP

**Estado previo (sin mitigar):** sin un destino de logging configurado, los eventos generados por el switch (incluyendo las violaciones de Port-Security descritas en la sección 2) sólo quedan almacenados en el búfer local de memoria volátil del equipo, sin ninguna copia externa ni marca de tiempo confiable. Este es igualmente el estado por defecto del equipo, sin comandos adicionales de por medio.

**Mitigación aplicada:** se configura el envío de eventos del switch hacia el servidor SRV-SIEM, con marcas de tiempo sincronizadas mediante NTP contra el mismo servidor.

```
SW-LOG(config)#service timestamps log datetime msec
SW-LOG(config)#clock timezone CLT -4
SW-LOG(config)#ntp server 192.168.30.10
SW-LOG(config)#logging host 192.168.30.10
SW-LOG(config)#logging on
```

No se registró en este equipo una captura local de `show logging` posterior a la configuración. La correcta recepción de los eventos por parte del servidor quedó confirmada del lado de SRV-SIEM: en el panel Services > Syslog del servidor se observaron múltiples registros con HostName 192.168.99.2 (dirección de administración de SW-LOG), incluyendo mensajes del tipo %PORT_SECURITY-2-..., lo que evidencia que el switch efectivamente envió sus eventos al servidor centralizado.

## 5. Estado final del equipo

Al término de esta secuencia, el switch SW-LOG cuenta con la segmentación VLAN definida, el enlace troncal hacia R-LOG operativo, control de direcciones MAC habilitado en el puerto de operador (Fa0/1), los puertos sin uso desactivados y aislados en la VLAN 999, y el envío de eventos de seguridad hacia el servidor SIEM activo.
