---
layout: post
title: "Advanced Declarative Bindings in Knockout.js"
date: 2013-08-13 19:17
comments: true
categories: [javascript, coffeescript, knockoutjs]
---

### Introduction

[Knockout.js](http://knockoutjs.com/) has fantastic documentation and live exampels on using the framework.  While the documentation alludes to the power you wield, it leaves much unsaid.  For example there are not many examples given for custom knockout bindings or the fact that you can access anything in the window or current context in the data-bind syntax, even libraries such as underscore.

This post will be organized into a series of jsFiddle examples starting with the base code:

{% jsfiddle VbJGc default presentation %}
