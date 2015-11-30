CoreOS sample1
==============

Probando CoreOS.

Partiendo de la presentación del [Codemotion 2015](http://slides.com/luismartinezdebartolome/deck)

La idea es montar N nodos en un cluster en los cuales desplegar el contenedor [jomoespe/nano-container](https://hub.docker.com/r/jomoespe/nano-container/). La manera de ejecutar este contenedor será: docker run --rm -ti -p 8443:8443 jomoespe/nano-container


Si todo va bien, a este contenedor se accederá con https://<ip>:8443/
