---
title: OpenShift 4 Bare Metal Instalacion - User Provisioned Infraestructure (UPI)
description: Guia de instalacion de OpenShift 4 UPI, donde se explica paso a paso como realizar cada uno de los pasos de la instalacion.  
---

# OpenShift 4 Bare Metal Instalacion - User Provisioned Infraestructure (UPI)

## Guia de instalacion paso a paso de OpenShift 4 UPI

- [OpenShift 4 Bare Metal ](#openshift-4-bare-metal)
    - [Diagrama de arquitectura](#diagrama-de-arquitectura)
    - [Descargas](#descargas)
    - [Instalación y Configuración DNS](#instalacion-y-configuracion-del-dns)
    - [Instalación y Configuración HAProxy](#instalacion-y-configuracion-del-haproxy)
    - [Instalación y Configuración Apache](#instalacion-y-configuracion-del-apache)
    - [Maquinas Virtuales](#maquinas-virtuales)
    - [Instalación del cliente](#instalacion-del-cliente)
    - [Archivo de Instalación](#archivo-de-instalacion)
    - [Creacion de Ignition Files](#creacion-de-ignition-files)
    - [Despliege de OpenShift](#despliegue-de-openshift)

## Diagrama de arquitectura

![Arquitectura](./imagenes/arquitectura.png)

## Descargas

1. Red Hat Enterprise Linux [RHEL](https://access.redhat.com/downloads/content/479/)
1. Ingresar en [Create an OpenShift cluster](https://console.redhat.com/openshift)
1. Seleccionar "Create Cluster" - "Datacenter" - "Bare Metal (x86_64)" - "User-provisioned infrastructure"
1. Descargamos los siguientes archivos
    - OpenShift Installer
    - Pull Secret
    - Cliente de OpenShift (Command line interface)
    - Red Hat Enterprise Linux CoreOs (RHCOS)
        - rhcos-X.X.X-x86_64-metal.x86_64.raw.gz
        - rhcos-X.X.X-x86_64-installer.x86_64.iso

## Instalacion y Configuracion del DNS

Instalamos BIND que sera el servidor de DNS para todas las rutas de OpenShift.

```bash
    dnf install bind bind-utils -y
```

Se debe modificar el archivo /etc/named.conf con las siguientes configuraciones o remplazar el archivo con el que esta en el git.

> *Se deberan considerar las IPs de su Red* 

Se debe agregar la ip del servidor DNS, en este caso se comenta la red de IPv6
```
        listen-on port 53 {127.0.0.1; 10.2.82.1; };
#       listen-on-v6 port 53 { ::1; };
```

Agregamos los DNS de google y se agrega el segmento de red de las maquinas
```
        forwarders      {8.8.8.8; 8.8.4.4;};
        allow-query     { localhost; 10.2.82.0/24;};
```

**Zona Directa** Se debe agregar el archivo *name-directa* en la siguiente ruta "/var/named/"

Configuración en el archivo *named.conf*
```
zone "openshift.segob.gob.mx" IN {
        type master;
        file "directa";
        allow-update { none; };
};
```

Archivo *named-directa*
```
$TTL    86400
@               IN SOA openshift.segob.gob.mx. root (
42              ; serial
3H              ; refresh
15M             ; retry
1W              ; expiry
1D )            ; minimum
                IN NS           ns
;use IP address of named machine for ns
ns              IN A          10.2.82.1
api             IN      A       10.2.82.1 
api-int         IN      A       10.2.82.1 
*.apps          IN      A       10.2.82.1 
bootstrap       IN      A       10.2.82.2 
master0         IN      A       10.2.82.3 
master1         IN      A       10.2.82.4 
master2         IN      A       10.2.82.5 
worker0         IN      A       10.2.82.6 
worker1         IN      A       10.2.82.7 
worker2         IN      A       10.2.82.8
```

**Zona Inversa** Se debe agregar el archivo *name-inversa* en la siguiente ruta "/var/named/"

Configuración en el archivo *named.conf*
```
zone "82.2.10.in-addr.arpa" IN {
        type master;
        file "inversa";
        allow-update { none; };
};
```
Archivo *named-inversa*
```
$TTL    86400
@       IN      SOA     openshift.segob.gob.mx. root.openshift.segob.gob.mx.  (
    1997022700 ; serial
    28800      ; refresh
    14400      ; retry
    3600000    ; expire
    86400 )    ; minimum
@ IN      NS      ns.openshift.segob.gob.mx.
1 IN      PTR     api.openshift.segob.gob.mx. 
1 IN      PTR     api-int.openshift.segob.gob.mx. 
2 IN      PTR     bootstrap.openshift.segob.gob.mx. 
3 IN      PTR     master0.openshift.segob.gob.mx. 
4 IN      PTR     master1.openshift.segob.gob.mx. 
5 IN      PTR     master2.openshift.segob.gob.mx. 
6 IN      PTR     worker0.openshift.segob.gob.mx. 
7 IN      PTR     worker1.openshift.segob.gob.mx. 
8 IN      PTR     worker2.openshift.segob.gob.mx.
```

## Instalacion y Configuracion del HAProxy

Instalamos el balanceador que se encargara de mandar las peticiones a cada maquina de OpenShift.

```bash
   dnf install haproxy -y
```

Se debe modificar el archivo /etc/haproxy/haproxy.conf con las siguiente configuraciones o remplazar el archivo con el que esta en este git. 

> *Se deberan considerar las IPs de su Red* 

**Kubernetes API Server**
 ```
#---------------------------------------------------------------------
# frontend to OpenShift K8s API Server                                
#---------------------------------------------------------------------
frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
#---------------------------------------------------------------------
# backend to OpenShift K8s API Server                                 
#---------------------------------------------------------------------
backend openshift-api-server
    balance source
    mode tcp
#    server bootstrap  10.2.82.2:6443 check
    server master0  10.2.82.3:6443 check
    server master1  10.2.82.4:6443 check
    server master2  10.2.82.5:6443 check
```
**Machine Config Server**
```
#---------------------------------------------------------------------
# frontend to OpenShift Machine Config Server                         
#---------------------------------------------------------------------
frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
#---------------------------------------------------------------------
# backend to OpenShift Machine Config Server                          
#---------------------------------------------------------------------
backend machine-config-server
    balance source
    mode tcp
#    server bootstrap 10.2.82.2:22623 check
    server master0  10.2.82.3:22623 check
    server master1  10.2.82.4:22623 check
    server master2  10.2.82.5:22623 check
```
**HTTP Traffic**
```
frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server worker0 10.2.82.6:80 check
    server worker1 10.2.82.7:80 check
    server worker2 10.2.82.8:80 check
```
**HTTPS Traffic**
```
frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server worker0 10.2.82.6:443 check
    server worker1 10.2.82.7:443 check
    server worker2 10.2.82.8:443 check
 ```

## Servidor Apache

## Maquinas Virtuales

## Instalacion del cliente

## Archivo de Instalacion

## Creacion de Ignition Files

## Despliegue de OpenShift