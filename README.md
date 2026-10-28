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
