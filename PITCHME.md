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


--

## RxJS in de praktijk


---

### Pipe

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

