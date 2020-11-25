---
layout: post
title:  "Announcing quiver: a new commutative diagram editor for the web"
date:   2020-11-25 16:00:00
---
[![quiver]({{site.baseurl}}/assets/images/quiver.png "quiver: a modern commutative diagram
editor")](https://q.uiver.app)

I'm happy to announce the release of [**quiver**](https://q.uiver.app), a new
commutative diagram editor for the web. I've been working on **quiver** for the
past two years and, while I only now feel ready to share it more widely, it has
already become an essential tool in my own workflow. If you want to try it out
immediately, you can do so at [q.uiver.app](https://q.uiver.app); if you're
interested in how **quiver** came to be, or want a quick overview of its
features, continue reading.

[Commutative diagrams](https://en.wikipedia.org/wiki/Commutative_diagram) are a
diagrammatic tool in mathematics used to express complex relationships between
mathematical structures: in particular, commutative diagrams are used
prolifically, and to great effect, in the field of [category
theory](https://en.wikipedia.org/wiki/Category_theory). If you're not familiar
with commutative diagrams, I hope you'll nevertheless be able to appreciate the
diagrams purely from an aesthetic viewpoint (I imagine **quiver** works well
even as an editor for other sorts of diagrams too). I use commutative diagrams
frequently in my own work. However, unfortunately, typesetting complex diagrams
can be a time-intensive and tedious affair. Though there exist tools for dealing
with very simple commutative diagrams, any diagram involving [higher-dimensional
structure](https://en.wikipedia.org/wiki/Higher_category_theory) (intuitively,
diagrams having arrows between other arrows, rather than just between vertices),
such as [natural
transformations](https://en.wikipedia.org/wiki/Natural_transformation),
[adjunctions](https://en.wikipedia.org/wiki/Adjoint_functors),
[equivalences](https://en.wikipedia.org/wiki/Equivalence_of_categories), and so
on, requires manually writing elaborate LaTeX TikZ code. Large [pasting
diagrams](https://ncatlab.org/nlab/show/pasting+diagram) can take hours to
typeset perfectly. My desire was to be able to produce such diagrams on a
computer just as quickly as I could draw them by hand, so that my willingness to
spend hours struggling with LaTeX would no longer be a limiting factor[^1].

[^1]: Anyone who has used it will know that writing LaTeX, and spending hours struggling with it, are the same thing. My hope is that this will no longer be necessary *specifically when creating commutative diagrams* for LaTeX.

[**quiver**](https://q.uiver.app) is designed for exactly this purpose, making it
essentially painless to create large, complex commutative and pasting diagrams
for exporting to LaTeX, or to simply take a screenshot for sharing elsewhere.
I've spent a long time working on a slew of features I think are necessary for
an editor like this and so, while it would take too long to list all of the
features here, I'm going to share some of my favourites to give you an idea of
**quiver**'s capabilities.

### Creating and editing diagrams

Most diagrams can be drawn by clicking and dragging: dragging creates a new
arrow, with the endpoints being created automatically. (If you want a
disconnected object, you can double-click anywhere on the canvas.) Once a
diagram has been drawn, you can rearrange the objects by clicking and dragging
in the empty space around them; similarly, you can modify the source and target
of an arrow by dragging its endpoints.

<video src="{{site.baseurl}}/assets/videos/click-and-drag.mp4" class="small-video" autoplay muted loop alt="Creating a diagram by clicking and dragging"></video>

### Higher-dimensional diagrams

From its inception, a motivating goal for **quiver** has been to support the
design of higher-dimensional pasting diagrams with the same ease as that of
ordinary commutative diagrams. Concretely, this means that drawing higher cells
(arrows between arrows) is as easy as clicking and dragging from one arrow to
another.

For instance, here's a
[modification](https://ncatlab.org/nlab/show/modification) between natural
transformations. This took me less than 30 seconds to draw in **quiver**.

(You can click on any of the screenshots in this post to open
the diagram in [q.uiver.app](https://q.uiver.app).)

[![A modification between natural transformations]({{site.baseurl}}/assets/images/modification.png "A modification between natural transformations")](https://q.uiver.app?q=WzAsMixbMCwwLCJcXG1hdGhzY3IgQyJdLFsyLDAsIlxcbWF0aHNjciBEIl0sWzAsMSwiRiIsMCx7ImN1cnZlIjotNX1dLFswLDEsIkciLDIseyJjdXJ2ZSI6NX1dLFsyLDMsIlxcc2lnbWEiLDIseyJjdXJ2ZSI6MiwibGVuZ3RoIjo3MH1dLFsyLDMsIlxcdGF1IiwwLHsiY3VydmUiOi0yLCJsZW5ndGgiOjcwfV0sWzQsNSwiXFxHYW1tYSIsMCx7Imxlbmd0aCI6NzB9XV0=)

### Flexible grid, zooming, and panning

The grid cells in **quiver** aren't a fixed size: they will grow to fit their
content (just like in TikZ).

[![The flexible grid in action]({{site.baseurl}}/assets/images/flexible-grid.png "The flexible grid in action")](https://q.uiver.app?q=WzAsMyxbMCwwLCIoQSBcXG90aW1lcyBJKSBcXG90aW1lcyBCIl0sWzIsMCwiQSBcXG90aW1lcyAoSSBcXG90aW1lcyBCKSJdLFsxLDEsIkEgXFxvdGltZXMgQiJdLFswLDIsIlxccmhvX0EgXFxvdGltZXMgQiIsMl0sWzEsMiwiQSBcXG90aW1lcyBcXGxhbWJkYV9CIl0sWzAsMSwiXFxhbHBoYV97QSwgSSwgQn0iXV0=)

Along with the ability to zoom and pan the canvas,
this means that even large, complex diagrams[^2] are easily manipulated and
navigated.

[^2]: This diagram, and the one at the start of the post, are taken from Gurski's [Biequivalences in tricategories](https://arxiv.org/abs/1102.0979), which is a rich source of aesthetic pasting diagrams.

[![A large diagram]({{site.baseurl}}/assets/images/large-diagram.png "A large diagram")](https://q.uiver.app?q=WzAsMTYsWzAsNSwiZmciXSxbMSw0LCIoZkkpZyJdLFsxLDYsImYoSWcpIl0sWzIsMywiKGYoZ2YpKWciXSxbMiw3LCJmKChnZilnKSJdLFszLDIsIigoZmcpZilnIl0sWzQsMSwiKElmKWciXSxbNSwwLCJmZyJdLFszLDgsImYoZyhmZykpIl0sWzQsOSwiZihnSSkiXSxbNSwxMCwiZmciXSxbNCw1LCIoZmcpKGZnKSJdLFs2LDIsIkkoZmcpIl0sWzYsOCwiKGZnKUkiXSxbNyw1LCJJSSJdLFs5LDUsIkkiXSxbMCwxLCJyXlxcYnVsbGV0MSIsMV0sWzEsMiwiYSJdLFswLDIsIjFsXlxcYnVsbGV0IiwxXSxbMSwzLCIoMVxcYmV0YSkxIiwyXSxbMyw0LCJhIiwwLHsiY3VydmUiOi0xfV0sWzIsNCwiMShcXGJldGExKSJdLFszLDUsImFeXFxidWxsZXQxIiwyXSxbNSw2LCIoXFxhbHBoYTEpMSIsMl0sWzYsNywibDEiLDJdLFs0LDgsIjFhIl0sWzgsOSwiMSgxYSkiXSxbOSwxMCwiMXIiXSxbNSwxMSwiYSIsMCx7ImN1cnZlIjotMn1dLFs4LDExLCJhXlxcYnVsbGV0IiwyLHsiY3VydmUiOjJ9XSxbNiwxMiwiYSIsMix7ImN1cnZlIjoyfV0sWzcsMTIsImxeXFxidWxsZXQiXSxbMTEsMTIsIlxcYWxwaGExIiwyLHsiY3VydmUiOjJ9XSxbMTAsMTMsInJeXFxidWxsZXQiLDJdLFs5LDEzLCJcXGFscGhhXlxcYnVsbGV0IiwwLHsiY3VydmUiOi0yfV0sWzExLDEzLCIxYSIsMCx7ImN1cnZlIjotMn1dLFsxMiwxNCwiMVxcYWxwaGEiLDAseyJjdXJ2ZSI6LTF9XSxbMTMsMTQsIlxcYWxwaGExIiwyLHsiY3VydmUiOjF9XSxbMTQsMTUsImwiLDAseyJjdXJ2ZSI6LTN9XSxbMTQsMTUsInIiLDIseyJjdXJ2ZSI6M31dLFs2LDExLCJcXGNvbmciLDEseyJjdXJ2ZSI6LTIsInN0eWxlIjp7ImJvZHkiOnsibmFtZSI6Im5vbmUifSwiaGVhZCI6eyJuYW1lIjoibm9uZSJ9fX1dLFs5LDExLCJcXGNvbmciLDEseyJjdXJ2ZSI6Miwic3R5bGUiOnsiYm9keSI6eyJuYW1lIjoibm9uZSJ9LCJoZWFkIjp7Im5hbWUiOiJub25lIn19fV0sWzEzLDEyLCJcXGNvbmciLDEseyJjdXJ2ZSI6LTIsInN0eWxlIjp7ImJvZHkiOnsibmFtZSI6Im5vbmUifSwiaGVhZCI6eyJuYW1lIjoibm9uZSJ9fX1dLFswLDcsIjEiLDAseyJjdXJ2ZSI6LTV9XSxbMCwxMCwiMSIsMix7ImN1cnZlIjo1fV0sWzEwLDE1LCJcXGFscGhhIiwyLHsiY3VydmUiOjV9XSxbNywxNSwiXFxhbHBoYSIsMCx7ImN1cnZlIjotNX1dLFsxNiwxOCwiXFxtdSIsMCx7ImN1cnZlIjotMSwibGVuZ3RoIjo1MH1dLFsyMSwxOSwiXFxjb25nIiwxLHsiY3VydmUiOjEsImxldmVsIjoxLCJzdHlsZSI6eyJib2R5Ijp7Im5hbWUiOiJub25lIn0sImhlYWQiOnsibmFtZSI6Im5vbmUifX19XSxbMzgsMzksIlxcY29uZyIsMSx7Imxlbmd0aCI6NzAsInN0eWxlIjp7ImJvZHkiOnsibmFtZSI6Im5vbmUifSwiaGVhZCI6eyJuYW1lIjoibm9uZSJ9fX1dLFsyMiwyNSwiXFxwaSIsMCx7ImN1cnZlIjotNCwibGVuZ3RoIjo0MH1dLFszNCwxMCwiXFxyaG8iLDEseyJjdXJ2ZSI6LTEsImxlbmd0aCI6NTB9XSxbNDMsMjIsIlxcUGhpXnstMX0xIiwyLHsibGVuZ3RoIjo1MH1dLFsyNSw0NCwiMVxcUHNpIiwyLHsibGVuZ3RoIjo1MH1dLFszNiw0NiwiXFxjb25nIiwxLHsibGVuZ3RoIjo3MCwic3R5bGUiOnsiYm9keSI6eyJuYW1lIjoibm9uZSJ9LCJoZWFkIjp7Im5hbWUiOiJub25lIn19fV0sWzcsMzAsIlxcbGFtYmRhIiwxLHsiY3VydmUiOi0xLCJsZW5ndGgiOjUwfV0sWzM3LDQ1LCJcXGNvbmciLDEseyJsZW5ndGgiOjcwLCJzdHlsZSI6eyJib2R5Ijp7Im5hbWUiOiJub25lIn0sImhlYWQiOnsibmFtZSI6Im5vbmUifX19XV0=)

### Composable arrow styles

**quiver** has a wide range of arrowhead, body, and tail styles, all of which
can be composed seamlessly. Here are just a few, to give you a taste of the
flexibility. Some of the combinations are so obscure that I can't imagine
anyone seriously using them[^3], but I think it's freeing to have the option.

[^3]: Or perhaps, now that the possibility presents itself, the floodgates will open: maybe 2021 will be the year of the hook-tailed squiggly harpoon.

[![A selection of arrow styles]({{site.baseurl}}/assets/images/arrow-styles.png "A selection of arrow styles")](https://q.uiver.app?q=WzAsMTYsWzAsMCwiXFxidWxsZXQiXSxbMSwwLCJcXGJ1bGxldCJdLFswLDEsIlxcYnVsbGV0Il0sWzEsMSwiXFxidWxsZXQiXSxbMCwyLCJcXGJ1bGxldCJdLFsxLDIsIlxcYnVsbGV0Il0sWzAsMywiXFxidWxsZXQiXSxbMSwzLCJcXGJ1bGxldCJdLFszLDAsIlxcYnVsbGV0Il0sWzIsMCwiXFxidWxsZXQiXSxbMywxLCJcXGJ1bGxldCJdLFsyLDEsIlxcYnVsbGV0Il0sWzMsMiwiXFxidWxsZXQiXSxbMiwyLCJcXGJ1bGxldCJdLFszLDMsIlxcYnVsbGV0Il0sWzIsMywiXFxidWxsZXQiXSxbMCwxLCIiLDAseyJzdHlsZSI6eyJ0YWlsIjp7Im5hbWUiOiJtb25vIn0sImJvZHkiOnsibmFtZSI6ImRhc2hlZCIsImxldmVsIjoxfX19XSxbMiwzLCIiLDAseyJzdHlsZSI6eyJ0YWlsIjp7Im5hbWUiOiJtYXBzIHRvIn0sImJvZHkiOnsibmFtZSI6ImRvdHRlZCIsImxldmVsIjoxfSwiaGVhZCI6eyJuYW1lIjoiZXBpIn19fV0sWzQsNSwiIiwwLHsic3R5bGUiOnsidGFpbCI6eyJuYW1lIjoiaG9vayIsInNpZGUiOiJ0b3AifSwiYm9keSI6eyJuYW1lIjoic3F1aWdnbHkiLCJsZXZlbCI6MX0sImhlYWQiOnsibmFtZSI6ImhhcnBvb24iLCJzaWRlIjoidG9wIn19fV0sWzYsNywiIiwwLHsic3R5bGUiOnsidGFpbCI6eyJuYW1lIjoiaG9vayIsInNpZGUiOiJib3R0b20ifSwiYm9keSI6eyJuYW1lIjoiYmFycmVkIiwibGV2ZWwiOjF9LCJoZWFkIjp7Im5hbWUiOiJoYXJwb29uIiwic2lkZSI6ImJvdHRvbSJ9fX1dLFs4LDksImYiLDIseyJvZmZzZXQiOjJ9XSxbOCw5LCJnIiwwLHsib2Zmc2V0IjotMn1dLFsxMCwxMSwiIiwwLHsibGV2ZWwiOjIsInN0eWxlIjp7ImJvZHkiOnsibmFtZSI6InNxdWlnZ2x5IiwibGV2ZWwiOjF9fX1dLFsxMiwxMywiIiwwLHsibGVuZ3RoIjoyMCwic3R5bGUiOnsiYm9keSI6eyJsZXZlbCI6MX19fV0sWzE0LDE1LCIiLDAseyJjdXJ2ZSI6MiwibGV2ZWwiOjMsInN0eWxlIjp7ImJvZHkiOnsibmFtZSI6ImJhcnJlZCIsImxldmVsIjoxfSwiaGVhZCI6eyJuYW1lIjoiZXBpIn19fV1d)

### Keyboard controls

Once you feel comfortable using the mouse to create diagrams, it probably won't
be long before you want to pick up some speed. You'll be in luck, because
everything can be controlled from the keyboard, which can be significantly
faster than the mouse with a little practice. If you enable hints from the
toolbar, the keyboard shortcuts for various actions will be highlighted
on-screen, helping you pick up the controls whilst you're learning.

[![Keyboard controls]({{site.baseurl}}/assets/images/keyboard-controls.png "Keyboard controls")](https://q.uiver.app?q=WzAsMixbMCwwLCJBIl0sWzEsMCwiQiJdLFswLDEsImYiXV0=)

[![Creating a diagram with the keyboard]({{site.baseurl}}/assets/images/keyboard-creation.png "Creating a diagram with the keyboard")](https://q.uiver.app?q=WzAsMixbMCwwLCJcXGJ1bGxldCJdLFsxLDAsIlxcYnVsbGV0Il0sWzAsMV1d)

Even if you feel more comfortable with the mouse, the cell queue can be a handy
time-saver. Simply draw out your diagram first (ignorning the labels and styles
initially), then then press Tab (⇥) to select each cell in turn, filling out the
details of each cell. This two-stage process: draw, then edit, feels very
natural and should take little time to pick up, even if you don't feel so at
ease with the keyboard.

[![Using the cell queue]({{site.baseurl}}/assets/images/cell-queue.png "Using the cell queue")](https://q.uiver.app?q=WzAsNCxbMCwwLCJcXGJ1bGxldCJdLFsxLDAsIlxcYnVsbGV0Il0sWzEsMSwiXFxidWxsZXQiXSxbMCwxLCJcXGJ1bGxldCJdLFswLDFdLFsxLDJdLFswLDNdLFszLDJdXQ==)

### Multiple selection

You can select multiple cells (both objects and arrows) to edit them
simultaneously &ndash; for example if a collection of arrows should be styled
identically &ndash; by holding Shift (⇧) when selecting.

[![Editing multiple cells simultaneously]({{site.baseurl}}/assets/images/multiple-selection.png "Editing multiple cells simultaneously")](https://q.uiver.app?q=WzAsNCxbMCwwLCJDIl0sWzAsMSwiQSJdLFsxLDAsIkIiXSxbMSwxLCJBICtfQyBCIl0sWzEsM10sWzIsMywiIiwwLHsic3R5bGUiOnsiaGVhZCI6eyJuYW1lIjoiZXBpIn19fV0sWzAsMSwiZiIsMix7InN0eWxlIjp7ImhlYWQiOnsibmFtZSI6ImVwaSJ9fX1dLFswLDIsImciXSxbMywwLCIiLDEseyJzdHlsZSI6eyJuYW1lIjoiY29ybmVyIn19XV0=)

### Macro support

If you're using **quiver** to design diagrams for LaTeX, you'll likely have a
host of macro declarations defining various names and symbols. By default,
these will be rendered as invalid commands, but you can paste the URL of a text
file containing your `\newcommand` declarations to have them rendered correctly
in the editor.

Without macros:

[![Diagram without macros]({{site.baseurl}}/assets/images/macros-before.png "Diagram without macros")](https://q.uiver.app?q=WzAsMyxbMCwxLCJcXHNtY2F0IEMiXSxbMCwwLCJcXHBzaHtcXHNtY2F0IEN9Il0sWzEsMCwiXFxjYXQgRCJdLFswLDIsIkYiLDJdLFswLDEsIlxceW9fe1xcc21jYXQgQ30iXSxbMSwyLCJcXHdpZGV0aWxkZSBGIl0sWzMsMSwiIiwyLHsibGVuZ3RoIjo3MH1dXQ==)

With macros ([example macro file](https://gist.github.com/varkor/85193e9d74be60377b4c295755bcb290)):

[![Diagram with macros]({{site.baseurl}}/assets/images/macros-after.png "Diagram with macros")](https://q.uiver.app?q=WzAsMyxbMCwxLCJcXHNtY2F0IEMiXSxbMCwwLCJcXHBzaHtcXHNtY2F0IEN9Il0sWzEsMCwiXFxjYXQgRCJdLFswLDIsIkYiLDJdLFswLDEsIlxceW9fe1xcc21jYXQgQ30iXSxbMSwyLCJcXHdpZGV0aWxkZSBGIl0sWzMsMSwiIiwyLHsibGVuZ3RoIjo3MH1dXQ==&macro_url=https%3A%2F%2Fgist.githubusercontent.com%2Fvarkor%2F85193e9d74be60377b4c295755bcb290%2Fraw%2F07384e833e541706143f73b9192c27c81a7a3581%2Ffree-cocompletion-macros.tex)

### Exporting to LaTeX

While **quiver** is intended to render beautiful diagrams in-editor for viewing
and taking screenshots, it is important to be able to use the diagrams
seamlessly in LaTeX. This is facilitated by the accompanying LaTeX package
[`quiver.sty`](https://q.uiver.app/quiver.sty)[^4], which includes all the
dependencies for the exported diagrams. The generated LaTeX code is prefixed by
a [q.uiver.app](https://q.uiver.app) link to the diagram, which allows you to
easily tweak it in the future when you want to make changes.

[^4]: For now, the package is available directly from the editor, though in the future, when the package has stabilised, I will probably make it available through [CTAN](https://ctan.org/).

[![Exporting to LaTeX]({{site.baseurl}}/assets/images/export.png "Exporting to LaTeX")](https://q.uiver.app?q=WzAsOCxbMCwwLCJHeCJdLFsxLDAsIkZ4Il0sWzIsMCwiR3giXSxbMywwLCJGeCJdLFswLDEsIkd5Il0sWzEsMSwiRnkiXSxbMiwxLCJHeSJdLFszLDEsIkZ5Il0sWzAsMSwiXFxiZXRhX3giXSxbMSwyLCJcXGFscGhhX3giXSxbMiwzLCJcXGJldGFfeCJdLFswLDQsIkdmIiwyXSxbNCw1LCJcXGJldGFfeSIsMl0sWzUsNiwiXFxhbHBoYV95IiwyXSxbNiw3LCJcXGJldGFfeSIsMl0sWzMsNywiRmYiXSxbMSw1LCJGZiIsMV0sWzIsNiwiR2YiLDFdLFs0LDEsIlxcYmV0YV9mIiwwLHsibGVuZ3RoIjo1MCwibGV2ZWwiOjJ9XSxbNSwyLCJcXGFscGhhX2YiLDAseyJsZW5ndGgiOjUwLCJsZXZlbCI6Mn1dLFs2LDMsIlxcYmV0YSdmIiwwLHsibGVuZ3RoIjo1MCwibGV2ZWwiOjJ9XSxbMCwyLCIxIiwwLHsiY3VydmUiOi00fV0sWzUsNywiMSIsMix7ImN1cnZlIjo0fV0sWzEsMjEsIlxcdmFyZXBzaWxvbl94IiwwLHsibGVuZ3RoIjo1MH1dLFsyMiw2LCJcXGV0YV95IiwyLHsibGVuZ3RoIjo1MH1dXQ==)

## Implementation and bug reports

**quiver** has had a long gestation period in part because I'm developing it in
my free time (along with [other
side-projects](https://github.com/rust-lang/rust/issues/44580)), but also
because rendering aesthetic commutative diagrams is hard. I'm not aware of
another editor that has the same degree of composability and flexibility as
**quiver**: in fact, it can even be difficult to render some of the more exotic
combinations in TikZ directly. I think the implementation techniques for
aesthetic arrow rendering are interesting, and I may make a blog post in the
future going into more detail.

**quiver** is open source: you can find the [repository on
GitHub](https://github.com/varkor/quiver). This is also the place to [report any
bugs or feature requests](https://github.com/varkor/quiver/issues) you might
have. I'd like to make **quiver** the ideal tool for designing commutative
diagrams, so if you have a suggestion for how to make it better, I'd love to
hear about it. So far, **quiver** has had a relatively small userbase, and I
wouldn't be surprised to find I've overlooked a few rough edges.

## A final request

As I previously mentioned, **quiver** can be a little *too* flexible when it
comes to generating appropriate TikZ to replicate the diagrams in LaTeX. I am
not a TikZ expert, and so there are a few places where the LaTeX export feature
struggles, because I haven't been able to figure out how to produce satisfactory
results in TikZ: drawing curved triple arrows and shortened curved arrows are a
couple of examples. If you're experienced with TikZ, and would like to help make
this tool even better, I'd love to hear from you: you can contact me by opening
an issue [on GitHub](https://github.com/varkor/quiver/issues), or by sending me
a message [on Twitter](https://twitter.com/varkora).

I want **quiver** to be helpful for as many people as possible, so I'd be very
grateful if you shared it with anyone else who might find it useful!

---

Try **quiver** out: [q.uiver.app](https://q.uiver.app)

Follow **quiver** on Twitter for updates: [@q_uiver_app](https://twitter.com/q_uiver_app)

---