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
