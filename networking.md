#Networking en Docker

En esta entrada veremos el funcionamiento básico del networking en Docker y haremos algunas pruebas para ver el comportamiento de cada tipo de red y sus opciones mas comunes. 

##Default Networks
Cuando instalamos Docker, este crea en el host en cuestion tres redes de forma automatica. Estas son `bridge`, `host` y `none`.
Podemos verlas con el comando `docker network ls`
```
# docker network ls
NETWORK ID          NAME                   DRIVER              SCOPE
6748ebbe9c05        bridge                 bridge              local
edaef368590e        host                   host                local
aa2f9ec2168b        none                   null                local
```
###Default Bridge

La red `bridge` es la red por defecto a la que se conectaran nuestros contenedores salvo que indiquemos otra cosa.
Durante su instalacion Docker creará en nuestro host una inteface virtual de red llamada `docker0` que será usada para hacer una conexion en modo bridge con cada uno de nuestros contenedores. La ip por defecto de `docker0` es 172.17.0.1
```
docker0   Link encap:Ethernet  HWaddr 02:42:CB:6A:66:A9
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:cbff:fe6a:66a9/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:648 (648.0 B)
```
Creamos un nuevo contenedor llamado "bridgenw" y le hacemos un inspect para ver como se comporta:
```
# docker run -tdi --name=bridgenw busybox
1c8a0d8ebea7c089b50347266ead23257f098911e0f58a3e6ec8b8e2dcf5fc08
```
```
# docker inspect bridgenw
...
    {
        "Name": "/bridgenw",
...
			"NetworkSettings": {
            "Bridge": "",
            "Gateway": "172.17.0.1",
            "IPAddress": "172.17.0.3",
            "Networks": {
                "bridge": {
```
En el ejemplo de arriba se han seleccionado solo las lineas que nos insteresan. En ellas podemos ver que como `Gateway` tenemos la IP de la insterface `docker0` de nuestro host. La IP del contenedor está dento del mismo rango. Si crearamos otro contenedor, se le asignaria una IP dentro del mismo rango y abria comunicacion IP entre ellos. Dicho de otro modo, todos los contenedores conectados la misma red `bridge` podran comunicarse entre ellos.

El comando `docker network inspect` nos muestra mas información de la red en cuention así como los contenedores que en ella tenemos conectados:

```
# docker network inspect bridge 
[
    {
        "Name": "bridge",
        "Id": "6748ebbe9c0587cbf99cb7389ae6dd91225d75fdb8936f7d5e94dfdb5ac8cbb5",
        "Created": "2018-01-02T10:31:39.910208787Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "1c8a0d8ebea7c089b50347266ead23257f098911e0f58a3e6ec8b8e2dcf5fc08": {
                "Name": "conbridge",
                "EndpointID": "d5b97c5b7598ea596f1bd9d5893470746ef3735ee8c7505c143b0c98c13f5998",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "1eacbba15707952f2f6bc943fd2ca82d1bb28f138f4679d449ad1b5122033d85": {
                "Name": "hardcore_almeida",
                "EndpointID": "f8479f73d924efcacdcde55db329ca431f08e94892c0bc4c0befb113ef8f173c",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
```

La red `bridge` por defecto no tiene ningún tipo de `service discovery` ni DNS que resuelva los nombre de host o contenedor por lo que la comunicación entre contenedores tiene que ser mediente direccion IP. Si tenemos encuenta que nuestros contenedores pueden cambiar su IP con fecuencia, nos encontramos con el primer gran problema a nivel de networking. En la red default `bridge` estó se solucionaba en parte con el uso del flag `--link` pero es una función obsoleta, y aunque aún funciona, no está recomendado su uso. Mas info [aquí](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/).
Mś adelante veremos un poco más al respecto.

Aunque no es algo común tenemos la opción de deshabilitar la a red default `bridge` editando en archivo `daemon.json` y añadiendo las siguientes lineas:
```
"bridge": "none",
"iptables": "false"
```
###None
La red `none` no tiene mucho misterio. Simplemente la usamos si queremos que el contenedor no tenga funciones de red.
Para usar cualquier red que no sea la default bridge usamos el flag `--network` seguido del nombre de la red a la que queramos conectar nuestro contenedor:
```
# docker run -d --name sinred --network=none ubuntu
```
El comando anterior crea una contenedor llamado "sinred" conectado a la red `none`. Si le hacemos un `inspect` vemos que no tiene dirección ip y que efectivamente esta en la red none
```
# docker inspect sinred
"NetworkSettings": {
            ...
            "IPAddress": "",
            "IPPrefixLen": 0,
            "IPv6Gateway": "",
            "MacAddress": "",
            "Networks": {
                "none": {
            ...
```
###Host
La red ```host``` hace que los conetenedores que conectemos a ella, usen directamente el networking de nuestro host. Esto implica que no habra aislamiento a nivel de red entre el host y los contenedores que corran en el. Si levantamos un contenedor que ejecute un servidor web, este será accesible desde la IP de nuestro host sin necesidad de exponer ningun puerto:
```
# docker run -tdi --network=host nginx 
e40a186725587050a96eaa78bb15a3c8f4c0005f8024f60f1c2ee29fa3f9ef07```
```
# curl 192.168.99.100
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
##Bidge User Defined Networks
En Docker tenemos un segudo tipo de red birdge denominado `user defined` en la que, como su nombre indica, podemos defenir algunos paramatros.
Para crear nuestra propia red bridge usamos el siguiente comando:
```
# docker network create -d bridge my-bridge01
 ```
Ahora creamos dos contenedores y los conectamos a la red recien creada:
```
# docker run -tdi --name=my-doc1 --network=my-bridge01 busybox
cff825243deb32c669ab21005748ce38b5a5ae1636fd0c0caf820aa65b56d309

# docker run -tdi --name=my-doc2 --network=my-bride01 busybox
92b7b9efd7af3d5dcdc24b6b16e96bf028668ca6e9f2d9142f3ec0de5fc1ce25
```
Si hacemos un inspect de la red vemos los detalles de la nueva red y los dos contenedores que tenemos conectados:
```
# docker network inspect my-bridge01
[
    {
        "Name": "my-bridge01",
        "Id": "86a94aba67fde6c9e356bd503f44617ad7f9191610ce1af8ea2ca37470e1461f",
        "Created": "2018-01-04T22:29:17.152346541Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "Gateway": "172.21.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "92b7b9efd7af3d5dcdc24b6b16e96bf028668ca6e9f2d9142f3ec0de5fc1ce25": {
                "Name": "my-doc2",
                "EndpointID": "d60e24b095b49de03e42157dcb73df73342c63cf356659acc156e0a7b9ad7775",
                "MacAddress": "02:42:ac:15:00:03",
                "IPv4Address": "172.21.0.3/16",
                "IPv6Address": ""
            },
            "cff825243deb32c669ab21005748ce38b5a5ae1636fd0c0caf820aa65b56d309": {
                "Name": "my-doc1",
                "EndpointID": "eee52afa6c388e9a97a4523bce74b4ce9469c7cde4b58021a507e51d86973a67",
                "MacAddress": "02:42:ac:15:00:02",
                "IPv4Address": "172.21.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```
Un uso típico de este tipo de red es crear entornos aislados a nivel de red para un conjunto de contenedores. Un ejemplo muy común es la tipica aplicación web donde tenemos un contenedor con la aplicación y otro con una base de datos. Queremos que haya comunicación entre ellos pero no con el restro de nuestros contenedores. Creamos nuesta propia red `bridge user defined` y listo.

Una diferencia importante entre la red default brigde y una red bridge user defined, es que estas últimas ya incopora un servicio de DNS que resuelve nombres de contenedores en direcciones IP. Vamos a conectarnos a uno de los contenedores que creamos en el ejemplo anterior para una prueba:
```
# docker attach con1
/ # ping con2
PING con2 (172.20.0.3): 56 data bytes
64 bytes from 172.20.0.3: seq=0 ttl=64 time=0.128 ms
64 bytes from 172.20.0.3: seq=1 ttl=64 time=0.125 ms
64 bytes from 172.20.0.3: seq=2 ttl=64 time=0.154 ms
64 bytes from 172.20.0.3: seq=3 ttl=64 time=0.145 ms
```
Como vemos llegamos al otro contenedor haciendo ping por su nombre sin necesidad de haber configurado nada y sin necesidad de usar el flag `--link`

##Puertos
Para que nuestros contenedores sean accesibles desde el exterior tendremos que "exponer" sus puertos. Por ejemplo, si tenemos un servidor web y no exponemos el puerto 80, solo podremos acceder a nuestro servidor desde el propio host que ejecuta el contenedor, lo cual es poco útil.

Para exponer un puerto o puertos tenemos dos opciones, el flag `-p` (o `--publish`) y el flag `-P` (o `--publish-all`).

Con `-p` podemos especificar el puerto de nuestro host que expondremos. Para un servidor web lo tipico seria lo siguiente:
```
# docker run -tdi --name=webserver -p 80:80 nginx
```
```
# docker inspect webserver
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "20bc4c533fae0172d277a114b0ce81b59d0ed93580b2a6a53ee0f16b846148c4",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "80/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "80"
                    }
```
En el ejemplo anterior levantamos un contenedor y mapeamos el puerto 80 de nuestro host al puerto 80 del contenedor. Comprobamos con `inspect`.
Ya podriamos acceder al servidor web de nuestro contenedor dirigiendonos a la direccion IP del host que lo ejecuta.

El flag `-P` funciona igual, pero en lugar de elegir nosotros el puerto del host que expondremos, Docker buscará de forma aleatoria un puerto libre (por encima del 30000) para exponer:
```
# docker run -tdi --name=webserver2 -P nginx
```
```
# docker inspect webserver2
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "3e1e82b42d1e39dcdd10b334cabef89e0f3f7462b070613dcff99feb3b0042ee",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "80/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "32769"
                    }
```
Podemos ver con `inspect` que el puerto ahora es el 32769.
Con docker ps tambien podemos ver mas comodamente puertos que expone cada contenedor:
```
docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
74229c5e4f5d        nginx               "nginx -g 'daemon ..."   4 minutes ago       Up 4 minutes        0.0.0.0:32769->80/tcp   webserver2
2ce6db313b0c        nginx               "nginx -g 'daemon ..."   25 minutes ago      Up 25 minutes       0.0.0.0:80->80/tcp      webserver
```








