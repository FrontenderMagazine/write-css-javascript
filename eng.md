Sometimes programming is just using the right tool. This may be a framework,
library or as it happens in my case CSS preprocessor. You probably don't realize
it, but LESS or SASS have a lot of constraints. I managed to change that by 
writing my own CSS preprocessor. I stopped writing CSS and moved everything into
the JavaScript world. This article is about [AbsurdJS][1]: a small Node.js
module, which changed my workflow completely.

## The Concept {#the-concept}

![Write your CSS with JavaScript][2]

If you write a lot of CSS you probably use preprocessor. There are two popular
- LESS and SASS. Both tools accept something, which looks like CSS, do some 
magic and export normal, pure CSS. What I did is just to replace the instrument 
and the input format. I didn't want to invent a new language or syntax, because 
this is connected with a lot of things like parsing and compiling. Thankfully, 
Node.js is here and I decided to use it. Also, I had a lot of LESS type projects,
which means that I already use Node.js to compile my styles. It was much easier 
to replace a module instead of adding something completely new.

## The Input {#the-input}

![Write your CSS with JavaScript][3]

I think that the closest thing to the CSS format is JSON -- that's what
AbsurdJS accepts. Of course there are some cons of this transformation. You have
to put some properties in quotes and of course the values. This needs a little 
bit more time during the writing, but as you will see below it's worth it.

## In the Beginning was ... a JavaScript File {#in-the-beginning-was-a-javascript-file}

Here is how a simple LESS file looks like:

    .main-nav {
        background: #333;
        color: #000;
        font-weight: bold;
        p {
            font-size: 20px;
        }
    }
    

And here is its AbsurdJS equivalent. It's a simple Node.js module:

    module.exports = function(api) {
        api.add({
            ".main-nav": {
                background: "#333",
                color: "#000",
                "font-weight": "bold",
                p: {
                    "font-size": "20px"
                }
            }
        })
    }
    

You should assign a function to `module.exports`. It accepts a reference to the
API, which has several methods, but the most important one is `add`. Simply pass
a JSON object and it will be converted to CSS.

To compile the less file we need install LESS's compiler via 
`npm install -g less` and run 

    lessc .\css.less > styles.less.css
    

It's the almost the same with AbsurdJS. Installation is again via node package
manager
- `npm install -g absurd`. 

    absurd -s css.js -o styles.absurd.css
    

It accepts source and output; the result is the same.

## The Truth {#the-truth}

You may have really beautiful and nice-looking LESS or SASS files, but what
matters is the final compiled CSS. Unfortunately the result is not always the 
best one.

### Combining {#combining}

Let's get the following example:

    .main-nav {
        background: #333;
    }
    .main-content {
        background: #333;
    }
    

If you pass this to the current preprocessors you will get the same thing in
the end. However if you use AbsurdJS like that:

    module.exports = function(api) {
        api.add({
            ".main-nav": {
                background: "#333"
            },
            ".main-content": {
                background: "#333"
            }
        })
    }
    

After the compilation you will get 

    .main-nav, .main-content {
        background: #333;
    }
    

SASS has a feature called *placeholders* which does the same thing. However, it
comes with its own problems. Placeholders can't accept parameters and you should
repeat them in every selector which you want to combine. My solution just parses
the rules and combine them. Let's get a little bit more complex example:

    {
        ".main-nav": {
            background: "#333",
            ".logo": {
                color: "#9f0000",
                margin: 0
            }
        },
        ".main-content": {
            background: "#333"
        },
        section: {
            color: "#9f0000",
            ".box": {
                margin: 0
            }
        }
    }
    

The result is 

    .main-nav, .main-content {
        background: #333;
    }
    .main-nav .logo, section {
        color: #9f0000;
    }
    .main-nav .logo, section .box {
        margin: 0;
    }
    section .box {
        padding: 10px;
        font-size: 24px;
    }
    

All identical styles are combined into one single definition. I know that the
browsers are really fast nowadays and this is not exactly the most important 
optimization, but it could decrease the file size.

### Overwriting {#overwriting}

You know that if you have two identical selectors and they contain definition
of the same style the second one overwrites the first. The following code passed
through LESS/SASS stays the same:

    .main-nav {
       font-size: 20px;
    }
    .main-nav {
       font-size: 30px;
    }
    

However I think that this leaves one more operation for the browser: it has to
find out that there is another definition with the same selector and style and 
compute the correct value. Isn't it better to avoid this, so send that directly:

    .main-nav {
        font-size: 30px;
    }
    

AbsurdJS takes care about this and produces only one definition. The input may
look like that:

    {
        ".main-nav": {
            "font-size": "20px"
        },
        ".main-nav": {
            "font-size": "30px"
        }
    }
    

It also makes your debugging processes easier, because there is no so long
chain of overwrites.

### Flexibility {#flexibility}

Ok, we have mixins, variables, placeholders, functions, but once you start
using them to write a little bit more complex things you are stuck. Let's get 
the mixins. I want to create a mixin, which defines another mixin. That's 
currently not possible in LESS, because you can't use a mixin defined in another
mixin. I guess it's a scope problem. SASS has some [imperfections][4] regarding
the interpolation of the variables. Overall, it's hard to produce good 
architecture with less code. You have to write a lot and even then, you can't 
really achieve your goals. The main reason behind these problems is the fact 
that both, LESS and SASS, have to deal with new syntax, new rules and basically 
invent a new compiler. However, if we use JavaScript we don't have to think 
about these issues.

AbsurdJS has something called *storage*. It could save whatever you want and
make it available in other files. For example:

    // B.js
    module.exports = function(api) {
        api.storage("theme", function(type) {
            switch(type) {
                case "dark": return { color: "#333", "font-size": "20px" }; break;
                case "light": return { color: "#FFF", "font-size": "22px" }; break;
                default: return { color: "#999", "font-size": "18px" };
            }
        });
    }
    
    // A.js
    module.exports = function(api) {
        api
        .import(__dirname + "/B.js")
        .add({
            ".main-nav": [
                {
                    "font-size": "16px",
                    padding: 0,
                    margin: 0
                },
                api.storage("theme")("dark")
            ]
        });
    }
    

At the end you get:

    .main-nav {
        color: #333;
        font-size: 20px;
        padding: 0;
        margin: 0;
    }
    

Using the storage may be a little bit ugly. I mean, you need an array assigned
to the selector and then call `api.storage`. I used that for a while, but later
decided to implement something much nicer. It's a feature which I always wanted
- the ability to create your own properties and save tons lines. For example, 
let's create a new property called `theme` and process its value.

    // B.js - definition of the plugin 
    module.exports = function(api) {
        api.plugin('theme', function(api, type) {
            switch(type) {
                case "dark": return { color: "#333", "font-size": "20px" }; break;
                case "light": return { color: "#FFF", "font-size": "22px" }; break;
                default: return { color: "#999", "font-size": "18px" };
            }
        });
    }
    
    // A.js - its usage
    module.exports = function(api) {
        api
        .import(__dirname + "/B.js")
        .add({
            ".header": {
                theme: "light"
            }
        })
    }
    

Again, the result is similar:

    .header {
        color: #FFF;
        font-size: 22px;
    }
    

## Conclusion {#conclusion}

[AbsurdJS][5] is something really simple, but avoids the usage of popular CSS
preprocessors. It still has the same feature like nested selectors, media 
queries bubbling, file import, variables, mixins and so one. However, it brings 
more flexibility, because it is a pure JavaScript. It has even a
[GruntJS support][6]. I'd like to get some feedback and will be happy if you
take a part in the project. The official repository is available here
[https://github.com/krasimir/absurd][7].

 [1]: https://github.com/krasimir/absurd
 [2]: img/concept.gif
 [3]: img/input.jpg
 [4]: http://krasimirtsonev.com/blog/article/Two-handy-and-advanced-SASS-features-and-their-limitations
 [5]: https://github.com/krasimir/absurd#usage
 [6]: https://github.com/krasimir/absurd#using-with-grunt
 [7]: https://github.com/krasimir/absurd