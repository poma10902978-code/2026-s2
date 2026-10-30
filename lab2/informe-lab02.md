# Informe de laboratorio 2: FRRouting en Ubuntu

**Asignatura:** ETN1011 – Sistemas de Comunicación II  
**Estudiante:** Josue Ivan Poma Chavez  
**Entorno:** Ubuntu 26.04.1 LTS en Oracle VirtualBox  
**Fecha de realización:** 29 de octubre de 2026

## 1. Introducción

En este laboratorio se trabajó con FRRouting (FRR), un conjunto de servicios de enrutamiento para Linux. La práctica se desarrolló sobre Ubuntu en una máquina virtual de Oracle VirtualBox, adaptando las instrucciones de la guía, que originalmente contemplan Ubuntu sobre WSL2. Se verificó el funcionamiento del servicio FRR, se utilizó una interfaz virtual `dummy0` y se configuró una ruta estática de descarte (`blackhole`).

## 2. Objetivos

- Reconocer la diferencia entre la administración de interfaces en Linux y el enrutamiento gestionado por FRR.
- Verificar el estado del servicio FRR y consultar su tabla de rutas.
- Comprobar una interfaz virtual de laboratorio con dirección IPv4.
- Configurar y verificar una ruta estática `blackhole`.
- Registrar evidencias de funcionamiento y persistencia de la configuración.
- Relacionar los comandos de Linux/FRR con sus equivalentes conceptuales de Cisco IOS.

## 3. Datos del entorno

| Elemento | Información |
|---|---|
| Sistema operativo | Ubuntu 26.04.1 LTS |
| Kernel | Linux 7.0.0-42-generic |
| Plataforma | Oracle VirtualBox |
| Usuario | `ghost` |
| Equipo | `ghost-VirtualBox` |
| Interfaz principal | `enp0s3` |
| Modo de red | NAT |
| Dirección IPv4 principal observada | `10.0.2.15/24` |
| Software de enrutamiento | FRRouting (FRR) |

**Nota de adaptación:** La guía de referencia describe WSL2 y la interfaz `eth0`. En este laboratorio se utilizó VirtualBox con `enp0s3`, sin modificar la interfaz principal ni la ruta predeterminada.

## 4. Topología utilizada

```text
           Red NAT de VirtualBox
                    |
                enp0s3
                    |
          Ubuntu + FRRouting
                    |
                  dummy0
               10.10.10.1/24

 Ruta estática de prueba:
 198.51.100.0/24 -> blackhole
```

La interfaz `dummy0` simula una red local conectada. La ruta `blackhole` descarta tráfico dirigido a la red de documentación `198.51.100.0/24`; no establece comunicación con otro router.

## 5. Desarrollo del laboratorio

### 5.1. Verificación del servicio FRR

Se comprobó el estado del servicio mediante:

```bash
systemctl is-active frr
```

**Resultado observado:** `active`, lo que confirma que el servicio FRR estaba en ejecución.

### 5.2. Verificación de la interfaz `dummy0`

Se consultó la dirección de la interfaz virtual:

```bash
ip -br addr show dummy0
```

**Resultado observado:** la interfaz `dummy0` presentó la dirección `10.10.10.1/24` y el estado `UNKNOWN`, habitual en interfaces de tipo dummy. Este estado no significa por sí mismo que la interfaz esté averiada.

### 5.3. Verificación de la ruta estática

Se ejecutó:

```bash
sudo vtysh -c "show ip route 198.51.100.0/24"
```

**Resultado observado:** FRR mostró la ruta `198.51.100.0/24` como estática, instalada y con destino `unreachable (blackhole)`.

La ruta representa una regla de descarte, no una ruta hacia una red alcanzable.

### 5.4. Persistencia de la configuración

Se trabajó con la persistencia de FRR y se registró evidencia específica. En FRR, el comando:

```text
write memory
```

permite guardar la configuración del enrutador en su archivo de configuración. Esto **no** vuelve persistente una interfaz creada manualmente con `ip link`; son configuraciones administradas por mecanismos distintos.

### 5.5. Registro de evidencias

Se guardaron los resultados de las comprobaciones en archivos de texto. Entre los archivos obtenidos durante la práctica se encuentran:

- `frr-persistencia.txt`
- `frr-verificacion-final.txt`

**Aclaración:** En las capturas disponibles, estos dos archivos se encuentran dentro de `lab1/evidencias/`. Para presentar el laboratorio 2 de manera independiente, corresponde copiarlos a `lab2/evidencias/` sin borrar los originales.

### 5.6. Pruebas complementarias de la guía

La guía también propone consultar rutas conectadas, revisar contadores de `dummy0`, probar `ip route get`, examinar registros con `journalctl` y explorar OSPF. Estas actividades están contempladas en el material de referencia, pero **no se consignan como realizadas** porque no contamos con evidencia de su ejecución en las capturas revisadas.

## 6. Comparación con Cisco IOS

| Operación | Linux / FRR | Cisco IOS (equivalente aproximado) |
|---|---|---|
| Consultar interfaces | `ip -br addr` | `show ip interface brief` |
| Consultar rutas | `sudo vtysh -c "show ip route"` | `show ip route` |
| Consultar configuración | `show running-config` en `vtysh` | `show running-config` |
| Guardar configuración | `write memory` | `copy running-config startup-config` |
| Ruta de descarte | `ip route 198.51.100.0/24 blackhole` | `ip route 198.51.100.0 255.255.255.0 Null0` |

## 7. Resultados

Se verificó que FRRouting estaba activo en Ubuntu, que la interfaz virtual `dummy0` tenía la dirección `10.10.10.1/24` y que FRR reconocía la ruta estática `198.51.100.0/24` como `blackhole`. Se generaron archivos de evidencia para documentar el estado del entorno y la configuración.

No se realizó una demostración de conectividad entre routers: la topología empleada tiene un solo sistema FRR y una interfaz virtual local.

## 8. Dificultades encontradas

Durante el trabajo con el repositorio Git se presentaron errores de verificación de certificados SSL asociados a una fecha incorrecta en el sistema invitado. Tras ajustar temporalmente el reloj, se consiguió realizar `git push` y se comprobó que la rama `main` quedó actualizada y el árbol de trabajo limpio. Este inconveniente correspondió a la sincronización del repositorio, no al funcionamiento de FRR.

## 9. Conclusiones

1. FRRouting permite administrar rutas desde una interfaz de comandos similar en algunos aspectos a Cisco IOS, mientras que Linux mantiene las interfaces y la tabla de encaminamiento del sistema.
2. La interfaz `dummy0` permite realizar ejercicios de enrutamiento sin agregar otra máquina virtual.
3. La ruta `blackhole` es útil para estudiar rutas estáticas de descarte sin depender de una red externa.
4. La persistencia de FRR y la persistencia de las interfaces Linux deben gestionarse por separado.
5. La conservación de evidencias y su organización por laboratorio facilitan la revisión y entrega del trabajo.

## 10. Referencias

- Guía del curso: `lab2/guia_laboratorio_frr_wsl_ubuntu_cisco.md`.
- Material de apoyo: `lab2/curso_frrouting_wsl_ubuntu_un_router.md`.
- Documentación oficial de FRRouting: https://docs.frrouting.org/en/latest/
- Documentación de `vtysh`: https://docs.frrouting.org/en/latest/vtysh.html
