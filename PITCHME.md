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

```ts
import { Observable } from "rxjs";
import { combineLatest } from "rxjs/operators";
```

ipv

```ts
import { Observable } from "rxjs/Observable";
import { combineLatest } from "rxjs/add/operators";
```

+++

#### Imports

Vaak eenvoudiger:

```ts
import * as rx from "rxjs";

rx.Observable.from([1, 2, 3]);
new rx.ReplaySubject<number>(10, 100);
```

+++

#### Pipe

- Operatoren via `pipe`

- Verplicht vanaf RxJS 6

- Angular 6 steunt op RxJS 6

```ts
  event$.pipe(filter(e => e.type === "drop"))
```
ipv

```ts
  event$.filter(e => e.type === "drop")
```

+++

#### Combinatie-operatoren

```ts
    merge(a$, b$);
    combineLatest(a$, b$);
    concat(a$, b$);
    zip(a$, b$);
```

ipv

```ts
    a$.pipe(merge(b$));
    a$.pipe(combineLatest(b$));
    a$.pipe(concat(b$));
    a$.pipe(zip(b$));
```


---

### Subcribe

- In eerste instantie expliciete subscribes proberen te vermijden

- Indien gebruikt, steeds unsubscribe!

- Repetitief => in basisklasse


```ts
this.bindToLifeCycle(this.zoomClickedSubj).subscribe(zoom => ...);
```

+++

#### `bindToLifeCycle`

```ts
export abstract class KaartComponentBase implements AfterViewInit, OnInit, OnDestroy {
  private readonly destroyingSubj: rx.Subject<void> = new rx.ReplaySubject<void>(1); // laatkomers

  ngOnDestroy() {
    this.destroyingSubj.next();
    this.destroyingSubj.complete();
  }

  protected bindToLifeCycle<T>(source: rx.Observable<T>): rx.Observable<T> {
    return source ? source.pipe(takeUntil(this.destroyingSubj)) : source;
  }
}
```

