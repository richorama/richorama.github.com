---
layout:     post
title:      Enter the Browser
date:       2015-10-02 13:18:00
summary:    In this post we'll look at moving code into the browser, to make a single page application - the node way!
---

In this series of blog posts, we're looking at how to set up a simple web app in node.

In the [previous installment](http://richorama.github.io/2015/09/25/storing-data/) we looked at storing data, the node way.

In this post we'll look at moving code into the browser, to make a single page application - the node way!

## Creating an API

Let's add some extra routes to our express app, to allow data to be posted and retrieve in JSON format. The code is very simliar to the code we had before, so you can replace the existing routes, or add these as well:

{% highlight javascript %}
app.post('/api', function(req, res){
  db.put(guid(), req.body.todo, function(err){
    if (err) return res.json({error:err});
    res.json({ok:true});
  });
});

app.delete('/api/:key', function(req, res){
  db.del(req.params.key, function(err){
    if (err) return res.json({error:err});
    res.json({ok:true});
  });
});

app.get('/api', function(req, res){
  var list = [];
  var stream = db.createReadStream();
  stream.on('data', function(data) {
    list.push(data);
  });
  stream.on('end', function() {
    res.json(list);
  });  
});
{% endhighlight %}
(the only real difference is that we're calling `json` on the response object).

We'll also need to allow json to be posted. Let's set up the body parser middleware to handle JSON by adding this line (you can add it next to your existing  body parser line)

{% highlight javascript %}
app.use(bodyParser.json());
{% endhighlight %}

That's a simple JSON API set up, you should be able to see your todos if you start up the web server and open this page in your browser: `http://localhost:8080/api`.

Now let's focus our effort on the browser.

## Browser side

Let's create a new directory for the source code which will run in the browser (so it doens't get confused with the server-side code). 

{% highlight text %}
> mkdir client
{% endhighlight %}

One of the first things we want to do it connect to our API.

I've got a really lightweight JavaScript module I use to do this - which you can borrow :¬)

{% highlight javascript %}
function makeRequest(method, uri, body, cb){
    var xhr = new XMLHttpRequest();
    xhr.open(method,uri,true);
    xhr.onreadystatechange = function(){
        if(xhr.readyState !== 4) return
        if(xhr.status < 400) return cb(null, JSON.parse(xhr.responseText));
      cb(xhr.status); 
    };
    xhr.setRequestHeader("Content-Type","application/json");
    xhr.setRequestHeader("Accept","application/json");
    xhr.send(body);
};

module.exports.get = function(url, cb){
  makeRequest("GET", url, null, cb);
}

module.exports.post = function(url, data, cb){
    makeRequest("POST", url, JSON.stringify(data), cb);
}

module.exports.del = function(url, cb){
    makeRequest("DELETE", url, null, cb);
}
{% endhighlight %}

Let's call this `http-request.js` and put it in the `client` directory.

Next let's create an `index.js` file which we'll use to call this module.

{% highlight javascript %}
var http = require('./http-request')

var http.get('/api', function(err, todos){
  console.log(todos);
});
{% endhighlight %}

__Hold on!__ This looks like node.js code. We're using the `require` function to load our `http-request` module, which we're loading off the disk. `require` isn't available in the browser, so what's going on?

We're going to use [browserify](http://browserify.org/), which is a useful command line tool for bundling together multiple source files, to build a single bundle we can deploy to the browser. Browserify parses throuhgh out JavaScript code, and finds all the call to `require`, and bundles all of the referenced modules together into a single file. The fantastic thing with browserify is that we can tap into the npm registry, and use node.js modules in the browser.

Let's install browserify, as well as uglify (which is a JavaScript minifier):

{% highlight text %}
> npm install browserify uglify-js -g
{% endhighlight %}

> The `-g` option installs the module globally, which registers it as a command line tool

Now we can create the bundle on the command line:

{% highlight text %}
> browserify client/index.js | uglifyjs > public/index.min.js
{% endhighlight %}

This command will create an `index.min.js` file in the `public` directory with our bundled and minified JavaScript. You'll need to run this command every time you change the client JavaScript files.

> There are lots of automation tools in node, grunt and gulp being popular options. I normally create a makefile - but perhaps I'm old fashioned :¬)

Now let's create an `index.html` in the `public` directory which will load the script file:

{% highlight html %}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="//maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap.min.css"></style>
  </head>
  <body>
    <div class="container" id="content"></div>
    <script src="index.min.js"></script>
  </body>
</html>
{% endhighlight %}

> Note that I have abandoned the server-side views, this static HTML file will be served instead.

If you fire up the application, you should now see your todos written to the console (open the developer tools with `F12`).
