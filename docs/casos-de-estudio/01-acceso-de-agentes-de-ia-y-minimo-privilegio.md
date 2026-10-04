# Caso de Estudio - Acceso de Agentes de IA con Minimo Privilegio

## Contexto

Un asistente de IA participaba de la operacion de una infraestructura
productiva personal: diagnostico, preparacion de cambios, revision de
configuracion y documentacion. Para hacerlo necesitaba llegar a los hosts.

El planteo inicial era el habitual: darle una clave por host, con permisos
acotados. Funcionaba, y tenia un problema que no se ve hasta que se lo
nombra.

## Problema

**Un agente con claves en la estacion de trabajo del operador es, en la
practica, el operador.** Las claves viven en el mismo disco, se usan desde el
mismo lugar y tienen el mismo alcance. La distincion entre "lo hizo la
persona" y "lo hizo el asistente" queda en la confianza, no en el diseno.

Tres preguntas que el modelo anterior no respondia:

- Como se corta el acceso del agente **sin cortar el del operador**?
- Como se distingue en la evidencia quien hizo cada cosa?
- Que pasa si el material del agente se copia fuera del entorno controlado?

## Decision

Se reemplazo el acceso directo por **un unico camino, con un interruptor
fisico**.

| Decision | Motivo |
|---|---|
| Un host de salto dedicado, y ningun acceso directo | El agente entra por un solo lugar o no entra |
| El host de salto **no arranca solo**: lo enciende el operador | El interruptor vive en el hipervisor, fuera del alcance del agente |
| Las credenciales hacia el resto **viven solo dentro** del host de salto | Si se copian afuera, apagar el host deja de significar algo |
| Restriccion por origen en cada destino | Aun filtrada, la credencial no sirve desde otro lugar |
| **Sin reenvio de puertos ni de agente** en el host de salto | Con reenvio, la credencial vuelve a la estacion del operador y el interruptor pierde sentido |
| Un agente de recoleccion de evidencia dentro del host de salto | Los eventos salen hacia el SIEM en el momento, por usuario |
| **Sin via de emergencia** para el agente | Decision explicita: si el host de salto esta apagado, se pide encenderlo, no se busca otro camino |

El cuarto punto es el que sostiene todo. **Sin el, las tres primeras
decisiones son decorativas.**

## Lectura privilegiada sin privilegio de escritura

El modelo tenia un costo practico: casi todo el trabajo util del asistente
es **leer** -configuracion efectiva, estado de servicios, registros-, y un
permiso acotado a comandos concretos genera idas y vueltas constantes por
cosas inofensivas.

Se resolvio con **un envoltorio de solo lectura**: permite leer casi todo el
host con privilegio, y **deniega el material critico** -archivo de
contrasenas, claves privadas, archivos de entorno, estado del tunel, base
del gestor de contrasenas, claves de agentes del SIEM-.

Dos detalles hacen que sirva:

- **Normaliza la ruta antes de decidir**, asi que atajos como `..` o
  enlaces simbolicos no lo evaden.
- **Mira el contenido**, no solo el nombre: un archivo que parece inocente
  pero contiene una clave privada se deniega igual.

Y una regla de diseno que no es obvia: **no se implementa como "permitir el
comando de lectura"**. Autorizar un lector generico con privilegio es
equivalente a dar acceso total, porque permite leer el archivo de
contrasenas. El envoltorio es un programa propio con su propia lista de
exclusion.

## El hallazgo incomodo

Al medir el modelo ya construido aparecio algo que la documentacion no
decia: **dentro del host de salto, los agentes tenian privilegio total sin
restriccion.** No la lista acotada que tenian en el resto: acceso completo.

La regla escrita decia que el asistente no debia usarlo. Y no lo usaba.

**Pero una regla que el propio agente cumple voluntariamente no es un
control.**

El alcance real estaba acotado -desde el host de salto se sale igual con
credenciales limitadas, y el interruptor vive fuera de la maquina-, pero
habia una consecuencia concreta: **la vigilancia de ese host se podia
desactivar desde adentro**, y un agente podia leer el material del otro.

Se corrigio el mismo dia. **La lista de comandos permitidos no se invento:
salio de leer las invocaciones privilegiadas realmente registradas.** El
resultado fue que casi todo el uso real era lectura -cubierta por el
envoltorio-, algunas consultas nunca habian necesitado privilegio, y
quedaban dos acciones concretas.

**El permiso total nunca habia hecho falta.**

## Validacion

Se verifico **por lo que ahora falla**, no por lo que funciona:

- un lector generico con privilegio: **denegado**
- un interprete de comandos con privilegio: **denegado**
- un agente leyendo el material privado del otro: **denegado**
- el envoltorio de solo lectura: funciona
- el acceso a todos los hosts destino: funciona

Antes de retirar cualquier permiso se instalo y probo el reemplazo. **El
script que hace el cambio se niega a ejecutarse si el reemplazo no esta
funcionando**, porque en este entorno ya se habia perdido acceso dos veces
por retirar una credencial antes de tener lista la que la sustituia.

## Resultado

| Antes | Despues |
|---|---|
| Credenciales del agente en la estacion del operador | Solo dentro del host de salto |
| Acceso directo a cada host | Un unico camino, con interruptor fisico |
| Privilegio total en el host de salto | Lista acotada derivada de uso medido |
| Agentes indistinguibles entre si | Separados: ninguno lee el material del otro |
| "El agente no deberia hacer X" | El agente **no puede** hacer X |

## Lecciones

**Una restriccion que solo se cumple cuando es comoda no es una
restriccion.** Durante el propio trabajo hizo falta medir el uso privilegiado
dentro del host de salto, y el unico camino disponible era exactamente el
permiso que estaba prohibido usar. No se uso: se pidio el dato. **La
restriccion costo un paso real y por eso vale.**

**Un permiso se dimensiona midiendo, no imaginando.** La lista final salio
de leer que se invoco de verdad. Adivinarla habria dejado permisos de mas
por las dudas, o de menos, rompiendo automatizaciones en silencio.

**El control tiene que estar en la infraestructura, no en la conducta del
agente.** La diferencia entre "no debe" y "no puede" es la unica que
sobrevive a un error, a un cambio de version o a un modelo distinto.
