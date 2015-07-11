We begin with a flat array of data as it comes from the JSON API server.

```javascript
var elements = [{
  color: 'blue',
  type: 'flying',
  value: 1
}, {
  color: 'red',
  type: 'flying',
  value: 4
}, {
  color: 'blue',
  type: 'sleeping',
  value: 18
}, {
  color: 'green',
  type: 'sleeping',
  value: 32
}]
```

Each of these objects describe an enemy in an online game the client is
building.

```javascript
{
  color: 'blue',
  type: 'flying',
  value: 1  
}
```

You are tasked with building an interface that allows the client to see tallies
of the values for each of the categories in the data: `color` and `type`.

* _blue_ **19**
* _red_ **4**
* _green_ **32**
* _flying_ **5**
* _sleeping_ **50**

The useful part of the interface is that the client wants to be able to
hover over any element in the list and have everything else in the page update
itself to display new values based on the hovered context.

`example`

When the user hovers over `blue` in the list, the rest of the counts update
for only game objects that are blue.  

* _blue_ **19** _-hovered-_
* _red_ **0** _-updated-_
* _green_ **0** _-updated-_
* _flying_ **1** _-updated-_
* _sleeping_ **18** _-updated-_

* * *

_note_ I'm purposefully leaving out any SVG related code in this d3 article.  
I am just going to stick with HTML elements `div` and `span` to reduce the focus
on the graphics themselves and shift the focus to the code architecture that
generates the elements, handles events for those elements, and performs visual updates on those elements.

* * *

### setup, draw, update

Each element in the dataset has multiple categories.

```javascript
{
  color: 'blue',
  type: 'flying',
  value: 1  
}
```

We need to write a function that accepts the entire dataset array, parses it
based on the category we want to graph, then generates the graph based on the
tallies, by category, generated by the parser.

### Psuedocode
```
function create_chart(dataset (array), category (string)){

  function create_tallies(dataset){...}
  var counts = create_tallies(dataset)

  function create_graph(counts){...}
  var graph = create_graph(counts)

  function update_elements(dataset){...}
  update_elements(dataset)

  return {
    graph: graph,
    update: update_elements
  }

}
```

In this function `create_chart` we defined three new functions `create_tallies`
`create_graph` and `update_elements`.

### create_tallies(data)

```javascript
function main(data,category){

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

##### psuedo-code

`create_tallies` is a function that programmatically builds a javascript object by
* creating an object to return
* iterating over each element in an array,
* finding the value of `element[category]` storing it in `category_string`
* check to see if `return_counts[category_string]` is `undefined`
  * if true, initialize it with zero
* add the value of the element to the returned object for that key

```javascript
// from the original dataset
var elements = [{
  color: 'blue',
  type: 'flying',
  value: 1
}, {
  color: 'red',
  type: 'flying',
  value: 4
}, {
  color: 'blue',
  type: 'sleeping',
  value: 18
}, {
  color: 'green',
  type: 'sleeping',
  value: 32
}]

var counts_by_color = create_tallies(elements, 'color')
// {
//   "blue": 19,
//   "red": 4,
//   "green": 32
// }

var counts_by_type = create_tallies(elements, 'type')
// {
//   "flying": 5,
//   "sleeping": 50
// }
```
That `counts` object returned by create_tallies is what we use to create our graph.

### create_graph(counts)
```javascript

function main(data,category){

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
The visualization code here is not very graphical.  Running this code produces a list that looks like:
 * blue: -1
 * red: -1
 * green: -1

##### psuedo-code

1. select the body
2. for each `key` found in the counts object
  1. append a `parent` div to the body
  2. append a `label` span to the `parent` div
    * initialize the html with the `key`
  3. append a `count` span to the `parent` div
    * initialize the html with `-1`
  4. push the `parent` div and the `count` span on some selectors arrays

We `push` the `count` span on the `update_selectors` array because the `update_elements` function will iterate over all the `update_selectors` elements and call the `.html()` function with the correct values.

When we call the `.datum()` function on the `div_parent` and `span_count` variables, it is because we are going use the `key_name` associated with the element to access the data related to that element in the `update_elements` function.

### update_elements()
```javascript

function main(data,category){

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
##### psuedo-code

1. create a set of `new_counts` from an array
2. for each `selector` in the `update_selectors` array
  1. retrieve the stored `data` object from the selector
  2. if the key in that `data` object is also a key present in the counts object
    * true - set the displayed value using the `.html()` function to the value in the counts object for that key
    * false - set the displayed value using the `.html()` function to zero

When we created