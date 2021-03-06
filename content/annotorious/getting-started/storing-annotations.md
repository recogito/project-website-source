---
title: "Storing Annotations"
date: 2020-06-01T10:20:00+02:00
draft: false
layout: "section-page"
subsection: "getting-started"
blurb: "To store annotations, you need an annotation server. __Annotorious does not come with built-in storage__. Learn how you can use the JavaScript API to set up your own solution that fits your needs." 
weight: 4
meta_title: "Getting Started: Storing Annotations"
meta_description: "Learn how you can use the Annotorious JavaScript API to integrate a storage backend"
meta_link: "https://recogito.github.io/annotorious/getting-started/storing-annotations"
---

# Storing Annotations

To store annotations, you need an annotation server or database backend.
Annotorious - as well as its sister project [RecogitoJS](https://github.com/recogito/recogito) - do not 
include their own storage solution. They provide __the user interface functionality for annotation 
only__. To store annotations, you need to connect them to your own backend, using the [API](/annotorious/api-docs/).

When the user creates, updates or deletes an annotation, Annotorious emits the following events:

- __createAnnotation__
- __updateAnnotation__
- __deleteAnnotation__

Subscribe to these events, and execute the corresponding write operations to your backend. 

```javascript
anno.on('createAnnotation', function(annotation) {
  // This part depends entirely on how your backend works
  axios.post('http://www.example.com/annotations/).then((response) => {
    console.log('Stored.');
  });
});
```

## Using Cloud Storage Services

If you don't want to run your own application backend, cloud storage services offer a good alternative.
See [this guide for using Google Firebase](/guides/using-firebase-for-storage/) as your own free & personal 
annotation store.
