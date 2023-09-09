# SVG viewer written in SVG

This post is about how to write an [SVG](https://en.wikipedia.org/wiki/SVG)
viewer / browser / "slideshow" which is itself a self-contained SVG.

## Motivation

I've been working on a parallel processing pipeline. Each stage of the pipeline
is running on a separate thread, and it takes some work items from a queue in
front of it, processes them and then puts them in the queue in front of the next
stage in the pipeline.

In order to better understand what exactly is going on I thought I'd visualise
the pipeline including the length and contents of all queues as well as the
position each worker/stage is at in the queue it's processing.

For now lets just imagine that an SVG image is created every time interval. So
after a run of the pipeline we'll have a bunch of SVGs showing us how it evolved
over time.

Initially I was using the [`feh`](https://feh.finalrewind.org/) image viewer,
which if you pass it several images lets you navigate through them using the
arrow keys.

But then I wondered: how can I show these SVGs to somebody else over the web?

## Demo

Before I show you how I did it, let's have a look at the resulting pipeline
visualisation (you need to click on the image):

[![Demo](https://stevana.github.io/svg-viewer-in-svg/wordcount-pipeline.svg)](https://stevana.github.io/svg-viewer-in-svg/wordcount-pipeline.svg)

The arrows in the top left corner are clickable and will take you to the first,
next, previous and last SVG respectively.

What you are seeing is a run of a parallel word count pipeline, where lines are
coming in from stdin and the counts are being written to stdout at the end.

## The code

Let's start by having a look at the SVG itself.

```xml
<svg version="1.1" xmlns="http://www.w3.org/2000/svg">

  // The navigation menu for going to the first, previous, next and last
  // slide/image. There's also a progress bar here which shows which slide
  // we are currently on and how many there are in total.
  <g font-family="Times,serif" font-size="14.00">
    <text id="first"    x="20"  y="30">⇤</text>
    <text id="previous" x="50"  y="30">←</text>
    <text id="progress" x="80"  y="30"></text>
    <text id="next"     x="120" y="30">→</text>
    <text id="last"     x="150" y="30">⇥</text>
  </g>

  // Placeholder for the image.
  <g id="image"></g>

  // The index of the currently viewed image.
  <desc id="0"></desc>

  // The fact that we can embedd JavaScript into SVGs is what makes this
  // whole thing work.
  <script>
  // <![CDATA[

    // Let me move this out to its own code block, so we get syntax
    // highlighting.

  // ]]>
  </script>
</svg>
```

The following goes into the script tag above:

```javascript
    // Array holding the SVG images.
    const imgs = new Array(
      "<svg>...</svg>",
      "<svg>...</svg>",
      "<svg>...</svg>",
      );

    // Helper for registering onclick handlers.
    function registerClick(selector, f) {
        document.querySelector(selector).addEventListener("click", (e) => {
            f(e);
        });
    }

    // Set and return the value of our counter, this is abusing the id
    // of the desc tag...
    function setCounter(f) {
        const counter = document.querySelector("desc");
        counter.id = f(parseInt(counter.id));
        return counter.id;
    }

    // Updates our image placeholder by injecting the SVG into the
    // image tag. Also updates the progress bar.
    function setImage(i) {
        const img = document.querySelector("#image");
        img.setAttribute("href", imgs[i]);
        updateProgress();
    }

    // Update the progress bar in the menu.
    function updateProgress() {
        document.querySelector("#progress").innerHTML =
            document.querySelector("desc").id + "/" + (imgs.length - 1);
    }

    // We can now define our navigation functions in terms of setting
    // the counter and the image.

    function first() {
        setImage(setCounter((_) => 0));
    }

    function previous() {
        setImage(setCounter((i) => i <= 0 ? 0 : --i));
    }

    function next() {
        setImage(setCounter((i) => i >= imgs.length - 1 ? imgs.length - 1 : ++i));
    }

    function last() {
        setImage(setCounter((_) => imgs.length - 1));
    }

    // Finally, to kick things off: register onclick handlers for the
    // navigation buttons and set the image to the first image in the array.
    registerClick("#first",    (_) => first());
    registerClick("#next",     (_) => next());
    registerClick("#previous", (_) => previous());
    registerClick("#last",     (_) => last());
    setImage(0);

    // We could even add keyboard support...
    window.addEventListener("keydown", (e) => {
        // Left arrow or k.
        if (e.keyCode === 37 || e.keyCode === 75) {
            previous();
        }
        // Right arrow or j.
        else if (e.keyCode === 39 || e.keyCode === 74) {
            next();
        }
    });
```

Another thing worth mentioning is that in my application the thread that
collects the metrics runs about 1000 times per second. If there's no change in
the metrics then we probably don't want to display an image that's the same as
the previous one. So I keep a CRC32 checksum of the metrics that the last image
is generated from and if the next metrics data has the same checksum, I skip
generating that image (as it will be the same as the previous one).

The (inner) SVGs themselves are generated with graphviz via the
[dot](https://graphviz.org/doc/info/lang.html) language, the
[record-based](https://graphviz.org/doc/info/shapes.html#record) node shapes
turned out to be useful for visualing data structures.

It's quite annoying to populate the `imgs` array by hand, so I wrote a small
bash [script](src/svg-viewer-in-svg) which takes a bunch of SVGs and outputs a
single SVG which can be used to view the original images.

## Usage

The easiest way to get started is probably to clone the repository.

```bash
git clone https://github.com/stevana/svg-viewer-in-svg
cd svg-viewer-in-svg
```

In the `img/` directory there are three simple SVGs:

```bash
ls img/
circle.svg  ellipse.svg  rectangle.svg
```

We can combine them all into one a single SVG that is a "slideshow" of the
shapes as follows:

```bash
./src/svg-viewer-in-svg img/*.svg > /tmp/combined-shapes.svg
firefox /tmp/combined-shapes.svg
```

If you want to "install" the script, simply copy it to any directory that is in
your `$PATH`.

One last thing worth noting is that hosting these combined SVGs on GitHub is a
bit of a pain. Merely checking them into a repository and trying to include them
in markdown won't work, because GitHub appears to be doing some SVG script tag
sanitising for security reasons. Uploading them to gh-pages and linking to those
seems to work though[^1].

## Contributing

I hope I've managed to inspire you to think about how to visualise the execution
of your own programs! Feel free to copy and adapt the code as you see fit. If
you come up with some interesting modifications or better ways of doing things,
then please do share!

## See also

Brendan Gregg's [flamegraphs](https://www.brendangregg.com/flamegraphs.html)
also generates a clickable SVG. I got the idea of adding keyboard support from
looking at his SVG, there's probably more interesting stuff to steal there.


[^1]: The following [gist](https://gist.github.com/ramnathv/2227408) shows how
    to create gh-pages branch that doesn't have any history. Also see the GitHub
    pages documentation for how to enable gh-pages for a respository.
