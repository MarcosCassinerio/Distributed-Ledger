#+TITLE: Atomic Broadcast

* Idea general

Objetivo Concreto: Implementar un servicio de /broadcast atómico/ distribuido.

Asumiremos que un sistema distribuido es un conjunto de procesos, \(\Phi = \{
p_1, \ldots, p_n \}\), y que además los procesos se pueden comunicar entre sí, y
el canal en sí de es confiable, es decir, no se replicaran mensajes ni se
generarán mensajes espurios.

Más formalmente: Un canal de comunicación confiable es aquel que si un /proceso
correcto/ \(p\) envía un mensaje \(m\) a un proceso correcto \(q\),
eventualmente \(q\) recibe \(m\).

Tendremos un conjunto posible del contenido de cada mensaje \(\mathbb{M}\) y
además un conjunto arbitrario de identificadores de mensajes \(I\).
De esta manera podemos intercambiar mensajes únicos dentro del sistema como una
tripla \(Msj = \mathbb{M} \times I \times \Phi \) identificando el contenido del
mensaje, un identificador único, y el proceso que lo envía.

Comencemos por definir el espacio de problemas:
+ Broadcast Confiable: Se define una única primitiva \(broadcast\). La primitiva
  debe cumplir la propiedad que \(broadcast(m)\) será enviado a todos los
  procesos /correctos/ si el proceso \(sender(m)\) es /correcto/.
+ Consenso: Todo proceso \(p_i\) comienza proponiendo un valor \(v_i\), y todos
  los procesos correctos finalizan decidiendo un valor \(v \in \{v_1, \ldots,
  v_n\}\) entre todos los valores propuestos.
+ Broadcast Totalmente Ordenado (Broadcast Atómico): es un servicio de broadcast
  confiable que además garantiza un orden sobre los mensajes enviados. Todos los
  mensajes enviados por los procesos siguen el mismo orden.

Los problemas de consenso y broadcast atómico fueron probados equivalentes,
es decir, que dada una solución para uno, se puede construir una solución para
el otro, y lo mismo al revés.

Es fácil de ver que teniendo un algoritmo de consenso \(A\), podemos decidir el
orden de los mensajes consensuando qué mensaje enviar en cada momento usando
\(A\).
Y teniendo un servicio de broadcast atómico podemos consensuar un valor
simplemente tomando el primero de los valores que han sido enviados.

Lo que reduce entonces el espacio de problemas a 2, broadcast confiables y
broadcast atómicos.

* Especificación del problema:

Un servicio de broadcast atómico implementa dos operaciones
  + \(ABroadcast(m)\) : que envía el mensaje \(m\) a todos los nodos del sistema
  + \(ADeliver(m)\) : usado por el sistema distribuido para enviar el mensaje \(m\) a un nodo del sistema
  con \(m \in Msj\).
  Asumiremos además que \(ABroadcast(m)\) se ejecutará una única vez.

Cuando un proceso \(p\) invoca \(ABroadcast(m)\) diremos que \(p\) hace un
broadcast de \(m\), y cuando \(p\) invoca a \(ADeliver(m)\) diremos que \(p\)
envía \(m\).

Que cumple con las siguientes propiedades:
  + (Validity) Si un proceso correcto \(p\) hace un broadcast de \(m\) entonces
    eventualmente \(p\) envía \(m\).
  + (Uniform Agreement) Si un proceso envía un mensaje \(m)\) entonces todos los
    procesos correctos eventualmente envían \(m\).
  + (Uniform Integrity) Para cualquier mensaje \(m\), todo proceso envía \(m\) a
    lo sumo una vez, y si anteriormente \(sender(m)\) hizo un broadcast.
  + (Uniform Total Order) Si dos procesos \(p_i, p_j \in \Phi \) envían mensajes
    \(m,m' \in Msj\), entonces \(p_i\) envía \(m\) antes que \(m'\) si y solo si \(p_j\)
    envía \(m\) antes que \(m'\).

/Validity/ y /Uniform Agreement/ son propiedades de /liveness/ del sistema,
mientras que /Uniform Integrity/ y /Uniform Total Order/ son propiedades de
correctitud.

Lamentablemente la definición tiene un problema llamado de /contaminación/.
Todavía hay procesos que pueden llegar a estados inconsistentes (justo antes de
crashear). Aunque por el momento lo pasaremos por alto.

# Las propiedades de uniformidad aplican sobre todos los procesos, capaz que sea
# mejora relajarlas a simplemente procesos correctos.

# Y para evitar el problema de contaminación podemos modificar la propiedad de UTO a algo de la forma:
# Si algún proceso envía un mensaje \(m'\) después que \(m\) entonces un proceso
# envía \(m'\) solo si ya envió \(m\).

* Algoritmos
El espacio de soluciones del problema es muy amplio ya que la solución depende
de la configuración concreta del sistema, los canales de comunicación y
diferentes requerimientos que se puedan llegar a necesitar.

En este trabajo práctico nos enfocaremos en las soluciones que se adapten al
modelo de comunicación que nos brinda Erlang, que como vimos en la materia,
tiene bastantes garantías.

Adicionalmente no resolveremos el problema en el caso que haya nodos que se
comporten de manera extraña, es decir, de forma /bizantina/, pero si podremos
identificar cuando hay nodos que /crashean/.

La primer pregunta que podemos preguntar es: ¿quién genera el orden?. Más
concretamente quién nos entrega las primitivas básicas para generar un orden
entre los mensajes, como ser: timestamps o una secuencia de identificadores.

Podemos entonces categorizar 3 roles en los que un proceso puede participar :
 - emisor
 - destinatario
 - secuencializador

Un proceso emisor es un proceso que origina un mensaje, un destinatario es un
proceso al que se le enviará un mensaje, y un secuencializador es un proceso
involucrado en la selección del orden de los mensajes.

** Secuencializador Centralizado

Una forma es tener un nodo identificado que es el encargado de secuencializar
(elegir un orden) entre los mensajes que recibe y replicar a los demás nodos de
la red.

Dentro de la carpeta *example* pueden encontrar un [[./example][ejemplo]] que
consta de dos archivos:
+ [[./example/dest.erl][Destinatarios]] que implementa principalmente el
  funcionamiento de los nodos destinatarios y emisores del sistema.
+ [[./example/sec.erl][Secuencializador]] que implementa el funcionamiento del
  secuencializador.

Como primer actividad se propone realizar experimentos sobre la
implementación dada e identificar las limitaciones del mismo.

** Mediante Acuerdo de los Destinatarios (ISIS)

El nombre es debido a la empresa que diseño el algoritmo (aunque ya no existe).
El objetivo en este caso es eliminar el /secuencializador/ y alcanzar un acuerdo
entre los destinatarios.
En este caso el acuerdo es en el orden de los mensajes, es decir,
decidir el numero de mensaje que se enviará a continuación.

La descripción del algoritmo la podrán encontrar en su sección de [[https://es.wikipedia.org/wiki/Multidifusi%C3%B3n][Wikipedia]].

La implementación deberá estar debidamente documentada, y tiene que ser
tolerante a fallas.  Es decir, que los nodos pueden fallar pero no comportarse
erroneamente.

* Implementación de un Ledger Distribuido
Implementando un servicio de Broadcast Atómico podemos entonces implementar un
/ledger/ distribuido.

Un /ledger/ es una entidad (objeto) que representa una secuencia de /records/, y
que soporta dos operaciones:
+ \(get()\) : que devuelve la secuencia completa
+ \(append(x)\) : que concatena el elemento /x/ al final de la secuencia.

El objetivo es implementar un objeto ledger de forma distribuida que sea
tolerante a fallas utilizando el servicio de broadcast atómico implementado en
la sección anterior.

En particular se pide implementar el algoritmo propuesto en el fragmento de
código 8 para los nodos (o procesos \(p_i\)) y código 5 para los clientes del
articulo [[https://arxiv.org/abs/1802.07817][Formalizing and Implementing Distributed Ledger Objects]] .

* Bibliografía
Para el desarrollo del trabajo se utilizó el artículo [[https://dl.acm.org/doi/10.1145/1041680.1041682][Total Order Broadcast and
Multicast Algorithms: Taxonomy and Survey]], y como implementación concreta de
ISIS la entrada de Wikipedia de [[https://es.wikipedia.org/wiki/Multidifusi%C3%B3n][Multidifusión]].
