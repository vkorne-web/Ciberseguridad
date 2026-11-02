# Informe de Auditoría de Red Wi-Fi Insegura

**Autor:** Auditor de Seguridad Junior
**Fecha:** 2026
**Sitio analizado:** `http://neverssl.com`
**Clasificación:** Práctica educativa — Uso responsable

---

## 1. Introducción

Actualmente la mayor parte del tráfico web de un usuario promedio circula por redes
inalámbricas (hogar, café, aeropuerto, coworking, transporte público). En estos
entornos **no controlamos la red**: cualquier dispositivo conectado al mismo punto de
acceso (AP) podría estar en condiciones de observar, manipular o registrar nuestro
tráfico (ataques *Man-in-the-Middle* — MitM —, *sniffing* o *evil twin*).

La diferencia fundamental entre **HTTP** y **HTTPS** es el cifrado. HTTP viaja en texto
plano (*cleartext*) sobre el puerto 80, por lo que cualquier intermediario en la ruta
puede leer el contenido íntegro. HTTPS incorpora TLS (puerto 443) y cifra la
comunicación, garantizando **confidencialidad**, **integridad** y **autenticación** del
servidor mediante certificados.

El objetivo de este informe es **documentar técnicamente** qué información queda
expuesta al navegar por HTTP desde una red Wi-Fi pública y **demostrar cómo una VPN
mitiga** ese riesgo. Para ello analizamos el sitio `http://neverssl.com`, cuyo
propósito explícito es ser un sitio **que nunca redirige a HTTPS**, lo que lo convierte
en un caso ideal para observar tráfico sin cifrar.

---

## 2. Sitio analizado

**URL:** `http://neverssl.com`
**Motivo de elección:** es un sitio diseñado para *no* usar HTTPS, usado habitualmente
para *captive portals*. Permite observar directamente el comportamiento de una
conexión HTTP pura.

Las herramientas utilizadas para recoger la evidencia fueron:

- Navegador (F12 → pestaña **Network / Red** → Recargar → seleccionar la primera solicitud).
- Verificación complementaria con `curl -v` para inspeccionar los headers a nivel de protocolo.

---

## 3. Evidencia observada

### 3.1 Datos de la primera solicitud

| Campo | Valor observado |
|---|---|
| **URL solicitada** | `http://neverssl.com/` |
| **Método HTTP** | `GET` |
| **Host** | `neverssl.com` |
| **Protocolo** | `HTTP/1.1` (sin TLS — puerto 80) |
| **IP resuelta** | `34.223.124.45` (IPv4) / `2600:1f13:...:736e` (IPv6) |
| **Código de respuesta** | `200 OK` |

### 3.2 Headers enviados por el cliente (petición)

```http
GET / HTTP/1.1
Host: neverssl.com
User-Agent: curl/8.13.0
Accept: */*
```

> En el navegador, el header `User-Agent` revela además el sistema operativo, el motor
> del navegador y su versión, por ejemplo:
> `Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/xxx Safari/537.36`.

### 3.3 Headers devueltos por el servidor (respuesta)

```http
HTTP/1.1 200 OK
Date: Mon, 02 Nov 2026 22:11:17 GMT
Server: Apache/2.4.68 ()
Upgrade: h2,h2c
Connection: Upgrade
Last-Modified: Wed, 29 Jun 2022 00:23:33 GMT
ETag: "f79-5e28b29d38e93"
Accept-Ranges: bytes
Content-Length: 3961
Vary: Accept-Encoding
Content-Type: text/html; charset=UTF-8
```

### 3.4 Interpretación de la evidencia

- La solicitud **no incluye** `https://` ni ningún handshake TLS: el cuerpo viaja en claro.
- El header `Server: Apache/2.4.68` **expone la tecnología y versión del servidor**
  (útil para que un atacante busque exploits conocidos — *information leakage*).
- El `User-Agent` **identifica al dispositivo del usuario** (SO, navegador, versión).
- La URL completa, el método y el `Host` permiten reconstruir la navegación sin esfuerzo.
- Como no hay cifrado, **todo este contenido es legible con un simple `tcpdump`/Wireshark**
  capturado en la misma red Wi-Fi.

---

## 4. Riesgos encontrados

### Pregunta 1 — ¿Qué protocolo utiliza el sitio?

Neverssl utiliza **HTTP** (no HTTPS). La conexión se realiza sobre el **puerto 80** y el
esquema de la URL es `http://`. No existe capa TLS, por lo que el tráfico circula en
**texto plano**.

### Pregunta 2 — ¿Qué información puede observarse durante la solicitud?

Al menos la siguiente información es visible sin privilegios especiales:

1. **Host / dominio** → `neverssl.com` (y su IP).
2. **URL completa** → la ruta solicitada (`/`, query strings incluidos).
3. **Método HTTP** → `GET`.
4. **Headers** → `User-Agent`, `Accept`, `Server`, `Content-Type`, etc.
5. **Contenido del cuerpo** y de la respuesta (a diferencia de HTTPS, aquí es legible).

### Pregunta 3 — ¿Qué riesgos existen al navegar por HTTP en una Wi-Fi pública?

Un atacante conectado a la misma red (o controlando el punto de acceso) puede:

- **Interceptar (sniffing):** leer credenciales, cookies de sesión, formularios,
  números de tarjeta y todo el contenido HTTP.
- **Man-in-the-Middle (MitM):** insertarse en la comunicación y alterar las respuestas
  (inyectar publicidad, malware o phishing).
- **Robo de sesión (*session hijacking*):** al capturar cookies no cifradas.
- **Evil Twin:** montar un AP falso con el mismo SSID para capturar todo el tráfico.
- **Reconocimiento:** inferir SO, navegador y servidores a partir de los headers.
- **Rastreo pasivo:** registrar a qué sitios navega y por cuánto tiempo.

---

## 5. Cómo ayuda una VPN

Una **VPN (Virtual Private Network)** crea un **túnel cifrado** entre el dispositivo del
usuario y el servidor VPN antes de que el tráfico salga a Internet:

| Aspecto | Sin VPN (HTTP) | Con VPN (HTTP sobre túnel) |
|---|---|---|
| **Cifrado** | Ninguno: texto plano | El VPN cifra todo el tráfico (IPSec/WireGuard/OpenVPN) |
| **Túnel seguro** | No existe | Se establece un túnel extremo a extremo hasta el VPN |
| **Protección del tráfico** | Cualquiera en la Wi-Fi puede leerlo | El atacante solo ve paquetes cifrados ilegibles |
| **Privacidad** | Se expone IP real, dominio y contenido | Se oculta la IP real; el ISP ve solo el tráfico al VPN |
| **Integridad** | Alterable por MitM | Protegida por el cifrado y autenticación del túnel |

En el escenario de `neverssl.com`, incluso si el sitio sigue siendo HTTP, **la VPN envuelve
esa conexión dentro del túnel cifrado**: un atacante en la misma Wi-Fi vería únicamente
tráfico cifrado hacia el servidor VPN, sin poder identificar el host visitado ni leer el
contenido. La VPN **no reemplaza** a HTTPS, pero reduce drásticamente la superficie de
exposición en redes no confiables.

---

## 6. Mis 3 Reglas de Oro para redes Wi-Fi públicas

1. **Nunca uses HTTP para datos sensibles** (contraseñas, banca, correo). Verifica que
   el candado y `https://` estén presentes antes de introducir cualquier credencial.
2. **Usa siempre una VPN** al conectarte a una red pública (o evita redes abiertas),
   para cifrar todo el tráfico y proteger tu privacidad frente a sniffers y Evil Twins.
3. **Desactiva la conexión automática a Wi-Fi y la compartición de archivos**, y no
   confíes en redes con nombres sospechosos o clones del original hasta verificarlas.

---

## 7. Conclusión

La evidencia demuestra que en una conexión HTTP existe una **exposición completa** del
tráfico a nivel de red: host, URL, método, headers y contenido son legibles por un
tercero. En una Wi-Fi pública esto constituye un riesgo real de robo de credenciales y
de sesión. El uso de **HTTPS** es la protección base e imprescindible, y **una VPN** añade
una capa de confidencialidad sobre toda la conexión, siendo recomendable su uso en
cualquier red no confiable.

---

*Informe elaborado con fines exclusivamente educativos como parte de una práctica de
auditoría de seguridad junior.*
