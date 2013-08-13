---
layout: post
title: "Suppressing knockout.js evaluation errors"
date: 2013-08-12 19:53
comments: true
categories: [knockout, web, ui]
---

### Introduction

This is not intended to be a beginners guide or tutorial on knockout.js.  For those of you who are unfarmiliar with Knockout.js please see [Knockout.js](http://knockoutjs.com/) 

I've coded quite extensively in knockout.js, it isn't a large framework, and it does have a few minior pain points, but overall it is a highly effective framework that allows for significant productivity.

### Problem

You may venture into working with much more unstructured content with Knockout.  I personally prefer *schemaless*/*modeless* designs when working with projects that lend to that style.  When I say *schemaless* or *modeless* I mean passing json directly from the database through the *response* to Knockout.

{% codeblock lang:coffeescript %}
class @ViewModelProducts
    constructor: () ->
        @products = ko.observableArray([])

    fetch_products: () ->
        the_promise = $.get('/products', {somearg: undefined})
        $.when(the_promise).done( (response) =>
            # Remember the => is required in the nested ajax to access @
            # I'm passing in either the js obj if it is native or the parsed version if it was stringified
            @products if typeof response is "object" then return response else if response is "" then return undefined else return JSON.parse(response)
        ).fail( (response) ->
            console.log response.error
        )
{% endcodeblock %}

There is definitely a risk associated with this however, and that is Knockout failing on evaluating a field that doesn't exist, either by intention or not.   This risk would be a valid argument of those arguing for type safety, which could be somewhat imposed more strictly by using javascript classes.

However, I've found that creating a coffeescript class/viewmodel and using the [Knockout mapping plugin](http://knockoutjs.com/documentation/plugins-mapping.html) to map the json is not necessarily as efficient as simply reading the json straight into an *observable*.  Don't get me wrong, the Knockout mapping works fantastically, but there is a decent amount of code just to write each class that you want to map. 

{% codeblock lang:jade %}
table
    thead
        tr
            th Name
            th Price
    tbody(data-bind="foreach: viewModels.viewModelProducts.products")
        tr
            td(data-bind="text: name")
            td(data-bind="text: vendor")
            td(data-bind="text: price")
{% endcodeblock %}

Now let's say that the json returns several correct items then one with a  *name* and *price* but __not__ a *vendor* field.  This is going to cause Knockout.js to fail and not process anymore data-binds on the page.  This can be very inconvenient.  You may reflexibly think this is a good thing, and it very well may be, but there is definitely a use case for a backend model that may be missing fields unintentially or by design.

The above will result in the the products being listed up to the point of the one missing the vendor field, then none after that point:
{% codeblock %}
Uncaught Error: Unable to parse bindings.
Message: ReferenceError: vendor is not defined;
Bindings value: text: vendor
{% endcodeblock %}

This is the problem.  In the event of a developer mistake, or missing data, my opinion is it is better to continue processing instead of failing the rest of the page.

### Solution

The solution to the above problem requires hacking Knockout.js to prevent it from terminating early in the processesing of a page upon a data-bind evaluation error.

To contain all of my custom Knockout bindings I created a file called koBindingHandlers.coffee is included in my header after the knockout library.  In other works, knockout must first be available for knockout to be overridden.

{% codeblock lang:coffeescript %}
# Only one section overridden here, that is the catch of the 'parseBindingsString'
ko.utils.extend ko.bindingProvider.prototype, {
  'nodeHasBindings': (node) ->
    if node.nodeType is 1
      return node.getAttribute('data-bind') != null
    else if node.nodeType is 8
      return ko.virtualElements.virtualNodeBindingValue(node) != null
    else
      return false

  'getBindings': (node, bindingContext) ->
    bindingsString = this['getBindingsString'](node, bindingContext);
    if bindingsString
      return this['parseBindingsString'](bindingsString, bindingContext, node)
    else
      return null

  'getBindingsString': (node, bindingContext) ->
    if node.nodeType is 1
      return node.getAttribute('data-bind')
    else if node.nodeType is 8
      ko.virtualElements.virtualNodeBindingValue(node)
    else
      return null

  'parseBindingsString': (bindingsString, bindingContext, node) ->
    try
      bindingFunction = createBindingsStringEvaluatorViaCache(bindingsString, this.bindingCache)
      return bindingFunction(bindingContext, node);
    catch ex
      # Comment this out!
      # throw new Error("Unable to parse bindings.\nMessage: " + ex + ";\nBindings value: " + bindingsString)
      # Optionally uncomment this to debug problems, otherwise all errors are supressed.
      # console.warn "Unable to parse bindings.\nMessage: " + ex + ";\nBindings value: " + bindingsString
      return undefined
}
ko.bindingProvider['instance'] = new ko.bindingProvider()

createBindingsStringEvaluatorViaCache = (bindingsString, cache) ->
  cacheKey = bindingsString;
  return cache[cacheKey] || (cache[cacheKey] = createBindingsStringEvaluator(bindingsString))

createBindingsStringEvaluator = (bindingsString) ->
  rewrittenBindings = ko.expressionRewriting.preProcessBindings(bindingsString)
  functionBody = "with($context){with($data||{}){return{" + rewrittenBindings + "}}}"
  return new Function("$context", "$element", functionBody)
{% endcodeblock %}

In the above code, in *'parseBindingsString'* I commented out:
{% codeblock lang:coffeescript %}
throw new Error("Unable to parse bindings.\nMessage: " + ex + ";\nBindings value: " + bindingsString)
{% endcodeblock %}

When this js error is thrown it causes the early termination of processing of any more Knockout bindings.  The sideeffect is it does become very difficult at times to debug when you've made a small mistake.  In that case you can simply uncomment:

{% codeblock lang:coffeescript %}
console.warn "Unable to parse bindings.\nMessage: " + ex + ";\nBindings value: " + bindingsString
{% endcodeblock %}

In practice I find it rare that implementing the above workaround causes any development pain.  If something seems off or not rendering, I simply uncomment the *console.log* figure out what is breaking the Knockout binding, fix it, and continue. 

I hope this helps some of you other Knockout users since this was originally one of my pain points with the Library.
