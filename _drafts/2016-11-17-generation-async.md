---
layout: post
title: ES6中的异步编程：Generators函数（二）
date: 2016-11-16 16:00:24.000000000 +08:00
---

[访问原文地址](http://njfeng.com/2016/11/generation-async/)

generators主要作用就是提供了一种，单线程的，很像同步方法的编程风格，方便你把异步实现的那些细节藏在别处。这让我们可以用一种很自然的方式书写我们代码中的流程和状态变化，而不是被迫去遵循那些奇怪的异步编程风格。

换句话说，通过将我们generator逻辑中的一些值的运算操作和和异步处理(使用generators的迭代器iterator)这些值实现细节分开来写，我们可以非常好的把性能处理和业务关注点给分开。

结果呢？所有那些强大的异步代码，将具备跟同步编程一样的可读性和可维护性。

那么我们将如何完成这个壮举？

## 一个简单的异步


由于这个非常的简易，generators并不需要去额外的处理一些这段代码还没有了的异步功能。


举例下，加入现在的异步代码已经是这样的了：

```javascript
function makeAjaxCall(url, cb) {
	//do some ajax fun
	//call cb(result) when complete
}

makeAjaxCall('http://some.url.1', function(result1) {
	var data = JSON.parse(result1);
	
	makeAjaxCall('http://some.url.2/?id='+data.id, function(result2) {
		var resp = JSON.parse(result2);
		console.log('The value you asked for: '+ resp.value);
	});
});
```

用一个generator（不添加任何decoration）去重新实现一遍，代码看这里：

```javascript
function request(url) {
	// this is where we're hiding the asynchronicity,
	// away from the main code of our generator
	// it.next() 是generators的迭代器
	makeAjaxCall(url, function(response) {
		it.next(response);
	});
}

function *main() {
	var result1 = yield request('http://some.url.1');
	var data = JSON.parse(result1);
	
	var result2 = yield request('http://some.url.2/?id='+data.id);
	var resp = JSON.parse(result2);
	console.log('The value you asked for: '+ resp.value);
}

var it = main();
it.next(); //启动所有请求
```


让我们捋一下这是如何工作的


request(..)函数对makeAjaxCall(..)做了基本封装，让数据请求的回调函数中调用generator的iterator的next(...)方法


先来看调用request("..")，你会发现这里根本没有return任何值（换句话来说，这是undefined）。这么没什么大不了，但是它跟我们在这篇文章后面会讨论到的方法做对比是很重要的：我需要有效的在这里使用yield undefined.



这时我们来调用yield（这时还是一个undefined值），其实除了暂停一下我们的generators之外没有做别的了。它将等待知道下次再调用it.next(..) 才恢复，我们队列已经把它在安排在（作为一个回调）Ajax请求结束的时候。

但是当yield表达式执行返回结果后我们做了什么？我们把返回值赋值给了result1.。那为什么yield会有从第一个Ajax请求返回的值？


这是因为当it.next(..)在Ajax的callback中调用的是偶，它传递Ajax的返回结果。这说明这个返回值发送到我们的generators时，已经中间那句`result1 = yield .. `给暂停下来了。

That's really cool and super powerful. In essence, result1 = yield request(..) is asking for the value, but it's (almost!) completely hidden from us -- at least us not needing to worry about it here -- that the implementation under the covers causes this step to be asynchronous. It accomplishes that asynchronicity by hiding the pause capability in yield, and separating out the resume capability of the generator to another function, so that our main code is just making a synchronous(-looking) value request.

The exact same goes for the second result2 = yield result(..) statement: it transparently pauses & resumes, and gives us the value we asked for, all without bothering us about any details of asynchronicity at that point in our coding.

Of course, yield is present, so there is a subtle hint that something magical (aka async) may occur at that point. But yield is a pretty minor syntactic signal/overhead compared to the hellish nightmares of nested callbacks (or even the API overhead of promise chains!).

Notice also that I said "may occur". That's a pretty powerful thing in and of itself. The program above always makes an async Ajax call, but what if it didn't? What if we later changed our program to have an in-memory cache of previous (or prefetched) Ajax responses? Or some other complexity in our application's URL router could in some cases fulfill an Ajax request right away, without needing to actually go fetch it from a server?

We could change the implementation of request(..) to something like this:

var cache = {};

function request(url) {
    if (cache[url]) {
        // "defer" cached response long enough for current
        // execution thread to complete
        setTimeout( function(){
            it.next( cache[url] );
        }, 0 );
    }
    else {
        makeAjaxCall( url, function(resp){
            cache[url] = resp;
            it.next( resp );
        } );
    }
}
Note: A subtle, tricky detail here is the need for the setTimeout(..0) deferral in the case where the cache has the result already. If we had just called it.next(..) right away, it would have created an error, because (and this is the tricky part) the generator is not technically in a paused state yet. Our function call request(..) is being fully evaluated first, and then the yield pauses. So, we can't call it.next(..) again yet immediately inside request(..), because at that exact moment the generator is still running (yield hasn't been processed). But we can call it.next(..) "later", immediately after the current thread of execution is complete, which our setTimeout(..0) "hack" accomplishes. We'll have a much nicer answer for this down below.

Now, our main generator code still looks like:

var result1 = yield request( "http://some.url.1" );
var data = JSON.parse( result1 );
..
See!? Our generator logic (aka our flow control) didn't have to change at all from the non-cache-enabled version above.

The code in *main() still just asks for a value, and pauses until it gets it back before moving on. In our current scenario, that "pause" could be relatively long (making an actual server request, to perhaps 300-800ms) or it could be almost immediate (the setTimeout(..0) deferral hack). But our flow control doesn't care.

That's the real power of abstracting away asynchronicity as an implementation detail.

Better Async
The above approach is quite fine for simple async generators work. But it will quickly become limiting, so we'll need a more powerful async mechanism to pair with our generators, that's capable of handling a lot more of the heavy lifting. That mechanism? Promises.

If you're still a little fuzzy on ES6 Promises, I wrote an extensive 5-part blog post series all about them. Go take a read. I'll wait for you to come back. <chuckle, chuckle>. Subtle, corny async jokes ftw!

The earlier Ajax code examples here suffer from all the same Inversion of Control issues (aka "callback hell") as our initial nested-callback example. Some observations of where things are lacking for us so far:

There's no clear path for error handling. As we learned in the previous post, we could have detected an error with the Ajax call (somehow), passed it back to our generator with it.throw(..), and then used try..catch in our generator logic to handle it. But that's just more manual work to wire up in the "back-end" (the code handling our generator iterator), and it may not be code we can re-use if we're doing lots of generators in our program.
If the makeAjaxCall(..) utility isn't under our control, and it happens to call the callback multiple times, or signal both success and error simultaneously, etc, then our generator will go haywire (uncaught errors, unexpected values, etc). Handling and preventing such issues is lots of repetitive manual work, also possibly not portable.
Often times we need to do more than one task "in parallel" (like two simultaneous Ajax calls, for instance). Since generator yield statements are each a single pause point, two or more cannot run at the same time -- they have to run one-at-a-time, in order. So, it's not very clear how to fire off multiple tasks at a single generator yield point, without wiring up lots of manual code under the covers.
As you can see, all of these problems are solvable, but who really wants to reinvent these solutions every time. We need a more powerful pattern that's designed specifically as a trustable, reusable solution for our generator-based async coding.

That pattern? yielding out promises, and letting them resume the generator when they fulfill.

Recall above that we did yield request(..), and that the request(..) utility didn't have any return value, so it was effectively just yield undefined?

Let's adjust that a little bit. Let's change our request(..) utility to be promises-based, so that it returns a promise, and thus what we yield out is actually a promise (and not undefined).

function request(url) {
    // Note: returning a promise now!
    return new Promise( function(resolve,reject){
        makeAjaxCall( url, resolve );
    } );
}
request(..) now constructs a promise that will be resolved when the Ajax call finishes, and we return that promise, so that it can be yielded out. What next?

We'll need a utility that controls our generator's iterator, that will receive those yielded promises and wire them up to resume the generator (via next(..)). I'll call this utility runGenerator(..) for now:

// run (async) a generator to completion
// Note: simplified approach: no error handling here
function runGenerator(g) {
    var it = g(), ret;

    // asynchronously iterate over generator
    (function iterate(val){
        ret = it.next( val );

        if (!ret.done) {
            // poor man's "is it a promise?" test
            if ("then" in ret.value) {
                // wait on the promise
                ret.value.then( iterate );
            }
            // immediate value: just send right back in
            else {
                // avoid synchronous recursion
                setTimeout( function(){
                    iterate( ret.value );
                }, 0 );
            }
        }
    })();
}
Key things to notice:

We automatically initialize the generator (creating its it iterator), and we asynchronously will run it to completion (done:true).
We look for a promise to be yielded out (aka the return value from each it.next(..) call). If so, we wait for it to complete by registering then(..) on the promise.
If any immediate (aka non-promise) value is returned out, we simply send that value back into the generator so it keeps going immediately.
Now, how do we use it?

runGenerator( function *main(){
    var result1 = yield request( "http://some.url.1" );
    var data = JSON.parse( result1 );

    var result2 = yield request( "http://some.url.2?id=" + data.id );
    var resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
} );
Bam! Wait... that's the exact same generator code as earlier? Yep. Again, this is the power of generators being shown off. The fact that we're now creating promises, yielding them out, and resuming the generator on their completion -- ALL OF THAT IS "HIDDEN" IMPLEMENTATION DETAIL! It's not really hidden, it's just separated from the consumption code (our flow control in our generator).

By waiting on the yielded out promise, and then sending its completion value back into it.next(..), the result1 = yield request(..) gets the value exactly as it did before.

But now that we're using promises for managing the async part of the generator's code, we solve all the inversion/trust issues from callback-only coding approaches. We get all these solutions to our above issues for "free" by using generators + promises:

We now have built-in error handling which is easy to wire up. We didn't show it above in our runGenerator(..), but it's not hard at all to listen for errors from a promise, and wire them to it.throw(..) -- then we can use try..catch in our generator code to catch and handle errors.
We get all the control/trustability that promises offer. No worries, no fuss.
Promises have lots of powerful abstractions on top of them that automatically handle the complexities of multiple "parallel" tasks, etc.

For example, yield Promise.all([ .. ]) would take an array of promises for "parallel" tasks, and yield out a single promise (for the generator to handle), which waits on all of the sub-promises to complete (in whichever order) before proceeding. What you'd get back from the yield expression (when the promise finishes) is an array of all the sub-promise responses, in order of how they were requested (so it's predictable regardless of completion order).

First, let's explore error handling:

// assume: `makeAjaxCall(..)` now expects an "error-first style" callback (omitted for brevity)
// assume: `runGenerator(..)` now also handles error handling (omitted for brevity)

function request(url) {
    return new Promise( function(resolve,reject){
        // pass an error-first style callback
        makeAjaxCall( url, function(err,text){
            if (err) reject( err );
            else resolve( text );
        } );
    } );
}

runGenerator( function *main(){
    try {
        var result1 = yield request( "http://some.url.1" );
    }
    catch (err) {
        console.log( "Error: " + err );
        return;
    }
    var data = JSON.parse( result1 );

    try {
        var result2 = yield request( "http://some.url.2?id=" + data.id );
    } catch (err) {
        console.log( "Error: " + err );
        return;
    }
    var resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
} );
If a promise rejection (or any other kind of error/exception) happens while the URL fetching is happening, the promise rejection will be mapped to a generator error (using the -- not shown -- it.throw(..) in runGenerator(..)), which will be caught by the try..catch statements.

Now, let's see a more complex example that uses promises for managing even more async complexity:

function request(url) {
    return new Promise( function(resolve,reject){
        makeAjaxCall( url, resolve );
    } )
    // do some post-processing on the returned text
    .then( function(text){
        // did we just get a (redirect) URL back?
        if (/^https?:\/\/.+/.test( text )) {
            // make another sub-request to the new URL
            return request( text );
        }
        // otherwise, assume text is what we expected to get back
        else {
            return text;
        }
    } );
}

runGenerator( function *main(){
    var search_terms = yield Promise.all( [
        request( "http://some.url.1" ),
        request( "http://some.url.2" ),
        request( "http://some.url.3" )
    ] );

    var search_results = yield request(
        "http://some.url.4?search=" + search_terms.join( "+" )
    );
    var resp = JSON.parse( search_results );

    console.log( "Search results: " + resp.value );
} );
Promise.all([ .. ]) constructs a promise that's waiting on the three sub-promises, and it's that main promise that's yielded out for the runGenerator(..) utility to listen to for generator resumption. The sub-promises can receive a response that looks like another URL to redirect to, and chain off another sub-request promise to the new location. To learn more about promise chaining, read this article section.

Any kind of capability/complexity that promises can handle with asynchronicity, you can gain the sync-looking code benefits by using generators that yield out promises (of promises of promises of ...). It's the best of both worlds.

runGenerator(..): Library Utility
We had to define our own runGenerator(..) utility above to enable and smooth out this generator+promise awesomeness. We even omitted (for brevity sake) the full implementation of such a utility, as there's more nuance details related to error-handling to deal with.

But, you don't want to write your own runGenerator(..) do you?

I didn't think so.

A variety of promise/async libs provide just such a utility. I won't cover them here, but you can take a look at Q.spawn(..), the co(..) lib, etc.

I will however briefly cover my own library's utility: asynquence's runner(..) plugin, as I think it offers some unique capabilities over the others out there. I wrote an in-depth 2-part blog post series on asynquence if you're interested in learning more than the brief exploration here.

First off, asynquence provides utilities for automatically handling the "error-first style" callbacks from the above snippets:

function request(url) {
    return ASQ( function(done){
        // pass an error-first style callback
        makeAjaxCall( url, done.errfcb );
    } );
}
That's much nicer, isn't it!?

Next, asynquence's runner(..) plugin consumes a generator right in the middle of an asynquence sequence (asynchronous series of steps), so you can pass message(s) in from the preceding step, and your generator can pass message(s) out, onto the next step, and all errors automatically propagate as you'd expect:

// first call `getSomeValues()` which produces a sequence/promise,
// then chain off that sequence for more async steps
getSomeValues()

// now use a generator to process the retrieved values
.runner( function*(token){
    // token.messages will be prefilled with any messages
    // from the previous step
    var value1 = token.messages[0];
    var value2 = token.messages[1];
    var value3 = token.messages[2];

    // make all 3 Ajax requests in parallel, wait for
    // all of them to finish (in whatever order)
    // Note: `ASQ().all(..)` is like `Promise.all(..)`
    var msgs = yield ASQ().all(
        request( "http://some.url.1?v=" + value1 ),
        request( "http://some.url.2?v=" + value2 ),
        request( "http://some.url.3?v=" + value3 )
    );

    // send this message onto the next step
    yield (msgs[0] + msgs[1] + msgs[2]);
} )

// now, send the final result of previous generator
// off to another request
.seq( function(msg){
    return request( "http://some.url.4?msg=" + msg );
} )

// now we're finally all done!
.val( function(result){
    console.log( result ); // success, all done!
} )

// or, we had some error!
.or( function(err) {
    console.log( "Error: " + err );
} );
The asynquence runner(..) utility receives (optional) messages to start the generator, which come from the previous step of the sequence, and are accessible in the generator in the token.messages array.

Then, similar to what we demonstrated above with the runGenerator(..) utility, runner(..) listens for either a yielded promise or yielded asynquence sequence (in this case, an ASQ().all(..) sequence of "parallel" steps), and waits for it to complete before resuming the generator.

When the generator finishes, the final value it yields out passes along to the next step in the sequence.

Moreover, if any error happens anywhere in this sequence, even inside the generator, it will bubble out to the single or(..) error handler registered.

asynquence tries to make mixing and matching promises and generators as dead-simple as it could possibly be. You have the freedom to wire up any generator flows alongside promise-based sequence step flows, as you see fit.

ES7 async
There is a proposal for the ES7 timeline, which looks fairly likely to be accepted, to create still yet another kind of function: an async function, which is like a generator that's automatically wrapped in a utility like runGenerator(..) (or asynquence's' runner(..)). That way, you can send out promises and the async function automatically wires them up to resume itself on completion (no need even for messing around with iterators!).

It will probably look something like this:

async function main() {
    var result1 = await request( "http://some.url.1" );
    var data = JSON.parse( result1 );

    var result2 = await request( "http://some.url.2?id=" + data.id );
    var resp = JSON.parse( result2 );
    console.log( "The value you asked for: " + resp.value );
}

main();
As you can see, an async function can be called directly (like main()), with no need for a wrapper utility like runGenerator(..) or ASQ().runner(..) to wrap it. Inside, instead of using yield, you'll use await (another new keyword) that tells the async function to wait for the promise to complete before proceeding.

Basically, we'll have most of the capability of library-wrapped generators, but directly supported by native syntax.

Cool, huh!?

In the meantime, libraries like asynquence give us these runner utilities to make it pretty darn easy to get the most out of our asynchronous generators!

Summary
Put simply: a generator + yielded promise(s) combines the best of both worlds to get really powerful and elegant sync(-looking) async flow control expression capabilities. With simple wrapper utilities (which many libraries are already providing), we can automatically run our generators to completion, including sane and sync(-looking) error handling!

And in ES7+ land, we'll probably see async functions that let us do that stuff even without a library utility (at least for the base cases)!

The future of async in JavaScript is bright, and only getting brighter! I gotta wear shades.

But it doesn't end here. There's one last horizon we want to explore:

What if you could tie 2 or more generators together, let them run independently but "in parallel", and let them send messages back and forth as they proceed? That would be some super powerful capability, right!?! This pattern is called "CSP" (communicating sequential processes). We'll explore and unlock the power of CSP in the next article. Keep an eye out!

## 参考
-	[Asynchronous calls with ES6 generators](http://blog.mgechev.com/2014/12/21/handling-asynchronous-calls-with-es6-javascript-generators/)
- 	[\[Javascript\] Promise, generator, async與ES6](http://huli.logdown.com/posts/292655-javascript-promise-generator-async-es6)
-  [The Hidden Power of ES6 Generators: Observable Async Flow Control](https://medium.com/javascript-scene/the-hidden-power-of-es6-generators-observable-async-flow-control-cfa4c7f31435#.4cadgop5s)
- [Going Async With ES6 Generators](https://davidwalsh.name/async-generators)