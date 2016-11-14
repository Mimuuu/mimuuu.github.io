---
layout:     post
title:      "Combine AngularJS, React and Flux to make components-based applications"
date:       2015-04-24 00:00:00
author:     "Mimu"
---

In this post, I'll assume you know the basics of AngularJS, React and the Flux architecture.

**AngularJS** is a big complete framework that can handle everything from you. It is also on the way out as the google team announced several months ago that they'll dropped this framework and start another one: AngularJS 2 (which is a totally different one, even though they'll be migrations path I believe).

**React** is a "library for building user interface" (react website) and only handle the view of an application.

**Flux** comes with a "Dispatcher" that will handle the discussion between the views (what the user sees and interact with) and the stores (where the logic is), through events.

# Why combine those 3 together?

Well, a mix of things.

It'll allow you to make the transition between a full angular application and a **components-based applications**, which seem to be what we are heading to right now.

You'll **remove all of the watchers / $digest issues** you will encounter with any big AngularJS application. AngularJS will not handle the view anymore, which means that there will be no digest, or very rarely. Instead React will take over with its virtual DOM, which would also be faster. Win win.

You'll **replace the 2-way data bindings with the Flux architecture**. Instead of everything magically update (or not update, or badly update), you'll have full control over it.

There is one drawback though, you will have to load the whole AngularJS framework to only use little parts of it. Parts that probably could be replaced with micro frameworks to lighter the initial load of your page. However I'm sure there are some use to it, at least we had one. :)


# How to combine them?

First of all, I want to say that there was already a popular directive to combine AngularJS and React called [ngReact](https://davidchang.github.io/ngReact/docs/ngReact.html). I basically use the exact same code, except I chose to go with Flux and remove every watchers, where [ngReact](https://davidchang.github.io/ngReact/docs/ngReact.html) still watches values that might change.

Basically, the application will look like this (a simple example will follow to make it more clear):

  * **1 angular view = 1 page** (or 1 route, basically)
  * Inside these views, we'll have **ONE directive**, that will be **associated with a react component** and that will serve as a hub: The views will send actions to the Dispatcher, that will inform the directive about what to do, the directive will do it and then re-render the component and update the page the react way
  * These **directives will not have template**, the react component will serve as template

  
# A list of images with React and AngularJS

Let's say we want a page that display a list of images with their title.

First, you'll have the angular view, let's call it *images.html*:

{% highlight html %}

<div>
  <h1>Images</h1>
  
  <!-- Angular directive that will be associated with the react component -->
  <images-list></images-list>
</div>  

{% endhighlight %}

Pretty simple view, here is the ImagesList directive in its smallest form:

{% highlight javascript %}
'use strict';

function ImagesList(AppService) {
  return {
    restrict: 'E',
    
    link: function(scope, element) {
      // Retrieve the appropriate component
      var component = AppService.getReactComponent('ImagesList');
      if (!component) {return;}
      
      /**
      * Render the component with optional props
      * @params props - Will be the props received by the react component
      */
      var renderComponent = function(props) {
        AppService.renderComponent(component, element[0], props);
      }
      
      renderComponent();
    }      
  }    
}

ImagesList.$inject = ['AppService'];
angular.module('AppModule').directive('imagesList', ImagesList);

{% endhighlight %}

This code is pretty straightforward, the directive render the associated react component.

The code inside the AppService (below) is directly taken from the `ngReact` directive by the way. We'll see later that when we declare the react component associated to the directive, we put him inside the angular application as a `value`, which is why we can retrieve it with `$injector`.

{% highlight javascript %}

function AppService() {
  
  /**
  * Retrieve the React component
  * @param name - Name of the component
  */
  this.getReactComponent = function(name) {
    var rComponent = null;
    try {
      rComponent = $injector.get(name);
    } catch(e) {}
    
    return rComponent;
  }
  
  /**
  * Render the component into the page
  * @param component - React component
  * @param element - Element that will "receive" the component
  * @param props - Properties passed to the component
  */
  this.renderComponent = function(component, element, props) {
    React.render(React.createElement(component, props), element);
  }
}

{% endhighlight %}

So we are storing the initial component into angular to be able to retrieve it and display it with the React's methods.

However we need to ask the servers the images list, and pass it to the component to be able to display it.

So in our directive, instead of just calling `renderComponent()`, we'll make a request to an API and render the component only after we receive the information. We could also render the component instantly and display something when there is no images received yet, and then call it again once the images are received to update it. I'll stick to the first case here as I think it will be more clear:

{% highlight javascript %}

function ImagesList(AppService) {
  ...
  ...
  
  // Call the API to get the images
  var params = {
    listId: 'XX'
  }
  AppService.getImages(params).then(function(res) {
    renderComponent(res.data.imagesList);
  }, function(err) {
    console.log('error occured', err);
  });
  
}

{% endhighlight %}

Instead of rendering a component without properties, we call a method that will make a GET request to the API (using `$http.get`), then the API will send us back the list that we will pass onto our component. For the sake of this example, let's pretend that the API send this array back:

{% highlight javascript %}

[
  {
    id: '1',
    title: 'Title1'
    image: 'url'
  },
  {
    id: '2',
    title: 'Title2'
    image: 'url'
  },
  {
    id: '3',
    title: 'Title3'
    image: 'url'
  }  
]

{% endhighlight %}

We are done here on the Angular front. Almost done, we'll get back to it when the user interact with the image.

So now we need to create our React component, this is probably the easiest part, React being somewhat easy to understand and to use. Here we go:

{% highlight javascript %}

var ImagesList = React.createClass({

  propTypes: {
    list: React.PropTypes.array.isRequired
  },
  
  render: function() {
    
    // Data preparation, so if we don't have any images we display a "no image" text
    var list = [];
    for (var i=0; i<this.props.list.length; i++) {
      list.push(
        <div key={this.props.list[i].id} className='image'>
          <img src={this.props.list[i].image} />
          <h2>{this.props.list[i].title}</h2>
        </div>
      );
    }
    if (list.length === 0) {
       list.push(<p>No images</p>);
    }
    
    return (
      <div className='images-list'>
        {list.map(function(item) {
          return item;
        })}
      </div>
    );
  }

});

angular.module('AppModule').value('ImagesList', ImagesList);

{% endhighlight %}

Here is assume you know react, and it is actually pure react. As stated by the website React does not care where the data comes from and it's true, we just passed our array in our component and from then we are just writing JSX to display it in the way we want.
Notice the last line, you should write it **only** for components **associated with an angular directive**, it is useful to retrieve it. However for other component there is no need and no point to use memory for it.

You can of course use more components. For example here, we could create an `Image` component, and passed the individual object to it.

# An interactive list of images with Flux

Let's say we want to delete the image if the user clicks on it.
Let's take this opportunity to also do the Image component, as it would be easier to have simple component if you want to interact with it.

Our list component will look like that:

{% highlight javascript %}

var ImagesList = React.createClass({

  propTypes: {
    list: React.PropTypes.array.isRequired
  },
  
  render: function() {
    
    // Data preparation, so if we don't have any images we display a "no image" text
    var list = [];
    for (var i=0; i<this.props.list.length; i++) {
      list.push(
        <Image data={this.props.list[i]} />
      );
    }
    if (list.length === 0) {
       list.push(<p>No images</p>);
    }
    
    return (
      <div className='images-list'>
        {list.map(function(item) {
          return item;
        })}
      </div>
    );
  }

});

angular.module('AppModule').value('ImagesList', ImagesList);

{% endhighlight %}

And our new Image component will look like this:

{% highlight javascript %}

var Image = React.createClass({

  propTypes: {
    data: React.PropTypes.object.isRequired
  },
  
  _deleteImage: function() {
    Dispatcher.dispatch({
      actionType: 'IMAGE_DELETE',
      imageId: this.props.data.id
    });
  }
  
  render: function() {
    
    return (
      <div key={this.props.data.id} className='image' onClick={this._deleteImage}>
        <img src={this.props.data.image} />
        <h2>{this.props.data.title}</h2>
      </div>
    );
  }

});

{% endhighlight %}

In the list component, we pushed Image's components inside the `list` array instead of html, and we created an Image component that display the same html.

You will notice an additional method in the image component aswell as an `onClick` listener on the `div`.
When the user clicks on it, that will execute the `_deleteImage` function and send the object to the dispatcher. The dispatcher will then send it to the directive, so we need something to catch it and act accordingly.

We will need this function, which as renderComponent will be the same for each directive (the events inside will change though, obviously). For clarity reason, I'll just write the new function, it is to put inside the `link` function.

{% highlight javascript %}

var dispatcherToken = Dispatcher.register(function(payload) {
  var actions = {
  
    'IMAGE_DELETE': function(action) {
      var params = {
        imageId: action.imageId
      };
      AppService.deleteImage(params).then(function(res) {
        // API call returns the new images list
        renderComponent(res.data.imagesList);
      }, function(err) {
        console.log('something wrong happened', err);
      });
    },
    
    ... (more events if needed)
  
  }
});

// Unregister the Dispatcher event to prevent possible multiple calls in the future
scope.$on('$destroy', function() {
  Dispatcher.unregister(dispatcherToken);
});

{% endhighlight %}

So now, when the user clicks on the image, the Dispatcher gets inform, send the data to those who registered to the `'IMAGE_DELETE'` event (like we just did), so we can execute the code and delete the image. By calling the `renderComponent` once again with the updated list, we allow the components to re-render (in the virtual DOM) and react can modify the real DOM by comparing the differences.

**/!\ Important**    
Please pay attention to the last lines, we need to unregister the event when the directive is destroyed, otherwise if we come on the page again, the events will register once more and the code will be executed twice, and you probably don't want that.

# Full code example

An example will soon be available here: [https://github.com/Mimuuu/Angular-React-Flux-example](https://github.com/Mimuuu/Angular-React-Flux-example)
