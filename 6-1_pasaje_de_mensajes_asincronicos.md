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
Algorimos de heartbeat
----------------------
En un sistema jerárquico, los servers en niveles intermedios con frecuencia son también clientes de servers de más bajo nivel. Por ejemplo, el FS anterior podría procesar los pedidos de read y write comunicándose con un disk server como el que desarrollamos.
Examinamos una clase distinta de interacción server en la cual los servers al mismo nivel son peers que cooperan para proveer un servicio. Este tipo de interacción se da en computaciones distribuidas en la cual no hay un solo server que tiene toda la información necesaria para servir un pedido del cliente.  consideramos el problema de computar la topología de una red. Este problema es representativo de una gran clase de problemas de intercambio de información distribuida que se da en redes. Se supone una red de procesadores conectados por canales bidireccionales.  Cada procesador se puede comunicar solo con sus vecinos y conoce solo los links con sus vecinos.
Cada procesador es modelizado por un proceso, y los links de comunicación con canales compartidos. Resolvemos este problema asumiendo primero que todos los procesos tienen acceso a una memoria compartida, lo que no es realista para este problema. Luego refinamos la solución en una computación distribuida replicando variables globales y teniendo procesos vecinos interactuando para intercambiar su información local. En particular, cada proceso ejecuta una secuencia de iteraciones. En cada iteración, un proceso envía su conocimiento local de la topología a todos sus vecinos, luego recibe la información de ellos y la combina con la suya. La computación termina cuando todos los procesos aprendieron la topología de la red entera.
Llamamos a este tipo de interacción entre procesos algoritmo heartbeat pues las acciones de cada nodo son como el latido de un corazón: primero se expande, enviando información; luego se contrae, incorporando nueva información. El mismo tipo de algoritmo puede usarse para resolver otros problemas, especialmente los de computaciones iterativas paralelas.
### Topología de red: solución con variables compartidas
```c
  var top[1:n,1:n]: bool:= ([n*n] false)
  process node[p:1..n] {
    var links[1:n] : bool
    # inicialmente links[q] es true si q es un vecino de Node[p]
    fa q:= 1 to n st links[q] { top[p,q]:= true }
    top[p,1:n] = links[1:n]
  }
```
### Topología de red: solución distribuida
```c
  # CONOCIENDO EL DIAMETRO DE LA RED (distancia entre los nodos más lejanos)
  chan topology[1:n]([1:n,1:n] bool)
  process node[p:1..n]{
    var links[1:n] : bool
    # inicialmente links[q] true si q es vecino de Node[p]
    var top[1:n,1:n]: bool:= ([n*n]false) # links conocidos
    top[p,1..n] := links # llena la fila para los vecinos
    var r:= 0
    var newtop[1:n,1:n] : bool
    do r < D { # D es el diámetro de la red
      # envía el conocimiento local de la topología a sus vecinos
      fa q:= 1 to n st links[q] { send topology[q](top) }
      # recibe las topologías y hace or con su top
      fa q:= 1 to n st links[q] {
        receive topology[p](newtop)
        top := top or newtop
      }
      r := r + 1
    }
  }
```
```c
  # SIN CONOCER EL DIAMETRO. SE PARA CUANDO TODAS LAS FILAS DE LA MATRIZ TIENEN UN VALOR EN TRUE
  chan topology[1:n](sender: int; done: bool; top: [1:n,1:n] bool)
  process nodo[p:1..n] {
    var links[1:n]: bool
    var active[1:n]: bool:= links         # vecinos aún activos
    var top[1:n,1:n]: bool:= ([n*n]false) # links conocidos
    var r: int:= 0, done: bool:= false
    var sender: int, qdone: bool, newtop[1:n,1:n]: bool
    top[p,1..n] := links                  # llena la fila para los vecinos
    do not done {
      # envía conocimiento local de la topología a todos sus vecinos
      fa q := 1 to n st links[q] { send topology[q](p, false, top) }
      # recibe las topologías y hace "or" con su top
      fa q := 1 to n st links[q] {
        receive topology[p](sender, qdone, newtop)
        top := top or newtop
        if qdone { active[sender]:= false }
      }
      if all_row_true(top) { done:= true } # todas las filas de top tienen al menos 1 true
      r:= r + 1
    }
    # última ronda
    # envía topología a todos sus vecinos aún activos
    fa q:= 1 to n st active[q] { send topology[q](p, done, top) }
    # recibe un mensaje de cada uno para limpiar el canal
    fa q:= 1 to n st active[q] { receive topology[p](sender, d, newtop) }
  }
```
