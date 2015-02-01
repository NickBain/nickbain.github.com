---
layout: post
title: Constructing Objects in JavaScript
category: JavaScript
articleID : 1
---

I recently watched Doug Crockford's talk "[The Better Parts] [3]", during which he commented that he no longer prefers the `Object.create()` method to construct object - a technique he popularised some time ago. He now favours a pattern he calls **Parasitic Inheritance**. 

I want to take a look at this pattern and why we may use it to over conventional techniques.

##Constructor Invocation##
In most programming languages we create new objects from classes which can inherit and extend functionality from other classes.  JavaScript, however, is classless<sup>1</sup> so we need another method to define and construct reusable objects. 

The standard way is to use a constructor function. This will be familiar with those used to classical programming languages (Java, C#, C++...). The constructor function in JavaScript is similar to a Class - we typically declare it with a capital letter and it is instantiated with the `new` keyword
<!--break-->

{% highlight javascript linenos=table %}
var Bicycle = function(name){
    this.name = name;
};

var bmx = new Bicycle('BMX');
bmx.name; 		//=> "BMX"

var anotherBMX = new Bicycle('New BMX');
anotherBMX.name; 	//=> "New BMX"

{% endhighlight %}

JavaScript uses [prototypal inheritance] [1] to reuse functionality from one object to another. A Constructor has a **Prototype** object property and this object is where we can define properties and functions which will form part of any new instances of BMX.  Let's continue with our example and introduce a new type of bicycle - a Motorbike.

{% highlight javascript linenos=table %}
var Bicycle = function(name){
    this.name = name;
};

Bicycle.prototype.moveForward = function(){
	console.log(this.name + " Moved forward 1");
}

var Motorbike = function(name){
	this. name = name;
}

//Inherit properties and methods from Bicycle 
Motorbike.prototype = new Bicycle();

Motorbike.prototype.startEngine = function(){
	console.log("Engine Started");
}

{% endhighlight %}

Let's run it and check the outcome:

{% highlight javascript linenos=table %}

var bmx = new Bicycle('BMX');
bmx.moveForward();  //=> prints "BMX Moved forward 1"

var fastBike = new Motorbike('Yamaha');
fastBike.moveForward();  //=> prints "Yamaha Moved forward 1"
fastBike.startEngine();  //= >prints "Engine Started"

bmx.startEngine();  //=> Error: undefined

{% endhighlight %}

Looks good - the `Motorbike` constructor has inherited the properties from the `Bicycle` by replacing it's prototype with an instance of `Bicycle`. Therefore the `moveForward` function is available to both `bmx` and `motoBike` whereas `startEngine` is only available to objects created from the `Motorbike` function.

One drawback of this method comes down to that **new** keyword. Firstly, by forcing the concepts of classical inheritance, we mask the true power of JavaScript's prototypal inheritance model.  And then there is the issue if we were to accidentally miss out the `new` keyword. This will introduce a bug into our code and behavior can become unpredictable. Where you intended assignments to `this` to be with the scope of the new object, they will instead be polluting the global namespace. *(Yes, strict mode can help here)*

Luckily we have another option - `Object.create()` to the rescue!

[1]: http://en.wikipedia.org/wiki/Prototype-based_programming "Prototype-based programming"

##Object.create()##

 ES5 officially introduced the [Object.create()] [2] method of constructing objects and achieving inheritance. Let's refactor our example above to use the `Object.create()` method.

 {% highlight javascript linenos=table %}

    var bicycle = {
    name : 'name',

    init : function(name){
        this.name = name;
    },

    moveForward : function(){
        console.log(this.name + " Moved forward 1");
    }
};

var motorBike = Object.create(bicycle);

motorBike.startEngine = function(){
    console.log("Engine Started");
}

//Inherit properties and methods from Bicycle 
var bmx = Object.create(bicycle);
bmx.init("BMX");
bmx.moveForward(); 	//=> prints "BMX Moved forward 1"


var fastBike = Object.create(motorBike);
fastBike.init('Yamaha')
fastBike.moveForward();  //=> prints "Yamaha Moved forward 1"
fastBike.startEngine();  //= >prints "Engine Started"

{% endhighlight %}

Let's break this down; `Object.create()` accepts an object as a parameter and returns a new object whose prototype equals the object we passed in.

In our example above `bicycle` defines the prototype and `bmx` is an instance of an object with the `bicycle` prototype. We are no longer using a constructor function to create an object so have no need to use the `new` keyword. 

[2]:  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create "Object.create()"

##Parasitic Inheritance##

Douglas Crockford - the man who popularised Object.create before it became part of the official ECMA specification - recently commented in his talk, "[The Better Parts] [3]", that he no longer used `Object.Create()`. His current chosen method to construct objects is detailed below: 

{% highlight javascript linenos=table %}

function constructor(spec) { 
    //create an object from another construct if we wish
    var that = other_constructor(spec),
        member,
    
        method = function () {
            // spec, member, method all available here
        };
    that.method = method;
    return that;
}

{% endhighlight %}

There's a few things going on here but nothing too complex. We've defined our own constructor function - we call this function a *factory*. It will ultimately create and return an object (*`that`*).  Within the factory function, we can assign methods and properties to the object we are creating. 

Now let's apply this pattern to our example:

{% highlight javascript linenos=table %}

function bicycle(spec) {

    var that = {},                   
    name = spec.name,

    moveForward = function () {
        console.log(name + " Moved forward 1");
    };

   that.moveForward = moveForward; // Make it public, if desired
   return that;
}

function motorBike(spec) { 
    var that = bicycle(spec), //inherit from bicycle
        //a private method
        turnKey = function(){
            console.log('Turn Key');
        }, 

        startEngine = function(){
             turnKey();
             console.log("Engine Started");
        };

    that.startEngine = startEngine;
    return that;
}

//create a new bicycle object 
var bmx = bicycle( {name : 'myBMX'} );

//create a new motorbike
var fastBike = motorBike( {name : 'yamaha'} );
{% endhighlight %}

Now check the output

{% highlight javascript linenos=table %}
bmx.moveForward();  // => prints "myBMX Moved forward 1"

fastBike.startEngine(); // => prints "Turn Key / Starts Engine"

fastBike.moveForward(); // => prints "yamaha Moved forward 1"

fastBike.turnKey() // Error undefined

{% endhighlight %}

Hopefully you noticed that we added an extra method `turnKey`. This is a *private* or *privileged* method. It cannot be invoked publicly, by calling `bmx.turnKey()` for example. 

Note that the `turnKey()` method can be called from with the motoBike factory  itself, even after the function has been called and returned the new object. This is thanks to the use of [closures in JavaScript][4].

Any  methods we do want to be public are explicitly assigned to the `that` object e.g. `that.startEngine = startEngine;`

The factory accepts an object - `spec` - as a parameter. Anything which is part of the 'spec' object can be referenced from within the function.

And when it comes to inheritance we can simply call another factory function first and then assign any additional properties or methods to it. 

So we have a nice way to define public and private methods, inherit from other factory, pass in a variable number of parameters via the `spec`, and in addition to avoiding the `new` keyword we also don't assign values using the `this` keyword. In doing so, we avoid any ambiguity to which namespace `this` is truly referencing. 

[3]: https://www.youtube.com/watch?v=bo36MrBfTk4 "The Better Parts"
[4]: https://developer.mozilla.org/en/docs/Web/JavaScript/Guide/Closures "Closures"

##Conclusion##

We've seen three ways to to construct object in JavaScript, and although the examples are slightly contrived it has been done to highlight the differences in each.

Hopefully this has given you a better understanding on different techniques to create objects and that the standard (read constructor functions) may not necessarily provide the best solution for your situation.


*<sup>1</sup> [Classes are coming in ES6](http://wiki.ecmascript.org/doku.php?id=strawman:maximally_minimal_classes), but the debate rages whether this is a good or bad addition.*


