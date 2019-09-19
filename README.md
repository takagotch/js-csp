### js-csp
---
https://github.com/ubolonton/js-csp

```js
// test/csp.js



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
});







```

```
```

```
```


