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
