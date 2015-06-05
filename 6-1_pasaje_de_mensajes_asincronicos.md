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
