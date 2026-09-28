# Análisis y Resolución del Sistema DNS

| | |
|---|---|
| **Módulo** | 0375 Servicios de red |
| **Ciclo formativo** | ASIX |
| **Alumno** | Felipe Rodríguez Gómez |

## Objetivo

Instalar y configurar servidores de DNS.

---

# 1. Introducción

### ¿Qué función cumple el servicio DNS en la red y por qué es esencial para el funcionamiento de Internet?

El DNS actúa como la "guía telefónica" de Internet: traduce los nombres de dominio que usamos los humanos (ej. `google.com`) a las direcciones IP que entienden las máquinas. Es esencial porque, sin él, tendríamos que memorizar direcciones IP numéricas para acceder a cualquier servicio, lo cual sería impracticable.

### ¿Cómo se estructura la jerarquía del DNS y qué rol cumple cada nivel?

El DNS se organiza como un árbol invertido con varios niveles:

- **Raíz (`.`)**: es el nivel más alto. Sabe quién gestiona cada dominio de primer nivel.
- **TLD (Top Level Domain)**: extensiones como `.com`, `.org`, `.es`. Saben quién es el servidor autoritativo de cada dominio de segundo nivel.
- **Servidores autoritativos**: contienen los registros finales de un dominio (ej. `google.com`). Son la fuente de verdad de esa zona.
- **Subdominios**: niveles adicionales como `www.google.com` o `mail.google.com`. Cada nivel delega la responsabilidad en el siguiente.

### ¿Cuál es la diferencia entre un servidor DNS iterativo y un servidor recursivo?

- **Iterativo**: responde con la mejor información que conoce. Si no tiene la respuesta final, devuelve una referencia al siguiente servidor al que hay que preguntar. Los servidores raíz y los TLD funcionan así.
- **Recursivo**: hace todo el trabajo por el cliente. Si no tiene la respuesta en su caché, consulta él mismo a los servidores raíz, TLD y autoritativos hasta obtener la respuesta final y se la devuelve ya resuelta. Los resolutores de los ISPs suelen ser recursivos.

### ¿Qué son los registros de recursos (RR) en DNS y cuáles son los más importantes?

Son las unidades de información que componen un archivo de zona. Los más importantes son:

| Registro | Función |
|----------|---------|
| **SOA** | Define el servidor primario de la zona |
| **NS** | Indica los servidores autoritativos |
| **A** | Asocia un nombre a una dirección IPv4 |
| **AAAA** | Asocia un nombre a una dirección IPv6 |
| **CNAME** | Crea un alias hacia otro nombre |
| **MX** | Especifica los servidores de correo |
| **PTR** | Se usa para la resolución inversa (IP → nombre) |
| **TXT** | Almacena información textual (verificaciones, SPF, etc.) |

### ¿Qué vulnerabilidades existen en el servicio DNS y qué mecanismos de seguridad pueden mitigarlas?

| Vulnerabilidad | Descripción | Mitigación |
|----------------|-------------|------------|
| Envenenamiento de caché | Inyección de datos falsos en la caché de un resolutor | DNSSEC, actualización de software |
| Spoofing | Suplantación de la identidad de un servidor DNS | DNSSEC, DNS over TLS/HTTPS (DoT/DoH) |
| Ataques de amplificación | Uso de servidores abiertos para inundar a una víctima (DDoS) | Restringir la recursión (`allow-recursion`) |
| Transferencias no autorizadas | Obtención de una copia completa de la zona | Limitar con `allow-transfer` |

---

# 2. Reporte técnico

## 2.1 Características de hardware y sistema operativo

| Componente | Valor |
|------------|-------|
| **Sistema operativo** | Ubuntu 26.04.1 LTS (Resolute) |
| **RAM** | 4 GB |
| **CPU** | 2 |
| **Disco** | 20 GB |
| **Interfaz NAT** | `enp0s3` → 10.0.2.15/24 (DHCP) |
| **Interfaz Red Interna** | `enp0s8` → 192.168.1.10/24 (estática) |

## 2.2 Desarrollo

### Configuración de red (Netplan)

```bash
sudo more /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
      optional: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.1.10/24
```

Aplicado con:

```bash
sudo netplan apply
```

### Instalación de BIND9

```bash
sudo apt update
sudo apt install -y bind9 bind9-utils bind9-doc bind9-dnsutils
```

### Configuración global (`/etc/bind/named.conf.options`)

```conf
options {
    directory "/var/cache/bind";

    listen-on { any; };
    listen-on-v6 { none; };

    allow-query { localhost; 192.168.1.0/24; 192.168.56.0/24; };
    allow-recursion { localhost; 192.168.1.0/24; 192.168.56.0/24; };
    allow-transfer { none; };

    forwarders {
        8.8.8.8;
        1.1.1.1;
    };

    dnssec-validation auto;
    querylog yes;
};
```

### Declaración de zonas (`/etc/bind/named.conf.local`)

```conf
zone "tudominio.local" {
    type master;
    file "/etc/bind/zones/db.tudominio.local";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

### Archivos de zona

**Zona directa** (`/etc/bind/zones/db.tudominio.local`)

```dns
$TTL 86400
@   IN  SOA ns1.tudominio.local. admin.tudominio.local. (
        2024092801  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Negative Cache TTL
)
;
@           IN  NS      ns1.tudominio.local.
;
ns1         IN  A       192.168.1.10
servidor1   IN  A       192.168.1.20
pc-cliente  IN  A       192.168.1.30
;
www         IN  CNAME   servidor1.tudominio.local.
;
@           IN  MX  10  mail.tudominio.local.
mail        IN  A       192.168.1.40
```

**Zona inversa** (`/etc/bind/zones/db.192.168.1`)

```dns
$TTL 86400
@   IN  SOA ns1.tudominio.local. admin.tudominio.local. (
        2024092801  ; Serial
        3600
        1800
        604800
        86400
)
;
@   IN  NS  ns1.tudominio.local.
;
10  IN  PTR ns1.tudominio.local.
20  IN  PTR servidor1.tudominio.local.
30  IN  PTR pc-cliente.tudominio.local.
```

### Validación de sintaxis

```console
$ sudo named-checkconf
$ sudo named-checkzone tudominio.local /etc/bind/zones/db.tudominio.local
zone tudominio.local/IN: loaded serial 2024092801
OK
$ sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1
zone 1.168.192.in-addr.arpa/IN: loaded serial 2024092801
OK
```

### Pruebas de resolución

**Consulta directa (registro A)**

```console
$ dig @localhost servidor1.tudominio.local

; <<>> DiG 9.20.24-1ubuntu0.3-Ubuntu <<>> @localhost servidor1.tudominio.local
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2828
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;servidor1.tudominio.local.     IN      A

;; ANSWER SECTION:
servidor1.tudominio.local. 86400 IN     A       192.168.1.20

;; Query time: 2 msec
;; SERVER: 127.0.0.1#53(localhost) (UDP)
;; WHEN: Mon Sep 28 18:27:52 UTC 2026
;; MSG SIZE  rcvd: 98
```

**Consulta inversa (registro PTR)**

```console
$ dig @localhost -x 192.168.1.20

; <<>> DiG 9.20.24-1ubuntu0.3-Ubuntu <<>> @localhost -x 192.168.1.20
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 36362
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;20.1.168.192.in-addr.arpa.     IN      PTR

;; ANSWER SECTION:
20.1.168.192.in-addr.arpa. 86400 IN     PTR     servidor1.tudominio.local.

;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(localhost) (UDP)
;; WHEN: Mon Sep 28 18:27:59 UTC 2026
;; MSG SIZE  rcvd: 121
```

**Consulta de alias (CNAME)**

```console
$ dig @localhost www.tudominio.local

; <<>> DiG 9.20.24-1ubuntu0.3-Ubuntu <<>> @localhost www.tudominio.local
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17722
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;www.tudominio.local.           IN      A

;; ANSWER SECTION:
www.tudominio.local.    86400   IN      CNAME   servidor1.tudominio.local.
servidor1.tudominio.local. 86400 IN     A       192.168.1.20

;; Query time: 2 msec
;; SERVER: 127.0.0.1#53(localhost) (UDP)
;; WHEN: Mon Sep 28 18:28:09 UTC 2026
;; MSG SIZE  rcvd: 116
```

## 2.3 Análisis de incidencias técnicas

**Aviso `.local`:** al hacer `dig` aparecía el mensaje `WARNING: .local is reserved for Multicast DNS`. No es un error, pero es una mala práctica en entornos reales.

## 2.4 Conclusiones

He dejado un servidor DNS funcional con BIND9 sobre Ubuntu Server, con zonas directa e inversa configuradas, y registros A, CNAME, MX y PTR verificados mediante `dig`. He aprendido a configurar direcciones IP estáticas con Netplan en escenarios con varios adaptadores de red, a distinguir los tipos de red de VirtualBox (NAT, red interna y Host-Only) y a diagnosticar problemas de resolución con herramientas como `dig`, `ss` y `ufw`.

---

# 3. Manual de usuario

## Archivos de configuración

| Archivo | Función |
|---------|---------|
| `/etc/bind/named.conf.options` | Configuración global del servidor |
| `/etc/bind/named.conf.local` | Declaración de las zonas |
| `/etc/bind/zones/db.tudominio.local` | Zona directa (nombre a IP) |
| `/etc/bind/zones/db.192.168.1` | Zona inversa (IP a nombre) |

## Configuración global

En `named.conf.options` se define:

- `listen-on { any; }` para escuchar en todas las interfaces.
- `allow-query` y `allow-recursion` limitados a localhost y a `192.168.1.0/24`.
- `allow-transfer { none; }` para no permitir transferencias de zona.
- `forwarders` apuntando a `8.8.8.8` y `1.1.1.1` para consultas externas.
- `dnssec-validation auto` y `querylog yes`.

## Declaración de zonas

En `named.conf.local` se declaran:

- **Zona directa:** `tudominio.local` → archivo `db.tudominio.local`.
- **Zona inversa:** `1.168.192.in-addr.arpa` → archivo `db.192.168.1`.

## Contenido de las zonas

- **Zona directa:** SOA con serial `2024092801`, NS `ns1.tudominio.local`, registros A para `ns1` (192.168.1.10), `servidor1` (192.168.1.20) y `pc-cliente` (192.168.1.30), un CNAME para `www` hacia `servidor1`, y un MX con su registro A para `mail` (192.168.1.40).
- **Zona inversa:** SOA con el mismo serial, NS `ns1.tudominio.local` y los PTR de las IPs **10, 20 y 30**.

## Añadir un host

1. Editar `db.tudominio.local` y añadir una línea tipo:
```dns
   nuevo-host  IN  A  192.168.1.50
```
2. Subir el serial del SOA, por ejemplo a `2024092802`.
3. Si se quiere resolución inversa, añadir el PTR en `db.192.168.1`.
4. Validar y reiniciar.

## Validar y reiniciar

```bash
sudo named-checkconf
sudo named-checkzone tudominio.local /etc/bind/zones/db.tudominio.local
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1
sudo systemctl restart bind9
```

> Si los tres primeros comandos devuelven `OK`, se puede reiniciar sin problema.

## Comandos útiles

| Comando | Función |
|---------|---------|
| `sudo systemctl status bind9` | Ver estado |
| `sudo systemctl restart bind9` | Reiniciar |
| `sudo named-checkconf` | Validar configuración |
| `sudo named-checkzone zona archivo` | Validar una zona |
| `dig @localhost nombre.dominio` | Consulta directa |
| `dig @localhost -x IP` | Consulta inversa |
| `sudo journalctl -u bind9 -f` | Ver logs |

## Diagnóstico rápido

| Problema | Solución |
|----------|----------|
| No arranca | Revisar con `sudo named-checkconf` |
| No resuelve | Revisar zonas y validarlas con `named-checkzone` |
| No responde a la red | Abrir el puerto 53: `sudo ufw allow 53/udp` y `sudo ufw allow 53/tcp` |
| Falla la inversa | Comprobar que la zona sea la IP invertida + `.in-addr.arpa` |

## Seguridad

- Restringir `allow-query` a la red local.
- `allow-transfer { none; }` para no permitir transferencias.
- No permitir recursión abierta.
- Mantener el sistema actualizado.

---

# Enlaces

1. Instalar BIND9: <https://www.isc.org/bind/>
2. Configurar las IP con Netplan: <https://netplan.io>
3. Guía de referencia: <https://punkymo.gitbook.io>
