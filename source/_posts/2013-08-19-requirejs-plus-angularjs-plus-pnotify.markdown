---
layout: post
title: "RequireJS + AngularJS + pnotify"
date: 2013-08-19 21:05
comments: true
categories: [javascript, coffeescript, requirejs, angularjs, pnotify]
---

### Introduction

Let's add pnotify: [Pines Notify](http://pinesframework.org/pnotify/) to an AngularJS webapp using RequireJS.  Now typically with an AngularJS app you would do this per the pnotify [directions](https://github.com/sciactive/pnotify#getting-started).  When using RequireJS the steps are a bit different however.

### Basic Example

#### Steps:

* From [pnotify/Github](https://github.com/sciactive/pnotify) Copy jquery.pnotify.js to your lib/notify directory and jquery.pnotify.default.css and jquery.pnotify.default.icons.css to your css directory.  You can choose the directory structure as you see fit, as long as it is setup properly in RequireJS

* Add the css to your index, no need to add the pnotify js however, that will be handled by require.  

{% codeblock lang:jade %}
!!! transitional
html(lang="en", ng-app="")
    head
        title Data Glue
        meta(name="viewport",content="width=device-width, initial-scale=1.0")
        script(data-main="js/main",src="js/lib/require/require.js")
        link(rel="stylesheet",type="text/css",media="all",href="css/bootstrap.min.css")
        link(rel="stylesheet",type="text/css",media="all",href="css/app.css")
        link(rel="stylesheet",type="text/css",media="all",href="css/pines/jquery.pnotify.default.css")
        link(rel="stylesheet",type="text/css",media="all",href="css/pines/jquery.pnotify.default.icons.css")
 
{% endcodeblock %}

* Your main.js should looks similar to:

{% codeblock lang:javascript %}
require.config({
    paths: {
        jquery: 'lib/jquery/jquery-2.0.3',
        angular: 'lib/angular/angular',
        text: 'lib/require/text',
        bootstrap: 'lib/bootstrap/bootstrap',
        'jquery.pnotify': 'lib/pines/jquery.pnotify'
        // You could also reference the github path
        //'jquery.pnotify': 'https://raw.github.com/sciactive/pnotify/master/jquery.pnotify.min'

    },
    baseUrl: 'js',
    shim: {
        'angular' : {'exports' : 'angular'},
        'angularMocks': {deps:['angular'], 'exports':'angular.mock'},
        'bootstrap': ["jquery"],
        // Nothing too special here, just remember to depend on jquery
        'jquery.pnotify': ["jquery"]
    },
    priority: [
        "angular"
    ],
    // http://stackoverflow.com/questions/8315088/prevent-requirejs-from-caching-required-scripts
    urlArgs: "bust=" + (new Date()).getTime()
});

require([
    'angular',
    'app',
    'routes',
    'bootstrap'
], function(angular, app, routes) {
    'use strict';
    angular.bootstrap(document, [app['name']]);
});
{% endcodeblock %}

* Now for defining a Controller using RequireJS that works with AngularJS.  The key here is to depend on jquery and jquery.pnotify in the RequireJS define, not in the second AngularJS []

{% codeblock lang:coffeescript %}
define ["jquery", "jquery.pnotify"], ($) ->
  ['$scope', ($scope) ->
    $.pnotify
        title: 'Regular Notice',
        text: 'Check me out! I\'m a notice.'

    # because this has happened asynchronously we've missed
    # Angular's initial call to $apply after the controller has been loaded
    # hence we need to explicityly call it at the end of our Controller constructor
    $scope.$apply();
  ]
{% endcodeblock %}

* This works great, however we can make this even more consice and modular by turning the pnotify into a service.

Open your services.js make sure that jquery and jquery.pnotify are dependencies, then add a factory for the pnotify.

{% codeblock lang:javascript %}
define(['angular', 'jquery', 'jquery.pnotify'], function (angular, $) {
    'use strict';
    
    angular.module('myApp.services', [])
        .value('version', '0.1')
        .service('sharedProperties', function () {
            return {
                foo: 'bar'
            };
        })
        .factory('notificationService', ['$http', function($http){
            return {
                notify: function(hash) {
                    $.pnotify(hash);
                }
            };
        }]);
});
{% endcodeblock %}

The benefit here is not only does this result in less code in the controllers, but also that the notifications are abstracted so you could easily switch out the notification framework or add additional logging or the like without modifying your controllers.

Speaking of controllers, let's change the previous controller definition to:

{% codeblock lang:coffeescript %}
define [], () ->
  ['$scope', '$http', 'notificationService', ($scope, $http, notificationService) ->
    notificationService.notify
      title: 'Regular Notice',
      text: 'Check me out! I\'m a notice.'

    # because this has happened asynchroneusly we've missed
    # Angular's initial call to $apply after the controller has been loaded
    # hence we need to explicityly call it at the end of our Controller constructor
    $scope.$apply();
  ]
{% endcodeblock %}

That's it, you can depend on that notificationService in Angular now for easy notifications.
