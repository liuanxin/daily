最近的业余时间在看 js 相关的书, 也在极客时间上买了前端相关的专栏, 对于一个非 jser 的人来说, 时时会有一种感觉: js 社区是真的激进和浮燥, 这帮规则的制定者似乎从来也不知道克制为何物. 有很多时候固有的东西是可以处理好的, 但是偏偏喜欢人为制造一些概念和语法糖, 人为的建起一座又一座的高山, 似乎你没跨过就是个 "菜鸡"

请原谅我的恶毒, 看《js 语言精粹》的时候, 这种感觉非常的强烈. 作者是业内的大牛, 还制定了 json, 但是每一章还在最开头引一句莎翁的话, 每一句话要表达的内容总有一种可以将话说成 10 分让一个普通人都能看明白的却偏不只说 3 分剩下的自己去体会. 换成以前, 会觉得这人是一座好大的高山要进行膜拜, 这几年虽然自己技术依然不那么好, 但是还是喜欢去思考一些内在的东西, 更在努力一点一点把心里的权威崇拜给去掉, 再看到这些的时候, "..." 这几个标点符号很容易印在脑子里.

说回标题上来, 除了看书, 看专栏, 找资料, 很久都还是没有把 generator 和 async/await 给理解透, 于是自己试着整个梳理了一下运行的流程

### Generator

我先试着在 yield 后面不跟任何的东西, 可以直接复制到控制台输出
```js
function *f0(param) {
    console.log('n: ' + param);
    yield;
    console.log('i');
    let l = yield;
    console.log('l: ' + l);
}
let v0 = f0('p');
console.log(v0.next(1)); // 输出  n: p  和  {value: undefined, done: false}
console.log('----');
console.log(v0.next(2)); // 输出  i  和  {value: undefined, done: false}
console.log('----');
console.log(v0.next(3)); // 输出  l: 3  和  {value: undefined, done: true}
console.log('----');
console.log(v0.next(4)); // 输出 {value: undefined, done: true}
console.log('----');
console.log(v0.next(5)); // 输出 {value: undefined, done: true}
```
在上面的基础上给方法 return 值
```js
function *f1() {
    console.log('n');
    yield;
    console.log('i');
    let l = yield;
    console.log('l: ' + l);
    return '?';
}
let v1 = f1();
console.log(v1.next(1));     // 输出  n  和  {value: undefined, done: false}
console.log('----');
console.log(v1.next(11));    // 输出  i  和  {value: undefined, done: false}
console.log('----');
console.log(v1.next(111));   // 输出  l: 111  和  {value: '?', done: true}
console.log('----');
console.log(v1.next(1111));  // 输出 {value: undefined, done: true}
console.log('----');
console.log(v1.next(11111)); // 输出 {value: undefined, done: true}
```

然后我试着在 yield 的后面加上内容
```js
function *f2(param) {
    console.log('0: ' + param);
    let f = yield 1;
    console.log('1: ' + f);
    let s = yield f + 2;
    console.log('2: ' + s);
    let t = yield (s + 3);
    console.log('3: ' + t);
    let fo = (yield s) + 4;
    console.log('4: ' + fo);
}
let v2 = f2('p');
console.log(v2.next('N')); // 输出  0: p  和  {value: 1, done: false}
console.log('----');
console.log(v2.next('I')); // 输出  1: I  和  {value: "I2", done: false}
console.log('----');
console.log(v2.next('L')); // 输出  2: L  和  {value: "L3", done: false}
console.log('----');
console.log(v2.next('S')); // 输出  3: S  和  {value: "L", done: false}
console.log('----');
console.log(v2.next('H')); // 输出  4: H4  和  {value: undefined, done: true}
console.log('----');
console.log(v2.next('I')); // 输出  {value: undefined, done: true}
console.log('----');
console.log(v2.next('T')); // 输出  {value: undefined, done: true}
```
最后, 在上面的基础上给方法 return 值
```js
function *f3() {
    console.log('0');
    let y1 = yield 1;
    console.log('1: ' + y1);
    let y2 = yield y1 + 2;
    console.log('2: ' + y2);
    let y3 = yield (y2 + 3);
    console.log('3: ' + y3);
    let y4 = (yield y3) + 4;
    console.log('4: ' + y4);
    return '??';
}
let v3 = f3();
console.log(v3.next('N')); // 输出  0  和  {value: 1, done: false}
console.log('----');
console.log(v3.next('I')); // 输出  1: I  和  {value: "I2", done: false}
console.log('----');
console.log(v3.next('L')); // 输出  2: L  和  {value: "L3", done: false}
console.log('----');
console.log(v3.next('S')); // 输出  3: S  和  {value: "S", done: false}
console.log('----');
console.log(v3.next('H')); // 输出  4: H4  和  {value: "??", done: true}
console.log('----');
console.log(v3.next('I')); // 输出  {value: undefined, done: true}
console.log('----');
console.log(v3.next('T')); // 输出  {value: undefined, done: true}
```

大致上就清楚 yield 的运行逻辑了, 以上面的 f3 为例, 对照上面的输出来看, 它其实是将一个方法分成了这样几段来执行
```js
function *f3() {
    // 调用第 1 次 next('N') 时运行的代码
    console.log('0');
    let y1 = yield 1;
    return 1;                          // | 封装成 {value: 1, done: false} 返回
                                       // |
                                       // | 这两行等同于 let y1 = yield 1;
    // 调用第 2 次 next('I') 时运行的代码 // |
    let y1 = 'I';                      // |
    console.log('1: ' + y1);
    return y1 + 2;                     // | 封装成 {value: "I2", done: false} 返回
                                       // |
                                       // | 这两行等同于 let y2 = yield y1 + 2;
    // 调用第 3 次 next('L') 时运行的代码 // |
    let y2 = 'L';                      // |
    console.log('2: ' + y2);
    return y2 + 3;                     // | 封装成 {value: "L3", done: false} 返回
                                       // |
                                       // | 这两行等同于 let y3 = yield (y2 + 3);
    // 调用第 4 次 next('S') 时运行的代码 // |
    let y3 = 'S';                      // |
    console.log('3: ' + y3);
    return y3;                         // | 封装成 {value: "S", done: false} 返回
                                       // |
                                       // | 这两行等同于 let y4 = (yield y3) + 4;
    // 调用第 5 次 next('H') 时运行的代码 // |
    let y4 = 'H'                       // |
    console.log('4: ' + y4);
    return '??';                       // | 封装成 {value: "??", done: true} 返回
}
```
再回头想一想就知道了, 第一次运行 next('N') 的时候, 传进去的 N 是会被忽略的, 因为第一次 next() 传的值没有 yield 前面来接收. 再去看书也好, 看查到的文章也好, 第一次 next() 都是没有传过参数

感觉 yield 就是为了迭代而生的, 但是却绕成这样, 也不知道这是为哪般! 就这还能新弄一个 for of 玩出花来, 因为每执行 next() 才会执行那一段段, 还美其名曰 "咱们终于可以异步了"


### async/await
再是 es7 开始有的这俩关键字, 看了一个大厂的面试题, 自己为了加深对这两个关键字的理解改了一下成下面这样
```js
async function async1() {
    console.log('A');
    console.log(await async2());
    return 'B';
}
async function async2() {
    console.log('C');
    return 'D';
}
console.log('E');
setTimeout(function() {
    console.log('F');
}, 0);
async1().then(function(r) {
    console.log(r);
});
new Promise(function(resolve, reject) {
    console.log('G');
    resolve();
}).then(function() {
    console.log('H');
});
console.log('I');
```

在 chrome 73.0.3683.75 底下的输出是:
```
// 这个 undefined 的意思应该主要是用来分隔宏任务的, 也就是前面的主线和任务队列是在一起的
E  A  C  G  I  D  H  B  undefined  F
```

在 firefox 60.5.1 底下的输出
```
// 这个 undefined 的意思应该只是用来分隔主线的, 任务队列和宏任务在一起了
E  A  C  G  I  undefined  H  D  B  F
```

在 opera 58.0.3135.107 底下的输出是:
```
// 这个 undefined 应该跟 chrome 里面是一样的
E  A  C  G  I  H  D  B  undefined  F
```
明显 D H B 是比较合理的. 在 firefox 和 opera 的实现中明显是有问题的, 想像得到, 低版本一点的 chrome 也可能是后面的结果


### 终

还有像 var let const 这种一个简单的赋值都能玩出这么多花样(当然, 这可以说是历史遗留问题导致的)

老实说, 我觉得这更多是为了: "别的语言有, 咱们这么前卫的语言当然应该要有!"


...

就这么样个语言, 居然可以流行成这样, 只能说这个世界是真奇妙
