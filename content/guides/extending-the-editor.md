---
title: "Extending the Editor"
date: 2020-09-24T14:24:22+02:00
draft: false
layout: "section-page"
subsection: "guides"
blurb: "This guide explains how you can extend the Annotorious/RecogitoJS editor popup with your own UI components."
weight: 2
meta_title: "Extending the Annotorious/RecogitoJS editor"
meta_description: "This guide explains how you can extend the Annotorious/RecogitoJS editor popup with your own UI components."
meta_link: "https://recogito.github.io/guides/extending-the-editor"
---

# Extending the Editor

Per default, the Annotorious/RecogitoJS editor popup features two widgets:

- A __comment widget__ for writing comments and replies
- A __tag widget__ for adding tags, either freetext or supported by a pre-configured vocabulary and an autosuggest dropdown

![Editor popup](/images/guides/editor-popup-with-vocab.png)

You can can customize this setup when you initialize Annotorious/RecogitoJS using the 
`widgets` config option. The configuration below will give you the editor shown in the 
image above.

```js
var anno = Annotorious.init({
  image: 'hallstatt',
  widgets: [
    'COMMENT',
    { widget: 'TAG', vocabulary: [ 'Animal', 'Building', 'Waterbody'] }
  ]
});
```

Alternatively, the configuration below will give you an editor with just a comment widget, but no tag widget.

```js
var anno = Annotorious.init({
  image: 'hallstatt',
  widgets: [
    'COMMENT'
  ]
});
``` 

Each item in the `widgets` array can be one of the following:

- The name of a __built-in widget__ - currently either `COMMENT` or `TAG`
- An editor __plugin__ (we'll learn more about that below)
- An object with a `widget` property that has the widget name or plugin as value, and 
  additional configuration arguments for the widget

## Building Your Own Widgets

You can extend the editor with your own __widgets__. A widget can be:

1. an ordinary JavaScript function that takes a set of input arguments and returns 
   a DOM element, which will be inserted into the editor
2. a [React](https://reactjs.org/docs/introducing-jsx.html) component conforming to the 
   same interface conventions as the built-in widgets. See __[this example React plugin](/guides/example-react-plugin)__ 
   for more information.

### Annotation Bodies

> Before you start writing your own widgets, you should first familiarize yourself with the
> the [W3C Web Annotation specification](/annotorious/getting-started/web-annotation), in 
> particular the concepts of __annotation bodies__ and body __purposes__.

In the terminology of the  [W3C Web Annotation](https://www.w3.org/TR/annotation-model/) spec, an 
annotation consists of multiple __bodies__ - data structures that each represent one "piece" of the
annotation. For example, if an annotation consists of one comment and multiple tags, then each is 
encoded as one body in the data structure.

```json
{ 
  "@context": "http://www.w3.org/ns/anno.jsonld",
  "id": "#ce0ed291-766b-4763-8e91-90ce1d04e706",
  "type": "Annotation",
  "body": [{
    "type": "TextualBody",
    "value": "This is a comment",
    "purpose": "commenting"
  }, {
    "type": "TextualBody",
    "value": "A Tag",
    "purpose": "tagging"
  }, {
    "type": "TextualBody",
    "value": "Another Tag",
    "purpose": "tagging"
  }],

  ...

}
```

To specify the body type, the W3C model has a `type` field, and an extra (optional) 
`purpose`. For example, tags and comments are both of type `TextualBody`, but with different 
purposes - `commenting` and `tagging`.

### Editor Widgets

The [Editor](https://github.com/recogito/recogito-client-core/blob/master/src/editor/Editor.jsx)
handles basic general concerns, like positioning of the popup window and managing the editing state. 
Otherwise though, it is just a shell for __widgets__. __Widgets__ implement user interface representations for
particular types of __bodies__. The built-in
[CommentWidget](https://github.com/recogito/recogito-client-core/blob/master/src/editor/widgets/comment/CommentWidget.jsx)
handles `TextualBody` nodes with purpose `commenting` or `replying`. The built-in
[TagWidget](https://github.com/recogito/recogito-client-core/blob/master/src/editor/widgets/tag/TagWidget.jsx)
takes care of `TextualBody` nodes with purpose `tagging`. Want to support a new type of body? Make a new widget for it!

## A 'Hello World' Widget

As an example, let's create a minimal editor widget that adds a color selector. Clicking the
selector creates a new annotation body with the purpose `highlighting`.

![Editor popup](/images/guides/colorselector-widget.png)
 
```js
var ColorSelectorWidget = function(args) {

  // 1. Find a current color setting in the annotation, if any
  var currentColorBody = args.annotation ? 
    args.annotation.bodies.find(function(b) {
      return b.purpose == 'highlighting';
    }) : null;

  // 2. Keep the value in a variable
  var currentColorValue = currentColorBody ? currentColorBody.value : null;

  // 3. Triggers callbacks on user action
  var addTag = function(evt) {
    if (currentColorBody) {
      args.onUpdateBody(currentColorBody, {
        type: 'TextualBody',
        purpose: 'highlighting',
        value: evt.target.dataset.tag;
      });
    } else { 
      args.onAppendBody({
        type: 'TextualBody',
        purpose: 'highlighting',
        value: evt.target.dataset.tag;
      });
    }
  }

  // 4. This part renders the UI elements
  var createButton = function(value) {
    var button = document.createElement('button');

    if (value == currentColorValue)
      button.className = 'selected';

    button.dataset.tag = value;
    button.style.backgroundColor = value;
    button.addEventListener('click', addTag); 
    return button;
  }

  var container = document.createElement('div');
  container.className = 'colorselector-widget';
  
  var button1 = createButton('RED');
  var button2 = createButton('GREEN');
  var button3 = createButton('BLUE');

  container.appendChild(button1);
  container.appendChild(button2);
  container.appendChild(button3);

  return container;
}
```

Here's what the code does, explained step by step:

1. We grab the first body with `purpose: 'highlighting'` from the annotation.
2. If a body exists, we store the body `value` in the `currentColorValue` variable.
3. The `addTag` function is called whenever the user makes a selection in the widget. 
   (We'll create the widget later in the code). If the annotation already has a
   `highlighting` body, `addTag` updates this body. If the annotation does not have
   a `highlighting` body yet, `addTag` appends a new one.
4. The remainder of the code renders the user interface elements: 3 identical buttons
   in different colors. Clicking a button triggers `addTag`.
5. Just add a bit of CSS for style

Since the `highlighting` body is now stored in the annotation, we can write a 
[formatter](/annotorious/api-docs/annotorious/#formatters) that renders highlighted 
annotations in different colors. 

```js
/** A matching formatter that sets the color according to the 'highlighting' body value **/
var ColorFormatter = function(annotation) {
  var highlightBody = annotation.bodies.find(function(b) {
    return b.purpose == 'highlighting';
  });

  if (highlightBody)
    return highlightBody.value;
}
```

Finally, we add the plugin and formatter to Annotorious on initialization, and apply some
CSS for styling.

```js
var anno = Annotorious.init({
  image: 'hallstatt',
  widgets: [ 
    ColorSelectorWidget,
    'COMMENT',
    'TAG'
  ],
  formatter: ColorFormatter
});
``` 

```css
/* 5. CSS styles for the color selector widget */
.colorselector-widget {
  padding:5px;
  border-bottom:1px solid #e5e5e5;
}

.colorselector-widget button {
  outline:none;
  border:none;
  display:inline-block;
  width:20px;
  height:20px;
  border-radius:50%;
  cursor:pointer;
  opacity:0.5;
  margin:4px;
}

.colorselector-widget button.selected,
.colorselector-widget button:hover {
  opacity:1;
}

svg.a9s-annotationlayer .a9s-annotation.RED .a9s-outer {
  stroke:red;
  stroke-width:3;
  fill:rgba(255, 0, 0, 0.3);
}

svg.a9s-annotationlayer .a9s-annotation.GREEN .a9s-outer {
  stroke:green;
  stroke-width:3;
  fill:rgba(0, 255, 0, 0.3);
}

svg.a9s-annotationlayer .a9s-annotation.BLUE .a9s-outer {
  stroke:blue;
  stroke-width:3;
  fill:rgba(0, 0, 255, 0.3);
}

svg.a9s-annotationlayer .a9s-annotation.RED .a9s-inner,
svg.a9s-annotationlayer .a9s-annotation.GREEN .a9s-inner,
svg.a9s-annotationlayer .a9s-annotation.BLUE .a9s-inner {
  fill:transparent;
  stroke:none;
}
```

## Extending the Editor with React

If you want to extend the editor with a React component, 
see [this example](/guides/example-react-plugin).