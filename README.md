# Ataque DHCP Spoofing

**Autor:** Roger Rodriguez  
**Matrícula:** 20250757  
**Fecha:** Junio 2026  
**Link:** https://youtu.be/BYf8ochMK4M

---

## Objetivo del laboratorio

Demostrar cómo un atacante puede suplantar un servidor DHCP legítimo en la red,
respondiendo a las solicitudes DHCP de las víctimas con información falsa como
Gateway y DNS maliciosos, logrando redirigir todo el tráfico a través del atacante.

---

## Objetivo del script

El script `dhcp_spoof.py` realiza las siguientes acciones:

1. Escucha paquetes DHCP Discover en la red
2. Responde con un DHCP Offer falso antes que el servidor legítimo
3. Asigna una IP a la víctima pero con Gateway y DNS apuntando al atacante
4. Confirma la asignación con un DHCP ACK falso

---

## Parámetros usados

| Parámetro | Valor | Descripción |
|---|---|---|
| `INTERFAZ` | `eth0` | Interfaz de red del atacante |
| `IP_FALSA` | `20.25.7.200` | IP del servidor DHCP falso |
| `GW_FALSO` | `20.25.7.100` | Gateway falso (IP del atacante) |
| `DNS_FALSO` | `20.25.7.100` | DNS falso (IP del atacante) |
| Puerto escucha | `67/68` | Puertos DHCP |

---

## Requisitos para utilizar la herramienta

### Software
- Kali Linux
- Python 3.x
- Librería Scapy

### Instalación de dependencias
```bash
sudo apt update && sudo apt install python3-scapy -y
```

### Permisos
```bash
sudo python3 dhcp_spoof.py
```

---

## Documentación del funcionamiento del script

### ¿Cómo funciona DHCP Spoofing?

El protocolo DHCP no tiene mecanismo de autenticación. Cuando un cliente
envía un DHCP Discover en broadcast, cualquier servidor puede responder.
El cliente acepta la primera respuesta que reciba, por lo que si el atacante
responde más rápido que el servidor legítimo, la víctima usará la configuración
maliciosa.

### Flujo del ataque

```
NORMAL:
PC1 --DISCOVER--> Broadcast
PC1 <--OFFER----- Servidor DHCP legítimo (GW: 20.25.7.1)
PC1 --REQUEST-->  Servidor DHCP legítimo
PC1 <--ACK------- Servidor DHCP legítimo

CON ATAQUE:
PC1 --DISCOVER--> Broadcast
PC1 <--OFFER----- Kali (GW: 20.25.7.100) ← más rápido
PC1 --REQUEST-->  Broadcast
PC1 <--ACK------- Kali (GW: 20.25.7.100)
PC1 ahora enruta TODO su tráfico por el Kali
```

### Diagrama del ataque

```
   PC1                ATTACKER (Kali)         DHCP Legítimo
20.25.7.10            20.25.7.100              20.25.7.1
    |                      |                       |
    |----DISCOVER---------->|---------------------->|
    |<---OFFER falso--------|                       |
    |    GW=20.25.7.100     |       (llega tarde)   |
    |    DNS=20.25.7.100    |                       |
    |----REQUEST----------->|---------------------->|
    |<---ACK falso----------|                       |
    |                       |                       |
    | PC1 usa Kali como GW y DNS                    |
```

### Pasos del script

1. **Sniffing:** Escucha paquetes UDP en puertos 67/68
2. **Detección Discover:** Identifica paquetes DHCP tipo 1
3. **OFFER falso:** Responde con IP asignada y GW/DNS del atacante
4. **Detección Request:** Identifica paquetes DHCP tipo 3
5. **ACK falso:** Confirma la asignación maliciosa

---

## Documentación de la red

### Topología

```
                    Router-GW
                   20.25.7.1/24
                        |
                    SW-CORE
                   /         \
            SW-ACCESS-1    SW-ACCESS-2
            /       \            \
        ATTACKER    PC1          PC2
      20.25.7.100  20.25.7.10  20.25.7.20
```

### Interfaces y direccionamiento

| Dispositivo | Interfaz | IP | VLAN |
|---|---|---|---|
| Router-GW | e0/0 | 20.25.7.1/24 | 1 |
| ATTACKER | eth0 | 20.25.7.100/24 | 1 |
| PC1 | eth0 | 20.25.7.10/24 | 1 |
| PC2 | eth0 | 20.25.7.20/24 | 1 |

---

## Ejecución del ataque

```bash
# Clonar el repositorio
git clone https://github.com/tu-usuario/dhcp-spoofing-attack

# Entrar al directorio
cd dhcp-spoofing-attack

# Ejecutar el script
sudo python3 dhcp_spoof.py
```

### Resultado esperado
```
==================================================
 DHCP Spoofing Attack
 Autor    : Roger Rodriguez
 Matricula: 20250757
 IP Falsa : 20.25.7.200
 GW Falso : 20.25.7.100
 DNS Falso: 20.25.7.100
==================================================
[*] Esperando paquetes DHCP... Ctrl+C para detener
[*] DHCP Discover recibido de 00:50:79:66:68:06
[+] DHCP Offer falso enviado -> GW: 20.25.7.100 DNS: 20.25.7.100
[*] DHCP Request recibido
[+] DHCP ACK falso enviado -> IP: 20.25.7.87 GW: 20.25.7.100
```

### Activar en PC1 para probar
```
VPCS> dhcp
```

---

## Contra-medida

### Descripción
Habilitar **DHCP Snooping** en el switch para que solo los puertos
de confianza puedan enviar respuestas DHCP. Los puertos de acceso
donde están los clientes quedan bloqueados para respuestas DHCP.

### Implementación en SW-CORE
```
enable
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 1
no ip dhcp snooping information option
interface e0/0
 ip dhcp snooping trust
end
write memory
```

### Verificación
```
SW-CORE# show ip dhcp snooping
Switch DHCP snooping is enabled
DHCP snooping is operational on following VLANs: 1
```

### ¿Por qué funciona?
DHCP Snooping marca como **trusted** solo el puerto conectado al servidor
DHCP legítimo. Cualquier respuesta DHCP que llegue por un puerto no confiable
(como el puerto del atacante) es descartada automáticamente.

---

## Capturas de pantalla

| Archivo | Descripción |
|---|---|
| <img width="973" height="547" alt="image" src="https://github.com/user-attachments/assets/c7094bf2-c382-4189-af33-f6a0bddb82c3" />|Topología en EVE-NG|
| <img width="933" height="518" alt="image" src="https://github.com/user-attachments/assets/e783b02f-ce61-4364-8738-6f3e646ad5d3" />| Script recibiendo DHCP Discover y enviando Offer falso |
| <img width="319" height="111" alt="image" src="https://github.com/user-attachments/assets/8d3f38e2-8e2e-4e9d-88ec-bd759182a75a" />| PC1 ejecutando comando dhcp |
|<img width="865" height="455" alt="image" src="https://github.com/user-attachments/assets/181fd04d-c668-425d-b62c-3b99ed764495" /> | DHCP Snooping activo en SW-CORE |

---

## Referencias

- [DHCP Spoofing Attack](https://www.imperva.com/learn/application-security/dhcp-spoofing/)
- [Cisco DHCP Snooping](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst6500/ios/12-2SX/configuration/guide/book/snoodhcp.html)
- [Scapy Documentation](https://scapy.readthedocs.io/)
