# AWX_ansible_githab

Playbooks de red para Ansible AWX, reorganizados. Contiene **solo lo que está en
uso**: los playbooks por fabricante y los envoltorios que combinan ambos.

Sustituye al repositorio `AnsibleGitHab`, que queda intacto como histórico. Lo
que no se migró está listado al final, con el motivo.

## Estructura

```
playbook/
├── backup.yml            envoltorios multi-fabricante: una plantilla de AWX
├── reiniciar.yml         cubre MikroTik y Cisco en la misma ejecución
├── snmp_activar.yml
├── snmp_desactivar.yml
├── cisco/                un playbook por tarea, play limitado a EquiposCisco
│   └── vars/             perfiles de puertos y troncales
└── mikrotik/             un playbook por tarea, play limitado a EquiposMikrotik
    └── vars/

ansible.cfg               timeouts de conexión persistente
collections/requirements.yml   colecciones fijadas por versión
inventario/hosts.ini.ejemplo   solo para pruebas a mano; el inventario real está en AWX
```

Las rutas `playbook/cisco/...` y `playbook/mikrotik/...` son idénticas a las del
repositorio anterior, así que las plantillas de AWX existentes siguen
funcionando salvo las seis que se renombraron (ver más abajo).

## Convenciones

- Cada play fija su grupo: `EquiposCisco` o `EquiposMikrotik`. Los envoltorios
  importan los dos fabricantes y el play que no tiene hosts no hace nada ni da
  error, así que con **Límite** = un equipo solo corre lo que le toca.
- `gather_facts: false` y `serial: 1` en lo que toca configuración, para no
  dejar media red a medio aplicar.
- MikroTik declara `connection: ansible.netcommon.network_cli` en el play; Cisco
  la hereda del inventario de AWX.
- En Cisco, `write memory` está comentado a propósito en todos los playbooks:
  los cambios se validan antes de hacerlos permanentes. Lo que no se guarde se
  pierde al reiniciar el switch.
- En MikroTik no aplica: RouterOS persiste los cambios al momento.

## Catálogo

### Envoltorios (plantilla única para los dos fabricantes)

| Playbook | Qué hace |
|---|---|
| `playbook/backup.yml` | Respalda MikroTik y Cisco. Pensado para programarse por horario. |
| `playbook/reiniciar.yml` | Reinicia los equipos del límite, de uno en uno. |
| `playbook/snmp_activar.yml` | Comunidad SNMP + destino de traps, en el orden correcto. |
| `playbook/snmp_desactivar.yml` | Retira la huella del panel, sin apagar SNMP del equipo. |

### Cisco SF/SG/CBS

| Playbook | Qué hace |
|---|---|
| `probar_conexion.yml` | `show version`, para validar credenciales y conexión. |
| `backup.yml` | `running-config` a `/backups/<equipo>/<fecha>.cfg`. |
| `crear_vlans.yml` | Crea VLANs. |
| `configurar_trunks.yml` | Puertos troncales por perfil (`vars/trunk_profiles.yml`). |
| `configurar_access.yml` | Puertos de acceso por perfil (`vars/port_profiles.yml`). |
| `configurar_rstp.yml` | Activa RSTP. |
| `configurar_gateway.yml` | Gateway predeterminado. |
| `configurar_ip_administracion.yml` | IP de administración. Corta la sesión en curso. |
| `configurar_hostname.yml` | Hostname. |
| `configurar_ntp.yml` | Cliente SNTP. |
| `configurar_snmp.yml` / `configurar_snmp_traps.yml` | SNMPv2c y destino de traps. |
| `desactivar_snmp.yml` | Retira comunidad y destino del panel. |
| `reiniciar.yml` | Reinicio. Con `guardar_antes: true` guarda primero. |

### MikroTik RouterOS

| Playbook | Qué hace |
|---|---|
| `backup.yml` | `/export` a `/backups/<equipo>/<fecha>.rsc`. |
| `configurar_ssh_key.yml` | Usuario dedicado para AWX con su clave pública. |
| `configurar_bridge_vlans.yml` | Bridge y tabla VLAN, sin tocar puertos físicos. |
| `configurar_ports_trunks.yml` | Puertos troncales por perfil (`vars/trunk_profiles.yml`). |
| `configurar_ports_access.yml` | Puertos de acceso por perfil (`vars/port_profiles.yml`). |
| `configurar_rstp.yml` | RSTP global en el bridge. |
| `configurar_gateways.yml` | Gateways IPv4. |
| `configurar_ip_administracion_bridge.yml` | IP de administración en el bridge. Corta la sesión. |
| `configurar_hostname.yml` | Hostname. |
| `configurar_ntp.yml` | Cliente NTP. |
| `configurar_snmp.yml` / `configurar_snmp_traps.yml` | SNMP y destino de traps. |
| `desactivar_snmp.yml` | Retira el destino de traps del panel. |
| `actualizar.yml` | RouterOS al último `stable` y luego el firmware RouterBOARD. |
| `reiniciar.yml` | Reinicio. |

El orden al preparar un equipo nuevo es: bridge y VLANs, RSTP, troncales,
puertos de acceso, y al final lo que corta la conexión (IP de administración).
Ese flujo se arma como workflow en la interfaz de AWX, excluyendo los playbooks
que dejan el equipo inalcanzable a mitad de la cadena.

## Cargarlo en AWX

1. Subir esta carpeta a un repositorio privado y crear un proyecto de AWX que
   apunte a él.
2. `collections/requirements.yml` fija las versiones de `ansible.netcommon`,
   `community.routeros` y `community.ciscosmb`. Comprobar que el entorno de
   ejecución las instale o ya las traiga.
3. Una plantilla por playbook, tipo **Run**, con el inventario de AWX y las
   credenciales del fabricante correspondiente. Los envoltorios necesitan las
   credenciales de los dos a la vez.
4. Para trabajos programados: inventario fijo en la plantilla y **"Preguntar al
   iniciar" desmarcado** en ese campo. Un trabajo por horario no puede
   responder preguntas y falla si la plantilla pide el inventario al lanzar.
5. El firmware de MikroTik no está en el repositorio: `actualizar.yml` usa
   `/system package update download` y lo baja el propio equipo.

## Qué cambió respecto a `AnsibleGitHab`

Renombrados (hay que actualizar la ruta en su plantilla de AWX):

| Antes | Ahora | Motivo |
|---|---|---|
| `cisco/pruebd e conecion.yml` | `cisco/probar_conexion.yml` | espacios en el nombre y dos erratas |
| `cisco/configurar_NTP.yml` | `cisco/configurar_ntp.yml` | mayúsculas, el lado MikroTik ya era minúscula |
| `cisco/configurar_RSTP.yml` | `cisco/configurar_rstp.yml` | igual |
| `mikrotik/Actualizar.yml` | `mikrotik/actualizar.yml` | igual |
| `mikrotik/mikrotik_set_hostname.yml` | `mikrotik/configurar_hostname.yml` | prefijo redundante dentro de `mikrotik/` |
| `mikrotik/mikrotik_ssh_key_pub.yml` | `mikrotik/configurar_ssh_key.yml` | igual |

No migrados:

- **Los ~30 playbooks sueltos en la raíz de `playbook/`** (`mikrotik_DNS.yml`,
  `mikrotik_NTP.yml`, `mikrotik_RSTP.yml`, `mikrotik_gateway.yml`,
  `mikrotik_vlans_*.yml`, `mikrotik_nuevo_equipo.yml`,
  `mikrotik_nuevo_completo.yml`, `mikrotik_print_data.yml`,
  `mikrotik_ssh_key.yml`, `mikrotik_actualizar.yml`, `cisco_vlans*.yml`,
  `cisco_nuevo_equipo.yml`, `vlan2.yml`, `vlan3.yml`, `vlans_nueva.yml`,
  `peuab vlan.yml`, `ping.yml`, `backup_all.yml`, `cragar_imagen.yml`,
  `etapa_actualizacion.yml`, `actualizar_routeros.yml`): versiones anteriores de
  lo que hoy vive en `cisco/` y `mikrotik/`. Ninguno se tocó después del
  2026-10-19; los de las carpetas por fabricante son todos del 2026-10-27 o
  posterior.
- **`playbook/correctos/`**: otra copia de ese mismo material antiguo.
- **`playbook/iso_mikrotiks/routeros-7.23.7.npk`**: 20 MB de firmware en el
  repositorio. Ningún playbook vigente lo usa.
- **`playbook/cisco/copiapostaccese.yml`**: archivo vacío.
- **`ping_all.yml`**: compara `ansible_network_os` con `community.ciscosmb`,
  mientras el inventario usa `community.ciscosmb.ciscosmb`, así que la rama
  Cisco nunca se ejecuta. Se dejó fuera en lugar de arrastrarlo roto.
- **`inventario/` con `group_vars`/`host_vars` cifrados**: definían los grupos
  `switchs_cisco` y `switchs_mikrotik`, que ya no existen — los playbooks usan
  `EquiposCisco` y `EquiposMikrotik`. Las credenciales están en AWX, así que no
  tiene sentido traer secretos cifrados con su contraseña de Vault a un
  repositorio nuevo. En su lugar está `inventario/hosts.ini.ejemplo`.

## Pendientes conocidos

- `mikrotik/actualizar.yml` no declara `connection` en el play, a diferencia del
  resto de los playbooks de MikroTik; depende de que el inventario de AWX la
  defina.
- En los `reiniciar.yml`, `puerto_ssh` se calcula con `ansible_port` en vez de
  `hostvars[inventory_hostname].ansible_port`, igual que en `actualizar.yml`.
  Dentro de `delegate_to: localhost` eso resuelve al puerto de localhost; hoy no
  se nota porque nadie define `ansible_port` y cae al 22.
- En `mikrotik/reiniciar.yml` la espera inicial es de 30 s. Si a los 30 s el
  equipo todavía no cerró el SSH, `wait_for` da éxito sin que el reinicio haya
  empezado. `actualizar.yml` usa 60 s para lo mismo.
- Ningún `reiniciar.yml` falla si el reinicio no ocurrió: la comparación de
  uptime al final es un `debug`, no una comprobación.
- Los SF350 fallan de forma aleatoria al escribir por SSH con libssh 0.9.6 y
  cifrado chacha20. Si una tarea de escritura falla sin patrón, revisar el
  cifrado negociado en el entorno de ejecución.
