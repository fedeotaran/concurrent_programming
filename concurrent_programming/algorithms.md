Paralelismo iterativo
=====================
Solución paralela por strip
---------------------------
```c
  Process worker[w:1..p] {
    int primera:= (w-1) * (n/p) + 1;
    int ultima:= primera + (n/p) - 1;
    for i:= primera to ultima {
      for j:= 1 to n {
        c[i,j]:= 0;
        for k:= 1 to n {
        c[i,j]:= c[i,j] + (a[i,k] * b[k,j]);
        }
    }
  }
  }
```
Problema de la sección crítica
==============================
Spinlocks
---------
```c
  bool: lock;
  process sc[i:1..n] {
    while (ts(lock)) { skip; }
    # Uso de la sección crítica
    lock:= false;
  }

  function ts(bool: lock) {
    < bool: inicial := lock;
    lock:= true
    return inicial >
  }
```
Algoritmo Tie-breaker
---------------------
```c
  var in[1:n]:= ([n] 0), last[1:n]:= ([n] 0);
  Process p[i:1..n] {
    do true {
      for j:=1 to n-1 {
        in[i]:= j;
        last[j]:= i;
        for k:= 1 to n st i <> k {
          do in[k] >= in[i] and last[j]= i { skip; }
        }
      }
      # Uso de la sección crítica
      in[i]:= 0;
      # Sección no crítica
    }
  }
```
Algoritmo Ticket
----------------
```c
  var number:= 1, next:= 1, turn[1..n]:= ([n] 0);
  Process p[i:1..n] {
    do true {
      turn[i]:= fa(number, 1);
      do turn[i] <> next { skip; }
      # Uso de la sección crítica
      next:= next + 1;
      # Sección no crítica
    }
  }

  function fa(int: var, int incr) {
    < temp:= var;
    var:= var + incr;
    return temp >
  }
```
Algoritmo Bakery
----------------
```c
  var turn[1..n]:= ([n] 0);
  Process p[i:1..n] {
    do true {
      turn[i]:= 1;
      turn[i]:= max(turn[1..n]) + 1;
      for j:= 1 to n st <> i {
        do turn[j] <> 0 and (turn[i],i) > (turn[j],j) { skip; }
      }
      # Uso de la sección crítica
      turn[i]:= 0;
      # Uso d ela sección no crítica
    }
  }
```
Sincronización barrier
======================
Contador compartido
-------------------
```c
  # GRANO GRUESO
  var count:= 0, passed[1:n]:= ([n] false);
  Process worker[i:1..n] {
    do true {
      # Código para implementar la tarea
      < count:= count + 1 >
      < await count = n -> passed[i]:= true >
    }
  }
  # GRANO FINO
  var count:= 0, passed[1:n]:= ([n] false);
  Process worker[i:1..n] {
    do true {
      # Código para implementar la tarea
      fa(count, 1);
      do count <> n { skip; }
    }
  }
```
Flags y coordinadores
---------------------
```c
  var arrive[1:n]:= ([n] 0), continue[1:n]:= ([n] 0);
  Process worker[i:1..n] {
    do true {
      # Código para implementar la tarea
      arrive[i]:= 1;
      < await continue[i] = 1 >
      continue:= 0;
    }
  }

  Process cordinator {
    do true {
      for i:= 1 to n {
        < await arrive[i] = 1 >;
        arrive[i]:= 0;
      }
      for i:= 1 to n { continue:= 1; }
    }
  }
```
* *TODO:* Sincronizacion Barrier: Arboles (combining tree barrier)
* *TODO:* Barreras simétricas
* *TODO:* Butterfly barrierhttp://telefe.com/bases-y-condiciones/elegidos/
Algoritmos paralelos de datos
-----------------------------
### Computación de prefijos paralela
```c
  var a[1:n], sum[1:n], old[1:n]
  Process sum[i:1..n] {
    var d:= 1
    sum[i]:= a[i]
    barrier
    do d < n {
      old[i]:= sum[i]
      barrier
      if i-d >= 1 { sum[i]:= old[i-d] + sum[i] }
      barrier
      d:= 2 * d
    }
  }
```
Semáforos
=========
```c
  # Impementacion de semáforos de grano grueso
  P(s) { < await s > 0 -> s:= s - 1 > }

  V(s) { < s:= s + 1 > }
```
```c
  # Semáforo binario
  P(b) { < await b > 0 -> b:= b - 1 > }

  V(b) { < await b < 1 -> b:= b + 1 > }
```
Problema de la sección crítica con semáforos
--------------------------------------------
```c
  var mutex: sem:= 1
  P[i:1..n] {
    do true {
      P(mutex);
      // Sección crítica
      V(mutex);
      // Sección no crítica
    }
  }
```
Productores y consumidores: semáforos binarios
----------------------------------------------
```c
  var buf: t
  var empty: sem:= 1, full: sem:= 0

  Process Producer[i:1..m] {
    do true {
      // producir
      P(empty)
      buf:= m
      V(full)
    }
  }
  Process Consumer[i:1..n] {
    do true {
      P(full)
      m:= buf
      V(empty)
      // consumir el mensaje m
    }
  }
```
Buffers Limitados: Contador de recursos
---------------------------------------
```c
  var buf[1:n]: T
  var front:= 0, rear:= 0
  var empty: sem:= n, full: sem:= 0
  var mutexD: sem:= 1, mutexF: sem:= 1
  Process Producer[i:1..m] {
    do true {
      // producir mensaje m
      P(empty)
      P(mutexD)
      buf[rear]:= m
      rear:= rear mod n + 1
      V(mutexD)
      V(full)
    }
  }
  Process Consumer[i:1..n] {
    do true {
      P(full)
      P(mutexF)
      m:= buf[front]
      front:= front mod n + 1
      P(mutexF)
      V(empty)
      // consumir mensaje
    }
```
Exclusion mutua selectiva
-------------------------
### Problema de los filósofos
```c
  var fork[1:5]: sem:= ([5] 1)
  Philosopher[i:1..4] {
    do true {
      P(fork[i]); P(fork[i + 1])
      // come
      V(fork[i]); V(fork[i + 1])
      // piensa
    }
  }
  Philosopher[5] {
    do true {
      P(fork[1]); P(fork[5])
      // come
      V(fork[1]); V(fork[5])
    }
  }
```
### Lectores escritores
```c
  var nr:= 0, mutex_r: sem:= 1, rw: sem:= 1
  Reader[i:1..m] {
    do true {
      P(mutex_r)
      nr:= nr + 1
      if (nr = 1) { P(rw) }
      V(mutex_r)
      // lee la BD
      P(mutex_r)
      nr:= nr - 1
      if (nr = 1) { V(rw) }
      V(mutex_r)
    }
  }
  Writer[j:1..n] {
    do true {
      P(rw)
      // escribe la BD
      V(rw)
    }
  }
```
Este algoritmo es llamado de preferencia de los lectores. Este término denota el hecho de que, si algún lector está accediendo a la BD y llega un lector y un escritor a sus entry protocols, entonces el nuevo lector tiene preferencia sobre el escritor.
Dado lo mencionado anteriormente, la solución NO ES FAIR. Esto es porque una serie continue de lectores puede evitar permanentemente que los escritores accedan a la BD.
Sincronización por condicion general
------------------------------------
### Lectores y escritores final
```c
  var nr:= 0, nw:= 0
  var e: sem:= 1, r: sem:= 0, w: sem:= 0
  var dr:= 0, dw:= 0
  Reader[i:1..m] {
    do true {
      P(e)
      if nw > 0 { dr:= dr + 1; V(e); P(r) }
      nr:= nr + 1
      if dr > 0 { dr:= dr - 1; V(r) }
         n = 0 { V(e) }
      // lee la BD
      nr:= nr - 1
      if nr = 0 and dw > 0 { dw:= dw -1; V(w) }
         nr > 0 or dw = 0 { V(e) }
    }
  }
  Writer[j:1..n] {
    do true {
      P(e)
      if nr > 0 or nw > 0 { dw:= dw +1; V(e); P(w) }
      nw:= nw + 1
      V(e)
      // escribe la BD
      P(e)
      nw:= nw - 1
      if dr > 0 { dr:= dr - 1; V(r) }
         dw > 0 { dw:= dw - 1; V(w) }
         dr = 0 and dw = 0 { V(e) }
    }
  }
```
En este algoritmo se puede ver el uso de la técnica de *Passing the baton*.
Alocacion de recursos
---------------------
### Alocación Shortest-job-next
```c
  var free:= true, e: sem:= 1; b[1..n]: sem:= ([n] 0)
  var p: set of (int, int):= 0
  request(time,id) {
    P(e)
    if not free {
      p.insert(time, id)
      V(e)
      P(b[id])
    }
    free:= false
    V(e)
  }
  release {
    P(e)
    free:= true
    if p.any? { p.remove(time, id); V(b[id]) }
      p.empty? { V(e) }
  }
```
* *TODO:* PRODUCTORES CONSUMIDORES CON BROADCAST
* *TODO:* VARIANTES DEL PROBLEMA DE LOS FILOSOFOS
* *TODO:* EL PROBLEMA DE LOS BAÑOS
* *TODO:* EL PROBLEMA DE LA MOLECULA DE AGUA
* *TODO:* EL PUENTE DE UNA SOLA VIA
* *TODO:* SEARCH - INSERT - DELETE SOBRE UNA LISTA
* *TODO:* DRINKING PHILOSOPHERS
Monitores
=========
Buffer limitado
---------------
```c
  monitor bounded_buffer {
    var buf[1..n]: T
    var front:= 1, rear:= 0, count:= 0
    var not_full: cond
    var not_empty: cond
    procedure deposit(data: T) {
      do count = n { wait(not_full) }
      buf[rear]:= data; rear:= (rear mod n) + 1; count:= count + 1
      signal(not_empty)
    }
    procedure fetch(var result: T) {
      do count = 0 { wait(not_empty) }
      result:= buf[front]; front:= (front mod n) + 1; count:= count - 1
      signal(not_full)
    }
  }
```
Shortest job next
-----------------
```c
  monitor shortest_job_next {
    var free:= true
    var turn: cond
    procedure request(time: int) {
      if free { free:= false
      } else {
        wait(turn, time)
      }
    }
    procedure release() {
      if empty(turn) {
        free:= true
      } else {
        signal(turn)
      }
    }
  }
```
Implementación de semáforos
---------------------------
```c
  monitor semaphore {
    var s:= 0, pos: cond
    procedure P() {
      do s = 0 { wait(pos) }
      s:= s - 1
    }
    procedure V() {
      s:= s + 1
      signal(pos)
    }
  }
```
Semaforos (Passing the condition)
---------------------------------
```c
  monitor semaphore {
    var s:= 0, pos: cond

    procedure P() {
      if s > 0 {
        s:= s -1
      } else {
        wait(pos)
      }
    }

    procedure V() {
      if empty(pos) {
        s:= s + 1
      } else {
        signal(pos)
      }
    }
  }
```
Lectores y escritores
---------------------
```c
  monitor rw_controller {
    var nr:= 0, nw:= 0
    var ok_to_read: cond
    var ok_to_write: cond

    procedure request_read() {
      do nw > 0 { wait(ok_to_read) }
      nr:= nr + 1
    }

    procedure release_read() {
      nr:= nr - 1
      if nr = 0 { signal(ok_to_write) }
    }

    procedure request_write() {
      do nr > 0 or nw > 0 { wait(ok_to_write) }
      nw:= nw + 1
    }

    procedure release_write() {
      nw:= nw - 1
      signal(ok_to_write)
      signal_all(ok_to_read)
    }
  }
```
Reloj lógico 1 (covering conditions)
------------------------------------
```c
  monitor timer {
    var curret_time:= 0
    var check: cond

    procedure wait(int interval) {
      int time_to_wakeup:= current_time + interval;
      do time_to_wakeup > current_time { wait(check) }
    }

    procedure tick() {
      current_time:= current_time + 1
      signal_all(check)
    }
  }
```
Relog lógico 2 (wait con prioridad)
-----------------------------------
```c
  monitor timer {
    var curret_time:= 0
    var check: cond

    procedure wait(int interval) {
      int time_to_wakeup:= current_time + interval;
      if time_to_wakeup > current_time { wait(check, time_to_wakeup) }
    }

    procedure tick() {
      current_time:= current_time + 1
      do not empty(check) and min(check) <= current_time { signal(check) }
    }
  }
```
Problema del sleeping barber (rendezvous)
-----------------------------------------
Una ciudad tiene una pequeña peluquería con dos puertas y unas pocas sillas. Los cliente entran por una puerta y salen por la otra. Dado que el negocio es chico, a lo sumo un cliente o el peluquero se pueden mover en él a la vez. El peluquero pasa su vida atendiendo clientes, uno por vez. Cuando no hay ninguno, el peluquero duerme en su silla. Cuando llega un cliente y encuentra que el peluquero está durmiendo, el cliente lo despierta, se sienta en la silla del peluquero, y duerme mientras el peluquero le corta el pelo. Si el peluquero está ocupado cuando llega un cliente, el cliente se va a dormir en una de las otras sillas. Después de un corte de pelo, el peluquero abre la puerta de salida para el cliente y la cierra cuando el cliente se va. Si hay clientes esperando, el peluquero despierta a uno y espera que el cliente se siente. Sino, se vuelve a dormir hasta que llegue un cliente.
```c
  monitor barber_shop { 
    var barber:= 0, chair:= 0, open:= 0
    var barber_available: cond
    var chair_accupied: cond
    var door_open: cond
    var customer_left: cond

    # llamado por los clientes
    procedure get_haircut() {
      do barber = 0 { wait(barber_available) }
      barber:= barber - 1
      chair:= chair + 1
      signal(chair_occupied)
      do open = 0 { wait(door_open) }
      open:= open -1
      signal(customer_left)
    }

    # llamado por el peluquero
    procedure get_next_customer() {
      barber:= barber + 1
      signal(barber_available)
      do chair = 0 { wait(chair_occupied) }
      chair:= chair - 1
    }

    #llamado por el peluquero
    procedure finished_cut() {
      open:= open + 1
      signal(door_open)
      do open > 0 { wait(customer_left) }
    }
  }
```
Scheduling de disco
-------------------
### Monitor separado
```c
  monitor disk_scheduler { 
    var position:= -1, c:= 0, n:= 1
    var scan[0:1]: cond

    procedure request(cyl: int) {
      if position = -1 {
        position:= cyl
      } elseif position != -1 and cyl > position {
        wait(scan[c],cyl)
      } elseif position != -1 and cyl <= position {
        wait(scan[N], cyl)
      }
    }

    procedure release() {
      if not empty(scan[c]) {
        position:= min(scan[c])
      } elseif empty(scan[c]) and not empty(scan[n]) {
        c:= n
        position:= min(scan[c])
      } elseif empty(scan[c]) and empty(scan[n]) {
        position:= -1
      }
    }
  }
```
### Con intermediario
```c
  monitor disk_interface {
    var position:= -2, c:= 0, n:= 1, scan[0:1]: cond
    var arg_area: arg_type, result_area: result_type, args:= 0, results:= 0
    var args_stored, result_stored, result_retrived: cond

    procedure use_disk(cyl: int, transfer_params, result_params) {
      if position:= -1 {
        position:= cyl
      } elseif position != -1 and cyl > position {
        wait(scan[c], cyl)
      } elseif position != -1 and cil <= position {
        wait(scan[n], cyl)
      }
      arg_area:= transfer_params
      args:= args + 1
      signal(args_stored)
      do results = 0 { wait(result_stored) }
      result_params:= result_area
      results:= results - 1
      signal(results_retrived)
    }

    procedure get_next_request(transfer_params) {
      if not empty(scan[c]) {
        position:= min(scan[c])
      } elseif empty(scan[c]) and not empty(scan[n]) {
        c:= n
        position:= min(scan[c])
      } elseif empty(scan[c]) and empty(scan[n]) {
        position:= -1
      }
      signal(scan[c])
      do args = 0 { wait(args_stored) }
      transfer_params:= arg_area
      args:= args - 1
    }

    procedure finished_transfer(result_params) {
      result_area:= result_params
      results:= results + 1
      signal(result_stored)
      do results > 0 { wait(results_retrieved) }
    }
  }
```
Problema del sleeping barber (rendezvous)
-----------------------------------------
Una ciudad tiene una pequeña peluquería con dos puertas y unas pocas sillas. Los cliente entran por una puerta y salen por la otra. Dado que el negocio es chico, a lo sumo un cliente o el peluquero se pueden mover en él a la vez. El peluquero pasa su vida atendiendo clientes, uno por vez. Cuando no hay ninguno, el peluquero duerme en su silla. Cuando llega un cliente y encuentra que el peluquero está durmiendo, el cliente lo despierta, se sienta en la silla del peluquero, y duerme mientras el peluquero le corta el pelo. Si el peluquero está ocupado cuando llega un cliente, el cliente se va a dormir en una de las otras sillas. Después de un corte de pelo, el peluquero abre la puerta de salida para el cliente y la cierra cuando el cliente se va. Si hay clientes esperando, el peluquero despierta a uno y espera que el cliente se siente. Sino, se vuelve a dormir hasta que llegue un cliente.
```c
  monitor barber_shop { 
    var barber:= 0, chair:= 0, open:= 0
    var barber_available: cond
    var chair_accupied: cond
    var door_open: cond
    var customer_left: cond

    # llamado por los clientes
    procedure get_haircut() {
      do barber = 0 { wait(barber_available) }
      barber:= barber - 1
      chair:= chair + 1
      signal(chair_occupied)
      do open = 0 { wait(door_open) }
      open:= open -1
      signal(customer_left)
    }

    # llamado por el peluquero
    procedure get_next_customer() {
      barber:= barber + 1
      signal(barber_available)
      do chair = 0 { wait(chair_occupied) }
      chair:= chair - 1
    }

    #llamado por el peluquero
    procedure finished_cut() {
      open:= open + 1
      signal(door_open)
      do open > 0 { wait(customer_left) }
    }
  }
```
Pasaje de mensajes
==================
Pasaje de mensajes asincrónicos
-------------------------------
### Filtros: una red de ordenación

