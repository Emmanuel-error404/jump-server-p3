## Video demostrativo

**[PENDIENTE: enlace del video demostrativo actualizado con Kali, vpnc y Firefox]**

El video debe presentar la configuración mediante GUI del FortiGate, la conexión desde Kali con vpnc, los accesos de los dos perfiles y RemoteApp desde Firefox. [PENDIENTE: video y URL del repositorio].

# Laboratorio de acceso remoto con FortiGate VPN y RemoteApp

**Autor:** Emmanuel Orlando Rodríguez Núñez  
**Matrícula:** 20250798  
**Institución:** Instituto Tecnológico de Las Américas  
**Plataforma:** GNS3

## Propósito

Implementar una red segmentada en la que los usuarios acceden por VPN a un Jump Server. Este servidor concentra el acceso al Web Server mediante HTTPS, SSH y RDP. Las aplicaciones publicadas en RemoteApp se asignan según el perfil de cada usuario.

La documentación describe la arquitectura y la configuración del laboratorio. Las capturas y los resultados nuevos quedan pendientes; no se considera demostrado el funcionamiento por la sola descripción de la configuración.

## Documentación técnica

- [Consultar el informe técnico en PDF](docs/Documentacion_tecnica_FortiGate.pdf).
- [Descargar el informe técnico editable en Word](docs/Documentacion_tecnica_FortiGate.docx).
- [Consultar las capturas de configuración](images/).
- [Consultar los diagramas](diagrams/).

El informe contiene el plan de direccionamiento, la configuración Cisco, las interfaces y políticas del FortiGate, la configuración descrita de los servidores y los anexos de referencia. Conserva las figuras vigentes, identifica topología y NAT como históricos, actualiza los diagramas y retira las antiguas capturas del cliente. Añade incidentes y una tabla de 15 pruebas sin resultados inventados.

## Componentes del laboratorio

| Componente | Función |
| --- | --- |
| R1 Cisco | ISP simulado y enlace hacia la WAN del FortiGate. |
| R2 Cisco | Gateway de VLAN 10, DHCP y NAT con sobrecarga. |
| Switch Ethernet de GNS3 | Conexión de los clientes en VLAN 10 y enlace dot1q hacia R2. |
| FortiGate | VPN IPsec, separación de las LAN y políticas de acceso. |
| Jump Server con Windows Server 2022 | Acceso remoto y publicación de aplicaciones con RDS, RemoteApp y Web Client. |
| Web Server | Sistema de Caja y servicios HTTPS, SSH y RDP descritos en la guía. |
| Cliente Kali Linux (vpnc) | VPN con vpnc 0.5.3, VLAN 10 por DHCP y Firefox para RD Web Client; perfiles básico y privilegiado. |
| Cloud1 | Acceso de gestión al FortiGate mediante port1. |

Una VM Kali se utiliza en sesiones separadas para los dos perfiles de usuario. [PENDIENTE: captura actual de la topología]. El esquema del Web Server utiliza Ubuntu, Apache, OpenSSH y xrdp.

## Topología

![Topología histórica del laboratorio en GNS3](images/topologia_gns3.png)

La imagen conserva la etiqueta del cliente anterior; no demuestra el uso de Kali. **[PENDIENTE: actualizar el nodo y reemplazar la captura de GNS3]**. El Mermaid siguiente ya representa el cliente final.

```mermaid
flowchart TD
    C["Cliente Kali Linux con vpnc · VLAN 10"] --> S["Switch Ethernet"]
    S --> R2["R2 · DHCP y NAT"]
    R2 --> R1["R1 · ISP simulado"]
    R1 --> F["FortiGate · port2 WAN"]
    G["Cloud1 · Gestión"] -. "port1" .-> F
    F -->|port3| J["Jump Server · 10.7.98.2"]
    F -->|port4| W["Web Server · 10.7.98.10"]
```

Kali establece la VPN contra `20.25.98.2` mediante vpnc. El FortiGate termina el túnel y controla el acceso al Jump. Las dos LAN de servidores utilizan interfaces y subredes independientes.

## Direccionamiento

| Segmento | Red | Direcciones asignadas |
| --- | --- | --- |
| ISP entre R1 y R2 | `20.25.7.0/30` | R1 f0/0: `20.25.7.1`; R2 f0/0: `20.25.7.2`. |
| ISP hacia FortiGate | `20.25.98.0/30` | R1 g1/0: `20.25.98.1`; FortiGate port2: `20.25.98.2`. |
| Usuarios VLAN 10 | `10.20.25.0/25` | R2 g1/0.10: `10.20.25.1`; DHCP: `10.20.25.10` a `10.20.25.126`. |
| LAN Jump Server | `10.7.98.0/29` | FortiGate port3: `10.7.98.1`; Jump: `10.7.98.2`. |
| LAN Web Server | `10.7.98.8/29` | FortiGate port4: `10.7.98.9`; Web: `10.7.98.10`. |
| Clientes VPN | Pool `10.20.25.128/25`; IP de interfaz `/32` | `tun0`: IP del rango `10.20.25.130` a `10.20.25.170`, según la sesión. |
| Gestión del FortiGate | `192.168.23.0/24` | port1: `192.168.23.152`. |
| Loopback del ISP | `8.8.8.8/32` | R1 Loopback0. |

Kali obtiene la IP física en `eth0` por DHCP: **[PENDIENTE: IP DHCP de Kali]**. `tun0` recibe una dirección `/32`, por ejemplo `10.20.25.130/32` o `10.20.25.131/32`; **[PENDIENTE: IP de tun0 de la sesión documentada]**. La máscara `/25` vista en el adaptador del cliente anterior no aplica al túnel de Linux.

La tabla NAT de la Figura 5 es histórica: su origen `10.20.25.10` no confirma la IP actual de Kali. **[PENDIENTE: asociación DHCP y tabla NAT actuales]**.

Las direcciones `20.25.x.x` representan los enlaces públicos dentro de la simulación. La dirección `8.8.8.8` está configurada como loopback de R1.

## Configuración de red

R2 termina VLAN 10 mediante `encapsulation dot1Q 10`, entrega direcciones con el pool `USUARIOS_V10` y utiliza `10.20.25.1` como gateway. El NAT con sobrecarga traduce la red de usuarios a `20.25.7.2`. Su ruta por defecto apunta a `20.25.7.1`.

En el FortiGate, la ruta por defecto utiliza `20.25.98.1` mediante `WAN-ISP (port2)`. Las interfaces `lan-jump (port3)` y `lan-web (port4)` actúan como gateways de sus servidores. La configuración y la demostración del FortiGate se realizan mediante GUI.

## VPN y perfiles de usuario

El túnel `VPN-REMOTE` se asocia con port2 y utiliza el pool `10.20.25.130` a `10.20.25.170`. El destino autorizado del acceso remoto es el Jump Server `10.7.98.2`.

| Perfil | Cuenta | Grupo del FortiGate | Acceso VPN al Jump | Aplicaciones asignadas en RemoteApp |
| --- | --- | --- | --- | --- |
| Básico | `usr_basico` | `grp_vpn_basico` | HTTPS. | Navegador Web. |
| Privilegiado | `usr_priv` | `grp_vpn_priv` | HTTPS y RDP. | Navegador Web, PuTTY y Escritorio remoto. |

El grupo `grp-vpn-all` incluye ambas cuentas para la autenticación VPN. La guía describe los grupos de Windows `GG-RA-BASICO` y `GG-RA-PRIV` para asignar las aplicaciones. Las cuentas del FortiGate y de Windows pertenecen a sistemas de autenticación distintos.


### Parámetros actuales de vpnc

| Parámetro | Valor documentado |
| --- | --- |
| SO y cliente | Kali Linux, vpnc 0.5.3. [PENDIENTE: salida completa de `vpnc --version`]. |
| Negociación | IPsec IKEv1, modo agresivo, PSK + XAuth, NAT-T. |
| Gateway e identificador | `20.25.98.2`; IPSec ID `vpnc-lab`. |
| Archivo | `/etc/vpnc/fortigate-lab.conf`; sin secretos en la versión pública. |
| Túnel | `tun0`, dirección `/32` del pool `10.20.25.130–10.20.25.170`. |
| Split tunnel | Destino autorizado `10.7.98.2/32`, `jumblab.lab.local`. |
| Fase 1 | `des-sha1`, observada según la actualización aportada. |
| Fase 2 | [PENDIENTE: confirmar propuesta final y cambio exacto]. `aes256-sha1` es un ejemplo, no un resultado confirmado. |
| Selectores y PFS | [PENDIENTE: confirmar valores finales en GUI]. |

`des-sha1` es la negociación reportada para esta práctica. Se informan limitaciones del vpnc 0.5.3 utilizado para SHA256 y DH14 completo; las compilaciones posteriores pueden variar. DES no es el único cifrado de vpnc y esta propuesta no demuestra incompatibilidad con AES. Se recomienda, como mejora futura, strongSwan con AES/SHA256/DH14 compatibles en ambos extremos. Esta recomendación no describe una configuración aplicada.

Los datos nuevos no se atribuyen a las capturas históricas del túnel. **[PENDIENTE: captura GUI de los parámetros actuales y evidencia de asociación de grupo]**.

## Políticas del FortiGate

| Política | Flujo | Servicio | Acción |
| --- | --- | --- | --- |
| `DENY-SSH-BASICO` | VPN del perfil básico hacia Jump. | SSH. | DENY. |
| `VPN-BASICO-A-JUMP` | VPN del perfil básico hacia Jump. | `SG-VPN-BASICO`: HTTPS. | ACCEPT. |
| `VPN-PRIV-A-JUMP` | VPN del perfil privilegiado hacia Jump. | `SG-VPN-PRIV`: HTTPS y RDP. | ACCEPT. |
| `JUMP-A-WEB` | Jump en port3 hacia Web en port4. | `SG-JUMP-A-WEB`: HTTPS, RDP y SSH. | ACCEPT. |
| `DENY-SSH-BASICO2` | VPN del perfil básico hacia Web. | SSH. | DENY. |
| `test` | VPN hacia Jump. | ALL. | Deshabilitada. |
| `Implicit Deny` | Tráfico sin coincidencia con una regla de permiso. | ALL. | DENY. |

La captura documenta NAT desactivado en las reglas de aceptación y ninguna regla ACCEPT desde VPN-REMOTE hacia la LAN Web. El cierre visible es una denegación implícita. La asignación de aplicaciones dentro del Jump se controla en RemoteApp. **[PENDIENTE: confirmar si se añadió `DENY-ALL` o se habilitó `fwpolicy-implicit-log`]**. La matriz conserva el cierre observado hasta recibir evidencia GUI del cambio.

![Políticas históricas del FortiGate registradas en la práctica](images/politicas_fortigate.png)

## Servicios y aplicaciones

| Servicio | Destino | Uso |
| --- | --- | --- |
| HTTPS, TCP 443 | Jump y Web Server. | Portal de acceso remoto y sitio Sistema de Caja. |
| SSH, TCP 22 | Web Server desde Jump. | Administración mediante PuTTY para el perfil privilegiado. |
| RDP, TCP 3389 | Jump y Web Server. | Acceso remoto y conexión mediante la aplicación publicada. |

El sitio del Web Server se identifica como `https://10.7.98.10`. La guía describe una página de Apache denominada Sistema de Caja. El acceso a RemoteApp se organiza mediante `https://jumblab.lab.local/RDWeb` y `https://jumblab.lab.local/RDWeb/webclient`, con resolución de `jumblab.lab.local` hacia `10.7.98.2`.


## RD Gateway Web Client y certificados

Según la actualización aportada, `Get-RDServer` lista `RDS-RD-SERVER`, `RDS-CONNECTION-BROKER`, `RDS-WEB-ACCESS` y `RDS-GATEWAY`. RD Gateway está instalado y `GatewayExternalFqdn = jumblab.lab.local`. En este despliegue permite abrir la sesión del Web Client por WebSocket seguro sobre TCP 443. Cargar la lista de apps no prueba que la sesión funcione.

`Get-RDWebClientPackage` informa el paquete `rd-html5`, versión `2.1.85.0`, publicado como `Production`. El certificado RDS es autofirmado y `Get-RDCertificate` informa Level **No es de confianza**. Se importó el certificado público en las autoridades de Firefox en Kali. **[PENDIENTE: confirmar importación en el servidor, asignaciones RDS y capturas de las consultas, CAP y RAP]**.

```powershell
# Consultas en el Jump Server
Get-RDServer
Get-RDDeploymentGatewayConfiguration
Get-RDCertificate
Get-RDWebClientPackage
```

## Configuración del cliente Kali

La entrada de `/etc/hosts` es:

```text
10.7.98.2 jumblab.lab.local
```

Archivo usado: `/etc/vpnc/fortigate-lab.conf`. Esquema público de referencia, con líneas de secretos omitidas; no equivale a una exportación real:

```text
IPSec gateway 20.25.98.2
IPSec ID vpnc-lab
IKE Authmode psk
Xauth username usr_basico
# Para el otro perfil: Xauth username usr_priv
# [PENDIENTE: opciones reales de NAT-T, DES, DH, PFS y rutas]
```

Las credenciales se proporcionan localmente. **[PENDIENTE: copia real saneada y captura del archivo con la PSK y la contraseña completamente ocultas]**. Confirmar `Enable Single DES` en el archivo real antes de documentarlo como opción aplicada.

```bash
sudo vpnc --no-detach --debug 1 /etc/vpnc/fortigate-lab.conf
ip a show eth0
ip a show tun0
ip route get 10.7.98.2
curl -kI --connect-timeout 5 https://jumblab.lab.local/RDWeb/
nc -zv -w 5 10.7.98.2 443
ssh -o ConnectTimeout=5 usr_basico@10.7.98.2
xfreerdp /v:jumblab.lab.local /u:usr_priv /d:lab.local
firefox https://jumblab.lab.local/RDWeb/webclient
# Al terminar la sesión
sudo vpnc-disconnect
```

**[PENDIENTE: nuevas capturas de eth0 y tun0, salida `got address`, SSH con timeout, hosts, autoridades de Firefox y RemoteApp por perfil]**. No incluir secretos en comandos ni capturas. `curl -kI` omite la validación TLS: no demuestra confianza del certificado ni una sesión RemoteApp.

Para comprobar el bloqueo directo al Web Server con split tunnel:

```bash
sudo ip route add 10.7.98.10/32 dev tun0
ip route get 10.7.98.10
curl -kI --connect-timeout 5 https://10.7.98.10
nc -zv -w 5 10.7.98.10 3389
ssh -o ConnectTimeout=5 usr_basico@10.7.98.10
# Retirar la ruta de prueba
sudo ip route del 10.7.98.10/32 dev tun0
```

La ruta no concede acceso ni modifica los selectores IPsec. Si Fase 2 solo acepta el Jump, el tráfico al Web puede descartarse antes de la política. Para acreditar `DENY-SSH-BASICO2`, verificar llegada al firewall, perfil XAuth y política mediante Forward Traffic en GUI. El usuario del comando SSH no cambia la identidad VPN.

## Problemas encontrados y solución

1. **Identidad del cliente anterior:** `Deny: policy violation`, Source User vacío en Forward Traffic, aunque `diagnose vpn ike gateway list` mostraba `xauth-user` correcto. XAuth funcionaba, pero la identidad no llegaba a la política.
2. **Asociación con vpnc:** el gateway list reporta `groups: grp_vpn_priv`. **[PENDIENTE: confirmar Source User en los logs actuales]**. Esa salida no demuestra por sí sola el campo de Forward Traffic.
3. **Fase 2:** `NO_PROPOSAL_CHOSEN` porque FortiGate ofrecía solo `des-sha256` y vpnc proponía SHA1/MD5. Se reporta corrección al ajustar Fase 2. **[PENDIENTE: confirmar cambio exacto y propuesta final]**.
4. **Web Client:** la lista de apps cargaba, pero la sesión caía con `connection to the remote PC was lost`. Se revisaron FQDN del gateway, certificado autofirmado y RD Gateway/CAP/RAP. **[PENDIENTE: causa final, solución y evidencia de sesión estable]**.
5. **Sesiones VPN huérfanas:** reportadas al cerrar vpnc sin `vpnc-disconnect`. Se comunicó `diagnose vpn ike gateway clear name <NOMBRE_DEL_TUNEL>` para limpieza; se registra como antecedente, sin sustituir la demostración GUI del FortiGate. **[PENDIENTE: evidencia GUI del estado actual y desconexión limpia]**.
6. **SSH y perfil:** bajo VPN de `usr_priv`, SSH directo al Jump cae en Implicit Deny con log Disabled. La prueba de `DENY-SSH-BASICO` exige VPN de `usr_basico`. **[PENDIENTE: captura del DENY explícito con usuario y política]**.

## Pruebas de funcionamiento con cliente Kali

Resultados esperados separados de resultados observados. Todas las pruebas quedan pendientes hasta recibir evidencias. Las pruebas 03 y 14 se repiten para ambos perfiles; 13 y 15 verifican los tres servicios. Desconectar antes de cambiar la cuenta XAuth. Anotar hora, IP de tun0 y perfil. Las comprobaciones locales de IP/ruta no generan un log de flujo.

| Prueba y comando o acción | Resultado esperado | Política esperada | Usuario en el log | Resultado observado |
| --- | --- | --- | --- | --- |
| 01 Dirección física — ip a show eth0 | DHCP de VLAN 10; IP exacta pendiente | `No aplica` | No aplica | [PENDIENTE: captura y resultado] |
| 02 Dirección VPN — ip a show tun0 | IP /32 del pool .130–.170 | `No aplica` | No aplica | [PENDIENTE: captura y resultado] |
| 03 Conexión IPsec — sudo vpnc --no-detach --debug 1 /etc/vpnc/fortigate-lab.conf | Conexión y got address; Fase 2 pendiente | `Autenticación IKE y XAuth` | usr_basico / usr_priv; registro IKE | [PENDIENTE: captura y resultado] |
| 04 Ruta al Jump — ip route get 10.7.98.2 | Salida por tun0 | `No aplica` | No aplica | [PENDIENTE: captura y resultado] |
| 05 Ruta de prueba al Web — sudo ip route add 10.7.98.10/32 dev tun0; ip route get 10.7.98.10 | Ruta por tun0; selector por confirmar | `No aplica` | No aplica | [PENDIENTE: captura y resultado] |
| 06 HTTPS básico al Jump — curl -kI --connect-timeout 5 https://jumblab.lab.local/RDWeb/; nc -zv -w 5 10.7.98.2 443 | TCP 443 permitido; respuesta HTTPS | `VPN-BASICO-A-JUMP` | XAuth usr_basico; Source User pendiente | [PENDIENTE: captura y resultado] |
| 07 SSH básico al Jump — ssh -o ConnectTimeout=5 usr_basico@10.7.98.2 | DENY explícito; validar log, no solo timeout | `DENY-SSH-BASICO` | XAuth usr_basico; Source User pendiente | [PENDIENTE: captura y resultado] |
| 08 SSH básico al Web — ssh -o ConnectTimeout=5 usr_basico@10.7.98.10 | DENY si llega al firewall y pasa selector | `DENY-SSH-BASICO2 si coincide` | XAuth usr_basico; Source User pendiente | [PENDIENTE: captura y resultado] |
| 09 RDP básico al Jump — nc -zv -w 5 10.7.98.2 3389 | Bloqueado; log implícito deshabilitado | `Implicit Deny; cierre por confirmar` | XAuth usr_basico; sin log esperado | [PENDIENTE: captura y resultado] |
| 10 HTTPS privilegiado al Jump — curl -kI --connect-timeout 5 https://jumblab.lab.local/RDWeb/ | TCP 443 permitido | `VPN-PRIV-A-JUMP` | XAuth usr_priv; Source User pendiente | [PENDIENTE: captura y resultado] |
| 11 RDP privilegiado al Jump — xfreerdp /v:jumblab.lab.local /u:usr_priv /d:lab.local | RDP permitido; sesión tras autenticación | `VPN-PRIV-A-JUMP` | XAuth usr_priv; Source User pendiente | [PENDIENTE: captura y resultado] |
| 12 SSH privilegiado al Jump — ssh -o ConnectTimeout=5 usr_priv@10.7.98.2 | Bloqueado; no valida DENY del básico | `Implicit Deny; cierre por confirmar` | XAuth usr_priv; sin log esperado | [PENDIENTE: captura y resultado] |
| 13 Acceso directo privilegiado al Web — curl -kI --connect-timeout 5 https://10.7.98.10; nc -zv -w 5 10.7.98.10 3389; ssh -o ConnectTimeout=5 usr_priv@10.7.98.10 | HTTPS RDP SSH bloqueados; llegada por confirmar | `Sin ACCEPT VPN a Web; Implicit Deny si se evalúa` | XAuth usr_priv; sin log esperado | [PENDIENTE: captura y resultado] |
| 14 RemoteApp por perfil — firefox https://jumblab.lab.local/RDWeb/webclient | Básico solo Web; privilegiado Web PuTTY RDP; sesiones funcionales | `VPN-BASICO-A-JUMP / VPN-PRIV-A-JUMP; asignación RDS` | usr_basico y usr_priv; contrastar RDS y Forward Traffic | [PENDIENTE: captura y resultado] |
| 15 Jump hacia Web — Desde RemoteApp en Jump: navegador https://10.7.98.10; PuTTY SSH 10.7.98.10; RDP 10.7.98.10 | HTTPS SSH RDP permitidos desde 10.7.98.2; SSH y RDP con perfil privilegiado | `JUMP-A-WEB` | No inferir Source User; origen Jump y usuario en RDS | [PENDIENTE: captura y resultado] |


Para `JUMP-A-WEB`, el origen es `10.7.98.2`; no se infiere el usuario del RemoteApp en Source User. Correlacionar la sesión RDS. Un timeout aislado o la ausencia de log no demuestran una política específica; confirmar trayecto, selector y regla.

## Organización del repositorio

| Ruta | Contenido |
| --- | --- |
| `README.md` | Video, propósito, arquitectura y navegación. |
| `docs/` | Documentación técnica en PDF y Word. |
| `images/` | Capturas de la topología y de la configuración. |
| `diagrams/` | Diagramas de topología lógica y flujos de acceso. |
| `configs/` | Running-configs reales de R1 y R2 y respaldo del FortiGate. |
| `scripts/` | Scripts efectivamente usados en servidores y pruebas desde Kali. |
| `video/` | Material complementario o referencia al video. |

## Archivos de configuración y scripts

Las exportaciones reales de los routers deben incorporarse en `configs/` como `R1-running-config.txt` y `R2-running-config.txt`. El respaldo del FortiGate se obtiene desde la GUI. Los comandos de referencia incluidos en los anexos del informe no sustituyen estas exportaciones.

En `scripts/` se conservan los archivos utilizados para configurar el Web Server y el Jump Server, como Netplan, el contenido Web y los scripts de Bash o PowerShell. Incluir una copia saneada de `/etc/vpnc/fortigate-lab.conf` en `configs/`, con todas las líneas de PSK y contraseñas eliminadas. Los secretos y certificados con claves privadas se excluyen de la versión pública. **[PENDIENTE: running-configs, respaldo GUI y scripts reales saneados]**.

## Preparación de la entrega

1. Colocar este `README.md` en la raíz del repositorio y conservar las carpetas del paquete adjunto.
2. Completar el enlace del video actualizado al inicio del README y la URL del repositorio.
3. Incorporar los running-configs, el respaldo del FortiGate y los scripts reales en sus carpetas.
4. Subir el conjunto al repositorio y comprobar los enlaces desde su página principal.

La documentación conserva la guía y las evidencias vigentes y registra los datos nuevos aportados sobre Kali y RDS. Los resultados, las capturas nuevas y el video siguen pendientes. No se afirma que se haya publicado o actualizado GitHub.


## Pendientes para cerrar la documentación

- Video actualizado y URL del repositorio; nueva topología GNS3 y enlace a sus figuras.
- IP DHCP de Kali, IP de tun0 por sesión, asociación DHCP y NAT actuales.
- Compilación completa de vpnc, archivo real saneado y opciones NAT-T/DES/DH/PFS/rutas.
- Propuesta final y cambio exacto de Fase 2, selectores y captura GUI del túnel.
- Source User en Forward Traffic; evidencia de la asociación de grupo y del DENY con usr_basico.
- Importación del certificado en el servidor, asignaciones y capturas de roles, gateway, paquete, certificado, CAP y RAP.
- Causa y solución final del Web Client; sesiones de Firefox y RemoteApp de ambos perfiles.
- Capturas de eth0/tun0, got address, hosts, autoridades de Firefox y SSH con timeout.
- Confirmación de DENY-ALL o fwpolicy-implicit-log, y actualización de matrices si corresponde.
- Resultados y capturas de las 15 pruebas, evidencia local del Web Server y desconexión limpia.
- Configuraciones y scripts reales saneados; archivos de images/ y diagrams/ incluidos y enlaces comprobados.

## Referencias técnicas

- [Manual de vpnc 0.5.3r550 en Debian](https://manpages.debian.org/stretch/vpnc/vpnc.8.en.html).
- [Configuración del cliente web de Escritorio remoto en Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remote-desktop-web-client-admin).

Estas referencias respaldan criterios técnicos y no sustituyen pruebas del laboratorio.
