# Seguridad y Modelo de Accesos

> Estado descrito: septiembre de 2026.

## Proposito

Documentar el enfoque de acceso remoto y de minimo privilegio del homelab
sin exponer detalles sensibles.

## Modelo de seguridad

- segmentacion por rol
- minimo privilegio, dimensionado midiendo el uso real
- administracion no expuesta publicamente
- **ningun puerto entrante en el borde**
- acceso automatizado separado del acceso del operador
- excepciones documentadas cuando un flujo entre zonas es necesario
- **todo control se verifica haciendolo fallar**
- separacion entre documentacion privada y publica

## Que no se publica

- claves, secretos, tokens y credenciales
- endpoints reales de administracion
- configuracion de la malla, del firewall, del SIEM o del monitoreo
- rutas internas de backups o logs
- destinos y detalles de la copia fuera del sitio

## Acceso administrativo

- SSH **solo por clave** en los hosts de infraestructura; la autenticacion
  por contrasena esta deshabilitada y verificada contra la configuracion
  efectiva, no contra el archivo escrito
- rescate por consola del hipervisor verificado, para no depender de la red
- sin publicacion a Internet de ninguna interfaz administrativa
- cuentas en desuso retiradas, con el orden: reemplazo, prueba, retiro

## Acceso a aplicaciones

- servicios internos por nombre, a traves de un proxy inverso
- las aplicaciones propias escuchan solo en la interfaz local de su host y se
  alcanzan por tunel; nunca se publican directamente

## Acceso remoto por malla superpuesta

**El modelo cambio en 2026.**

| Antes | Ahora |
|---|---|
| Tunel punto a punto con concentrador propio | **Malla superpuesta con identidad por nodo** |
| Un puerto entrante publicado en el router de borde | **Ningun puerto entrante**: cada nodo sale hacia el plano de control |
| Una zona de red dedicada | Una capa por encima del direccionamiento, sin zona propia |
| Quien entra al tunel, entra a la red | **Politica como codigo**, denegacion por defecto, permisos por puerto |

**Por que se cambio:** el modelo anterior obligaba a abrir el borde, que es
lo que el resto del diseno evita. El concentrador se dio de baja y su
segmento dejo de anunciarse.

Decisiones del modelo actual:

- **rutas por host, no la subred completa.** Anunciar la subred vuelve el
  entorno inalcanzable desde redes ajenas que usan el mismo rango privado,
  que es el caso mas comun
- la puerta de enlace domestica no se anuncia
- un nodo sirve de salida a internet para redes no confiables
- **la lista de rutas se reemplaza entera en cada cambio**, y la salida a
  internet vive en esa lista: reanunciar sin incluirla la desactiva sin
  error ni alerta. Paso una vez y quedo como aviso escrito
- **bloqueo de la malla con nodos firmantes**: un equipo nuevo no entra
  aunque tenga credenciales validas si un nodo firmante no lo autoriza
- la publicacion de servicios hacia internet que ofrece la malla esta
  prohibida
- las claves de nodo vencen; el vencimiento esta registrado con fecha para
  que no caduquen todas juntas
- una auditoria periodica del estado de la malla emite una metrica, y la
  alerta avisa si **deja de correr**

## Hardening por rol

Un hardening generico no sirve para todos los hosts.

| Tipo de host | Enfoque |
|---|---|
| DNS interno | puertos de resolucion y administracion minima |
| NAS / storage | shares y paneles segun origen |
| SIEM | solo los puertos del dashboard y de los agentes |
| Host de contenedores | tratamiento especial: un firewall generico rompe la red de contenedores y el proxy |
| Hipervisor | control fino de firewall, reenvio y transito |
| Puerta de la malla | reenvia trafico por diseno; su control es la politica de la malla |

**Hallazgo real:** al relevar el filtrado para instalar una herramienta
nueva, la mayoria de los hosts ya filtraban y no estaba documentado. El plan
cambio: se completa el filtrado donde falta, con la herramienta que cada
host ya usa.

## Acceso de automatizacion y de agentes de IA

Las cuentas automatizadas, incluidos los asistentes de IA, **no comparten el
modelo de acceso de la persona**. Apagar uno no debe apagar el otro.

| Propiedad | Como se resuelve |
|---|---|
| Un solo camino | todo el acceso automatizado pasa por un host de salto dedicado |
| Interruptor manual | ese host **no arranca solo**: lo enciende el operador desde el hipervisor |
| Credenciales confinadas | las credenciales hacia el resto viven solo dentro del host de salto |
| Restriccion por origen | una credencial filtrada no sirve desde otro lugar |
| Sin reenvio | ni de puertos ni de agente |
| Trazabilidad por cuenta | los eventos van al SIEM distinguiendo que cuenta hizo cada cosa |
| Acceso de solo lectura al codigo | el agente lee el remoto de codigo con un token de solo lectura que vive en el host de salto; la escritura esta probada como rechazada |
| Sin via de emergencia | decision explicita: si el host esta apagado, se pide encenderlo |

### Lectura privilegiada sin escritura

La mayor parte del trabajo util de un agente es **leer**. Autorizar un
lector generico con privilegio equivale a dar acceso total, porque permite
leer el archivo de contrasenas. Se resuelve con un envoltorio propio de solo
lectura, con lista de exclusion para material critico, que normaliza rutas e
inspecciona contenido en vez de confiar en el nombre del archivo.

**Ningun host tiene un permiso privilegiado generico**, tampoco el de salto.
La lista de comandos permitidos salio de medir que se invocaba realmente.
Detalle en el caso de estudio de este repositorio.

## Verificacion de controles

Un control que no se prueba es una suposicion documentada. Todo script de
seguridad comprueba **lo que tiene que fallar**:

- el envoltorio de lectura verifica que deniega el archivo de contrasenas,
  una clave privada y una ruta que intenta evadirlo
- la reduccion de privilegios verifica que un lector generico y un
  interprete quedan denegados
- la rotacion verifica que la credencial vieja **deja de funcionar**
- el endurecimiento de SSH consulta la configuracion efectiva, no el
  archivo

## Riesgos conocidos

| Riesgo | Mitigacion actual | Pendiente |
|---|---|---|
| DNS con un solo resolver | resolver central monitoreado | segundo resolver |
| Filtrado por host incompleto | aplicado en la mayoria de los hosts | completar los restantes |
| Aplicaciones propias sin autenticacion propia | solo alcanzables por tunel | agregar autenticacion |
| Endurecimiento de los equipos del operador | claves por equipo y bloqueo de la malla | completar el endurecimiento |

## Idea central

La seguridad del acceso remoto de este homelab no se apoya en una
herramienta. Se apoya en: ninguna exposicion entrante, segmentacion y minimo
privilegio medido, automatizacion separada del operador, controles probados
haciendolos fallar, y honestidad sobre lo que sigue abierto.
