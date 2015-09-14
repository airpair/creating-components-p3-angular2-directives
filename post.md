Welcome to Part 3 of our 3-part series on content container components!

In this tutorial, we will be converting the Angular 1.3 directive from [Part 2](https://www.airpair.com/angularjs/posts/creating-container-components-part-2-angular-1-directives) into an Angular 2 component.  As such, if you are just joining in, you will probably want to visit:

*  [Part 1](https://www.airpair.com/javascript/posts/creating-container-components-part-1-shadow-dom) by [Rachael L Moore](https://twitter.com/morewry) for an introduction to the `ot-site` component and background on the Shadow DOM.

*  [Part 2](https://www.airpair.com/angularjs/posts/creating-container-components-part-2-angular-1-directives) for information about directives and advanced transclusion.

Let's get started!

## Rebuilding `ot-site`

In [Part 2](https://www.airpair.com/angularjs/posts/creating-container-components-part-2-angular-1-directives), we used Angular 1.3 to build this lovely `ot-site` container component:

![Empty ot-site container](https://8604d17a51d354cba084d27f632b78fe46e70205.googledrive.com/host/0Bws_6WaNR1DWelh6X1hLcTlBR1E/otsite-empty.png)

As a consistent layout for all OpenTable micro-apps, it had to allow multiple insertion points for custom user content (in the header, menu, and the body). 

Achieving that wasn't easy.  We had to build custom transclusion functionality from scratch and match elements from the user's template to our directive template completely manually.  This required in-depth knowledge of several concepts in Angular: transclusion, the directive lifecycle, and the way scopes interact, to name a few.  Our final implementation (before refactoring into a service) looked like this:

```javascript
angular.module("ot-components")

.directive("otSite", function() {
  return {
    scope: {},
    transclude: true,
    template: `
      <div id="site">													
        <header transclude-id="head">	
          <svg id="logo"></svg>
        </header>					
        <nav transclude-id="menu"></nav> 								
        <main transclude-id="body"></main>								
        <footer>					
          © 2015 OpenTable, Inc.
        </footer>				
      </div>`,	
    link: function(scope, elem, attr, ctrl, transclude) {
      transclude(function(clone) {
        angular.forEach(clone, function(cloneEl) {
          var destinationId = cloneEl.attributes["transclude-to"].value;
          var destination = elem.find('[transclude-id="'+ destinationId +'"]');
          if (destination.length) {
            destination.append(cloneEl);
          } else { 
            cloneEl.remove();
          }
        });
      });
    }
  };
});
```

Pretty bulky, right?  Clearly, a lot of custom configuration was required to get it off the ground.  

## From Angular 1 to Angular 2

Implementing the same functionality in Angular 2 looks *considerably better* -- far cleaner and easier to understand -- because Angular 2 leverages the power of web components.   And as you saw earlier in [Part 1](https://www.airpair.com/javascript/posts/creating-container-components-part-1-shadow-dom), web components can be pretty powerful.  

To recreate our 1.3 directive, we'll be building an Angular 2 **component**.  Angular 2 components combine the best of both worlds: web component standards and the advantages of Angular - bindings, DI, and so on. 

We will need to re-wire three key things: how we're inserting custom content, how we're defining scope, and how we're registering our directive. 


![Differences between Angular 1.3 and Angular 2.0](https://8604d17a51d354cba084d27f632b78fe46e70205.googledrive.com/host/0Bws_6WaNR1DWelh6X1hLcTlBR1E/Screen%20Shot%202015-03-22%20at%2012.24.49%20PM.png)


### Transclusion &rarr; Shadow DOM

First thing to nuke in our old directive: transclusion. 

In Angular 2, the concept of transclusion is simply no longer necessary.  Because component directives are built on top of web components, we can simply take advantage of the built-in web component functionality instead -- `content` tags.

If we use `content` tags with `select` attributes to filter the user-provided template, our directive already has all that it needs natively to handle multiple insertion points.  All of that custom transclusion code in the `link` function -- as well as our `transclude-id` attributes -- can be deleted.

While we wait for broader web component support, we also have the option of using the Angular emulation of content insertion - `ng-content` tags.  Like regular `content` tags, they accept a `select` attribute to filter the distributed nodes.

If we remove all the old transclusion code (`transclude: true` and the `link` function) and add in `ng-content` tags, our new directive definition looks like this:


```javascript
angular.module("ot-components")

.directive("otSite", function() {
  return {
    scope: {},
    template: `
      <div id="site">													
        <header>	
          <svg id="logo"></svg>
          <ng-content select="[head]"></ng-content>
        </header>					
        <nav>
          <ng-content select="[menu]"></ng-content>
        </nav> 								
        <main>
          <ng-content select="[body]"></ng-content>
        </main>								
        <footer>					
          © 2015 OpenTable, Inc.
        </footer>				
      </div>`
  };
});
```

Already getting much slimmer!


### Manual Scope &rarr; Sensible Default Contexts

The second thing that's changing in Angular 2 is how we think about execution context. In order to make our directive work in Angular 1.3, we had to fully understand transcluding scopes and isolate scopes, and how to ensure they interact harmoniously (especially when it comes to components nested within components).

Ultimately, you end up making scope diagrams like the one below, just to keep it all straight:

![Scope diagram](https://8604d17a51d354cba084d27f632b78fe46e70205.googledrive.com/host/0Bws_6WaNR1DWelh6X1hLcTlBR1E/Screen%20Shot%202015-03-17%20at%205.34.28%20PM.png)

*Design credit: Simon Attley*

In contrast, Angular 2 provides some sensible defaults depending on which type of directive you’ve declared, which reduces much of that uncertainty.  Components have a completely isolated execution context by default.  You don’t have to set it yourself or even understand the scope hierarchy.  Additionally, if you elect to use `content` tags to allow user-provided templates- your bindings will still work as expected. 

This means we can also remove that `scope:{}` line from our directive definition:


```javascript
angular.module("ot-components")

.directive("otSite", function() {
  return {
    template: `
      <div id="site">													
        <header>	
          <svg id="logo"></svg>
          <ng-content select="[head]"></ng-content>
        </header>					
        <nav>
          <ng-content select="[menu]"></ng-content>
        </nav> 								
        <main>
          <ng-content select="[body]"></ng-content>
        </main>								
        <footer>					
          © 2015 OpenTable, Inc.
        </footer>				
      </div>`
  };
});
```

Which leaves us with only two pieces of configuration to create this complex directive: the name of the directive and the template itself. Pretty cool!


### One DDO &rarr; Class & Annotations

The last thing we need to do to convert our directive to Angular is to update the registration syntax.

In Angular 1.3, to register a directive,  we have to use somewhat cumbersome Directive Definition Object (DDO).

**Angular 1.3 registration syntax**

```javascript
angular.module("ot-components")

.directive("otSite", function() {
  return {
    // all of the things!
  };
});
```

We have to call a `directive` function -- to which we pass a factory function -- which returns an object -- that contains everything we ever want associated with the directive -- templates, callback functions, configurations, etc etc.  

Instead of this one large config, Angular 2 provides a simpler interface for component authors -- splitting the definition out into two logical parts: the component class and its meta-data annotations.

** Component Class **

The first half of creating a directive is defining the component class. It controls the component's business logic and can expose a public API to other components.  For example, if we were creating a dropdown component,  this is where our open and close methods would live.

If you're using ES5, you can simply use a constructor function for this definition:

 ```javascript
function OtSite() {
};
 
/* public methods here
OtSite.prototype.someFunction = function() {...};
*/

```

...or if you're using ES6 or TypeScript, you can use the built-in class syntax:


```javascript
class OtSite() {
  constructor () {
  }

  // public methods here
}
```

ES6 syntax is what we'll use for this example.  

It's worth noting that normally there would be methods in our definition, but as our `ot-site` directive is used as an inert container for our layout (and it doesn't actually *do* anything),  we don’t currently need any.

What we do need is to record our two pieces of meta-data: the name of the directive and its template.  Which brings us to the second half of the definition: annotations.


** Meta-data annotations **

So, what the heck are annotations?  The short answer is that annotations serve to succinctly communicate the intent of the associated class - what is it and what does it need?  In our case, we want to define that our class represents a component directive named "ot-site" and that it needs the "ot-site.html" template.

Annotations are often thought of as the scary new part of Angular 2 directive syntax because they don't seem to fit naturally with anything in Angular 1.x.  But they're really nothing new - they are simply Angular 2's way of separating meta-data -- like your directive's name, its type, its associated dependencies and templates etc -- out from the business logic of the component.  We had most of this information previously - it was just buried in the complex DDO.

To "annotate" in ES6, we simply set a static property on the `OtSite` class we just created and call it "annotations", and Angular will pick it up.


```javascript
class OtSite() {
  constructor () {
  }
  // public methods here
}

OtSite.annotations = [];

```

So what do we put in here?  First, let's add the meta-data for the directive specifically.  This is where we indicate that we are creating a component, and we pass in the selector that Angular can use to identify it. 

```javascript
class OtSite() {
  constructor () {
  }
  // public methods here
}

OtSite.annotations = [
  new Component({
    selector: "ot-site"
  })
];

```
 You’ll notice that we no longer are using a normalized directive name like "otSite" or the `restrict` property to register the directive. We can just use a CSS selector.

Next, we can add any view configurations:

```javascript
class OtSite() {
  constructor () {
  }
  // public methods here
}

OtSite.annotations = [
  new Component({
    selector: "ot-site"
  }),
  new View({
    templateUrl: "ot-site.html"
  })
];

```

It's hard to believe, but at this point, we've added all the code required to make our component work.  That's all you need to make a fully functional container component!

## Converting to TypeScript

 But we can make it even cleaner.  If we were to convert this to TypeScript, we get even more syntactic sugar.  


The new version of TypeScript has decorators built in, so we don't have to manually define an annotations property at all. Instead, we can simply decorate our component class with annotations using the @ decorator shorthand*.

* If you'd like more information about the difference between decorators and annotations, I'd recommend [Pascal Precht's blog post here](http://blog.thoughtram.io/angular/2015/05/03/the-difference-between-annotations-and-decorators.html).


```javascript
@Component({
  selector: "ot-site"
})
@View({
  templateUrl: "ot-site.html"
})

class OtSite() {
  constructor () {
  }

  // public methods here
}
```

You'll also notice that we moved the meta-data up to the top of the definition.  This is ideal because it prevents you from having to scroll through all your class code just to see a component's selector or associated template.  

And **that’s it**! With just those lines, we've re-created `ot-site`.

So given the following component user markup:

```markup
<ot-site>
	<div head>
      I render in head.
    </div>
    <div menu>
      I render in menu.
    </div>
    <div body>
      I render in body.
    </div>
</ot-site>

```

We can reach the expected output:

![Final ot-site output](https://8604d17a51d354cba084d27f632b78fe46e70205.googledrive.com/host/0Bws_6WaNR1DWelh6X1hLcTlBR1E/otsite-final.png)

## Conclusion

Angular 2 clearly has a lot to offer - what took lines and lines of custom code in Angular 1.x is now incredibly simple.  

See a working example of the code on Github [here](https://github.com/kara/angular2-ot-site).

For further reading on Angular 2, see [Angular's new website](https://angular.io/) and the [current Github repo](https://github.com/angular/angular/tree/master/modules/angular2).  

Happy coding!

