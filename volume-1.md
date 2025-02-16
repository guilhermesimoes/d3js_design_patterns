# a d3js design pattern

![d3js-design-pattern](https://cloud.githubusercontent.com/assets/432483/8653869/c78d15e0-293b-11e5-8167-4eaed96b7048.png)

## intro & theory

Larger interactive d3 programs can be difficult to organize.  After all your elements are on the screen you want to glue them together inside event callbacks.  The conventional methods of calling `d3.selectAll()` against a large DOM tree end up resulting in a lot of additional code to make sure your selections are accurate.  This is especially true if your goal is re-usable components that would put multiple identical graph types on the screen (`small multiples` or `sparklines`).

The use of closures will allow you to expose only the elements you want to attach to mouse and touch events, and provide an interface for updating your DOM elements from a parent scope downstream in your code.

Implementing the features of the [revealing module pattern](http://addyosmani.com/resources/essentialjsdesignpatterns/book/#revealingmodulepatternjavascript) you write a function that creates DOM elements and returns a closure around them. The closure gives you access to those elements of the graph without having to set up some kind of book-keeping for later selecting by `id` and `class` attributes.  The closure also gives you a function that when passed a new dataset, will update all the pertinent elements in the chart.

The goal of the pattern is to enable you to design self-contained interactive graphs that only expose useful elements to the parent scope.

## code & application

http://codepen.io/billautomata/pen/WvJvGL

Our source data is a flat array fresh from the JSON API server.

```javascript
var elements = [{
  color: 'blue',
  moving: 'flying',
  value: 1
}, {
  color: 'red',
  moving: 'flying',
  value: 4
}, {
  color: 'blue',
  moving: 'sleeping',
  value: 18
}, {
  color: 'green',
  moving: 'sleeping',
  value: 32
}]
```

Each of these objects describe an enemy in an online game the client is
building.

```javascript
{
  color: 'blue',
  moving: 'flying',
  value: 1  
}
```

You are tasked with building an interface that allows the client to see tallies
of the values for each of the categories in the data: `color` and `moving`.

* * *

`example output`

* _blue_ **19**
* _red_ **4**
* _green_ **32**
* _flying_ **5**
* _sleeping_ **50**

The useful part of the interface is that the client wants to be able to
hover over any element in the list and have everything else in the page update
itself to display new values based on the hovered context.

`example interaction behavior`

When the user hovers over `blue` in the list, the rest of the counts update
for only game objects that are blue.  

* _blue_ **19** _-hovered-_
* _red_ **0** _-updated-_
* _green_ **0** _-updated-_
* _flying_ **1** _-updated-_
* _sleeping_ **18** _-updated-_

_note_ I'm purposefully leaving out any SVG related code in this d3 article.  
I am just going to stick with HTML elements `div` and `span` to reduce the focus on the graphics themselves and shift the focus to the code architecture that generates the elements, handles events for those elements, and performs visual updates on those elements.

# setup, draw, update

### overview

Each element in the dataset has multiple categories.

```javascript
{
  color: 'blue',
  moving: 'flying',
  value: 1  
}
```

We will write a function `main` that accepts our data array, parses it based on the category we want to graph, then generates the graph based on the tallies, by category, generated by the parser.

What makes this design pattern useful is that the function returns the elements of the graph itself, and an interface function other code can use to update the elements without having to know anything about those elements.

##### pseudo-code
```javascript
function chart(dataset (array), category (string)){

  // setup
  function create_tallies(dataset){...}
  var counts = create_tallies(dataset)

  // draw
  function create_graph(counts){...}
  var graph = create_graph(counts)

  // updates
  function update_elements(dataset){...}
  update_elements(dataset)

  return {
    graph: graph,
    update: update_elements
  }

}
```

In this function `chart` we defined three new functions `create_tallies` `create_graph` and `update_elements`.

### create_tallies(data)

```javascript
function chart(data,category){
  // define the function
  function create_tallies(data) {
    // create an empty object
    var return_counts = {}
    // iterate over each element in the array
    data.forEach(function(element) {

      var category_string = element[category]

      if (return_counts[category_string] === undefined) {
        return_counts[category_string] = 0
      }

      return_counts[category_string] += element.value

    })

    return return_counts

  }

  // run the function to generate the category counts
  var counts = create_tallies(data)  

  // ... ... ...

}
```

##### pseudo-code

`create_tallies` is a function that programmatically builds a javascript object by
* creating an empty object to return
* iterating over each element in an array
* finding the value of `element[category]` storing it in `category_string`
* check to see if `return_counts[category_string]` is `undefined`
  * if true, initialize it with zero
* add the value of the element to the returned object for that key

```javascript
// from the original dataset
var elements = [{
  color: 'blue',
  moving: 'flying',
  value: 1
}, {
  color: 'red',
  moving: 'flying',
  value: 4
}, {
  color: 'blue',
  moving: 'sleeping',
  value: 18
}, {
  color: 'green',
  moving: 'sleeping',
  value: 32
}]

var counts_by_color = create_tallies(elements, 'color')
// {
//   "blue": 19,
//   "red": 4,
//   "green": 32
// }

var counts_by_moving = create_tallies(elements, 'moving')
// {
//   "flying": 5,
//   "sleeping": 50
// }
```
That `counts` object returned by create_tallies is what we use to create our initial DOM elements in our graph.  Every key in the counts object (`flying` `sleeping`) will have a corresponding DOM element.

### create_graph(counts)
```javascript

function chart(data,category){

  // create_tallies code...

  var update_selectors = []
  var parent_selectors = []

  function create_graphs(counts) {

    var body = d3.select('body')

    Object.keys(counts).forEach(function(key_name) {

      var div_parent = body.append('div')

      div_parent.datum({ key: key_name })

      var span_label = div_parent.append('span').html(key_name + ': ')
      var span_count = div_parent.append('span').html(-1)

      span_count.datum({ key: key_name })

      parent_selectors.push(div_parent)
      update_selectors.push(span_count)

    })

  }

  // ...

}
```
The visualization code here is not very graphical.  Running this code produces a list of `div` elements that look like
* blue: -1
* red: -1
* green: -1

##### pseudo-code

1. select the body
2. for each `key` found in the counts object
  1. append a `parent` div to the body
  2. append a `label` span to the `parent` div
    * initialize the html with the `key`
  3. append a `count` span to the `parent` div
    * initialize the html with `-1`
  4. push the `parent` div and the `count` span on some selectors arrays

We `push` the `count` span on the `update_selectors` array because the `update_elements` function will iterate over each selector in that array and call the `.html()` function with the correct values.  Meaning, `update_elements` will fix the content of all the spans going from the initial `-1` to the accurate value.

When we call the `.datum()` function on the `div_parent` and `span_count` objects, it is because we are going use the `key_name` associated with the element to access the data for that element in the tallies in the `update_elements` function.

### update_elements(data)

```javascript
function chart(data,category){

  // create_tallies code...
  // create_graphs code...

  function update_elements(data) {

    var new_counts = create_tallies(data)

    update_selectors.forEach(function(selector) {

      var data = selector.datum()
      if (new_counts[data.key] !== undefined) {
        selector.html(new_counts[data.key])
      } else {
        selector.html(0)
      }

    })

  }

}
```
##### pseudo-code

1. create a set of `new_counts` from an array
2. for each `selector` in the `update_selectors` array
  1. retrieve the stored `data` object from the selector
  2. if the key in that `data` object is also a key present in the counts object
    * true - set the displayed value using the `.html()` function to the value in the counts object for that key
    * false - set the displayed value using the `.html()` function to zero

When we created our graph we pushed the `span_count` objects we created onto the `update_selectors` array.  We knew to save them on the array because this span holds the value that will change later during mouse events.


### the closure

```javascript
function chart(data,category){

  // create_tallies code...
  // create_graphs code...
  // update_elements code...

  return {
    selectors: parent_selectors,
    update: update_elements
  }

}
```
The `chart` function returns an object that is effectively a portal (a closure) from the parent scope to the important stuff in the chart scope.  This returned object contains a reference to an array of DOM elements you want to attach touch and mouse event code to.  The closure also contains a reference to a function that, when passed a similarly structured data array, will parse out the relevant information and update the appropriate elements without having to be told anything about what it needs to go find.

This closure is powerful because you don't need to know anything about the DOM elements you want to update when the data changes.  

You don't need to call functions that look like `d3.selectAll('g#foo').selectAll('text#bar')...`.  

You just pass your new data to the returned `update` function and `update` knows what to do.

If you want to use the same chart twice on a page, the closure protects the two graphs from contaminating the global state and thus ruining the ability to update the elements in only one instance of the graph.  

Another benefit of this modularization technique is that a `chartA.js` and a `chartB.js` do not need to know about each other in order to be glued together in `main.js`.  If the `chart.js` code returns everything needed to attach mouse events and update itself with similar looking input data, then `main.js` has an easier time of organizing things.  

Everything relevant about that graph is contained in the object that `chart.js` returns.  If you need something extra in `main.js`, put it in that object.

![d3js-design-pattern](https://cloud.githubusercontent.com/assets/432483/8653869/c78d15e0-293b-11e5-8167-4eaed96b7048.png)
