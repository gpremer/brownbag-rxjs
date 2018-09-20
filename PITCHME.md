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

- Redeneren op hoger niveau

- Niet zonder valkuilen

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

Vaak eenvoudiger

```ts
import * as rx from "rxjs";

rx.Observable.from([1, 2, 3]);
new rx.ReplaySubject<number>(10, 100);
```

+++

#### Pipe

- Operatoren via `pipe`

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

- anti-pattern: member variabele zetten in subscribe!

- Indien gebruikt, steeds unsubscribe!

- Repetitief => `bindToLifeCycle` in basisklasse


```ts
this.bindToLifeCycle(this.zoomClickedSubj).subscribe(zoom => ...);
```
- Niet nodig als observable zelf stopt

+++

#### `bindToLifeCycle`

```ts
export abstract class KaartComponentBase implements AfterViewInit, OnInit, OnDestroy {
  private readonly destroyingSubj: rx.Subject<void> = new rx.ReplaySubject<void>(1);

  ngOnDestroy() {
    this.destroyingSubj.next();
    this.destroyingSubj.complete();
  }

  protected bindToLifeCycle<T>(source: rx.Observable<T>): rx.Observable<T> {
    return source ? source.pipe(takeUntil(this.destroyingSubj)) : source;
  }
}
```

+++

#### Initialisatie in constructor

- Gelocaliseerde code

- niet mogelijk om `super.ngOnInit()` te vergeten

```ts
this.initialising$.pipe(
  switchMap(() => this.modelChanges$.pipe(...))
).subscribe(...);
```

+++

#### `initialising$`

```ts
export abstract class KaartComponentBase implements AfterViewInit, OnInit, OnDestroy {
  private readonly destroyingSubj: rx.Subject<void> = new rx.ReplaySubject<void>(1);
  private readonly initialisingSubj: rx.Subject<void> = new rx.ReplaySubject<void>(1);

  ngOnInit() {
    this.initialisingSubj.next();
    this.initialisingSubj.complete();
  }

  ngOnDestroy() {
    this.destroyingSubj.next();
    this.destroyingSubj.complete();
  }

  protected get initialising$(): rx.Observable<void> {
    return this.initialisingSubj.pipe(takeUntil(this.destroyingSubj));
  }
```

- vb van observable die zelf stopt


---

### Alternatief voor `subscribe`

- `async` pipe

```html
<awv-kaart-achtergrond-tile 
  *ngFor="let laag of (backgroundTiles$ | async)">
```

- Gebruik met `OnPush` change detection strategie

+++

#### `async` gotcha

- Opletten voor expressies in de `async` pipe!

```html
<div *ngIf="enabled$ | async">Ok</div> 
<div *ngIf="enabled$.pipe(map(e => !e)) | async">Niet ok</div> 
```

- async kijkt ook naar de referentie van de expressie

+++

#### 'switchMap`

Gebruik switchMap ipv flatMap

- Unsubscribe van binnenste observable


---

### Opletten voor

+++

#### `distinctUntilChanged`

- Vaak gebruikt om effectieve changes te krijgen

- Maar werkt op object referentie

```ts
this.aanHetTekenen$.pipe(distinctUntilChanged()); // Ok
viewInstellingen$.pipe(distinctUntilChanged(), map(vi => vi.zoom)) // Nok
viewInstellingen$.pipe(
  distinctUntilChanged((vi1, vi2) => vi1.zoom === vi2.zoom), 
  map(vi => vi.zoom)
); // Ok
viewInstellingen$.pipe(
  map(vi => vi.zoom),
  distinctUntilChanged(), 
); // Not really Ok
```

+++

#### hergebruik van observables

- Zelfde observable meer dan eens gebruikt

- Meerdere subscribes op de bron => extra overhead of zelfs neveneffecten

- `share` of `shareReplay`

```ts
 this.opties$ = this.modelChanges.uiElementOpties$.pipe(
    filter(o => o.naam === LagenUiSelector),
    map(o => o.opties as LagenUiOpties),
    startWith(DefaultOpties),
    shareReplay(1)
 );
```

+++

#### Tonen van state in de UI

- UI moet vaak niet alle wijzingen direct tonen

- Gebruik `debounceTime`

```ts
stableReferentielagen$ = 
  this.modelChanges.lagenOpGroep.get("Voorgrond.Laag").pipe(debounceTime(250));
```

+++

#### Angular ChangeDetection

- Angular gebruikt zone.js

- In Angular zone worden objecten geïnstrumenteerd voor ChangeDetection

- Overbodig met Observables

- Gebruik `observeOutsideAngular` en `observeOnAngular`

```ts
this.internalMessage$.pipe(
   ofType<VerwijderTekenFeatureMsg>("VerwijderTekenFeature"), //
   observeOnAngular(this.zone)
)
```

+++

#### `observeOutsideAngular`

```ts
export function observeOutsideAngular<T>(zone: ZoneLike) {
  return (source: rx.Observable<T>) =>
    new rx.Observable<T>(observer => {
      return source.subscribe({
        next(x) {
          zone.runOutsideAngular(() => observer.next(x));
        },
        error(err) {
          observer.error(err);
        },
        complete() {
          observer.complete();
        }
      });
    });
}
```

+++

#### `observeOnAngular`

```ts
export function observeOnAngular<T>(zone: ZoneLike) {
  return (source: rx.Observable<T>) =>
    new rx.Observable<T>(observer => {
      return source.subscribe({
        next(x) {
          zone.run(() => observer.next(x));
        },
        error(err) {
          observer.error(err);
        },
        complete() {
          observer.complete();
        }
      });
    });
}
```

+++ 

#### Errors

- Exceptie in event stream stopt observable

- En dus updates in UI (voor UI observable)

```ts
catchError(error => {
  kaartLogger.error("Fout bij opvragen weglocatie", error);
  // bij fout toch zeker geldige observable doorsturen, anders geen volgende events
  return rx.of(XY2AdresError(`Fout bij opvragen weglocatie: ${error}`));
})
```

- evt ook `retry`, `retryWhen`

#### `BehaviourSubject`

- Subject die state bijhoudt. Kan opgevraagd worden

- Observable + member variable

- Anti-patern!

- Om state bij te houden: gebruik `scan`

```ts
numBusy$: rx.Observable<number> = mergedDataloadEvent$.pipe(
  scan((numBusy: number, evt: DataLoadEvent) => {
    switch (evt.type) {
      case "LoadStart":
        return numBusy + 1;
      case "LoadComplete":
        return numBusy - 1;
      case "LoadError":
        return numBusy - 1;
    }
  }, 0)
);
```

- Om initiële toestand te zetten: `startWith`

---

