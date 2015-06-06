Pasaje de mensajes
==================
Hay 4 clases básicas de procesos en un programa distribuido: filtros, clientes, servidores y pares (peers). Un filtro es un transformador de datos: recibe streams de valores de datos desde sus canales de entrada , realiza alguna computación sobre esos valores, y envía streams de resultados a sus canales de salida. A causa de estos atributos, podemos diseñar un filtro independiente de otros procesos. Más aún, fácilmente podemos conectar filtros en redes que realizan computaciones más grandes: lo único que se requiere es que cada filtro produzca salidas que cumplan las suposiciones de entrada de los filtros que consumen esas salidas.
Un cliente es un proceso triggering (disparador); un servidor es un proceso reactivo. Los cliente hacen pedidos que disparan reacciones de los servidores. Un cliente inicia la actividad, a veces de su elección; con frecuencia luego se demora hasta que su pedido fue servido. Un servidor espera que le hagan pedidos, luego reacciona a ellos. La acción específica que toma un servidor puede depender de la clase de pedido, los parámetros del mensaje de pedido, y el estado del servidor; el servidor podría ser capaz de responder a un pedido inmediatamente, o podría tener que salvar el pedido y responder luego. Un servidor con frecuencia es un proceso que no termina y provee servicio a más de un cliente. Por ejemplo, un file server en un sistema distribuido maneja un conjunto de archivos y pedidos de servicios de cualquier cliente que quiere acceder a esos archivos.
Un peer es uno de un conjunto de procesos idénticos que interactúan para proveer un
servicio o para resolver un problema. Por ejemplo, dos peers podrían manejar cada uno una copia de
un archivo replicado e interactuar para mantenerlas consistentes. O varios peers podrían interactuar
para resolver un problema de programación paralela, resolviendo cada uno una parte del problema.
Pasaje de mensajes asincrónicos
-------------------------------
Se proveen un número de técnicas útiles para diseñar programas distribuidos. Describimos los siguiente paradigmas de interacción:
* flujo de datos en una dirección en redes de filtros
* pedidos y respuestas entre clientes y servidores
* interacción back-and-forth (heartbeat) entre procesos vecinos
* broadcasts entre procesos en grafos completos
* token-passing por las aristas en un grafo
* coordinación entre procesos servidores descentralizados
* workers replicados compartiendo una bolsa de tareas
### Ejemplo de proceso filtro
```c
  chan input(char), output([1:MAX_LINE] char)
  process char_to_line {
    var line[1:MAX_LINE]: char, i:= 1
    do true {
      receive input(line[i])
      do line[i] != CR and i < MAX_LINE {
        i:= i + 1
        receive input(line[i])
      }
      send output(line)
      i:= 1
    }
  }
```
### Filtros: una red de ordenación
```c
  # Proceso short
  process sort {
    < Recibe todos los números de un canal de entrada >
    < Ordena los números >
    < Envía los números ordenados por el canal de salida >
  }
```
```c
  # Proceso Merge
  char in_one(int), in_two(int), out(int)
  process merge {
    int num1, num2
    receive in_one(num1)
    receive in_two(num2)
    do num1 != EOS and num2 != EOS {
      if num1 <= num2 {
        send out(num1)
        receive in_one(num1)
      } else {
        send out(num2)
        receive in_two(num2)
      }
    }
    if num2 != EOS {
      do num2 != ESO { send out(num2); receive in_two(num2) }
    }
    if num2 != EOS {
      do num1 != ESO { send out(num1); receive in_two(num1) }
    }
    # Agregar el centinela
    send out(EOS)
  }
```
Para formar una red de ordenación, empleamos una colección de procesos Merge y arreglos de canales de entrada y salida. Asumiendo que el número n de valores de entrada es potencia de 2, los procesos y canales son conectados de manera que el patrón de comunicación resultante forma un árbol.
Un atributo clave de los filtros como Merge es que podemos interconectarlos de distintas maneras. Todo lo que se necesita es que la salida producida por un filtro cumpla las suposiciones de entrada del otro. Una consecuencia importante de este atributo es que podemos reemplazar un proceso filtro (o una red de filtros) por una red diferente. Por ejemplo, podemos reemplazar el proceso Sort descripto antes por una red de procesos Merge más un proceso (o red) que distribuye los valores de entrada a la red de merge.
Las redes de filtros pueden usarse para resolver una variedad de problemas de programación paralela.
### Clientes y servidores
#### Monitores activos (1 operación)
```c
  chan request(int, values_type)
  chan reply[1:n](results_type)

  process server {
    var index : int, values, results
    < código de inicialización >
    do true {
      receive request(index, values)
      < cuerpo de op >
      send reply[index](results)
    }
  }

  process client[i:1..n]{
    send request(i, arg_values)
    receive reply[i](arg_results)
    ......
  }
```
#### Monitores activos, múltiples operaciones
```c
  type op_kind = enum(op1, ..., opn)
  type arg_type = union(arg1: atype1 , ..., argn: atypen)
  type result_type = union(res1: rtype1, ..., resn: rtypen)
  chan request(int, op_kind, arg_type)
  chan reply[1:n](res_type)
  process server {
    var index: int, kind: op_kind, args: arg_type, results: res_type
    <código de inicialización >
    do true {
      receive request(index, kind, args)
      if kind = op1 { < cuerpo de op1 > }
      ...
      elseif kind = opn { < cuerpo de opn > }
      send reply[index](results)
    }
  }
  process client[i:1..n] {
    var myargs : arg_type, myresults : result_type
    < poner los valores de los argumentos en myargs >
    send request(i, op i , myargs)
    receive reply[i](myresults)
    ...
  }
```
* *TODO:* PASAJE DE UN MONITOR CON VARIABLES CONDICIÓN A UN PROCESO SERVER EN PMA
* *TODO:* SERVER DE SELF-SCHEDULING DE DISCO

#### File servers: continuidad conversacional
Como ejemplo final de la interacción cliente/servidor, presentamos una manera de implementar file servers, que son procesos que proveen acceso a archivos en almacenamiento secundario (por ej, archivos en disco). Para acceder a un archivo, un cliente primero abre el archivo.  Si la apertura es exitosa (el archivo existe y el cliente tiene permiso para acceder a él) entonces el cliente hace una serie de pedidos de read y write. Eventualmente el cliente cierra el archivo.
```c
  type kind= enum(READ, WRITE, CLOSE)
  chan open(file_name: string, client_id: int)
  chan access[1:n](kind: int, arg_types)
  chan open_reply[1:m](int)
  chan access_reply[1:m](result_types)

  process file_server[i:1..n] {
    var file_name: string, client_id: int
    var k: kind, args: arg_types
    var more:= false
    var resuls
    do true {
      receive open(file_name, client_id)
      < abre el archivo y si tiene exito >
      send open_reply[client_id](i)
      more:= true
      do more {
        receive access[i](k,args)
        if k = READ { < procesa pedido de lectura > }
        elseif k = WRITE { < procesa pedido de escritura > }
        elseif k = CLOSE { < cierra archivo >; more:= false }
      }
    }
  }

process client[j:1..m] {
  send open("file", j)
  receive open_reply[j](server_id)
  {{ usar el archivo }}
  send access[server_id](args)
  receive access_reply[j](results)
  ...
}
```
La interacción entre un cliente y un server es un ejemplo de continuidad conversacional.  En particular, un cliente comienza una “conversación” con un FS cuando ese server recibe el pedido de apertura. Luego el cliente sigue conversando con el mismo server. Esto se programa haciendo que el server primero reciba de open, luego repetidamente recibe de su elemento de access.
Pares interactuantes: intercambio de valores
--------------------------------------------
Los procesadores están conectados por tres modelos de arquitectura: *cantralizado*, *simétrico* y en *anillo circular*.
PROBLEMA: cada proceso tiene un dato local "V" y los N procesos deben saber cuál es el menor y cuál el mayor de los valores.
### Solución centralizada
La arquitectura centralizada es apta para una solución en que todos envían su dato local "V" al procesador central, éste ordena los N datos y reenvía la información del mayor y menor a todos los procesos.
El algoritmo debe realizar 2*(N-1) mensajes. Si el procesador central dispone de una primitiva broadcast se reduce a N mensajes.
```c
  chan values(int)
  chan results[n](int min, int max)

  process p[0] {
    int value, new, min, max
    value= < obtengo mi valor >
    min= value
    max= value
    for i= 1 to n-1 {
      receive values(new)
      if new < min { min= new }
      if new > max { max= new }
    }
    for i= 1 to n-1 { send results[i](min, max) }
  }

  process p[i= 1..n-1] {
    int value, min, max
    value= < obtengo mi valor >
    send values(value)
    receive results[i](min, max)
  }
```
### Solución simétrica
En la arquitectura simétrica o "full conected" hay un canal entre cada par de procesos. Todos los procesos ejecutan el mismo algoritmo.
Cado proceso transmite su dato local "V" a los n-1 procesos restantes. Luego recibe y procesa los n-1 datos que le faltan, de modo que *en paralelo* toda la arquitectura está calculando el mínimo y el máximo y toda la arquitectura tiene acceso a los n datos.
El algoritmo realiza n*(n-1) mensajes. Si disponemos de una primitiva de broadcast, serán n mensajes
```c
  chan values[n](int)

  process p[i= 0..n-1] {
    int value, new, min, max
    value= < obtengo mi valor >
    min= value
    max= value
    for j= 0 to n-1 st k != i { send values[j] }
    for j= 0 to n-1 st k != i {
      receive values[j](new)
      if new < min { min:= new }
      if new > max { max:= new }
    }
  }
```
### Solución en anillo circular
Un tercer modo de organizar la solución es tener un anillo donde p[i] recibe mensajes de p[i-1] y envía mensajes a p[i+1]. p[n-1] tiene como sucesor a p[0]. Es un esquema de dos etapas. En la primera cada proceso recibe dos valores y los compara con su valor local, transmitiendo un máximo local y un mínimo local a su sucesor. En la segunda etapa todos deben recibir la circulación del máximo y el mínimo global. p[0] deberá ser algo diferenta para empezar el procesamiento. Se requerirán 2*(n-1) mensajes.
```c
  chan values[n](int min, int max)

  process p[0] {
    int value, min, max
    value= < obtengo mi valor >
    min= value
    max= value
    send values[1](min, max)
    receive values[0](min, max)
    send values[1](min, max)
  }

  process p[i= 1..n-1] {
    int value, min, max, new
    value= < obtengo mi valor >
    min= value
    max= value
    receibe values[i](min, max)
    if value < min { min:= value }
    if value > max { max:= value }
    send values[i+1 MOD n](min, max)
    # espera el min y max global
    receive values[i](min, max)
    if i < n-1 { send values[i+1](min, max) }
  }
```
### Conclusión
La solución simétrica es la mas corta y sensilla de programar, pero usa el mayor número de mensajes. Puede transmitirse en parelelo si la red soporta transmiciones concurrentesm pero el overhead de comunicación acota el speedup.
La solución centralizada y anillo usan un número lineal de mensajes, pero tienen distintos patrones de comunicación que llevan a distintas performance.
En centralizada los mensajes al coordinador se envían casi al mismo tiempo. En anillo, todos los procesos son productores y consumidores. El último tiene que esperar a que todos los otros reciabn un mensaje, hacer poco conputo, y enviar su resultado. Solución inherentemente lineal y lenta para este problema, pero puede funcionar si cada proceso tiene mucho cómputo.
Algorimos de heartbeat
----------------------
En un sistema jerárquico, los servers en niveles intermedios con frecuencia son también clientes de servers de más bajo nivel. Por ejemplo, el FS anterior podría procesar los pedidos de read y write comunicándose con un disk server como el que desarrollamos.
Examinamos una clase distinta de interacción server en la cual los servers al mismo nivel son peers que cooperan para proveer un servicio. Este tipo de interacción se da en computaciones distribuidas en la cual no hay un solo server que tiene toda la información necesaria para servir un pedido del cliente.  consideramos el problema de computar la topología de una red. Este problema es representativo de una gran clase de problemas de intercambio de información distribuida que se da en redes. Se supone una red de procesadores conectados por canales bidireccionales.  Cada procesador se puede comunicar solo con sus vecinos y conoce solo los links con sus vecinos.
Cada procesador es modelizado por un proceso, y los links de comunicación con canales compartidos. Resolvemos este problema asumiendo primero que todos los procesos tienen acceso a una memoria compartida, lo que no es realista para este problema. Luego refinamos la solución en una computación distribuida replicando variables globales y teniendo procesos vecinos interactuando para intercambiar su información local. En particular, cada proceso ejecuta una secuencia de iteraciones. En cada iteración, un proceso envía su conocimiento local de la topología a todos sus vecinos, luego recibe la información de ellos y la combina con la suya. La computación termina cuando todos los procesos aprendieron la topología de la red entera.
Llamamos a este tipo de interacción entre procesos algoritmo heartbeat pues las acciones de cada nodo son como el latido de un corazón: primero se expande, enviando información; luego se contrae, incorporando nueva información. El mismo tipo de algoritmo puede usarse para resolver otros problemas, especialmente los de computaciones iterativas paralelas.
### Topología de red: solución con variables compartidas
```c
  bool top[1:n,1:n]= ([n*n] false)

  process node[p:1..n] {
    boll links[1:n]
    # inicialmente links[q] es true si q es un vecino de Node[p]
    fa q= 1 to n st links[q] { top[p,q]= true }
    top[p,1:n]= links[1:n]
  }
```
### Topología de red: solución distribuida
```c
  # CONOCIENDO EL DIAMETRO DE LA RED (distancia entre los nodos más lejanos)
  chan topology[1:n]([1:n,1:n] bool)

  process node[p:1..n]{
    bool links[1:n]
    # inicialmente links[q] true si q es vecino de Node[p]
    bool top[1:n,1:n]= ([n*n]false) # links conocidos
    top[p,1..n]= links # llena la fila para los vecinos
    int r= 0
    bool newtop[1:n,1:n]
    do r < D { # D es el diámetro de la red
      # envía el conocimiento local de la topología a sus vecinos
      fa q= 1 to n st links[q] { send topology[q](top) }
      # recibe las topologías y hace or con su top
      fa q= 1 to n st links[q] {
        receive topology[p](newtop)
        top= top OR newtop
      }
      r= r + 1
    }
  }
```
```c
  # SIN CONOCER EL DIAMETRO. SE PARA CUANDO TODAS LAS FILAS DE LA MATRIZ TIENEN UN VALOR EN TRUE
  chan topology[1:n](sender: int; done: bool; top: [1:n,1:n] bool)
  process nodo[p:1..n] {
    bool links[1:n]
    bool active[1:n]= links         # vecinos aún activos
    bool top[1:n,1:n]= ([n*n]false) # links conocidos
    int r= 0
    bool done= false
    int sender, bool qdone, bool newtop[1:n,1:n]
    top[p,1..n]= links                  # llena la fila para los vecinos
    do not done {
      # envía conocimiento local de la topología a todos sus vecinos
      fa q= 1 to n st links[q] { send topology[q](p, false, top) }
      # recibe las topologías y hace "or" con su top
      fa q= 1 to n st links[q] {
        receive topology[p](sender, qdone, newtop)
        top= top OR newtop
        if qdone { active[sender]= false }
      }
      if all_row_true(top) { done= true } # todas las filas de top tienen al menos 1 true
      r= r + 1
    }
    # última ronda
    # envía topología a todos sus vecinos aún activos
    fa q= 1 to n st active[q] { send topology[q](p, done, top) }
    # recibe un mensaje de cada uno para limpiar el canal
    fa q= 1 to n st active[q] { receive topology[p](sender, d, newtop) }
  }
```
Algoritmos probe/echo
---------------------
Un probe es un mensaje enviado por un nodo a su sucesor; un echo es un reply subsecuente.  Dado que los procesos ejecutan concurrentemente, los probes se envían en paralelo a todos los sucesores. El paradigma probe/echo es el análogo en programación concurrente a DFS, Primero ilustramos el paradigma probe mostrando cómo hacer broadcast de información a todos los nodos de una red. Luego agregamos el paradigma echo desarrollando un algoritmo distinto para construir la topología de una red.
### Broadcast en una red
Existe dos variantes para este tipo de problemas. Por un lado podemos resolver el problema contando con las siguiente premisas. Asumimos que el nodo i tiene una copia local top de la topología completa de la red, computada. Entonces una manera eficiente para que i haga broadcast de un mensaje es primero construir un spanning tree de la red, con él mismo como raíz del árbol. Un spanning tree de un grafo es un árbol cuyos nodos son todos los del grafo y cuyas aristas son un subconjunto de las del grafo.
Dado un spanning tree T, el nodo i puede hacer broadcast de un mensaje m enviando m junto con T a todos sus hijos en T. Luego de recibir el mensaje, cada nodo examina T para determinar sus hijos en el árbol, luego hace forward de m y T a todos ellos. El spanning tree es enviando junto con m pues los nodos que no son i no sabrían qué árbol usar.
```c
  chan probe[1:n](int span_tree[1:n, 1:n], message_type)

  process node[p:1..n] {
    int span_tree[1:n,1:n], message_type message
    receive probe[p](span_tree, message)
    fa q= 1 to n st q es un hijo de p en span_tree { send probe[q](span_tree, message) }
  }

  process initiator {
    int i
    int top[1:n, 1:n]
    int T[1:n, 1:n], message_type message
    < computar el spanning tree de top y almacenarlo en T >
    message= < mensaje a emitir >
    send probe[i](T, message)
  }
```
La otra solución a plantear es con un algoritmo simétrico. Dejamos que cada nodo que recibe m la primera vez envía m a todos sus vecinos, incluyendo al que le envió m. Luego el nodo recibe copias redundantes de m desde todos sus vecinos; estos los ignora.
```c
  chan probe[1:n](message_type)

  process node[p:1..n] {
    bool links[1:n] # inicializar con los vecinos
    int num= < cantidad de vecinos >
    message_type message
    receive probe[p](m)
    # envía m a todos sus vecinos
    fa q= 1 to n st links[q] { send probe[q](message) }
    # recibe de todos sus vecinos y los ignora
    fa q= 1 to num-1 { receive probe[q](message) }
  }

  process initiator {
    int i # indice del nodo que inicia el broadcast
    message_type message
    message= < emitir mensaje >
    send probe[i](message)
  }
```
El algoritmo broadcast usando un spanning tree causa que se envíen n-1 mensajes, uno por cada arista padre/hijo en el spanning tree. El algoritmo usando conjuntos de vecinos causa que dos mensajes sean enviados sobre cada link en la red, uno en cada dirección. El número exacto depende de la topología de la red, pero en general el número será mucho mayor que n-1.  Sin embargo, el algoritmo del conjunto de vecinos no requiere que el iniciador conozca la topología y compute un spanning tree. En esencia, el spanning tree es construido dinámicamente; consta de los links por los que se envían las primeras copias de m. Además, los mensajes son más cortos en este algoritmo pues el spanning tree (n 2 bits) no necesita ser enviado en cada mensaje.
### Topología de red revisitada
En particular, tenemos un nodo iniciador que junta los datos de topología local de cada uno de los otros nodos y luego disemina la topología completa a los otros nodos. La topología es reunida en dos fases. Primero, cada nodo envía un probe a sus vecinos. Luego, cada nodo envía un echo conteniendo la información de topología local al nodo del cual recibió el primer probe. Eventualmente, el nodo que inicia reunió todos los echoes. Entonces puede computar un spanning tree para la red y broadcast la topología completa usando alguno de los algoritmos ya vistos.
Para el primer algoritmo vamos a asumir que la topología es acíclica.
```c
  const source= i # indice del nodo que inicia el algoritmo
  chan probe[1:n](int sender)
  chan echo[1:n](bool links[1:n, 1:n])
  chan final_echo[1:n](bool links[1:n, 1:n])

  process node[p:1..n] {
    bool links[1:n]
    bool local_top[1:n, 1:n]= ([n*n] false)
    local_top[p, 1:n]= links
    bool new_top[1:n, 1:n]
    int parent
    receive probe[p](parent)
    # envía probes
    fa q= 1 to n st links[q] and q != parent { send probe[q](p) }
    # recibe echoes
    fa q=1 to n st links[q] and q != parent {
      receive echo[q](new_top)
      local_top= local_top OR new_top
    }

    if p = source { send final_echo(local_top) }
    else { send echo[parent](local_top) }
  }

  process initiator {
    bool top[1:n, 1:n]
    send probe[source](source)
    receive final_echo(top)
  }
```
Para computar la topología de una red que contiene ciclos, generalizamos el algoritmo anterior como sigue. Luego de recibir un probe, un nodo lo envía a todos sus otros vecinos, luego espera un echo de cada uno. Sin embargo, a causa de los ciclos y que los nodos ejecutan concurrentemente, dos vecinos podrían enviar cada uno otros probes casi al mismo tiempo. Los probes que no son el primero pueden ser echoed inmediatamente. En particular, si un nodo recibe un probe subsecuente mientras está esperando echos, inmediatamente envía un echo conteniendo una topología nula. Eventualmente un nodo recibirá un echo en respuesta a cada probe. En este punto, echoes la unión de su conjunto de vecinos y los echos que recibió.  El algoritmo probe/echo general para computar la topología de red es el siguiente. Dado que un nodo podría recibir probes subsecuentes mientras espera echoes, los dos tipos de mensajes tienen que ser mezclados en un canal
```c
  const source= i
  type kind= enum(PROBE, ECHO)
  chan probe_echo[1:n](kind, int sender, int links[1:n, 1:n])
  chan final_echo[1:n](int links[1:n, 1:n])

  process node[p:1..n] {
    bool links[1:n]= [[ vecinos de p ]]
    bool local_top[1:n]= links
    bool new_top[1:n]
    int first
    kind k
    int sender
    int need_echo= [[ número de vecinos -1 i ]]
    receive probe_echo[p](k, sender, new_top)
    first= sender
    # envía prove a todos los vecinos
    fa 1= 1 to n st links[q] and q != first { send probe_echo[q](PROBE, p, nil) }
    do need_echo > 0 {
      # recibe echoes o probes
      receive probe_echo[p](k, sender, new_top)
      if k == PROBE { send probe_echo[q](ECHO, p, nil) }
      if k == ECHO { local_top= local_top OR new_top; need_echo= need_echo - 1 }
    }
    if p = source { send final_echo(local_top)  }
    else { send probe_echo[first](ECHO, p, local_top) }
  }

  process initiator {
    bool top[1:n, 1:n]
    send probe_echo[source](PROBE, source, nil)
    receive final_echo(top)
  }
```
Este algoritmo probe/echo para computar la topología de una red requiere menos mensajes que el algoritmo heartbeat. Para computaciones que diseminan o reúnen información en grafos, los algoritmos probe/echo son más eficientes que los heartbeat. En contraste, los algoritmos heartbeat son apropiados y necesarios para muchos algoritmos iterativos paralelos en los cuales los nodos repetidamente intercambian información hasta que convergen en una respuesta.

