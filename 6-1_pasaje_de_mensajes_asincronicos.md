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
  # Proceso sort
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
Algoritmos de broadcast
-----------------------
En la sección previa, mostramos cómo hacer broadcast de información en una red. En particular, podemos usar un algoritmo probe para diseminar información y un algoritmo probe/echo para reunir o ubicar información.
En la mayoría de las LAN, los procesadores comparten un canal de comunicación común tal como un Ethernet o token ring. En este caso, cada procesador está directamente conectado a cada uno de los otros. De hecho, tales redes con frecuencia soportan una primitiva especial llamada broadcast, la cual transmite un mensaje de un procesador a todos los otros. Esto provee una técnica de programación útil.
Podemos usar broadcasts para diseminar o reunir información; por ejemplo, con frecuencia se usa para intercambiar información de estado del procesador en LAN. También podemos usarlo para resolver muchos problemas de sincronización distribuidos. Esta sección ilustra el poder de broadcasts desarrollando una implementación distribuida de semáforos. La base para los semáforos distribuidos (y otros protocolos de sincronización descentralizados) es un ordenamiento total de los eventos de comunicación. Comenzamos mostrando cómo implementar relojes lógicos y luego mostramos cómo usar esos relojes para ordenar eventos.
### Relojes lógicos y ordenamiento de eventos
Los procesos en un programa distribuido ejecutan acciones locales y acciones de comunicación. Las acciones locales incluyen cosas tales como leer y escribir variables locales. No tienen efecto directo sobre otros procesos. Las acciones de comunicación son enviar y recibir mensajes. Estos afectan la ejecución de otros procesos pues comunican información y son el mecanismo de sincronización básico. Las acciones de comunicación son así los eventos significantes en un programa distribuido. Por lo tanto, usamos el término evento para referirnos a la ejecución de send o receive.
Hay un ordenamiento total entre eventos que se afectan uno a otro. Pero, hay solo un orden parcial entre la colección entera de eventos en un programa distribuido. Esto es porque las secuencias de eventos no relacionados (por ej, las comunicaciones entre distintos conjuntos de procesos) podría ocurrir antes, después, o concurrentemente con otros.  Un reloj lógico es un contador entero que es incrementado cuando ocurre un evento.  Asumimos que cada proceso tiene un reloj lógico y que cada mensaje contiene un timestamp. Los relojes lógicos son entonces incrementados de acuerdo a las siguientes reglas:
Reglas de Actualización de Relojes Lógicos. Sea lc un reloj lógico en el proceso A.
(1) Cuando A envía o broadcast un mensaje, setea el timestamp del mensaje al valor corriente
de lc y luego incrementa lc en 1.
(2) Cuando A recibe un mensaje con timestamp ts, setea lc al máximo de lc y ts+1 y luego
incrementa lc en 1.
### Semáforos distribuidos
Recordemos la definición básica de semáforos: en todo momento, el número de operaciones P completadas es a lo sumo el número de operaciones V completadas más el valor inicial. Así, para implementar semáforos, necesitamos una manera de contar las operaciones P y V y una manera de demorar las operaciones P. Además, los procesos que “comparten” un semáforo necesitan cooperar para mantener el invariante aunque el estado del programa esté distribuido.  Podemos cumplir estos requerimientos haciendo que los procesos broadcast mensajes cuando quieren ejecutar operaciones P y V y que examinen los mensajes que reciben para determinar cuando continuar. En particular, cada proceso tiene una cola de mensajes local mq y un reloj lógico lc. Para simular la ejecución de P o V, un proceso broadcast un mensaje a todos los procesos usuario, incluyendo él mismo. El mensaje contiene la identidad del emisor, un tag (P o V) y un timestamp.
Desafortunadamente, es irrealista asumir que broadcast es una operación atómica. Dos mensajes broadcast por dos procesos distintos podrían ser recibidos por otros en distinto orden. Más aún, un mensaje con un timestamp menor podría ser recibido después que un mensaje con un timestamp mayor. Sin embargo, distintos mensajes broadcast por un proceso serán recibidos por los otros procesos en el orden en que fueron enviados; estos mensajes tendrán también timestamps crecientes. Esto es porque (1) la ejecución de broadcast es la misma que la ejecución concurrente de send (que asumimos que provee entrega ordenada y confiable) y (2) un proceso incrementa su reloj lógico después de cada evento de comunicación.
El hecho de que mensajes consecutivos enviados por cada proceso tienen timestamps distintos nos da una manera de tomar decisiones de sincronización. Supongamos que una cola mq de mensajes de un proceso contiene un mensaje m con timestamp ts. Entonces, una vez que el proceso ha recibido un mensaje con un timestamp más grande de cada uno de los otros procesos, se asegura que nunca verá un mensaje con un timestamp menor. En este punto, el mensaje m se dice que está totalmente reconocido (fully acknowledged). Más aún, una vez que m está totalmente reconocido, entonces cualquier otro mensaje anterior a éste en mq también estará totalmente reconocido pues todos tendrán timestamps menores. Así, la parte de mq que contiene mensajes totalmente reconocidos es un prefijo estable: ningún mensaje nuevo será insertado en él.
Para completar la implementación de semáforos distribuidos, cada proceso simula las operaciones semáforo. Usa una variable local sem para representar el valor del semáforo. Cuando un proceso toma un mensaje ACK, actualiza el prefijo estable de su cola mq. Para cada mensaje V, el proceso incrementa sem y borra el mensaje V. Luego examina los mensajes P en orden de timestamp. Si sem > 0, el proceso decrementa sem y borra el mensaje P.
```c
  type kind= enum(V, P, ACK)
  chan sem_op[1:n](int sender, kind, int timestamp)
  chan go[1:n](int timestamp)

  process user[i:1..n] {
    int lc, ts
    lc= 0
    # ejecuta una operación V
    broadcast sem_op(i, V, lc)
    lc= lc + 1
    ...
    # ejecuta la operación P
    broadcast sem_op(i, P, lc)
    lc= lc + 1
    receive go[i](ts)
    lc= max(lc, ts+1)
    lc= lc + 1
    ...
  }

  process helper[i:1..n] {
    mq queue of (int, kind, int)
    int lc= 0
    int sem= [[ valor inicial ]]
    int sender, ts
    kind k
    do true {
      receive sem_op[i](sender, k, ts)
      lc= max(lc, ts+1)
      lc= lc + 1
      if k == P or k == V {
        mq.insert(sender, k, ts)
        broadcast sem_op(i, ACK, lc)
        lc= lc + 1
      } elseif k == ACK {
        [[ registrar que otro ACK fue visto ]]
        fa fully acknowledged V messages { mq.remove(sender, k, ts) }
        fa fully acknowledged P messages st sem > 0 {
          mq.remove(sender, k, ts)
          sem= sem + 1
          if sender == i { send go[i](lc); lc= lc + 1 }
        }
      }
    }
  }
```
Podemos usar semáforos distribuidos para sincronizar procesos en un programa distribuido esencialmente de la misma manera que usamos semáforos regulares en programas de variables compartidas.
Cuando se usan algoritmos broadcast para tomar decisiones de sincronización, cada proceso debe participar en cada decisión. En particular, un proceso debe oir de cada uno de los otros para determinar cuándo un mensaje está totalmente reconocido. Esto significa que estos algoritmos no son buenos para interacciones entre un gran número de procesos. También que deben ser modificados para contemplar fallas.
Algoritmos token-passing
------------------------
Un token es una clase especial de mensaje que puede ser usado o para dar permiso para tomar una acción o para reunir información de estado global. Ilustramos token-passing presentando soluciones a dos problemas adicionales de sincronización. Primero presentamos una solución distribuida simple al problema de la SC. Luego desarrollamos dos algoritmos para detectar cuándo una computación distribuida ha terminado. Token-passing también es la base para otros algoritmos; por ej, para sincronizar el acceso a archivos replicados.
### Exclusión Mutua Distribuida
Aunque el problema de la SC reside primariamente en programas de variables compartidas, también se da en programas distribuidos cuando hay un recurso compartido que puede usar a lo sumo un proceso a la vez. Más aún, el problema de la SC con frecuencia es una componente de un problema mayor, tal como asegurar consistencia en un archivo distribuido o un sistema de BD.
Aquí resolvemos el problema de una tercera manera usando un token ring. La solución es descentralizada y fair, como una que usa semáforos distribuidos, pero requiere intercambiar pocos mensajes. Además, la aproximación básica puede ser generalizada para resolver otros problemas de sincronización que no son fácilmente resueltos de otras maneras.
Sea P[1:n] una colección de procesos “regulares” que contienen SC y SNC. Necesitamos desarrollar entry y exit protocols que esos procesos ejecuten antes y después de su SC. También los protocolos deberían asegurar exclusión mutua, evitar deadlock y demora innecesaria, y asegurar entrada eventual (fairness).
Dado que los procesos regulares tienen otro trabajo que hacer, no queremos que también tengan que circular el token. Así emplearemos una colección de procesos adicionales, Helper[1:n], uno por proceso regular. Estos procesos helper forman un anillo.
Cuando Helper[i] recibe el token, chequea para ver si su cliente P[i] quiere entrar a su SC. Si no, Helper[i] pasa el token. En otro caso, Helper[i] le dice a P[i] que puede entrar a su SC, luego espera hasta que P[i] salga; luego, pasa el token.
```c
  chan token[1:n]()
  chan enter[1:n]()
  chan go[1:n]()
  chan exit[1:n]()

  process helper[1:1..n] {
    receive token[i]()
    if not empty(enter[i]) {
      receive enter[i]()
      send go[i]()
      receive exit[i]()
    }
    send token[(i mod n) + 1]() # pasa el token
  }

  process p[i:1..n] {
    send enter[i]()
    receive go[i]()
    SC
    send exit[i]()
    SNC
  }
```
Esta solución es fair, pues el token circula continuamente, y cuando un Helper lo tiene, P[i] tiene permiso para entrar si quiere hacerlo. Como está programado, el token se mueve continuamente entre los helpers. Esto es lo que sucede en una red física token-ring. Sin embargo, en un token-ring software, probablemente es mejor agregar algún delay en cada helper para que el token se mueva más lentamente en el anillo.
## Detección de Terminación en un Anillo
No es en absoluto fácil detectar la terminación de un programa distribuido. Esto es porque el estado global no es visible a ningún procesador. Más aún, puede haber mensajes en tránsito entre procesadores.
El problema de detectar cuándo una computación distribuida ha terminado puede ser resuelto de varias maneras. Por ejemplo, podríamos usar un algoritmo probe/echo o usar relojes lógicos y timestamps. Aquí desarrollamos un algoritmo token-passing, asumiendo que toda la comunicación entre procesos es a través de un anillo.
Nuestra tarea es obtener un algoritmo de detección de terminación en una computación distribuida arbitraria, sujeto solo a la suposición de que los procesos se comunican en un anillo. Claramente la terminación es una propiedad del estado global, el cual es la unión de los estados de los procesos individuales más los contenidos de los canales. Así, los procesos tienen que comunicarse uno con otro para determinar si la computación terminó.
Para detectar terminación, sea que hay un token, el cual es un mensaje especial que no es parte de la comunicación. El proceso que mantiene el token lo pasa cuando se convierte en ocioso.
Cuando el token hizo un circuito completo del anillo de comunicación, sabemos que cada proceso estaba ocioso en algún punto.
Para esto asociamos un color con cada proceso: blue para ocioso y red para activo. Inicialmente todos los procesos están activos, entonces están coloreados red. Cuando un proceso recibe el token, está ocioso, entonces se colorea a sí mismo blue y pasa el token. Si el proceso luego recibe un mensaje regular, se colorea red. Así, un proceso que está blue se ha vuelto ocioso, pasó el token, y se mantuvo ocioso pues pasó el token.
Luego, asociamos un valor con el token indicando cuántos canales están vacíos si P[1] está aún ocioso. Sea token el valor del token. Cuando P[1] se vuelve ocioso, se colorea a sí mismo azul, setea token a 0, y luego envía token a P[2]. Cuando P[2] recibe el token, está ocioso y ch[2] podría estar vacío. Por lo tanto, P[2] se colorea blue, incrementa token a 1, y envía el token a P[3]. Cada proceso P[i] a su turno se colorea blue e incrementa token antes de pasarlo.
Más aún, todos estos procesos se mantuvieron ociosos desde que pasaron el token. Así, si P[1] es aún blue cuando el token volvió a él, todos los procesos son blue y todos los canales están vacíos. Por lo tanto P[1] puede anunciar que la computación terminó.
Servidores replicados
---------------------
Hay ejemplos que involucran el uso de servers replicados, es decir, múltiples procesos server que hacen lo mismo. La replicación sirve a uno de dos propósitos. Primero, podemos incrementar la accesibilidad de datos o servicios teniendo más de un proceso que provea el mismo servicio. Estos servers descentralizados interactúan para proveer a los clientes la ilusión de que hay un único servicio centralizado. Segundo, a veces podemos usar replicación para acelerar el encuentro de la solución a un problema dividiendo el problema en subproblemas y resolviéndolos concurrentemente. Esto se hace teniendo múltiples procesos worker compartiendo una bolsa de subproblemas.

