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

- Basis: `Observable`, `Subscriber`, `Subscription`, `Subject`

- Gamma aan patronen (multi-casting, timing, transformatie, ...)

- In essentie inversion of control

- Gedsciplineerde manier voor callbacks

- Redeneren op hoger niveau

- Niet zonder valkuilen

@ulend

+++

### RxJS marbles

Interactieve [rxmarbles](http://rxmarbles.com/)


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
import { Observable } from "rxjs/Observable";
import { combineLatest } from "rxjs/add/operators";
```

wordt

```ts
import { Observable } from "rxjs";
import { combineLatest } from "rxjs/operators";
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
event$.filter(e => e.type === "drop")
```
wordt

```ts
event$.pipe(filter(e => e.type === "drop"))
```

+++

#### Combinatie-operatoren

```ts
a$.pipe(merge(b$));
a$.pipe(combineLatest(b$));
a$.pipe(concat(b$));
a$.pipe(zip(b$));
```

wordt

```ts
merge(a$, b$);
combineLatest(a$, b$);
concat(a$, b$);
zip(a$, b$);
```


---

### Subcribe

- Poog expliciete subscribes proberen te vermijden

- Anti-pattern: member variabele zetten in subscribe!

- Indien gebruikt, steeds unsubscribe!

- Repetitief => `bindToLifeCycle` in basisklasse

- Niet nodig als observable tijdig zelf stopt

```ts
this.bindToLifeCycle(this.zoomClicked$).subscribe(zoom => ...);
```

+++

#### Implementatie `bindToLifeCycle`

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
this.initialising$.subscribe(
  () => this.kaart.dispatch(VoegUiElementToe(TekenenUiSelector))
);
this.destroying$.subscribe(
  () => this.kaart.dispatch(VerwijderUiElement(TekenenUiSelector))
);
```

+++

#### Implementatie `initialising$`

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
    return this.initialisingSubj;
  }
```

+++

### Alternatief voor `subscribe`

- `async` pipe

- Gebruik met `OnPush` change detection strategie

```html
<awv-kaart-achtergrond-tile 
  *ngFor="let laag of (backgroundTiles$ | async)">
```

+++

#### `async` gotcha

- Opletten voor expressies in de `async` pipe!

- `async` kijkt ook naar de referentie van de expressie

```html
<div *ngIf="enabled$ | async">Ok</div> 
<div *ngIf="enabled$.pipe(map(e => !e)) | async">Niet ok</div> 
```


---

### Opletten voor

+++

#### 'switchMap`

Gebruik [switchMap](http://rxmarbles.com/#switchMap) ipv [flatMap/mergeMap](http://rxmarbles.com/#mergeMap)

- Unsubscribe van binnenste observable

- Bijv. `httpClient`, `xxxClicked$`

+++

#### `distinctUntilChanged`

- Om effectieve changes te krijgen

- Maar: werkt op object referentie!

```ts
this.aanHetTekenen$.pipe(distinctUntilChanged()); // Ok
viewInst$.pipe(distinctUntilChanged(), map(vi => vi.zoom)); // Nok

viewInst$.pipe(
  distinctUntilChanged((vi1, vi2) => vi1.zoom === vi2.zoom), 
  map(vi => vi.zoom)); // Ok
viewInst$.pipe(
  map(vi => vi.zoom),
  distinctUntilChanged()); // Not really Ok
```

+++

#### Hergebruik van observables

- Zelfde observable meerdere keren in HTML template

- Meerdere subscribes op de bron => extra overhead of zelfs neveneffecten

- `share` of `shareReplay`

- Eventueel te vermijden met `let`

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
  this.modelChanges.lagenOpGroep.get("Voorgrond.Laag")
    .pipe(debounceTime(250));
```

+++

#### Angular ChangeDetection

- Angular gebruikt zone.js

- Instrumenteert objecten voor ChangeDetection

- Gebruik `observeOutsideAngular` en `observeOnAngular`

- Met `subscribe`

```ts
this.internalMessage$.pipe(
   ofType<VerwijderTekenFeatureMsg>("VerwijderTekenFeature"), //
   observeOnAngular(this.zone)
)
```

+++

#### Implementatie `observeOutsideAngular`

```ts
export function observeOutsideAngular<T>(zone: ZoneLike) {
  return (source: rx.Observable<T>) =>
    new rx.Observable<T>(observer => {
      return source.subscribe({
        next(x: T) {
          zone.runOutsideAngular(() => observer.next(x));
        },
        error(err: any) {
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

#### Implementatie `observeOnAngular`

```ts
export function observeOnAngular<T>(zone: ZoneLike) {
  return (source: rx.Observable<T>) =>
    new rx.Observable<T>(observer => {
      return source.subscribe({
        next(x: T) {
          zone.run(() => observer.next(x));
        },
        error(err: any) {
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

- evt ook `retry`, `retryWhen`

```ts
catchError(error => {
  kaartLogger.error("Fout bij opvragen weglocatie", error);
  // bij fout toch zeker geldige observable doorsturen, anders geen volgende events
  return rx.of(XY2AdresError(`Fout bij opvragen weglocatie: ${error}`));
})
```

+++

#### `BehaviourSubject`

- Subject die state bijhoudt. Kan opgevraagd worden

- Observable + member variable

- Anti-patern!

- Om state bij te houden: gebruik `scan`, `startWith`

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


---

## Vbn uit ng-kaart

+++

### Services parallel bevragen

```ts
stableReferentielagen$.pipe(
  switchMap(lgn =>
    stableInfoServices$.pipe(
      switchMap(svcs =>
        geklikteLocatie$.pipe(
          switchMap(locatie =>
            rx.from(allSvcCalls(lgn, svcs, locatie)).pipe(
              mergeAll(5), //
              scan(srv.merge)
            )
          )
        )
      )
    )
  )
)
```

+++

### Custom Observable

```ts
function observableFromOlEvents<A extends ol.events.Event>(olObj: ol.Object, ...eventTypes: string[]): rx.Observable<A> {
  return rx.Observable.create((subscriber: rx.Subscriber<A>) => {
    let eventsKeys: ol.EventsKey[];
    try {
      eventsKeys = olObj.on(eventTypes, (a: A) => subscriber.next(a)) as ol.EventsKey[];
    } catch (err) {
      eventsKeys = [];
      subscriber.error(err);
    }
    return () => {
      kaartLogger.debug("release events ", eventTypes, eventsKeys);
      ol.Observable.unByKey(eventsKeys);
    };
  });
}
```
