+++
date = "2015-08-11T15:53:56-07:00"
title = "introducing gindoro, a clean, minimal hugo theme"

+++

gindoro means poplar in Japanese.

Gindoro is the theme running this site. It's a hugo theme, [found here (by the way, mouse over these links!)](https://github.com/cdipaolo/gindoro), and it does some cool things.

We don't have syntax highlighting (to improve render times) but you'll still probably like the curreny format (I hope!)

```go
// long lines will cause the block to be scrollable. This is preferable to wrapping lines because you wouldn't be able to tell when it happened!
func main() {
    // most people don't know
    // about the print() builtin
    print("hello, github!")
}
```

Math support:

<div>
    \[ \int{\sin^2{x}dx = \frac{1}{2}\int{\left(1 - \cos{\left(2x\right)}\right)dx}} \]
    \[ = \frac{1}{2}\left( x - \frac{1}{2}\sin{\left(2x\right)} \right) \]
</div>

That's cool. Here's some general markdown tools:

I | am | a | table of variable length
----|----|---|---
that|is|true|
but||you|have revealed yourself too early
notice|the|light|gray lines dividing content!

but what about blockquotes????

> that's a good question. They are simple, but so am I.
> they can also have multiple lines in the markdown and
> it will auto-group the text when markdown renders

_italics_ and **bold text** also work, as per usual!

The font being used is Roboto Slab. By the way, did you notice how fast this rendered? The entire theme is only 36K and math support is using KaTeX (faster than MathJax) being brought in from a CDN.

You must be wondering about images, right about now. This image is large (I didn't feel like cropping it and self-hosting) so it probably took a little time to load:

![good question](https://images.unsplash.com/photo-1433354359170-23a4ae7338c6?q=80&fm=jpg&s=ae9f141fc85e050f6d68d1654c554236)

pretty clean, right?!

- this is an
- unordered
  * list
  * it uses the square bullet to look cleaner

You need to divide lists or markdown won't know what you were trying to do!

1. This is an ordered list
2. it uses [Hiragana (Japanese syllabic characters)](https://en.wikipedia.org/wiki/Hiragana)
3. as it's numbering (staring with ah, eee, ooo, A, oh in english pronounciation)
4. (this is the 'ay' pronounciation)
  * nobody needs those stupid numbers anyways
  * and this is pretty sweet
5. I hope you agree!
