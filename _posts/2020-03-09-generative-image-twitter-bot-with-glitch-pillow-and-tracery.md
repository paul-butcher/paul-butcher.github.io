---
layout: post
title: A Generative Image Twitter bot with Glitch, Pillow and Tracery
date: 2020-03-09 20:57 +0000
---

[@RandomTartan](https://twitter.com/RandomTartan) is a Twitter 
bot that posts a randomly generated tartan image every hour. It is 
driven by [CheapBotsDoneQuick](https://cheapbotsdonequick.com/) and 
[Glitch](https://glitch.com/), and uses [Pillow](https://pillow.readthedocs.io/en/stable/).

Tartans are defined by a formulaic [threadcount definition](https://www.tartanregister.gov.uk/threadcount)
making them an ideal candidate for a project that generates images from a text
description.

This project is in three parts - 1. A [Tracery](https://tracery.io) grammar
to generate a threadcount definition, hosted on cheapbotsdonequick; 2. A Python module
that parses a threadcount and returns a PIL image; 3. A 
Python project hosted on Glitch to respond to requests with the image.

Although Twitter does not currently accept `image/svg+xml` images, 
CheapBotsDoneQuick does allow you to generate [SVG](https://www.w3.org/Graphics/SVG/)
images directly using tracery, which it converts into a format that Twitter does
accept.  However, I wanted to post both the text and the image derived from it.
The easiest way to do this was to generate the text, then generate the image 
from the text, rather than to try to interleave both SVG and text generation 
in a tracery grammar.

## The Tracery part

Tracery uses a grammar description to generate text. This is a set of rewrite rules
similar to a [Context-free Grammar](https://en.wikipedia.org/wiki/Context-free_grammar).
It also offers the facility to save data for later reuse.  In generative storytelling
this might be used to generate the name of the protagonist, which can then be
used throughout the story.  @RandomTartan uses this facility to generate the threadcount
then output it twice - once in the body of the tweet, and once as part of the URL
of the image.

Here is the grammar used by @RandomTartan
```javascript
{
	"origin": ["[this_threadcount:#threadcount#]#this_threadcount# {img https://filly-more.glitch.me/tartan.png?threadcount=#this_threadcount#}"],
	"threadcount": ["[symmetrical:#symmetry#]#terminaldef# #threaddef# #threaddef# #threaddefs#"],
	"terminaldef":["#colour##symmetrical##count#"], 
	"threaddefs": ["#threaddef# #threaddefs#", "#terminaldef#"],
	"threaddef": ["#colour##count#"],
	"colour": [
		"LR", "R", "DR", "O", "DO", "LY", "Y", "DY", "LG", "G", "DG", "LB", "B",
		"DB", "LP", "P", "DP", "W", "LN", "N", "DN", "K", "LT", "T", "DT"
	],
	"count": ["#digit0##digit1#", "10", "20", "30"],
	"digit0": ["", "1", "2", "3", "", ""],
	"digit1": ["2", "4", "6", "8"],
	"symmetry": ["", "/"]
}
```

The above grammar generates output like this:

```text
DR10 B20 R18 R20 <img src="https://filly-more.glitch.me/tartan.png?threadcount=DR10 B20 R18 R20">
```

Some of the choices I made here were:

1. Only even numbered threadcounts. Because the number of terminals in the grammar
influences the output, I restricted the scope of `digit1` so that it didn't generate
too many similar results.

2. At least four stripes. [Legally](http://www.legislation.gov.uk/asp/2008/7/section/2),
a tartan has a minimum of two stripes, but at that minimum, it's essentially gingham.
Creating actually "tartan-looking" tartans, you need more stripes.

3. A maximum of 38 threads per stripe. This seemed like a reasonable maximum.  If I
allow it to be too large, it would be likely to generate a tartan that could not fit a 
full sett in a 512 pixel square.  The variation in width between really wide and really 
narrow stripes would also spoil the effect.

4. Separate entries for the tens.  I wanted the threadcounts to look normal, not like they had
been algorithmically generated.  I didn't want to have stripes of zero width, or zero-padded 
numbers under 10.

5. Multiple empty string entries for the first digit.  This is to add a bias in favour of 1-digit numbers.

## The Python part

The python module is documented on [readthedocs](https://tartan.readthedocs.io/en/latest/). and stored
on [Github](https://github.com/paul-butcher/tartan/blob/master/tartan/tartan.py).
To create the woven pattern, I construct an image each for warp and weft, then paste them together with
a checkerboard mask, signifying the alternating over/under pattern of the threads.

The image generation funciton returns a PIL image, so that the Glitch project can serialise it in the
most appropriate manner.

The colour palette was taken from the Scottish Register of Tartans.  They have a document listing a 
number of possible realisations per colour abbreviation that they use for rendering tartans.
I chose one colour for each, because I wanted the results to be consistent.

## The Glitch part

The [Glitch project](https://glitch.com/~filly-more) is very slim.  It is a [Flask](https://flask.palletsprojects.com/)
application that plumbs requests containing threadcounts together with responses from the tartan python
module.

The significant part of the project is the tartan image route. It saves the image to a BytesIO buffer,
and constructs a response with it.

```
@app.route("/tartan.png")
def get_image():
    buffer = BytesIO()

    img = threadcount_to_image(request.args['threadcount'], (512, 512))
    img.save(buffer, "PNG")

    response = app.make_response(buffer.getvalue())
    response.mimetype = "image/png"
    return response
```

By using Glitch simply to route requests to a python module installed form Pypi, The Glitch project
can easily be remixed to create other image generating bots, with changes to only a few lines. 
The python project can also be easily and independently extended and reused.
For example, the python project also exposes a CLI.

## Conclusions

Combining CheapBotsDoneQuick with Glitch is a spectacularly easy way to create a Twitter bot that posts
on a schedule or in response to @mentions

This project has resulted in some interesting tartans. Despite conforming to the legal definition of 
a tartan, and with my added constraint of having at least four stripes, many of the patterns do not 
match my intuitive concept of what a tartan looks like.

* Symmetrical tartans are more likely to look like tartan than asymmetric ones. 
![A symmetrical tartan DR/10 LP10 DN20 DN20 LN10 G10 DO/10](/assets/img/symmetrical_tartan.png)
![An asymmetric tartan DR10 LP10 DN20 DN20 LN10 G10 DO10](/assets/img/asymmetric_tartan.png)

* There are sequences of stripe widths that look more tartany than others:
![A tartan with a thin stripe DG32 T4 DB22 G10 LN10](/assets/img/different_width_tartan.png)
![A tartan with similar stripes DG20 T22 DB22 G20 LN18](/assets/img/similar_width_tartan.png)

* High contrast (e.g. light colours on dark backgrounds) narrow stripes also seem more tartany
![A tartan with pale thin stripes DG/32 T4 DB22 G10 LN/2](/assets/img/contrast_tartan.png)
![A tartan with all dark stripes DG/32 D4 DB22 G10 DN/2 ](/assets/img/low_contrast_tartan.png)

However, all this looking at tartan may have messed with my judgement.
