# Objetivo 2: Escaneo de Red (Nmap)

Para este objetivo tenía que usar Nmap para escanear un objetivo autorizado. En vez de usar un programa de bug bounty de Bugcrowd, elegí escanear **scanme.nmap.org**

## Preparación del entorno

Usé una máquina virtual con Linux (Ubuntu), donde primero verifiqué que tenía conexión al objetivo:

```bash
ping -c 4 scanme.nmap.org
```

La respuesta fue exitosa, con 0% de paquetes perdidos, confirmando que había conectividad hacia el servidor.

Como la VM no tenía Nmap instalado de entrada, lo instalé con:

```bash
sudo apt install nmap
```
<img width="1112" height="560" alt="image" src="https://github.com/user-attachments/assets/8d3c122e-a14e-4d8f-ac10-e2dcc26dac9d" />

## Comando ejecutado

Una vez instalado, corrí un escaneo básico con detección de versión de servicios:

```bash
nmap -sV scanme.nmap.org
```
<img width="988" height="310" alt="image" src="https://github.com/user-attachments/assets/c4985e3f-193f-4586-9055-c4efb6fa106f" />

La flag `-sV` hace que Nmap, además de detectar qué puertos están abiertos, intente identificar qué servicio y versión específica está corriendo en cada uno.

## Resultados obtenidos

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-11-12 17:38 -03
Nmap scan report for scanme.nmap.org (45.33.32.156)
Host is up (0.37s latency).
Other addresses for scanme.nmap.org (not scanned): 2600:3c01::f03c:91ff:fe18:bb2f
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       Apache httpd 2.4.7 ((Ubuntu))
9929/tcp open  nping-echo Nping echo
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 49.45 seconds
```

## Interpretación de los resultados

El escaneo encontró tres puertos abiertos:

- **Puerto 22/tcp — SSH (OpenSSH 6.6.1p1):** permite el acceso remoto seguro a la terminal del servidor. La versión detectada corresponde a una distribución Ubuntu bastante antigua, sería una señal para investigar si existen vulnerabilidades a esa versión.
- **Puerto 80/tcp — HTTP (Apache httpd 2.4.7):** indica que el servidor corre una página web sin cifrar, sobre Apache. 
- **Puerto 9929/tcp — nping-echo:** este es un servicio propio de las herramientas de Nmap (Nping), pensado justamente para que la gente practique y pruebe la herramienta contra él, lo cual tiene sentido dado que este servidor está armado específicamente para testing.

Nmap también detectó que el sistema operativo del host es Linux, en base a las características de la respuesta de red.

## Conclusión

Este ejercicio me sirvió para entender cómo un escaneo básico con Nmap ya aporta información valiosa en la fase de reconocimiento de un pentest: con un solo comando pude identificar los puertos abiertos, los servicios específicos corriendo en cada uno, sus versiones exactas, y hasta el sistema operativo del objetivo. Toda esta información es la base para decidir qué pruebas o ataques específicos tendrían sentido investigar después, dependiendo de las versiones de software encontradas.
