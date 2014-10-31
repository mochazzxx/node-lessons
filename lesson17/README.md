# 《使用 promise 替代回调函数》

## 目标

建立一个 lesson5 项目，在其中编写代码。

代码的入口是 `app.js`，当调用 `node app.js` 时，它会输出 CNode(https://cnodejs.org/ ) 社区首页的所有主题的标题，链接和第一条评论，以 json 的格式。

注意：与上节课不同，并发连接数需要控制在 5 个。

输出示例：

```js
[
  {
    "title": "【公告】发招聘帖的同学留意一下这里",
    "href": "http://cnodejs.org/topic/541ed2d05e28155f24676a12",
    "comment1": "呵呵呵呵"
  },
  {
    "title": "发布一款 Sublime Text 下的 JavaScript 语法高亮插件",
    "href": "http://cnodejs.org/topic/54207e2efffeb6de3d61f68f",
    "comment1": "沙发！"
  }
]
```

## 知识点

1. 理解 Promise 概念，为什么需要 promise
1. 学习 q 的API，利用 q 来替代回调函数

## 课程内容

第五课讲述了如何使用 async 来控制并发。async的本质是一个流程控制。其实在异步编程中，还有一个更为经典的模型，叫做Promise/Deferred模式。本节我们就来学习这个模型的代表实现[q](https://github.com/kriskowal/q)

首先，我们思考一个典型的异步编程模型，考虑这样一个题目：读取一个文件，在控制台输出这个文件内容。

```js
var fs = require('fs');
fs.readFile('sample.txt','utf8',function(err,data){
	console.log(data);
})
```

看起来很简单，再进一步: 读取两个文件，在控制台输出这两个文件内容。

```js
var fs = require('fs');
fs.readFile('sample01.txt','utf8',function(err,data){
	console.log(data);
	fs.readFile('sample02.txt','utf8',function(err,data){
		console.log(data);
	})
});
```

要是读取更多的文件呢?

```js
var fs = require('fs');
fs.readFile('sample01.txt','utf8',function(err,data){
	fs.readFile('sample02.txt','utf8',function(err,data){
		fs.readFile('sample03.txt','utf8',function(err,data){
			fs.readFile('sample04.txt','utf8',function(err,data){
			
			})
		})
	})
})
```

这段代码就是臭名昭著的邪恶金字塔(Pyramid of Doom)。可以使用async来改善这段代码，但是在本课中我们要用promise/defer来改善它。

## promise基本概念
先学习promise的基本概念。

*promise只有三种状态，未完成，完成(fulfiled)和失败(rejected)。
*promise的状态可以由未完成转换成完成，或者未完成转换成失败。
*promise的状态转换只发生一次

promise有一个then方法，then方法可以接受3个函数作为参数。前两个函数对应promise的两种状态fulfiled, rejected的回调函数。第三个函数用于处理进度信息。

```js
promiseSomething().then(function(fulfiled){
		//当promise状态变成fulfiled时，调用此函数
	},function(rejected){
		//当promise状态变成rejected时，调用此函数
	},function(progress){
		//当返回进度信息时，调用此函数
	});
```

学习一个简单的例子
```js
var Q = require('q');
var defer = Q.defer();
/**
 * 获取初始promise
 * @private
 */
function getInitialPromise() {
 return defer.promise;
}
/**
 * 为promise设置三种状态的回调函数
 */
getInitialPromise().then(function(success){
	console.log(success);
},function(error){
	console.log(error);
},function(progress){
	console.log(progress);
});
defer.notify('in progress');//控制台打印in progress
defer.resolve('resolve');   //控制台打印resolve
defer.reject('reject');		//没有输出。promise的状态只能改变一次
```

## promise的传递

then方法会返回一个promise，在下面这个例子中，我们用outputPromise指向then返回的promise。

```js
var outputPromise = getInputPromise().then(function(fulfiled){
	},function(rejected){
	});
```
现在outputPromise就变成了受function(fulfiled)或者function(rejected)控制状态的promise了。怎么理解这句话呢？

* 当function(fulfiled)或者function(rejected)返回一个值，比如一个字符串，数组，对象等等，那么outputPromise的状态就会变成fulfiled。

在下面这个例子中，我们可以看到，当我们把inputPromise的状态通过defer.resovle()变成fulfiled时，控制台输出fulfiled. 

当我们把inputPromise的状态通过defer.reject()变成rejected，控制台输出rejected

```js
	/**
	 * 通过defer获得promise
	 * @private
	 */
	function getInputPromise() {
		return defer.promise;
	}

	/**
	 * 当inputPromise状态由未完成变成fulfil时，调用function(fulfiled)
	 * 当inputPromise状态由未完成变成rejected时，调用function(rejected)
	 * 将then返回的promise赋给outputPromise
	 * function(fulfiled) 和 function(rejected) 通过返回字符串将outputPromise的状态由
	 * 未完成改变为fulfiled
	 * @private
	 */
	var outputPromise = getInputPromise().then(function(fulfiled){
		return 'fulfiled';
	},function(rejected){
		return 'rejected';
	});

	/**
	 * 当outputPromise状态由未完成变成fulfil时，调用function(fulfiled)，控制台打印'fulfiled'。
	 * 当outputPromise状态由未完成变成rejected, 调用function(rejected), 控制台打印'rejected'。
	 */
	outputPromise.then(function(fulfiled){
		console.log(fulfiled);
	},function(rejected){
		console.log(rejected);
	});

	/**
	 * 将inputPromise的状态由未完成变成rejected
	 */
	defer.reject();

	/**
	 * 将inputPromise的状态由未完成变成fulfiled
	 */
	//defer.resolve();
```
* 当function(fulfiled)或者function(rejected)抛出异常时，那么outputPromise的状态就会变成rejected

```JS
	var Q = require('q');
	var fs = require('fs');
	var defer = Q.defer();

	/**
	 * 通过defer获得promise
	 * @private
	 */
	function getInputPromise() {
		return defer.promise;
	}

	/**
	 * 当inputPromise状态由未完成变成fulfil时，调用function(fulfiled)
	 * 当inputPromise状态由未完成变成rejected时，调用function(rejected)
	 * 将then返回的promise赋给outputPromise
	 * function(fulfiled) 和 function(rejected) 通过抛出异常将outputPromise的状态由
	 * 未完成改变为reject
	 * @private
	 */
	var outputPromise = getInputPromise().then(function(fulfiled){
		throw new Error('fulfiled');
	},function(rejected){
		throw new Error('rejected');
	});

	/**
	 * 当outputPromise状态由未完成变成fulfil时，调用function(fulfiled)，控制台打印'[Error:fulfiled]'。
	 * 当outputPromise状态由未完成变成rejected, 调用function(rejected), 控制台打印'[Error:rejected]'。
	 */
	outputPromise.then(function(fulfiled){
		console.log(fulfiled);
	},function(rejected){
		console.log(rejected);
	});

	/**
	 * 将inputPromise的状态由未完成变成rejected
	 */
	defer.reject();

	/**
	 * 将inputPromise的状态由未完成变成fulfiled
	 */
	//defer.resolve();
```

* 当function(fulfiled)或者function(rejected)返回一个promise时，outputPromise就会成为这个新的promise.

这样做有什么意义呢? 主要在于聚合结果(Q.all)，管理延时，异常恢复等等

比如说我们想要读取一个文件的内容，然后把这些内容打印出来。可能会写出这样的代码：

```js
//错误的写法
var outputPromise = getInputPromise().then(function(fulfiled){
			fs.readFile('test.txt','utf8',function(err,data){
				return data;
			});
			},function(rejected){
			throw new Error('rejected');
			});
```

然而由于异步IO的原因，这样写是不行的。需要下面的方式:

```js
	var Q = require('q');
	var fs = require('fs');
	var defer = Q.defer();

	/**
	 * 通过defer获得promise
	 * @private
	 */
	function getInputPromise() {
		return defer.promise;
	}

	/**
	 * 当inputPromise状态由未完成变成fulfil时，调用function(fulfiled)
	 * 当inputPromise状态由未完成变成rejected时，调用function(rejected)
	 * 将then返回的promise赋给outputPromise
	 * function(fulfiled)将新的promise赋给outputPromise
	 * 未完成改变为reject
	 * @private
	 */
	var outputPromise = getInputPromise().then(function(fulfiled){
		var myDefer = Q.defer();
		fs.readFile('test.txt','utf8',function(err,data){
			if(!err && data) {
				myDefer.resolve(data);
			}
		});
		return myDefer.promise;
	},function(rejected){
		throw new Error('rejected');
	});

	/**
	 * 当outputPromise状态由未完成变成fulfil时，调用function(fulfiled)，控制台打印'[Error:fulfiled]'。
	 * 当outputPromise状态由未完成变成rejected, 调用function(rejected), 控制台打印'[Error:rejected]'。
	 */
	outputPromise.then(function(fulfiled){
		console.log(fulfiled);
	},function(rejected){
		console.log(rejected);
	});

	/**
	 * 将inputPromise的状态由未完成变成rejected
	 */
	//defer.reject();

	/**
	 * 将inputPromise的状态由未完成变成fulfiled
	 */
	defer.resolve();
```

## 方法传递

方法传递有些类似于Java中的try和catch。当一个异常没有响应的捕获时，这个异常会接着往下传递。

方法传递的含义是当一个状态没有响应的回调函数，就会沿着then往下找。

* 没有提供function(rejected)

```js
var outputPromise = getInputPromise().then(function(fulfiled){})
```
如果inputPromise的状态由未完成变成rejected, 此时对rejected的处理会由outputPromise来完成。

```js
	var Q = require('q');
	var fs = require('fs');
	var defer = Q.defer();

	/**
	 * 通过defer获得promise
	 * @private
	 */
	function getInputPromise() {
		return defer.promise;
	}

	/**
	 * 当inputPromise状态由未完成变成fulfil时，调用function(fulfiled)
	 * 当inputPromise状态由未完成变成rejected时，这个rejected会传向outputPromise
	 */
	var outputPromise = getInputPromise().then(function(fulfiled){
		return 'fulfiled'
	});
	outputPromise.then(function(fulfiled){
		console.log(fulfiled);
	},function(rejected){
		console.log(rejected);
	});

	/**
	 * 将inputPromise的状态由未完成变成rejected
	 */
	defer.reject('inputpromise rejected'); //控制台打印inputpromise rejected

	/**
	 * 将inputPromise的状态由未完成变成fulfiled
	 */
	//defer.resolve();
```

* 没有提供function(fulfiled)

```js
var outputPromise = getInputPromise().then(null,function(rejected){})
```

如果inputPromise的状态由未完成变成fulfiled, 此时对fulfil的处理会由outputPromise来完成。

```js
	var Q = require('q');
	var fs = require('fs');
	var defer = Q.defer();

	/**
	 * 通过defer获得promise
	 * @private
	 */
	function getInputPromise() {
		return defer.promise;
	}

	/**
	 * 当inputPromise状态由未完成变成fulfil时，传递给outputPromise
	 * 当inputPromise状态由未完成变成rejected时，调用function(rejected)
	 * function(fulfiled)将新的promise赋给outputPromise
	 * 未完成改变为reject
	 * @private
	 */
	var outputPromise = getInputPromise().then(null,function(rejected){
		return 'rejected';
	});

	outputPromise.then(function(fulfiled){
		console.log(fulfiled);
	},function(rejected){
		console.log(rejected);
	});

	/**
	 * 将inputPromise的状态由未完成变成rejected
	 */
	//defer.reject('inputpromise rejected');

	/**
	 * 将inputPromise的状态由未完成变成fulfiled
	 */
	defer.resolve('inputpromise fulfiled'); //控制台打印inputpromise fulfiled
```

* 可以使用fail(function(error))来专门针对错误处理，而不是使用then(null,function(error))

```js
 var outputPromise = getInputPromise().fail(function(error){})
```

看这个例子
```js
	var Q = require('q');
	var fs = require('fs');
	var defer = Q.defer();

	/**
	 * 通过defer获得promise
	 * @private
	 */
	function getInputPromise() {
		return defer.promise;
	}

	/**
	 * 当inputPromise状态由未完成变成fulfil时，调用then(function(fulfiled))
	 * 当inputPromise状态由未完成变成rejected时，调用fail(function(error))
	 * function(fulfiled)将新的promise赋给outputPromise
	 * 未完成改变为reject
	 * @private
	 */
	var outputPromise = getInputPromise().then(function(fulfiled){
		return fulfiled;
	}).fail(function(error){
		console.log(error);
	});
	/**
	 * 将inputPromise的状态由未完成变成rejected
	 */
	defer.reject('inputpromise rejected');//控制台打印inputpromise rejected

	/**
	 * 将inputPromise的状态由未完成变成fulfiled
	 */
	//defer.resolve('inputpromise fulfiled');
```

* 可以使用progress(function(progress))来专门针对进度信息进行处理，而不是使用then(function(success){},function(error){},function(progress){})

```js
var Q = require('q');
var defer = Q.defer();
/**
 * 获取初始promise
 * @private
 */
function getInitialPromise() {
 return defer.promise;
}
/**
 * 为promise设置progress信息处理函数
 */
var outputPromise = getInitialPromise().then(function(success){
	
}).progress(function(progress){
	console.log(progress);
});

defer.notify(1);
defer.notify(2); //控制台打印1，2
```

## promise链

promise链提供了一种让函数顺序执行的方法。

函数顺序执行是很重要的一个功能。比如知道用户名，需要根据用户名从数据库中找到相应的用户，然后将用户信息传给下一个函数进行处理。

```js
    var Q = require('q');
	var defer = Q.defer();
	
	//一个模拟数据库
	var users = [{'name':'andrew','passwd':'password'}];
	
	function getUsername() {
	return defer.promise;
	}

	function getUser(username){
		var user;
		users.forEach(function(element){
			if(element.name === username) {
				user = element;
			}
		});
		return user;
	}

	//promise链
	getUsername().then(function(username){
	 return getUser(username);
	}).then(function(user){
	 console.log(user);
	});

	defer.resolve('andrew');
```

我们通过两个then达到让函数顺序执行的目的。

then的数量其实是没有限制的。当然，then的数量过多，要手动把他们链接起来是很麻烦的。比如

```js
foo(initialVal).then(bar).then(baz).then(qux)
```

这时我们需要用代码来动态制造promise链

```js
var funcs = [foo,bar,baz,qux]
var result = Q(initialVal)
funcs.forEach(function(func){
	result = result.then(func)
})
return result
```

当然，我们可以再简洁一点

```js
var funcs = [foo,bar,baz,qux]
funcs.reduce(function(pre,current),Q(initialVal){
	return pre.then(current)
})
```

看一个具体的例子

```js
				function foo(result) {
					console.log(result);
					return result+result;
				}
				//手动链接
				Q('hello').then(foo).then(foo).then(foo);

				//动态链接
				var funcs = [foo,foo,foo];
				var result = Q('hello');
				funcs.forEach(function(func){
					result = result.then(func);
				});
				//精简后的动态链接
				funcs.reduce(function(prev,current){
					return prev.then(current);
				},Q('hello'));	
```

对于promise链，最重要的是需要理解为什么这个链能够顺序执行。如果能够理解这点，那么以后自己写promise链可以说是轻车熟路啊。

## promise组合

回到我们一开始读取文件内容的例子。如果现在让我们把它改写成promise链，是不是很简单呢？

```js
var Q = require('q'),
	fs = require('fs');
function printFileContent(fileName) {
	return function(){
		var defer = Q.defer();
		fs.readFile(fileName,'utf8',function(err,data){
		  if(!err && data) {
			console.log(data);
			defer.resolve();
		  }
		})
		return defer.promise;
	}
}
//手动链接
printFileContent('sample01.txt')()
			.then(printFileContent('sample02.txt'))
			.then(printFileContent('sample03.txt'))
			.then(printFileContent('sample04.txt'));
```

很有成就感是不是。然而如果仔细分析，我们会发现为什么要他们顺序执行呢，如果他们能够并行执行不是更好吗? 我们只需要在他们都执行完成之后，得到他们的执行结果就可以了。

我们可以通过Q.all([promise1,promise2...])将多个promise组合成一个promise返回。
注意：

1. 当all里面所有的promise都fulfil时，Q.all返回的promise状态变成fulfil
2. 当任意一个promise被reject时，Q.all返回的promise状态立即变成reject

我们来把上面读取文件内容的例子改成并行执行吧

```js
var Q = require('q'),
	fs = require('fs');
/**
 *读取文件内容
 *@private
 */
function printFileContent(fileName) {
		//Todo: 这段代码不够简洁。可以使用Q.denodeify来简化
		var defer = Q.defer();
		fs.readFile(fileName,'utf8',function(err,data){
		  if(!err && data) {
			console.log(data);
			defer.resolve(fileName + ' success ');
		  }else {
			defer.reject(fileName + ' fail ');
		  }
		})
		return defer.promise;
}

Q.all([printFileContent('sample01.txt'),printFileContent('sample02.txt'),printFileContent('sample03.txt'),printFileContent('sample04.txt')])
	.then(function(success){
		console.log(success);
	});
```

现在知道Q.all会在任意一个promise进入reject状态后立即进入reject状态。如果我们需要等到所有的promise都发生状态后(有的fulfil, 有的reject)，再转换Q.all的状态, 这时我们可以使用Q.allSettled

```js
var Q = require('q'),
	fs = require('fs');
/**
 *读取文件内容
 *@private
 */
function printFileContent(fileName) {
		//Todo: 这段代码不够简洁。可以使用Q.denodeify来简化
		var defer = Q.defer();
		fs.readFile(fileName,'utf8',function(err,data){
		  if(!err && data) {
			console.log(data);
			defer.resolve(fileName + ' success ');
		  }else {
			defer.reject(fileName + ' fail ');
		  }
		})
		return defer.promise;
}

Q.allSettled([printFileContent('nosuchfile.txt'),printFileContent('sample02.txt'),printFileContent('sample03.txt'),printFileContent('sample04.txt')])
	.then(function(results){
		results.forEach(
			function(result) {
				console.log(result.state);
			}
		);
	});
```

## 结束promise链

通常，对于一个promise链，有两种结束的方式。第一种方式是返回最后一个promise

如 return foo().then(bar);

第二种方式就是通过done来结束promise链

如 foo().then(bar).done()

为什么需要通过done来结束一个promise链呢? 如果在我们的链中有错误没有被处理，那么在一个正确结束的promise链中，这个没被处理的错误会通过异常抛出。

```js
					var Q = require('q');
					/**
					 *@private
					 */
					function getPromise(msg,timeout,opt) {
						var defer = Q.defer();
						setTimeout(function(){
						console.log(msg);
							if(opt)
								defer.reject(msg);
							else
								defer.resolve(msg);
						},timeout);
						return defer.promise;
					}
					/**
					 *没有用done()结束的promise链
					 *由于getPromse('2',2000,'opt')返回rejected, getPromise('3',1000)就没有执行
					 *然后这个异常并没有任何提醒，是一个潜在的bug
					 */
					getPromise('1',3000)
							.then(function(){return getPromise('2',2000,'opt')})
							.then(function(){return getPromise('3',1000)});
					/**
					 *用done()结束的promise链
					 *有异常抛出
					 */
					getPromise('1',3000)
							.then(function(){return getPromise('2',2000,'opt')})
							.then(function(){return getPromise('3',1000)})
							.done();

```
这次我们要介绍的是 async 的 `mapLimit(arr, limit, iterator, callback)` 接口。另外，还有个常用的控制并发连接数的接口是 `queue(worker, concurrency)`，大家可以去 https://github.com/caolan/async#queueworker-concurrency 看看说明。

这回我就不带大家爬网站了，我们来专注知识点：并发连接数控制。

对了，还有个问题是，什么时候用 eventproxy，什么时候使用 async 呢？它们不都是用来做异步流程控制的吗？

我的答案是：

当你需要去多个源(一般是小于 10 个)汇总数据的时候，用 eventproxy 方便；当你需要用到队列，需要控制并发数，或者你喜欢函数式编程思维时，使用 async。大部分场景是前者，所以我个人大部分时间是用 eventproxy 的。

正题开始。

首先，我们伪造一个 `fetchUrl(url, callback)` 函数，这个函数的作用就是，当你通过

```js
fetchUrl('http://www.baidu.com', function (err, content) {
  // do something with `content`
});
```

调用它时，它会返回 `http://www.baidu.com` 的页面内容回来。

当然，我们这里的返回内容是假的，返回延时是随机的。并且在它被调用时，会告诉你它现在一共被多少个地方并发调用着。

```js
// 并发连接数的计数器
var concurrencyCount = 0;
var fetchUrl = function (url, callback) {
  // delay 的值在 2000 以内，是个随机的整数
  var delay = parseInt((Math.random() * 10000000) % 2000, 10);
  concurrencyCount++;
  console.log('现在的并发数是', concurrencyCount, '，正在抓取的是', url, '，耗时' + delay + '毫秒');
  setTimeout(function () {
    concurrencyCount--;
    callback(null, url + ' html content');
  }, delay);
};
```

我们接着来伪造一组链接

```js
var urls = [];
for(var i = 0; i < 30; i++) {
  urls.push('http://datasource_' + i);
}
```

这组链接的长这样：

![](https://raw.githubusercontent.com/alsotang/node-lessons/master/lesson5/1.png)

接着，我们使用 `async.mapLimit` 来并发抓取，并获取结果。

```js
async.mapLimit(urls, 5, function (url, callback) {
  fetchUrl(url, callback);
}, function (err, result) {
  console.log('final:');
  console.log(result);
});
```

运行输出是这样的：

![](https://raw.githubusercontent.com/alsotang/node-lessons/master/lesson5/2.png)

可以看到，一开始，并发链接数是从 1 开始增长的，增长到 5 时，就不再增加。当其中有任务完成时，再继续抓取。并发连接数始终控制在 5 个。

完整代码请参见 app.js 文件。



