---
layout:     post
title:      "Speed up AngularJS: Remove watchers on unseen elements"
date:       2015-03-12 12:00:00
author:     "Mimu"
---

AngularJS is great, easy to use, somewhat easy to learn if you don't believe everything you read online and think a little bit, and most importantly allow you to do some bindings in a very simple way. Everything is updating in real time, you don't have to do anything, it's perfect.

However it doesn't come without drawbacks, the biggest one being **performance**.
With Angular 1.3 came the "bind once" syntax using the `::`syntax. This is great but sometimes you want to keep all of your bindings set because the values might change, and that can become an issue in long lists (for example an infinite scrolling `ng-repeat`).

#How to remove / re-add watchers

First, we need it to be transparent for the user. They need to be able to do whatever they want and don't feel any slowdown.

The first idea I had was to actually remove the elements outside the viewport, and put them back when needed using `$compile`, but it was **VERY SLOW** to the point that the page was very annoying to use.

So I decided to **use the debug info** created by default by Angular. It provides a shit load of stuff, including arrays of watchers associated with every elements. The idea was to store the arrays of watchers locally, empty the ones attached to the element, and fill them back when needed. 

It's actually very easy.  
An element has 2 kind of watchers associated to it, the ones from its `scope`and the ones from its `isolateScope`, so we need to store both of them.

* We need an array to store the watchers, we'll call it **wArray**.  
* The first element would be another array of 2 elements, the watchers from the the `scope`, the watchers from the `isolateScope`
* Then we will loop through every child element and do the same

In the end we will have an array like this.

{% highlight javascript %}

wArray = [
  [[scope watchers from the element], [isolateScope watchers from the element]],
  [[scope watchers from the first child], [isolateScope watchers from the first child]],
  ...
  [[scope watchers from the last child], [isolateScope watchers from the last child]]
]

{% endhighlight %}


That will allow us to put them back very easily.

Here is an example code:

{% highlight javascript %}

function getElemWatchers(element) {
  wArray[i] = [];
  wArray[i][0] = getWatchersFromScope(element.data().$isolateScope);
  wArray[i][1] = getWatchersFromScope(element.data().$scope);

  angular.forEach(element.children(), function (childElement) {
    i++;
    getElemWatchers(angular.element(childElement));
  });
}

  function getWatchersFromScope(scope) {
    if (scope) {
      var tmp = scope.$$watchers || [];
      scope.$$watchers.length = 0;
      return tmp;
    } else {
      return [];
    }
}
		
getElemWatchers(elem);
	
{% endhighlight %}
	

So we stored the watchers and remove them from the element. What do to when we need to enable them back?  
As you probably anticipate, we just have to do the exact same thing in reverse, we have our array with every watchers associated with every child of the element, so we loop through it and put fill the arrays:

{% highlight javascript %}

function setElemWatchers(element) {
  setWatchersFromScope(element.data().$isolateScope, 0);
  setWatchersFromScope(element.data().$scope, 1);

  angular.forEach(element.children(), function(childElement) {
    i++;
    setElemWatchers(angular.element(childElement));
  });
}

function setWatchersFromScope(scope, n) {
  if (scope) {
    scope.$$watchers = wArray[i][n];
  }
}

setElemWatchers(elem);
	
{% endhighlight %}	



#When to call all of this?

We can disable and enable the watchers of an element, now we need to be able to do this everytime an element end up in or outside the viewport.

For that we just listen to the scroll event, and test for each element is it's inside or outise the viewport, and act accordingly.

A few tips first:

* Consider adding a **debounce function to your listener**, so you actually do all the watchers thing when the user is done scrolling instead of every time the scrolling event triggers, which is A LOT
* Add a boolean to indicate if the element is hidden or not, so if its status does not change, as it won't for most of the elements, you don't do anything

With that in mind, we would have something like this as our listener:

{% highlight javascript %}

var checkElements = debounce(function() {
  scope.$broadcast('dwhCheckElements');
}, 250);

document.addEventListener('scroll', checkElements);

{% endhighlight %}

This code is pretty straightforward, every time the user scrolls, we broadcast an event to let the elements know they need to check if their "status/visibility" changed, and disable / enable their watchers.

So yes there are actually **2 directives** here.  
The first one on the parent element, the `ngRepeat`, to handle the scroll listener. The 2nd one of every child element you want to be disabled if invisible.

I also added a "range of error" of 1000 pixels, which means we consider the viewport to be the viewport itself + 1000 up and down so if an element is half visible it still works.

#Full code

You can find the full directives code below or [on github](https://github.com/Mimuuu/Disable-When-Hidden).

Also this code could probably be better, and if you want to use it you should probably be sure that using it is worth having the debug info in your app. Since 1.3 [you can disable it](https://docs.angularjs.org/guide/production) and I see no reason why you shouldn't.

{% highlight javascript %}

(function() {
  'use strict';

  var app = angular.module('app');

  // Parent directive
  // Broadcast an event to every listening child every time the user is scrolling	
  app.directive('disableWhenHidden', function() {
    return {
      restrict: 'A',

      link: function(scope) {

        function debounce(fn, delay) {
          var timer = null;
          return function () {
            var context = this, args = arguments;
            clearTimeout(timer);
            timer = setTimeout(function () {
              fn.apply(context, args);
            }, delay);
          };
        }
      
        var checkElements = debounce(function() {
          scope.$broadcast('dwhCheckElements');
        }, 250);
    
        document.addEventListener('scroll', checkElements);
      }
    };
  });

  app.directive('dwhElement', function() {
    return {
      restrict: 'A',

      link: function(scope, element) {
				
        var m = 1000; // Range of "errors" outside the viewport
        var wArray = []; // Array to store all the watchers
        var isHidden = false; // Used to prevent useless computation
        
         // Store and remove the watchers of the element
         var disableWatchers = function () {
           wArray.length = 0;
           leaveHimToDie(element);
           isHidden = true;
         };
         
          // Put the watchers back  
          var enableWatchers = function () {
            bringHimBack(element);
            isHidden = false;
          };
          
          // Listener
          scope.$on('dwhCheckElements', function () {
            var coordinates = element[0].getBoundingClientRect();
            if (coordinates.bottom > 0 - m && coordinates.top < window.innerHeight + m) {
              if (isHidden) {
                enableWatchers();
              }
            } else if (!isHidden) {
              disableWatchers();
            }
          });
           
           // Remove watchers from the element passed in parameter
           var leaveHimToDie = function (elem) {
             var i = 0;
             
             function getElemWatchers(element) {
               wArray[i] = [];
               wArray[i][0] = getWatchersFromScope(element.data().$isolateScope);
               wArray[i][1] = getWatchersFromScope(element.data().$scope);
               
               angular.forEach(element.children(), function (childElement) {
                 i++;
                 getElemWatchers(angular.element(childElement));
               });
             }
             
             function getWatchersFromScope(scope) {
               if (scope) {
                 var tmp = scope.$$watchers || [];
                 scope.$$watchers = [];
                 return tmp;
               } else {
                 return [];
               }
             }
             
             getElemWatchers(elem);
           };
           
           // Enable back watchers to the element passed in parameter
           var bringHimBack = function (elem) {
             var i = 0;
             
             function setElemWatchers(element) {
               setWatchersFromScope(element.data().$isolateScope, 0);
               setWatchersFromScope(element.data().$scope, 1);
               
               angular.forEach(element.children(), function (childElement) {
                 i++;
                 setElemWatchers(angular.element(childElement));
               });
             }
             
             function setWatchersFromScope(scope, n) {
               if (scope) {
                 scope.$$watchers = wArray[i][n];
               }
             }
             
             // Start the loop
             setElemWatchers(elem);
           };
         }
       };
     });
})();
	
{% endhighlight %}	