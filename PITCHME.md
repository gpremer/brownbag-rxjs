# RxJS in ng-kaart


---

## RxJS algemeen


---

### RxJS en Angular

Gebruikt intern

Minder magie in applicaties


---

### RxJS opfrissing

@ul

- Lid van Rx*-familie

- In essentie inversion of control

- Gestructureerde manier voor callbacks

- Gamma aan patronen (multi-casting, timing, transformatie, ...)

@ulend

+++

### RxJS marbles

Interactieve [Diagrammen](http://rxmarbles.com/)


---

## RxJS in de praktijk


---

### Voorbereiden op Angular 6

- Angular 5.1 steunt op RxJS 5.5

- Angular 6 steunt op RxJS 6

- RxJS 6 heeft compat library 

- Maar subset van RxJS 5.5 is forward compatible

+++

#### Imports

Vanaf RxJS 5.5 

```
import { Observable } from "rxjs";
import { combineLatest } from "rxjs/operators";
```

ipv

```
import { Observable } from "rxjs/Observable";
import { combineLatest } from "rxjs/add/operators";
```

+++

#### Imports

Vaak eenvoudiger:

```
import * as rx from "rxjs";
```

+++

#### Pipe

- Operatoren via `pipe`

- Verplicht vanaf RxJS 6

- Angular 6 steunt op RxJS 6

```
  event$.pipe(filter(e => e.type === "drop"))
```
ipv

```
  event$.filter(e => e.type === "drop")
```

+++

#### Combinatie-operatoren

```
    merge(a$, b$);
    combineLatest(a$, b$);
    concat(a$, b$);
    zip(a$, b$);
```

ipv

```
    a$.pipe(merge(b$));
    a$.pipe(combineLatest(b$));
    a$.pipe(concat(b$));
    a$.pipe(zip(b$));
```

