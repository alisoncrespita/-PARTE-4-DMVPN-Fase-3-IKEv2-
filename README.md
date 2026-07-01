# -PARTE-4-DMVPN-Fase-3-IKEv2-
# DMVPN Phase 3 with IPSec IKEv2 & OSPF Dynamic Routing

Este repositorio contiene los scripts oficiales de configuración para la migración e implementación de una red privada virtual dinámica multipunto de última generación (**DMVPN Fase 3**). Se ha elevado el nivel de seguridad del plano de control utilizando **IPSec IKEv2** y se optimizó el enrutamiento inter-sucursal mediante las directivas bajo demanda de NHRP.

**Estudiante:** Alison Domeirys Caines Bautista  
**Matrícula:** 2024-1113  
**Institución:** Instituto Tecnológico de Las Américas (ITLA)  

---

## 🛠️ Evolución Tecnológica (Fase 2 IKEv1 vs Fase 3 IKEv2)

1. **Seguridad (IKEv2):** Se migró la suite criptográfica a una arquitectura modular ultra segura utilizando **AES-CBC de 256 bits, firmas de integridad SHA256 y Grupo Diffie-Hellman 14**, reduciendo el intercambio inicial a solo 4 mensajes eficientes.
2. **Optimización de Rutas (Fase 3):** Mediante la inclusión de los comandos `ip nhrp redirect` en el Hub y `ip nhrp shortcut` en las sucursales, los Spokes ya no dependen del Hub para el plano de datos. El Hub delega el tráfico enviando un desvío, permitiendo que los Spokes establezcan atajos directos (*Spoke-to-Spoke*) de manera dinámica y temporal en su caché NHRP.

---

## 🏢 1. Script de Configuración - HUB (R2)

```cisco
! ####################################################
! CONFIGURACIÓN CENTRAL DMVPN FASE 3 - HUB (R2)
! Estudiante: Alison D. Caines B. - Matrícula 2024-1113
! ####################################################

configure terminal

! --- 1. Remoción de Políticas IKEv1 Anteriores ---
no crypto isakmp policy 10
no crypto isakmp key cisco123 address 0.0.0.0
no crypto ipsec profile DMVPN_PROFILE

! --- 2. Definición de Propuesta y Política IKEv2 (Fase 1) ---
crypto ikev2 proposal IKEv2_PROP 
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy IKEv2_POLICY 
 proposal IKEv2_PROP
exit

! --- 3. Creación de Llavero (Keyring) y Perfil IKEv2 ---
crypto ikev2 keyring DMVPN_KEYRING
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key local cisco123
  pre-shared-key remote cisco123
 exit
exit

crypto ikev2 profile IKEv2_PROFILE
 match identity remote address 0.0.0.0 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN_KEYRING
exit

! --- 4. Transform-Set y Perfil IPSec (Fase 2) ---
crypto ipsec transform-set TS_IKEv2 esp-aes 256 esp-sha256-hmac 
 mode transport
exit

crypto ipsec profile DMVPN_PROFILE_IKEv2
 set transform-set TS_IKEv2
 set ikev2-profile IKEv2_PROFILE
exit

! --- 5. Interfaz Tunnel0 Multipunto con Directiva REDIRECT (Fase 3) ---
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp authentication alison1113
 ip nhrp map multicast dynamic
 ip nhrp network-id 100
 ip nhrp redirect
 ip tcp adjust-mss 1360
 ip ospf network broadcast
 ip ospf priority 255
 ip ospf mtu-ignore
 tunnel source FastEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN_PROFILE_IKEv2
exit

! --- 6. Enrutamiento Dinámico OSPF Global ---
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 10.24.11.0 0.0.0.255 area 0
exit

end
write memory
🛰️ 2. Script de Configuración - SPOKE 1 (R4)
Cisco CLI
! ####################################################
! CONFIGURACIÓN SUCURSAL DMVPN FASE 3 - SPOKE 1 (R4)
! Estudiante: Alison D. Caines B. - Matrícula 2024-1113
! ####################################################

configure terminal

! --- 1. Remoción de Políticas IKEv1 Anteriores ---
no crypto isakmp policy 10
no crypto isakmp key cisco123 address 172.16.0.2
no crypto ipsec profile DMVPN_PROFILE

! --- 2. Definición de Propuesta y Política IKEv2 (Fase 1) ---
crypto ikev2 proposal IKEv2_PROP 
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy IKEv2_POLICY 
 proposal IKEv2_PROP
exit

! --- 3. Creación de Llavero (Keyring) y Perfil IKEv2 Abierto ---
crypto ikev2 keyring DMVPN_KEYRING
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key local cisco123
  pre-shared-key remote cisco123
 exit
exit

crypto ikev2 profile IKEv2_PROFILE
 match identity remote address 0.0.0.0 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN_KEYRING
exit

! --- 4. Transform-Set y Perfil IPSec (Fase 2) ---
crypto ipsec transform-set TS_IKEv2 esp-aes 256 esp-sha256-hmac 
 mode transport
exit

crypto ipsec profile DMVPN_PROFILE_IKEv2
 set transform-set TS_IKEv2
 set ikev2-profile IKEv2_PROFILE
exit

! --- 5. Interfaz Tunnel0 Cliente con Directiva SHORTCUT (Fase 3) ---
interface Tunnel0
 ip address 10.0.0.4 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp authentication alison1113
 ip nhrp map multicast 172.16.0.2
 ip nhrp map 10.0.0.1 172.16.0.2
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 100
 ip nhrp shortcut
 ip tcp adjust-mss 1360
 ip ospf network broadcast
 ip ospf priority 0
 ip ospf mtu-ignore
 tunnel source GigabitEthernet2/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN_PROFILE_IKEv2
exit

! --- 6. Enrutamiento Dinámico OSPF Global ---
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 10.4.11.0 0.0.0.255 area 0
exit

end
write memory
🛰️ 3. Script de Configuración - SPOKE 2 (R3)
Cisco CLI
! ####################################################
! CONFIGURACIÓN SUCURSAL DMVPN FASE 3 - SPOKE 2 (R3)
! Estudiante: Alison D. Caines B. - Matrícula 2024-1113
! ####################################################

configure terminal

! --- 1. Remoción de Políticas IKEv1 Anteriores ---
no crypto isakmp policy 10
no crypto isakmp key cisco123 address 172.16.0.2
no crypto ipsec profile DMVPN_PROFILE

! --- 2. Definición de Propuesta y Política IKEv2 (Fase 1) ---
crypto ikev2 proposal IKEv2_PROP 
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy IKEv2_POLICY 
 proposal IKEv2_PROP
exit

! --- 3. Creación de Llavero (Keyring) y Perfil IKEv2 Abierto ---
crypto ikev2 keyring DMVPN_KEYRING
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key local cisco123
  pre-shared-key remote cisco123
 exit
exit

crypto ikev2 profile IKEv2_PROFILE
 match identity remote address 0.0.0.0 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local DMVPN_KEYRING
exit

! --- 4. Transform-Set y Perfil IPSec (Fase 2) ---
crypto ipsec transform-set TS_IKEv2 esp-aes 256 esp-sha256-hmac 
 mode transport
exit

crypto ipsec profile DMVPN_PROFILE_IKEv2
 set transform-set TS_IKEv2
 set ikev2-profile IKEv2_PROFILE
exit

! --- 5. Interfaz Tunnel0 Cliente con Directiva SHORTCUT (Fase 3) ---
interface Tunnel0
 ip address 10.0.0.3 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp authentication alison1113
 ip nhrp map multicast 172.16.0.2
 ip nhrp map 10.0.0.1 172.16.0.2
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 100
 ip nhrp shortcut
 ip tcp adjust-mss 1360
 ip ospf network broadcast
 ip ospf priority 0
 ip ospf mtu-ignore
 tunnel source GigabitEthernet1/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN_PROFILE_IKEv2
exit

! --- 6. Enrutamiento Dinámico OSPF Global ---
router ospf 1
 network 10.0.0.0 0.0.0.255 area 0
 network 10.3.11.0 0.0.0.255 area 0
exit

end
write memory
