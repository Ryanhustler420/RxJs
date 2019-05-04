# RxJs
RxJs Under The Hood

```javaScript

const {from} = require('rxjs');

// Create a Generic Subscriber in RxJS

let observable$ = from([1, 2, 3, 4, 5]); /*? */

observable$.subscribe(value => {
  console.log(value);
});

const subscriber = {
  next: value => {
    console.log(value);
  },
  complete: value => {
    console.log(value);
  },
  error: value => {
    console.log(value);
  },
};

observable$.subscribe(subscriber);

```

```javaScript
// ==> Extend Subscriber to Override '_next' in RxJS

const {Subscriber} = require('rxjs');

class DoubleSubscriber extends Subscriber {
  _next(value) {
    console.log(value);
    // destination == subscriber object above
    this.destination.next(value * 2);
    // this.destination.complete (value * 2);
    // this.destination.error (value * 2);
  }
}

observable$.subscribe(new DoubleSubscriber(subscriber));
```

```javaScript
// ==> Connect a Source to a Subscriber with RxJS 'pipe

const {Observable} = require('rxjs');

// this is the code below which is different form original source

const o$ = new Observable();
o$.source = observable$;
o$.operator = {
  call(sub, source) {
    source.subscribe(new DoubleSubscriber(sub));
  },
};

o$.subscribe(subscriber);

// we can use pipe for that because it take a source and returns a source

observable$.pipe(source => source).subscribe(subscriber); // for revision

observable$
  .pipe(source => {
    //  dont't do it this way
    const o$ = new Observable();
    o$.source = source;
    o$.operator = {
      call(sub, source) {
        source.subscribe(new DoubleSubscriber(sub));
      },
    };
    return o$;
  })
  .subscribe(subscriber);
```

```javaScript
// ==> Use 'lift' to Connect a 'source' to a 'subscriber' in RxJS

const double = source =>
  source.lift({
    call(sub, source) {
      source.subscribe(new DoubleSubscriber(sub));
    },
  });

observable$.pipe(double).subscribe(subscriber);

```

```javaScript
// ==> Create a Reusable Operator from Scratch in RxJS

// you can pull this override class and multiply pipe into different file and export

class MultiplySubscriber extends Subscriber {
  constructor(subscriber, number) {
    super(subscriber);
    this.number = number;
  }
  _next(value) {
    console.log(this.number);
    console.log(value);
    // destination == subscriber object above
    this.destination.next(value * this.number);
    // this.destination.complete (value * 2);
    // this.destination.error (value * 2);
  }
}

const multiply = number => source =>
  source.lift({
    call(sub, source) {
      source.subscribe(new MultiplySubscriber(sub, number));
    },
  });

observable$.pipe(multiply(3)).subscribe(subscriber);
observable$.pipe(multiply(4)).subscribe(subscriber);

```

```javaScript
// ==> Create Operators from Existing Operators in RxJs

const {map} = require('rxjs/operators');

const multiply = number => map(value => value * number);

observable$.pipe(multiply(3)).subscribe(subscriber);

```

```javaScript
// ==> Implement the 'map' Operator from Scratch in RxJS

class MapSubscriber extends Subscriber {
  constructor(sub, fn) {
    super(sub);
    this.fn = fn;
  }
  _next(value) {
    this.destination.next(this.fn(value));
  }
}

const map = fn => source =>
  source.lift({
    call(sub, source) {
      // overriding the source value through function expression
      // here fn = value * number as passed withen map
      // console.log (source);
      source.subscribe(new MapSubscriber(sub, fn));
    },
  });

const multiply = number => map(value => value * number);

observable$.pipe(multiply(3)).subscribe(subscriber);
```

```javaScript
// ==> Chain RxJS Operators Togather with a Custom 'Pipe' Function using Array.reduce

const {pipe} = require('rxjs');
const {map, filter} = require('rxjs/operators');

const pipe = (...fns) => source => fns.reduce((acc, fn) => fn(acc), source);

const multiply = number =>
  pipe(
    map(value => value * number),
    filter(value => value < 10)
  );

observable$.pipe(multiply(3)).subscribe(subscriber);
```

```javaScript
// ==> Implement RxJS 'mergeMap' through inner Observables to Subscribe and Pass Values Through

// Pre Defined Way

const {fromEvent} = require('rxjs');
const {scan} = require('rxjs/operators');

observable$ = fromEvent(document, 'click').pipe(scan(i => i + 1, 0));

observable$.subscribe(subscriber);

// Creating Our Own Merge Map

class MyMergeMapSubscriber extends Subscriber {
  constructor(sub, fn) {
    super(sub);
    this.fn = fn;
  }

  _next(value) {
    console.log(`outer`, value);
    const o$ = this.fn(value);
    o$.subscribe({
      next: value => {
        console.log(` inner`, value);
        this.destination.next(value);
      },
    });
  }
}

const mergeMap = fn => source =>
  source.lift({
    call(sub, source) {
      source.subscribe(new MyMergeMapSubscriber(sub, fn));
    },
  });

const {fromEvent, of} = require('rxjs');
const {scan, delay} = require('rxjs/operators');
// const {scan, delay, mergeMap} = require('rxjs/operators');

observable$ = fromEvent(document, 'click').pipe(
  scan(i => i + 1, 0),
  mergeMap(value => of(value).pipe(delay(500)))
);

observable$.subscribe(subscriber);
```

```javaScript
// ==> Implement RxJS 'switchMap' by Canceling Inner Subscriptions as Values are Passed Through

class MySwitchMapSubscriber extends Subscriber {
  innerSubscription;

  constructor(sub, fn) {
    super(sub);
    this.fn = fn;
  }

  _next(value) {
    console.log(`outer`, value);

    if (this.innerSubscription) {
      this.innerSubscription.unsubscribe();
    }

    const o$ = this.fn(value);
    this.innerSubscription = o$.subscribe({
      next: value => {
        console.log(` inner`, value);
        this.destination.next(value);
      },
    });
  }
}

const switchMap = fn => source =>
  source.lift({
    call(sub, source) {
      source.subscribe(new MySwitchMapSubscriber(sub, fn));
    },
  });

const {fromEvent, of} = require('rxjs');
const {scan, delay} = require('rxjs/operators');
// const {scan, delay, switchMap} = require('rxjs/operators');

observable$ = fromEvent(document, 'click').pipe(
  scan(i => i + 1, 0),
  switchMap(value => of(value).pipe(delay(500)))
);

observable$.subscribe(subscriber);
```

```javaScript
// ==> Implement RxJS 'concatMap' by Waiting for Inner Subscriptions to Complete

class MyConcatMapSubscription extends Subscriber {
  constructor(sub, fn) {
    super(sub);
    this.innerSubscription = null;
    this.fn = fn;
    this.buffer = [];
  }

  _next(value) {
    const {isStopped} = this.innerSubscription || {isStopped: true};

    if (!isStopped) {
      this.buffer = [...this.buffer, value];
    } else {
      const o$ = this.fn(value);
      this.innerSubscription = o$.subscribe({
        next: value => {
          this.destination.next(value);
        },
        complete: () => {
          console.log(this.buffer);
          if (this.buffer.length) {
            const [first, ...rest] = this.buffer;
            this.buffer = rest;
            this._next(first);
          }
        },
      });
    }
  }
}

// Example

const arr = [1, 2, 3, 4, 6];
const [first, ...rest] = arr;
first; // 1
rest; // [2,3,4,5,6]

const MyConcatMap = fn => source =>
  source.lift({
    call(sub, source) {
      source.subscribe(new MyConcatMapSubscription(sub, fn));
    },
  });

const {fromEvent, of} = require('rxjs');
const {scan, delay} = require('rxjs/operators');
// const {scan, delay, concatMap} = require('rxjs/operators');

observable$ = fromEvent(document, 'click').pipe(
  scan(i => i + 1, 0),
  MyConcatMap(value => of(value).pipe(delay(500)))
);

observable$.subscribe(subscriber);
```
```javaScript
// ==> 'add' Inner Subscriptions toOuter Subscribers to 'unsubscribe' in RxJS

class MyConcatMapSubscription extends Subscriber {
  constructor(sub, fn) {
    super(sub);
    this.innerSubscription = null;
    this.fn = fn;
    this.buffer = [];
  }

  _next(value) {
    const {isStopped} = this.innerSubscription || {isStopped: true};

    if (!isStopped) {
      this.buffer = [...this.buffer, value];
    } else {
      const o$ = this.fn(value);
      this.innerSubscription = o$.subscribe({
        next: value => {
          this.destination.next(value);
        },
        complete: () => {
          console.log(this.buffer);
          if (this.buffer.length) {
            const [first, ...rest] = this.buffer;
            this.buffer = rest;
            this._next(first);
          }
        },
      });
    }

    this.add(this.innerSubscription);
  }
}

const MyConcatMap = fn => source =>
  source.lift({
    call(sub, source) {
      source.subscribe(new MyConcatMapSubscription(sub, fn));
    },
  });

const {fromEvent, of} = require('rxjs');
const {scan, delay, takeUntil} = require('rxjs/operators');

observable$ = fromEvent(document, 'click').pipe(
  scan(i => i + 1, 0),
  MyConcatMap(value => of(value).pipe(delay(500))),
  takeUntil(fromEvent(document, 'keydown'))
);

observable$.subscribe(subscriber);
```
