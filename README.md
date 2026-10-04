# Homelab - Acceso Remoto y Minimo Privilegio

Este repositorio documenta, de forma sanitizada, el modelo de acceso remoto
de un homelab personal: una malla superpuesta con identidad por nodo y sin
puertos entrantes, y un diseno de minimo privilegio especifico para agentes
de IA, con un unico camino de entrada y un interruptor manual.

Es parte de un portfolio tecnico pensado para entrevistas de trabajo. No es
documentacion operativa de un entorno en produccion: es una version
transformada -decisiones, patrones y aprendizajes- de un homelab real, sin
datos que permitan identificarlo o reproducirlo.

Lo que busca demostrar: capacidad de disenar acceso remoto sin exponer
puertos al borde, y un enfoque de minimo privilegio aplicado a asistentes de
IA que participan de tareas de operacion, donde la restriccion vive en la
infraestructura y no en la buena conducta del agente.

## Indice

- [Ficha rapida para quien evalua](contexto.md)
- [Seguridad y modelo de accesos](docs/01-seguridad-y-accesos.md)
- [Caso de estudio: acceso de agentes de IA con minimo privilegio](docs/casos-de-estudio/01-acceso-de-agentes-de-ia-y-minimo-privilegio.md)

## Licencia

Ver [LICENSE.md](LICENSE.md).
