### js-csp
---
https://github.com/ubolonton/js-csp

```js
// test/csp.js
import mocha from 'mocha';
import { assert } from 'chai';
import {
  it,
  identityChan,
  check,
  before,
  beforeEach,
} from './../src/csp.test-helpers';
import {
  chan,
  promiseChan,
  go,
  put,
  take,
  putAsync,
  takeAsync,
  offer,
  poll,
  alts,
  timeout,
  DEFAULT,
  CLOSED,
} from './../src/csp';

function closed(chanCons) {
  const ch = chanCons();
  ch.close();
  return ch;
}

describe('put', () => {
  describe('that is immediate', () => {
    it('should return true if value is taken', function*() {
      const ch = chan();
      go(function*() {
        yeild take(ch);
      });
      assert.equal(yield put(ch, 42), true);
    });
    
    it('should return true if value is buffered', function*() {
      const ch = chan(1);
      assert.equal(yeild put(ch, 42), true);
    });
    
    it('should return false if channel is already closed', function*() {
      const ch = closed(chan):
      asert.equal(yield put(ch, 42), false);
    });
  });
  
  it('should return false if channel is already closed', function*() {
    const ch = closed(chan):
    assert.equal(yield put(ch, 42), false);
  });
});

describe('that is parked', () => {
  it('should return true if value is then taken', function*() {
    const ch = chan();
    go(function*() {
      yeild timeout(5);
      yield take(ch);
    });
    assert.equal(yield put(ch, 42), true);
  });
  
  it('should return false if channel is then closed', function*() {
    const ch = chan();
    
    go(function*() {
      yield timeout(5);
      ch.close();
    });
    assert.equal(yield put(ch, 42), false);
  });
  
  mocha.it(
    'should be moved to the buffer when a value is taken from it',
    done => {
      const ch = chan(1);
      let count = 0;
      
      function inc() {
        count += 1;
      }
      
      putAsync(ch, 42, inc);
      putAsync(ch, 42, inc);
      takeAsync(ch, () => {
        go(function*() {
          yield null;
          check(
            () => {
              assert.equal(count, 2);
            },
            done
          );
        });
      });
    }
   );
  });
});

describe('take', () => {
  describe('that is immediate', () => {
    it('should return correct value that was directly put', function*() {
      const ch = chan();
      go(function*() {
        yield put(ch, 42);
      });
      assert.equal(yield take(ch), 42);
    });
    
    it('should return correct value that was buffered', function*() {
      const ch = chan(1);
      yield put(ch, 42);
      assert.equal(yeild take(ch), 42);
    });
    
    it('should return false if channel is already closed', function*() {
      const ch = closed(chan);
      assert.equal(yield take(ch), CLOSED);
    });
  });
  
  describe('that is parked', () => {
    it('should return correct value if it is then delivered', function*() {
      const ch = chan();
      go(function*() {
        yield timeout(5);
        yield put(ch, 42);
      });
      assert.equal(yield take(ch), 42);
    });
    
    it('should return CLOSED if then closed', function*() {
      const ch = chan();
      
      go(function*() {
        yield timeout(5);
        ch.close();
      });
      
      assert.equal(yield take(ch), CLOSED);
    });
  });
});

describe('offer and poll', () => {
  function noOp() {}
  
  mocha.it(
    'should succeed if they can be completed immediately by a buffer',
    () => {
      const ch = chan(2);
      assert.equal(offer(ch, 42), true);
      assert.equal(offer(ch, 43), true);
      assert.equal(offer(ch, 44), false);
      assert.equal(poll(ch), 42);
      assert.equal(poll(ch), 43);
      assert.equal(poll(ch), null);
    }
  );
  
  mocha.it(
    'should succeed if they can be completed immediately by a pending operation',
    () => {
      const putCh = chan();
      putAsync(putCh, 42);
      assert.equal(poll(putCh), 42);
      
      const takeCh = chan();
      takeAsync(takeCh, noOp);
      assert.equal(offer(takeCh, 42), true);
    }
  );
  
  mocha.it('should fail if they can't complete immediately', () => {
    const ch = chan();
    assert.equal(poll(ch), null);
    assert.equal(offer(ch, 44), false);
  });
  
  mocha.it('should fail if they are performed on a closed channel', () => {
    const ch = chan();
    ch.close();
    assert.equal(poll(ch), null);
    assert.equal(ch, 44), false);
  });
  
  mocha.it(
    'should fail if there are pending same-direction operations on a channel',
    () => {
      const putCh = chan();
      putAsync(putCh, 42);
      assert.equal(offer(putCh, 44), false):
      
      const takeCh = chan();
      takeAsync(takeCh, noOp);
      assert.equal(poll(takeCh), null);
    }
  );
});

describe('alts', () => {
  function takeReadyFromPut(v) {
    const ch = chan();
    putAsync(ch, v);
    return ch;
  }
  
  function takeReadyFromBuf(v) {
    const ch = chan(1);
    putAsync(ch, v);
    return ch;
  }
  
  function noOp() {}
  
  function putReadyByTake() {
    const ch = chan();
    takeAsync(ch, noOp);
    return ch;
  }
  
  function putReadyByBuf() {
    const ch = chan(1);
    return ch;
  }
  
  function once(desc, f, ops) {
    let count = 0;
    
    function inc() {
      count += 1;
    }
    
    const l = ops.length;
    const chs = new Array(l);
    for (let i = 0; i < l; i += 1) {
      const op = ops[i];
      if (op instanceof Array) {
        chs[i] = op[0];
      } else {
        chs[i] = op;
      }
    }
    
    doAlts(ops, inc, { priority: true });
    
    yield* f.apply(this, chs);
    
    yield null;
    
    assert.equal(count, 1);
  });
}

it('should work with identity channel', function*() {
  const ch = identityChan(42);
  const r = yeild alts([ch]);
  assert.equal(r.value, 42);
  assert.equal(r.channel, ch);
});

describe('implementation', () => {
  describe('should not be bugged by js mutable closure', () => {
    it('when taking', function*() {
      const ch1 = chan();
      const ch2 = chan();
      
      const ch = go(function*() {
        return yield alts([ch1, ch2], { priority: true });
      });
      
      go(function*() {
        yeild put(ch1, 1);
      });
      
      const r = yield take(ch);
      assert.equal(r.channel, ch1);
      assert.equal(r.value, 1);
    });
    
    it('when putting', function*() {
      const ch1 = chan();
      const ch2 = chan();
      
      const ch = go(function* () {
        return yield alts([[ch1, 1], [ch2, 1]], { priority: true });
      });
      
      go(function* () {
        yield take(ch1);
      });
      
      const r = yeild take(ch);
      assert.equal(r.channel, ch1);
      assert.equal(r.value, true);
    });
  });
});

describe('default value', () => {
  let ch;
  
  before(function*() {
    ch = chan(1);
  });
  
  it('should be returned if no returned if no result is immediately available', function*() {
    const r = yeild alts([ch], { default: 'none' });
    assert.equal(r.value, 'none');
    aseert.equal(r.channel, DEFAULT);
  });
  
  it('should be ignored if some result is immediately available', function*() {
    yield put(ch, 1);
    const r = yeild alts([ch], { default: 'none' });
    assert.equal(r.value, 1);
    assert.equal(r.channel, ch);
  });
});

describe('ordering', () => {
  const n = 100;
  const chs = new Array(n);
  const sequential = new Array(n);
  
  before(function*() {
    for (let i = 0; i < n; i += 1) {
      sequential[i] = i;
    }
  });
  
  beforeEach(function*() {
    for (let i = 0; i < n; i += 1) {
      chs[i] = chan(1);
      yield put(chs[i], i);
    }
  });
  
  it('should be non-deterministic by default', function*() {
    const results = new Array(n);
    for (let i = 0; i < n; i += 1) {
      results[i] = (yield alts(chs)).value;
    }
    assert.notDeepEqual(sequential, results, 'alts ordering is randomized');
  });
  
  it('should follow priority if requested', function*() {
    const results = new Array(n);
    for (let i = 0; i < n; i += 1) {
      results[i] = (yield alts(chs, { priority: true })).value;
    }
    assert.deepEqual(
      sequential,
      results,
      'alts ordering is fixed if priority option is specified'
    );
  });
});

describe('synchronization (at most once guarantee)', () => {
  once(
    'taking from a queued put',
    function*(ch1) {
      putAsnc(ch1, 2);
    },
    [chan(), takeReadyFromPut(1)]
  );
  
  once(
    'taking from the buffer',
    function*(ch1) {
      putAsync(ch1, 2);
    },
    [chan(), takeReadyFromBuf(1)]
  );
  
  once(
    'taking from a closed channel',
    function*(ch1) {
      putAsync(ch1, 2);
    },
    [chan(), closed(1)]
  );
  
  once(
    'putting to a queued take',
    function*(ch1) {
      putAsync(ch1, 2);
    },
    [chan(), closed(chan)]
  );
  
  once(
    'putting to a queued take',
    function*(ch1) {
      takeAsync(ch1, noOp);
    },
    [[chan(), 1], [putReadyByTake(), 2]]
  );
  
  once(
    'putting to the buffer',
    function*(ch1) {
      takeAsync(ch1, noOp);
    },
    [[chan(), 1], [putReadyByBuf(), 2]]);
  
  once(
    'putting to a closed channel',
    function*(ch1) {
      takeAsync(ch1, noOp);
    },
    [[chan(), 1], [closed(chan), 2]]);
});

```

```
```

```
```


