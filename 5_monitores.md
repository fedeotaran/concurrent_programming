Monitores
=========
Buffer limitado
---------------
```c
  monitor bounded_buffer {
    T buf[1..n]
    int front= 1, rear= 0, count= 0
    cond not_full
    cond not_empty

    procedure deposit(T data) {
      do count == n { wait(not_full) }
      buf[rear]= data; rear= (rear mod n) + 1; count= count + 1
      signal(not_empty)
    }

    procedure fetch(T result) {
      do count == 0 { wait(not_empty) }
      result= buf[front]; front= (front mod n) + 1; count= count - 1
      signal(not_full)
    }
  }
```
Shortest job next
-----------------
```c
  monitor shortest_job_next {
    bool free= true
    cond turn

    procedure request(time: int) {
      if free {
        free= false
      } else {
        wait(turn, time)
      }
    }

    procedure release() {
      if empty(turn) {
        free= true
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
    int s= 0
    cond pos

    procedure P() {
      do s == 0 { wait(pos) }
      s= s - 1
    }

    procedure V() {
      s= s + 1
      signal(pos)
    }
  }
```
Semaforos (Passing the condition)
---------------------------------
```c
  monitor semaphore {
    int s= 0
    cond pos

    procedure P() {
      if s > 0 {
        s= s -1
      } else {
        wait(pos)
      }
    }

    procedure V() {
      if empty(pos) {
        s= s + 1
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
