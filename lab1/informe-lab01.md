# Laboratorio 1 — Redes virtuales con Linux

**Materia:** ETN1011 — Sistemas de Comunicación II  
**Estudiante:** Josue Ivan Poma Chavez  
**Fecha:** 29 de octubre de 2026

## 1. Datos del entorno

El laboratorio se desarrolló utilizando Ubuntu instalado en una máquina virtual mediante Oracle VirtualBox.

- Sistema operativo: Ubuntu 26.04.1 LTS.
- Kernel: Linux 7.0.0-42-generic.
- Usuario del sistema: ghost.
- Herramientas: iproute2, ping, traceroute, tcpdump, Git y GitHub.
- Adaptador de red de VirtualBox: NAT.

Durante el reconocimiento inicial se identificó la interfaz `enp0s3`, con dirección IPv4 `10.0.2.15/24` y puerta de enlace `10.0.2.2`.

Las pruebas hacia `8.8.8.8` y `google.com` tuvieron 0 % de pérdida de paquetes, demostrando conectividad externa y resolución DNS.

## 2. Objetivo

Construir una red virtual aislada mediante network namespaces e interfaces VETH, configurar direcciones IPv4 y comprobar la comunicación entre dos nodos.

También se busca observar paquetes de red, identificar direcciones MAC, provocar una falla de conectividad, diagnosticarla y recuperar el funcionamiento normal.

## 3. Topología

Se crearon dos nodos virtuales independientes llamados `hostA` y `hostB`, conectados mediante un par de interfaces VETH.

```text
       Ubuntu Linux
            |
     +------+------+
     |             |
   hostA         hostB
     |             |
   vethA ======= vethB
     |             |
 10.10.1.1     10.10.1.2
    /30           /30
```

Cada namespace posee sus propias interfaces, direcciones IP, rutas y tabla de vecinos.

## 4. Direccionamiento

| Parámetro | Valor |
|---|---|
| Red | 10.10.1.0/30 |
| Máscara | 255.255.255.252 |
| Dirección hostA | 10.10.1.1 |
| Dirección hostB | 10.10.1.2 |
| Broadcast | 10.10.1.3 |
| Direcciones utilizables | 2 |

Ambos nodos pertenecen a la misma subred y pueden comunicarse directamente sin configurar una puerta de enlace.

## 5. Construcción

Se crearon los espacios de nombres:

```bash
sudo ip netns add hostA
sudo ip netns add hostB
ip netns list
```

Posteriormente, se creó el par de interfaces virtuales:

```bash
sudo ip link add vethA type veth peer name vethB
```

Cada interfaz fue asignada al nodo correspondiente:

```bash
sudo ip link set vethA netns hostA
sudo ip link set vethB netns hostB
```

Se configuraron las direcciones IPv4:

```bash
sudo ip netns exec hostA ip addr add 10.10.1.1/30 dev vethA
sudo ip netns exec hostB ip addr add 10.10.1.2/30 dev vethB
```

Finalmente, se activaron las interfaces:

```bash
sudo ip netns exec hostA ip link set lo up
sudo ip netns exec hostA ip link set vethA up
sudo ip netns exec hostB ip link set lo up
sudo ip netns exec hostB ip link set vethB up
```

## 6. Verificación

### Interfaces y direcciones

Se verificó que `vethA` y `vethB` estaban activas y tenían sus respectivas direcciones IPv4.

### Rutas

En ambos nodos apareció automáticamente la ruta hacia la red `10.10.1.0/30`.

No fue necesario configurar rutas adicionales porque existe una conexión directa.

### Vecinos

Mediante `ip neigh` se observaron las siguientes asociaciones:

| Nodo | IP del vecino | Dirección MAC |
|---|---|---|
| hostA | 10.10.1.2 | 66:72:25:84:99:66 |
| hostB | 10.10.1.1 | ce:49:7c:8e:b0:2e |

Las entradas presentaron el estado `STALE`, indicando que la dirección MAC era conocida, aunque su accesibilidad no se había confirmado recientemente.

### Conectividad

Se realizaron pruebas de ping en ambos sentidos:

```bash
sudo ip netns exec hostA ping -c 4 10.10.1.2
sudo ip netns exec hostB ping -c 4 10.10.1.1
```

Los resultados fueron satisfactorios: cuatro paquetes enviados, cuatro recibidos y 0 % de pérdida en ambos sentidos.

## 7. Captura y análisis de tráfico

Se utilizó la herramienta `tcpdump` para observar el tráfico de la interfaz `vethA`.

```bash
sudo ip netns exec hostA tcpdump -n -i vethA
```

La captura registró 14 paquetes, incluyendo tráfico ARP, ICMP e ICMPv6.

Se identificaron solicitudes ICMP Echo Request enviadas desde `hostB` y respuestas ICMP Echo Reply generadas por `hostA`.

También se observaron solicitudes y respuestas ARP para descubrir las direcciones MAC correspondientes a las direcciones IPv4 de los nodos.

La captura finalizó con cero paquetes descartados por el kernel.

## 8. Falla y diagnóstico

Se provocó una falla controlada desactivando la interfaz `vethB`.

```bash
sudo ip netns exec hostB ip link set vethB down
```

**Predicción:** al desactivar uno de los extremos del enlace VETH, la comunicación entre ambos nodos dejaría de funcionar.

La prueba de ping desde `hostA` hacia `hostB` produjo mensajes `Destination Host Unreachable`, con cuatro paquetes enviados, cero recibidos y 100 % de pérdida.

Durante el diagnóstico se observó:

- `vethA` presentaba `NO-CARRIER` y estado `DOWN`.
- La dirección IPv4 `10.10.1.1/30` seguía configurada.
- La ruta hacia `10.10.1.0/30` mostraba `linkdown`.
- La entrada de vecinos correspondiente a `10.10.1.2` mostraba `FAILED`.

La causa fue la desactivación de `vethB`, que interrumpió el enlace virtual.

Para recuperar la conexión se ejecutó:

```bash
sudo ip netns exec hostB ip link set vethB up
```

Posteriormente, un nuevo ping confirmó cuatro paquetes enviados, cuatro recibidos y 0 % de pérdida.

La conectividad se recuperó sin modificar las direcciones IP ni las rutas.

## 9. Uso de OpenCode

**Pendiente:** realizar y documentar una consulta utilizando OpenCode, si corresponde a los requisitos de la actividad.

Durante el desarrollo se utilizó ChatGPT como herramienta de apoyo para comprender comandos, interpretar resultados de terminal y organizar la documentación.

Los resultados técnicos descritos fueron comprobados mediante comandos ejecutados en Ubuntu.

## 10. Conclusiones

Se consiguió implementar una red virtual entre dos namespaces independientes utilizando interfaces VETH.

Las pruebas demostraron que ambos nodos podían comunicarse correctamente dentro de la subred configurada.

La observación de tráfico permitió identificar el funcionamiento de ICMP y ARP. Además, la falla controlada permitió comprender la importancia del estado físico y lógico de las interfaces.

Finalmente, se consiguió recuperar la conectividad y registrar evidencias del trabajo mediante Git y GitHub.

## Evidencias

Los archivos de configuración y verificación están disponibles en la carpeta `evidencias/` del repositorio.

Se incluyen registros de interfaces, direcciones IPv4, tablas de rutas, vecinos, namespaces y la prueba de recuperación.

**Commit de evidencias:** `32aef65`.
