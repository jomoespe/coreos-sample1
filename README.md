CoreOS sample1
==============

## Introducción

Prueba de concepto de montaje de un cluster [CoreOS](https://coreos.com/) en [Digital Ocean](https://www.digitalocean.com).

Partiendo de la presentación del [Codemotion 2015](http://slides.com/luismartinezdebartolome/deck) y el tutorial [Getting Started with CoreOS de Digital Ocean](https://www.digitalocean.com/community/tutorial_series/getting-started-with-coreos-2).


## Objetivo

Montar un cluster de 4 nodos. En tres de ellos se desplegará un servicio REST básico, y en el cuarto de ellos un servidor NGinX que actue como balanceador de carga.

El servicio REST es el PoC de [contenedor con servicio HTTP/2](https://github.com/jomoespe/docker-nano-container). El proyecto está disponible en DockerHub como **jomoespe/nano-container**.


## Creación del cluster

### Generar una nueva discovery URL

Acceder a  [https://discovery.etcd.io/new](https://discovery.etcd.io/new) y recuperar el valor devuelto, que es una URL única.

    curl -W "\n" "https://discovery.etcd.io/new"
    
    https://discovery.etcd.io/<discovery id>


Cuando tenemos un cluster ya activo, si necesitamos recuperar el id del cluster_

    $ grep DISCOVERY /run/systemd/system/etcd2.serviced/20-cloudint.conf


### Crear el fichero cloud-config

Crear un fichero [cloud-config](./cloud-config), que posteriormente utilizaremos en la creación de los Droplets de Digital Ocean.

    #cloud-config
    
    coreos:
      etcd2:
        # Recuperar la URL generada de https://discovery.etcd.io/new
        discovery: https://discovery.etcd.io/<discovery_id>
        advertise-client-urls: http://$private_ipv4:2379,http://$private_ipv4:4001
        initial-advertise-peer-urls: http://$private_ipv4:2380
        listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
        listen-peer-urls: http://$private_ipv4:2380
      units:
        - name: etcd2.service
          command: start
        - name: fleet.service
          command: start


### Crear los Droplets en DigitalOcean

Se crear los nodos de manera normal, teniendo en cuenta los siguientes detalles:

  - Para las imágenes de CoreOS hay que tener una SSH Key registrada,
  - La imagen ha de ser CoreOS (Obvio!).
  - Todos los nodos han de estar en el **mismo *data center region* **.
  - Ha de configurarse **private networking**.
  - Hay que seleccionar **user-data**, y en el contenido de user-data añadir el contenido del fichero **cloud-config**.


## Administración del cluster 

Una vez creadas las máquinas se ha de administrar las máquinas, desplegar servicios, etc.

### Acceder

Se accede al cluster conectándose vía SSH a cualquiera de los nodos que lo componen.

    $ ssh -A core@<node_ip>

Una vez dentro, por medio de [fleet](https://coreos.com/using-coreos/clustering/) manejaremos el cluster.


Listar las máquinas del cluster:

    $ fleetctl list-machines

Listar las unidades del cluster:

    $ fleetctl list-units


### Creación de servicios

Por cada servicio que se vaya a manejar hay que crear un fichero de configuración de systemd. Dicho servicio se encargará **fleet** de administrarlos y distribuirlo por los nodos del cluster.

En nuestro ejemplo creamos el fichero [nano-container@.service](./nano-container@.service), en el cual se definirá el ciclo de vida del contenedor **nano-container**.


### Ciclo de vida de un servicio 

  1. Enviar (*submit*) el servicio al cluster
  2. Cargar (*load*) el servicio
  3. Iniciar (*start*) 
  4. Parar (*stop*) 
  5. Descargar (*unload*) 
  5. Destruir (*destroy*) 

Ejemplo, para nuestro servicio, para una instancia:

    $ fleetctl load nano-container@1
    $ fleetctl submit nano-container@1
    $ fleetctl start nano-container@1
    $ fleetctl stop nano-container@1
    $ fleetctl unload nano-container@1
    $ fleetctl destroy nano-container@1


Para arrancar diferentes instancias en diferentes nodos del cluster hay que nombrar el servicio con un nombre diferente. Fleet se encargará de distribuir cada una de las instancias del servicio por los nodos del cluster. Los pasos de cartga, envío y arranque se pueden llevar a cabo en un único paso al hacer un **start**.

    $ fleetctl start nano-container@1
    $ fleetctl start nano-container@2
    $ fleetctl start nano-container@3


Se arrancarán tres instancias del servicio, cada una de las ellas en un nodo del cluster. Se podrá acceder a ellos por medio de la url **https://<node_public_ip>:8443/**



## Notas

Está pendiente ver como configurar el reverse proxy con load balancing, con NGinX.  

  - [NGinX Load balancing](http://nginx.org/en/docs/http/load_balancing.html)
  - [NGinX Reverse proxy](https://www.nginx.com/resources/admin-guide/reverse-proxy/)
  - Arrancar el contenedor de NGinX (esto está funcionando con los servicios desplegados actualmente en DigitalOcean). Falta ver [como reconfigurarlos](https://www.digitalocean.com/community/tutorials/how-to-use-confd-and-etcd-to-dynamically-reconfigure-services-in-coreos):  

    docker run --rm --name balancer -p 443:443 -p 80:80 -v /home/jmoreno/projects/lab/coreos/sample1:/etc/nginx:ro nginx


