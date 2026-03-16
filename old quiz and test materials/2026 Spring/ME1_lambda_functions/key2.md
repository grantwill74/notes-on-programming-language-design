
1. `and3 = `$\lambda$`x y z . x (y z False) False`
2. `4 = `$\lambda$` f x . f (f (f (f x)))`, `4 id = `$\lambda$` x. id (id (id (id x)))` =
   $\lambda$ `x . x = id`
3. 3 + 2 = 5
4. (in Javascript)
```javascript
let switchNegative =
    (x) => (y) => (z) =>
        x < 0 ? y : z;
```
alternatively
```javascript
let switchNegative =
    (x) => x < 0 ?
        (y) => (_) => y :
        (_) => (z) => z
```