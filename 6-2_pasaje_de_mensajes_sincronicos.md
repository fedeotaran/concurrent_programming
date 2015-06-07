Pasaje de mensajes asincrónicos
===============================
Si un proceso trata de enviar a un canal, se demora hasta que otro proceso esté esperando recibir por ese canal. De esta manera, un emisor y un receptor sincronizan en todo punto de comunicación. Si el emisor sigue, entonces el mensaje fue entregado, y los mensajes no tiene que ser buffereados. En esencia, el efecto de la comunicación sincrónica es una sentencia de asignación distribuida, con la expresión siendo evaluada por el proceso emisor y luego siendo asignada a una variable en el proceso receptor.
Hay así un tradeoff entre AMP y SMP. Por un lado, SMP simplifica la resolución de algunos problemas y no requiere alocación de buffer dinámica. Por otro lado, es más difícil, como veremos, programar algoritmos heartbeat y broadcast usando SMP.
La notación es similar a la introducida por Hoare en 1978 en el paper sobre CSP. Uno de los principales conceptos que introdujo Hoare en el artículo es lo que llamamos comunicación guardada (o waiting selectivo). La comunicación guardada combina pasaje de mensajes con sentencias guardadas para producir una notación de programa elegante y expresiva.
Comunicación guardada
---------------------
Una sentencia de comunicación
guardada tiene la forma general:
```c
  B; C → S
```
Aquí B es una expresión booleana opcional, C es una sentencia de comunicación opcional, y S es una lista de sentencias. Si B se omite, tiene el valor implícito de true. Si C se omite, una sentencia de comunicación guardada es simplemente una sentencia guardada.  Juntos, B y C forman la guarda. La guarda tiene éxito (succeds) si B es true y ejecutar C no causaría una demora; es decir, algún otro proceso está esperando en una sentencia de comunicación matching. La guarda falla si B es falsa. La guarda se bloquea si B es true pero C no puede ser ejecutada sin causar demora.
Las sentencias de comunicación guardadas aparecen dentro de sentencias if y do. Ahora una sentencia if es ejecutada como sigue. Si al menos una guarda tiene éxito, una de ellas es elegida no determinísticamente. Primero se ejecuta la sentencia de pasaje de mensajes, y luego la correspondiente lista de sentencias. Si todas las guardas fallan, el if termina. Si ninguna guarda tiene éxito y algunas guardas están bloqueadas, la ejecución se demora hasta que alguna guarda tenga éxito. Dado que las variables no son compartidas, el valor de una expresión booleana en una guarda no puede cambiar hasta que el proceso ejecute sentencias de asignación. Así, una guarda bloqueada no puede tener éxito hasta que algún otro proceso alcanza una sentencia de comunicación matching. 
Una sentencia do es ejecutada de manera similar. La diferencia es que el proceso de selección se repite hasta que todas las guardas fallan.
Como un ejemplo de comunicación guardada tenemos la implementación de un semáforo binario con PMS:
```c
  process c[1:n] { 
    do true { 
      Sem! P()
      SC
      Sem! V()
      SNC
    }
  }

  process sem {
    do (i:1..n) c[i]? P() {
      c[i]? V()
    }
  }
```
Redes de filtros
----------------
Las redes de filtros son programadas con SMP de manera similar a lo hecho con AMP.
Así, la diferencia esencial entre usar SMP versus AMP es la ausencia de buffering implícito. Esto puede ocasionalmente ser usado para beneficio pues el emisor de un mensaje sabe cuándo fue recibido. La ausencia de buffering puede también complicar algunos algoritmos. Por ejemplo, en un algoritmo heartbeat, puede resultar deadlock si dos procesos vecinos tratan ambos primero de enviar uno al otro y luego de recibir uno del otro.
Esta sección desarrolla soluciones paralelas a dos problemas: generación de números primos y multiplicación matriz/vector. Ambas soluciones emplean redes de filtros. El primer algoritmo usa un arreglo de procesos, donde cada proceso se comunica con sus dos vecinos. El segundo algoritmo usa una matriz de procesos, y cada proceso se comunica con sus 4 vecinos. Como siempre sucede con redes de filtros, la salida de cada proceso es una función de su entrada.
### Generación de Números Primos: La Criba de Eratóstenes
Para la generación de números primos utilizaremos la técnica de la criba de Erastótenes. Y lo implementaremos empleando un pipeline de procesos filtro. Cada filtro en esencia ejecuta el cuerpo del loop externo del algoritmo secuencial. En particular, cada filtro del pipeline recibe un stream de números de su predecesor y envía un stream de números a su sucesor. El primer número que recibe un filtro es el próximo primo más grande; le pasa a su sucesor todos los números que no son múltiplos del primero.
```c
  # Por cada canal, el primer número es primo y todos los otros números
  # no son múltiplo de ningún primo menor que el primer número
  process sieve[1] {
    int p= 2
    int i
    # pasa los número impares a Sieve[2]
    fa i= 3 to n by 2 { sieve[2]! i }
  }

  process sieve[i:2..L] {
    int p, next
    sieve[i-1]? p # p es primo
    do true {
      # recibe el próximo candidato
      sieve[i-1]? next
      # pasa next si no es múltiplo de p
      if next mod p != 0 { sieve[i+1]! next }
    }
  }
```
El primer proceso, Sieve[1], envía todos los números impares desde 3 a n a Sieve[2].  Cada uno de los otros procesos recibe un stream de números de su predecesor. El primer número p que recibe el proceso Sieve[i] es el i-ésimo primo. Cada Sieve[i] subsecuentemente pasa todos los otros números que recibe que no son múltiplos de su primo p. El número total L de procesos Sieve debe ser lo suficientemente grande para garantizar que todos los primos hasta n son generados.
El programa anterior termina en deadlock. Podemos fácilmente modificarlo para que termine normalmente usando centinelas, como en la red de filtros merge.
### Multiplicación matriz/vercor 
Consideremos el problema de multiplicar una matriz a por un vector b. Por simplicidad, asumiremos que a tiene n filas y n columnas, y por lo tanto que b tiene n elementos. Nuestra tarea es computar el producto matriz/vector.
Esto requiere computar n productos internos, uno por cada fila de a con el vector b.
Como solución distribuida podemos implementar un proceso por posición de la matriz.
Cada proceso P[i,j] tiene un elemento de a, digamos a[i,j]. Primero, cada proceso recibe el valor de b[i] desde su vecino Norte y lo pasa a su vecino Sur. El proceso luego recibe una suma parcial desde su vecino Oeste, le suma a[i,j] * b[i], y envía el resultado a su vecino Este.
```c
  process p[i:1..n, j:1..n] {
    real sum, b
    sum= 0
    p[i-1, j]? b; p[i+1, j]! b
    p[i, j-1]? sum; p[i, j+1]!(sum + a[i, j] * b)
  }
```
No se muestran los procesos en el borde de la red. Como se muestra en la figura, los proceso en el borde norte envía los elementos de b, y los procesos en el borde Oeste envían ceros.  Los procesos en el borde Sur solo reciben los elementos de b; los procesos en el borde Este reciben los elementos del vector resultado x. Estos procesos se necesitan para que las sentencias de comunicación no queden en deadlock.
Procesos paralelos interactuantes
---------------------------------
En los algoritmos paralelos de la sección anterior, la información fluía a través de una red de procesos. En esta sección, examinamos algoritmos paralelos en los cuales los procesos intercambian información hasta que han computado una solución.
### Sorting paralelo: Algoritmo Heartbeat
Desarrollaremos una solución en la cual los procesos vecinos intercambian información repetidamente.
#### Algoritmo compare and exchange
Asumimos que hay dos procesos, P1 y P2, y que cada proceso inicialmente tiene n/2 valores arbitrarios (por simplicidad asumimos que n es par). Luego podemos ordenar el conjunto entero de valores en orden no decreciente por el siguiente algoritmo heartbeat. Primero, cada proceso usa un algoritmo secuencial tal como quicksort para ordenar sus n/2 valores. Luego los procesos intercambian repetidamente valores hasta que P1 tiene los n/2 valores menores y P2 los n/2 mayores. En particular, en cada paso, P1 le da a P2 una copia de su mayor valor, y P2 le da a P1 una copia de su valor más chico. Cada proceso inserta el nuevo valor en el lugar apropiado en su lista ordenada de valores, descartando el viejo valor si es necesario. El algoritmo termina cuando el mayor valor de P1 no es mayor que el menor valor de P2. El algoritmo se garantiza que termina pues cada intercambio (excepto el último) le da dos valores más a los procesos.
```c
# Antes de cada intercambio, a1 y a2 están ordenados
# P1 y P2 repetidamente intercambian a1[largest] y a2[smallest]
# hasta que a1[largest] ≤ a2[smallest]
process p1 {
  int a1[1:n/2] # inicializado con n/2 valores
  const largest= n/2
  int new
  [[ ordenar a1 en orden no decreciente ]]
  p2! a1[largest]; p2? new # intercambia valores con P2
  do a1[largest] > new {
    [[ poner new en el lugar correcto en a1, descantando el viejo a1[largest] ]]
    p2! a1[largest]; P2? new
  }

  process p2 {
    int a2[1:n/2] # inicializado con n/2 valores
    const smallest= 1
    int new
    [[ ordenar a2 en orden no decreciente ]]
    p1? new; p1! a2[smallest] # intercambia valores con P1
    do a2[smallest] < new {
      [[ poner new en el lugar correcto en a2, descantando el viejo a2[smallest] ]]
      p1? new; a2[smallest] # intercambia valores con P1
    }
  }
```
#### Algoritmo odd/even exchange sort
Supongamos que usamos k procesos P[1:k], por ejemplo, porque tenemos k procesadores.  Inicialmente, cada proceso tiene n/k valores. Asumimos que ponemos los procesos en una secuencia lineal desde P[1] hasta P[k] y que cada proceso primero ordena sus n/k valores. Luego podemos ordenar los n elementos usando aplicaciones paralelas repetidas del algoritmo de dos procesos compare-and-exchange.
Cada proceso ejecuta una serie de rondas. En las rondas impares, cada proceso con número impar juega el rol de P1, y cada proceso con número par el de P2. En particular, para todos los valores impares de i, P[i] intercambia valores con P[i+1]. (Si k es impar, P[k] no hace nada en las rondas impares).
En las rondas pares, cada proceso numerado par juega el rol de P1, y cada proceso impar el rol de P2. En este caso, para todos los valores pares de i, P[i] intercambia valores con P[i+1]. (En las rondas pares, P[1] no hace nada, y, si k es par, P[k] tampoco hace nada).
Para el ejemplo de este algoritmo asumimos que tanto k como n son el mismo número.
```c
  process p[i:1..n] {
    int value= [[ mi valor ]]
    int new # valor intercambiado
    for j= 1 to k { # con k rondas no aseguramos que el algoritmo terminara ordenando todo.
      if j MOD 2 != 0 { # ronda impar
        if i MOD 2 =! 0 {
          # rol de p1
          p[i+1]! value; p[i+1]? new
          if value < new { value= new }
        } else {
          # rol de p2
          p[i-1]? new; p[i-1]? value
          if value > new { value= new }
        }
      } else { # ronda par
        if i MOD 2 = 0 {
          if i != n {
            # rol de p1
            p[i+1]! value; p[i+1]? new
            if value < new { value= new }
          }
        } else {
          if i != 1 {
            # rol de p2
            p[i-1]? new; p[i-1]? value
            if value > new { value= new }
          }
        }
      }
    }
  }
```
Para algoritmos con n > k cada proceso tieme más de un valor y por lo tanto deberá intercambiar los valores siguiendo la misma lógica que el algoritmo *compare and exchange*

