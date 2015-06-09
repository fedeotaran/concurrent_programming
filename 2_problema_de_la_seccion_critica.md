Problema de la sección crítica
==============================
Spinlocks
---------
```c
  bool lock
  process sc[i:1..n] {
    while true {
      while (ts(lock)) { skip; }
      # Uso de la sección crítica
      lock= false;
    }
  }

  function ts(bool lock) {
    < bool inicial= lock;
    lock= true
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
