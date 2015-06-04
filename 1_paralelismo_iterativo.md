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
