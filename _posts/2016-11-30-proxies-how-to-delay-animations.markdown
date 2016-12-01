---
layout: post
title:  "Proxies: How to Delay Animations"
date:   2016-11-30 22:13:48 +0100
categories: angularjs angular2 proxy
---

# Proxies: How to Delay Animations

When working with animations in a large application, we often need to simulate slow responses from webservers 
This is also to show how ES6 [Proxy](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Proxy) is great and teach you how it works, by example. 

## AngularJS without Proxy

We can use angulars `$q` service to slow down or simulate a webservice as a temporary mock. 
In my case i needed to work on some animations and check that they looked nice in the app with various load times. 
So by utilizing the `$q` service and `setTimeout` we can quickly make the response as slow as we like. 
I used a local webserver so the response was too quick for real life appliance. 

```typescript
export class WebService {

  static $inject: string[] = ["$q", "$http"];
  constructor(
        private $q,
        private $http
    ) {}

    someMethod() {
        let promise = this.$http.get("someurl", (response) => {
            // Do something with the response
        })

        // The normal code would return promise here
        // return promise

        let deferred = this.$q.defer();

        setTimeout(() => {
                console.log("Fixing Stuff");
                deferred.resolve(promise);
            },
            2000);
        return deferred.promise;

    }

}

app.service("WebService", WebService)
```

Arguably, this is not the best example as it modifies production code to debug, which should be avoided when possible. 



## Using Proxies 
We can do better using `Proxy`. The following is also more framework agnostic. 

This other method of defering has some benefits, such as: 

 * it is less obtrusive
 * it can easily be disabled

The downside is it relies heavily on ES6 (but is that a problem anymore?)

### How does Proxies work?
The `Proxy` type allows for easy manipulation of objects. 
it has the following interface.

```javascript
var p = new Proxy(target, handler);
```
The `target`is the object we like to manipulate and the `handler` is a object that defines several traps. 
In the example below we use the `get` trap, which as you might hve guessed, is called each time we get a property. 
In JavaScript this includes calling a method, as we actually get the property and then call the method. 
There is not "call" trap as the proxy cannot know if we call the property. 


### Proxy example
Lets say we have some service defined as a class:

```javascript
let nameService = {
    getHelloString : (firstname, lastname) => {
        return new Promise((resolve, reject) => {
      	    resolve("Hello, " + firstname + " " + lastname)
      })
   }
}
```
Of course the promise would normally be some asynchronious call to a webserver or similar. 
The typicall usecase for such a service is to inject it into some component. 


```javascript 
let nameServiceProxy = new Proxy(nameService, {

    // Define a get trap. 
    // The params are the target (i.e. NameService) and the name of the property.
    get: (target, name) => {

        // Test if it is a function
        if(isFunction(target[name])) {

            // Return a method than returns a promise
            return (...args) => new Promise((resolve, reject) => {

                setTimeout(() => {
                    // After 2000ms invoke the original method with the args
                    resolve(target[name](...args))
                }, 2000)
            })
        }

        // In case it isn't a function, return the target
        return target[name]
    }
})

// included here for completeness
let isFunction = (obj) => {
  return !!(obj && obj.constructor && obj.call && obj.apply);
};

```

Ideally I would like to extend `Proxy` to create a generic version easily, 
however, Proxy is a special type in javascript and 
does not have a `[[prototype]]` property we can extend. 

For th example to be usefull we need: 

 * A way to set the timeout via the console. 
 * Make it generic and easy to put into code. 
 * Ensure it never reaches production 
 * Make it not fail for methods that do not return promises

## A much more usefull example
The above examples show how proxies work, 
but they do not create a good solution for solving this problem.

The following example is easy to use as it comes with a generator that makes it easy to create a proxy. 

```javascript

// Define a get trap. 
// The params are the target (i.e. NameService) and the name of the property.
var getTrap = (target, name) => {

    // Test if it is a function
    if(!isFunction(target[name])) {
        // In case it isn't a function, return the target
        return target[name]
    }

    // Return a method than returns a promise
    return (...args) => { 
        
        // call the original method
        let result = target[name](...args);

        // Test if the result is a promise (has a then method)
        if (!isFunction(result["then"])) { 
            // If not we just return it 
            return result
        }

        return new Promise((resolve, reject) => {
            setTimeout(() => {
                // After 2000ms invoke the original method with the args
                resolve(result)
            }, getDelayTime())
        
        })
    }
}


let createProxy = (target) => {
    return new Proxy(target, {
        get: getTrap
    })
}


// included here for completeness
let isFunction = (obj) => {
    return !!(obj && obj.constructor && obj.call && obj.apply);
};

// Fetch delay time from sessionStorage
let getDelayTime = () => {
    return window.sessionStorage.getItem("delay") || 2000
}

```

Creating a new proxy is simple

```javascript
var nameServiceProxy = createProxy(nameService);
```

This example solves all the issues except for disabling in production.
But ensuring this is very context dependent and should not pose a problem for a developer. 


