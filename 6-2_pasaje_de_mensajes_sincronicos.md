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
    do true → Sem! P()
      SC
      Sem! V()
      SNC
    od
  }

  process sem {
    do (i:1..n) c[i]? P() →
      c[i]? V()
    od
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
