#References:
-  JakeVide0 2018 JSCONF(little complicated): https://www.youtube.com/watch?v=cCOL7MC4Pl0
    +  His article with animated examples for promise(microtask queues understanding): https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
- Philip Roberts: What the heck is the event loop anyway (All content is picked from this video mostly): What the heck is the event loop anyway



#1) What is javascript
- Its a
    + Single threaded
    + Non-blocking
    + asynchronous
    + concurrent
language

- It has 
    + call stack
    + an event loop
    + callback queue
    + some other apis
    

#2) What is V8?
- As of now 5/5/2018, its JS runtime environment for Chrome
![](event_loop/event_loop0.png)
![](event_loop/event_loop1.png)

#3) Is javascript single threaded
Yes, see image how it runs in a single thread:
![](event_loop/event_loop2.png)
![](event_loop/event_loop3.png)

#4) What are blocking instructions
Anything which is relatively slow is blocking. We don't have a hard definition of slow here. Some examples
- console.log() -> NON-BLOCKING
- n/w request -> BLOCKING
- while loop 1-1000000000 is BLOCKING

#5) What would have happend if Javascript was blocking(no async) ?
Conside image ![](event_loop/event_loop4.png),
First n/w req $.getSync('//foo.com') will move on stack and then finish execution and pop out of stack

Then again second n/w req will be added to stack and so on

So problem here is that while the n/w request is being processed on stack, browser UI will be freezed as other js related to UI(like button will be freezed) will be blocked.

#6) How does javascript deal with blocking n/w requsets on a single thread stack ?
- using asynchronous callbacks
- See: ![](event_loop/event_loop5.png) . The setTimeout function has a callback defined which is called after 5 seconds.
- So first console.log("Hi") is pushed and processed(popped out). Then timeout function call is placed on stack. But while it is waiting this call popped from execution stack and is pushed on to callback queue(will discuss this later). 
- Then console.log("JsconfEU") is pushed on stack and processed(popped out of stack)
- Then event loop(will discuss later) pushes the timeout function call from callback queue on top of execution stack and processes it.

#7) How does concurrency actually happen in browser with blocking n/w requests
- See image: ![](event_loop/event_loop6.png)
- Chrome has a JS runtime environment shown as stack and heap
- At the same time there is a WEB-API section which handles DOM, AJAX requests and setTimeout function calls.
- So this AJAX and setTimeouts can execute on a parallel thread offered by chrome. This might be done to make execution of browser faster for n/w requests

For Example see the images for the example mentioned above
```
console.log("Hi");

setTimeout(function cb() {
    console.log("There")
    });

console.log("JSConfEU");
```

- console.log("Hi"); is pushed on stack ->   [![](event_loop/event_loop7.png)
- After previous statement is processed and popped out, setTimeout(....) is pushed on top of stack   [![](event_loop/event_loop8.png)]
    + Now setTimeout() is an API provided by Chrome browser, so it is passed on to the WebAPi section and next call console.log("JSConfEU") is placed on to stack : [![](event_loop/event_loop11.png)]
    + console.log("JSConfEU"); gets executed and pooped out of stack  [![](event_loop/event_loop9.png)]
    + When WebAPI is done processing timeout of 5 seconds, it places the callback on task queue [![](event_loop/event_loop10.png)]

**NOTE: Job of the EVENT LOOP is to monitor the stack and task queue. As soon as a callback is placed on event queue, it starts monitoring stack to become empty. once stack gets empty it places the callback back onto stack**

    + Since the stack got empty after printing "JSConfEU" and callback is placed on task queue, event loop does its job of moving callback from task queue to stack of execution where the callback function gets executed. [![](event_loop/event_loop12.png)]

Your XHR/Network request also works in the same way in chrome: [![](event_loop/event_loop13.png)]


#8) If setTimeout(callbackFunction, 8000) is called 4 times then,  will the 4th timeout command execute exactly after 36000ms ?
- No the value provided in timeout is NOT the exact time to execution. It is the minimum amount after which the callback will be processed.
- Thats because say, when 4th timeout goes into task queue and at that time two previous timeouts are already waiting ahead of 4th timeout.
- S0 although 4th timeout was placed on event queue on time i.e after 36000ms, there might be an additional delay due to task queue having additional commands to process . See: [![](event_loop/event_loop14.png)]  


#9) What are callbacks ?
- Callback is generally a function that another function calls
OR
- Callback can be an asynchronous callback which will be pushed on to the task queue


- Consider the code below

```
//Synchronous code
[1,2,3,4].forEach( function(i){
    console.log(i + "processing sync");
    delay();
    });

//Asynchronous
function asyncForEach(array, callback) {
    array.forEach(function() {
        setTimeout(callback, 0)
        })
}

asyncForEach( [1,2,3,4], function() {
    console.log("processing async");
    delay;
    })

```


#10) What is a render queue
- Apart from task queue, there is a render queue which is responsible for painting the DOM on the screen. 
- It has a higher priority for being processed than the task queue
- Like task queue, it needs to move its paint job on the stack of execution to paint out the DOM updation
- But like the task queue, it gets a chance to go to stack of execution only when stack is empty.
- Whenever the browser senses the need to update DOM it passes in a job to Render queue
- Normally screen refreshes every 1/60 th of a second(a job is pushed to render queue every 1/60th of second)
- Example is if you have a gif running on your page, then render queue is responsible for rendering its animation every 1/60th of a second

# 11) How can render queue be blocked
- If a loop is called in synchronously, the render queue gets blocked and the content on the page freezes.
- In the above example 
```
//Synchronous code
[1,2,3,4].forEach( function(i){
    console.log(i + " processing sync");
    delay();
    });
```

- while the above code runs the stack first places the forEach function call on stack
S1: forEach
- Then callback function is called for each of the item in array and gets processed synchronously
- S2: function(1) . It logs console.log(i + " processing sync"); -> 1 processing sync
- function(1) completes and function(2) is pushed on top of stack
S2: function(2) .... and this goes on till 4
- Finally S2: function(4) is popped and then S1: forEach() is popped
- But all this time while stack was not empty, the render queue could not execute its job of painting and UI will look freezed for this duration
- When stack is finally empty after the sync code executes, render queue pushes its paint job on stack of execution and repaints the DOM

#12) how does async code helps unblock render queue
- Consider code
```
//Asynchronous
function asyncForEach(array, callback) {
    array.forEach(function(i) {
        setTimeout(callback, i)
        })
}

asyncForEach( [1,2,3,4], function() {
    console.log("processing async");
    delay;
    })

```

S1: script main is pushed on stack (executing file is considered a function in itself [main function])
S2: asyncForEach() function call is pushed on screen
S3: forEach() call is placed
S4: setTimeout(callback, 1) is called for loop:1
- setTimeout being an async call is immediately sent to WebAPI for handling its timeout and pushed off the stack
**NOTE: This is it. Here we are not blocking the callstack but allowing our time taking callback function(which contains delay) to be moved to task queue**
- so S4 is poppped for loop:1
S4: setTimeout(callback, 2) is called for loop:2
- again its popped immediately ..... and so on
- S4 is therafter similarly pushed in for setTimeout(callback, 3), setTimeout(callback, 4) and then popped out for it to behandled by WebAPI
- S3 forEach() is complete so it pops out
- S2 asyncForEach() is completed processing so it pops out
- main() function is done processing all lines so it pops ot

- All this while, the four callbacks for loop(1,2,3,4) is queued in task queue and are waiting be processed as and when stack becomes empty.

- But here is the catch, render queue is also waiting for DOM refresh event after 1/60th of second. So it gets the priority of executing on stack

- Now once render queue job gets processed, stack is empty again and this time the callback for loop:1 gets its chance to move on stack of execution

- And subsequently render queue and callback queue will take turns to execute on stack of execution.

**So this is how async code does not block rendering of DOM**

#13) Is there a seperate queue for promises ?
- Yes I am not getting much deep into it right now
- But yes there is a separate queue for microtasks(promises happen to be a microtask only)
- microtask queue has precedence over task queue for execution
- promises in microtask queue will wait for stack to be empty and once it gets empty they are processed first before any other TASK on task queue gets a chance.

For more: refer to Jake's video and blog refereed on top of this file.

Thanks!!
