被单元测试偶尔故障的问题困扰了一下午，问题体现为多个测试并发执行导致数据被污染。在stackoverflow上看到的解答：

****

- Operation synchronous, Mocha test synchronous.

```js
it("reads foo", function () {
    fs.readFileSync("foo");
    // ssert what we want to assert.
});

```
No problem.

- Operation synchronous, Mocha test asynchronous.

```js
it("reads foo", function (done) {
    fs.readFileSync("foo");
    // Assert what we want to assert.
    done();
});
```
It is pointless to have the Mocha test be asynchronous, but no problem.

- Operation asynchronous, Mocha test asynchronous.

```js
it("reads foo", function (done) {
    fs.readFile("foo", function () {
        // Assert what we want to assert.
        done();
    });
});

```
No problem.

Operation asynchronous, Mocha test synchronous.

```js
it("reads foo", function () {
    fs.readFile("foo", function () {
        // Assert what we want to assert.
    });
});

```

This is a problem. Mocha will return right away from the test callback and call it successful (assuming fs.readFile does not raise an exeption). The asynchronous operation will still be scheduled and the call back may still be called later. One important point here: Mocha does not have the power to make the operations it tests synchronous. So making the Mocha test synchronous has no effect on the operations in the test. If they are asynchronous, they will remain asynchronous, no matter what we tell Mocha.

****

简单总结一下就是：

mocha执行时分为同步和异步两种，通过`it(title,fn)`中的fn.length，也就是fn的args长度来判断是否异步执行。若声明为异步执行，则会等待整个it执行结束后再执行下一个it。而我犯的错正是示例中的第四种，mocha并不知道我执行的异步测试，于是就简单地执行了fn，挂了个异步任务没有报错后，认为it执行结束了，马上去执行下一个it，导致两个it同时执行，就会出现一定几率数据污染的测试错误。

如果it方法是`it(title,fn(done))`这样的话如果不执行`done()`异步是不会停止的，而这里的`done`如果接受到参数则会视为报错。