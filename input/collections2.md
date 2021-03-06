home: index.html
prev: collections1.html
next: collections3.html
---
# 8. Implementing a model

What's a model? Roughly, a model does a couple of things:

*   **Data**. A model contains data.
*   **Events**. A model emits change events when data is altered.
*   **Persistence**. A model can be stored persistently, identified uniquely and loaded from storage.

That's about it, there might be some additional niceties, like default values for the data.

## Defining a more useful data storage object (Model)


<table>
<tr><td class="left" style="background-color: #fbfbfb;">

  <div class="code">
    <div>function Model(attr) {</div>
    <div>  this.reset();</div>
    <div>  attr &amp;&amp; this.set(attr);</div>
    <div>};</div>
    <div>Model.prototype.reset = function() {</div>
    <div class="hl green">  this._data = {};</div>
    <div class="hl blue">  this.length = 0;</div>
    <div>  this.emit('reset');</div>
    <div>};</div>
  </div>

</td><td class="right" style="background-color: #fbfbfb;">

  <h2>Model.reset()</h2>

  <div class="green"><p><i>_data</i>: The underlying data structure is a object. To keep the values stored in the object from conflicting with property names, let's store the data in the <code>_data</code> property</p></div>
  <div class="blue"><p>Store length: We'll also keep a simple length property for quick access to the number of elements stored in the Model.</p></div>
</td></tr>
<tr style="border-top: 1px solid #E5E5EE;"><td class="left" style="background-color: #fcfcfc;">

<div class="code">
  <div> </div>
  <div>Model.prototype.get = function(key) {</div>
  <div>  return this._data[key]; </div>
  <div>};</div>
  <div> </div>
</div>

</div>

</td><td class="right" style="background-color: #fcfcfc;">

<h2>Model.get(key)</h2>

<p>This space intentionally left blank.</p>

</td></tr>

<tr style="border-top: 1px solid #E5E5EE;"><td class="left">

<div class="code">
  <div>Model.prototype.set = function(key, value) {</div>
  <div >  var self = this;</div>
  <div class="hl orange">  if(arguments.length == 1 &amp;&amp; key === Object(key)) {</div>
  <div class="hl orange">    Object.keys(attr).forEach(function(key) {</div>
  <div class="hl orange">      self.set(key, attr[key]);</div>
  <div class="hl orange">    });</div>
  <div class="hl orange">    return;</div>
  <div class="hl orange">  }</div>
  <div>  if(!this._data.hasOwnProperty(key)) {</div>
  <div>    this.length++;</div>
  <div>  }</div>
  <div class="hl red">  this._data[key] = (typeof value == 'undefined' ?</div>
  <div class="hl red">    true : value);</div>
  <div>};</div>
</div>

</td><td class="right">

<h2>Model.set(key, value)</h2>

<div class="orange">
  <p>Setting multiple values: if only a single argument <code>Model.set({ foo: 'bar'})</code> is passed, then call <code>Model.set()</code> for each pair in the first argument. This makes it easier to initialize the object by passing a hash.
  </p>

<p>Note that calling <code>Model.set(key)</code> is the same thing as calling <code>Model.set(key, true)</code>.</p>

<p>What about ES5 getters and setters? <a href="http://jsperf.com/es5-getters-setters-versus-getter-setter-methods/5">Meh</a>, I <a href="http://code.google.com/p/v8/issues/detail?id=1239">say</a>.</p>

</div>

<div class="red"><p>Setting a single value: If the value is undefined, set to true. This is needed to be able to store null and false.</p>

</div>

</td></tr>

<tr><td class="left">
<div class="code">
<div>Model.prototype.has = function(key) { </div>
<div class="hl yellow">  return this._data.hasOwnProperty(key);</div>
<div>};</div>
<div></div>
<div>Model.prototype.remove = function(key) {</div>
<div class="hl purple">  this._data.hasOwnProperty(key) &amp;&amp; this.length--;</div>
<div>  delete this._data[key];</div>
<div>};</div>
<div> </div>
<div class="hl blue">module.exports = Model;</div>
</div>
</td><td class="right">

<h2>Model.has(key), Model.remove(key)</h2>


<div class="yellow">
  <p>Model.has(key): we need to use hasOwnProperty to support false and null.</p>
</div>

<div class="purple">
  <p>Model.remove(key): If the key was set and removed, then decrement .length.</p>
</div>

<div class="blue">
  <p>That's it! Export the module.</p>
</div>

</td></tr>
</table>

## Change events

Model accessors (get/set) exist because we want to be able to intercept changes to the model data, and emit `change` events. Other parts of the app -- mainly views -- can then listen for those events and get an idea of what changed and what the previous value was. For example, we can respond to these:

*   a set() for a value that is used elsewhere (to notify others of an update / to mark model as changed)
*   a remove() for a value that is used elsewhere

We will want to allow people to write `model.on('change', function() { .. })` to add listeners that are called to notify about changes. We'll use an EventEmitter for that.

If you're not familiar with EventEmitters, they are just a standard interface for emitting (triggering) and binding callbacks to events (I've written more about them [in my other book](http://book.mixu.net/node/ch9.html).)

<table>
<tr><td class="left">

<div class="code">
  <div class="hl green">var util = require('util'),</div>
  <div class="hl green">    events = require('events');</div>
  <div> </div>

  <div>function Model(attr) {</div>
  <div>  // ...</div>
  <div>};</div>
  <div> </div>
  <div class="hl green">util.inherits(Model, events.EventEmitter);</div>
  <div> </div>

  <div>Model.prototype.set = function(key, value) {</div>
  <div>  var self = this, oldValue;</div>
  <div>  // ...</div>
  <div>  oldValue = this.get(key);</div>
  <div class="hl blue">  this.emit('change', key, value, oldValue, this);</div>
  <div>  // ...</div>
  <div>};</div>
  <div>Model.prototype.remove = function(key) {</div>
  <div class="hl blue">  this.emit('change', key, undefined, this.get(key), this);</div>
  <div>  // ...</div>
  <div>};</div>


</div>

</td><td class="right">

<div class="green"><p>The model extends <code>events.EventEmitter</code> using Node's <code>util.inherits()</code> in order to support the following API:</p>

<ul>
  <li>on(event, listener)</li>
  <li>once(event, listener)</li>
  <li>emit(event, [arg1], [...])</li>
  <li>removeListener(event, listener)</li>
  <li>removeAllListeners(event)</li>
</ul>

<p>For in-browser compatibility, we can use one of the many API-compatible implementations of Node's EventEmitter. For instance, I wrote one a while back (<a href="https://github.com/mixu/miniee">mixu/miniee</a>).</p>

</div>

<div class="blue">
<p>When a value is <code>set()</code>, <code>emit('change', key, newValue, oldValue)</code>. </p>
<p> This causes any listeners added via on()/once() to be triggered.</p>
<p>When a value is <code>removed()</code>, <code>emit('change', key, null, oldValue)</code>.</p>
</div>

</td></tr>

</table>

## Using the Model class

So, how can we use this model class? Here is a simple example of how to define a model:

```
function Photo(attr) {
  Model.prototype.apply(this, attr);
}

Photo.prototype = new Model();

module.exports = Photo;
```

Creating a new instance and attaching a change event callback:

```
var badger = new Photo({ src: 'badger.jpg' });
badger.on('change', function(key, value, oldValue) {
  console.log(key + ' changed from', oldValue, 'to', value);
});
```

Defining default values:

```
function Photo(attr) {
  attr.src || (attr.src = 'default.jpg');
  Model.prototype.apply(this, attr);
}
```

Since the constructor is just a normal ES3 constructor, the model code doesn't depend on any particular framework. You could use it in any other code without having to worry about compatibility. For example, I am planning on reusing the model code when I do a rewrite of my window manager.

## Differences with Backbone.js

I recommend that you read through [Backbone's model implementation](http://documentcloud.github.com/backbone/docs/backbone.html#section-27) next. It is an example of a more production-ready model, and has several additional features:

*   Each instance has a unique cid (client id) assigned to it.
*   You can choose to silence change events by passing an additional parameter.
*   Changed values are accessible as the `changed` property of the model, in addition to being accessible as events; there are also many other convenient methods such as changedAttributes and previousAttributes.
*   There is support for HTML-escaping values and for a validate() function.
*   .reset() is called .clear() and .remove() is .unset()
*   Data source and data store methods (Model.save() and Model.destroy()) are implemented on the model, whereas I implement them in separate objects (first and last chapter of this section).
