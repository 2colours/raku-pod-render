# Rendering Pod Distribution
>A generic distribution to render Pod in a file (program or module) or in a cache (eg. the Raku documentation collection)


----
## Table of Contents
[Creating a Renderer](#creating-a-renderer)
[Pod Source Configuration](#pod-source-configuration)
[Templates](#templates)
[String Template](#string-template)
[Block Templates](#block-templates)
[Partials and New Templates](#partials-and-new-templates)
[Change the Templating Engine](#change-the-templating-engine)
[Custom Pod and Template](#custom-pod-and-template)
[Para](#para)
[Rendering Strategy](#rendering-strategy)
[Rendering many Pod Sources](#rendering-many-pod-sources)
[Methods Provided by ProcessedPod](#methods-provided-by-processedpod)
[modify-templates](#modify-templates)
[rewrite-target](#rewrite-target)
[process-pod](#process-pod)
[render-block](#render-block)
[render-tree](#render-tree)
[delete-pod-structure](#delete-pod-structure)
[file-wrap](#file-wrap)
[source-wrap](#source-wrap)
[Individual component renderers](#individual-component-renderers)
[Public Class Attributes](#public-class-attributes)
[Minimum Template Set](#minimum-template-set)

----
This distribution ('distribution' because it contains several modules and other resources) provides a generic class `ProcessedPod`, which accepts templates, and renders one or more Pod trees. The class collects the information in the Pod files to create page components, such as _Table of Contents_, _Glossary_, _Metadata_ (eg. Author, version, etc), and _Footnotes_.

The output depends entirely on the templates. The body of the text, TOC, Glossary, and Footnotes can be output or suppressed, and their position can be controlled using a combination of templates, or in the case of HTML, templates and CSS. It also means that the same generic class can be used for HTML and MarkDown.

Two other modules are provided: `Pod::To::HTML` and `Pod::To::MarkDown`. For more information on them, see [Pod::To::HTML](Pod2HTML.md). These have the functionality and default templates to be used in conjunction with the **raku** (aka perl6) compiler option `--doc=name`.

The aim of ProcessedPod is to allow for a more flexible mechanism for rendering POD. For example, when multiple POD6 files are combined each individual source generates TOC, Glossary, Footnotes, and Metadata information. There is no single way these can be combined, and different uses will need a different approach. For example, a flat-file HTML page will need separate pages for each source, with a landing page and global TOC and glossary pages to link them all.

The `Pod::To::HTML` has a simple way of handling customised CSS, but no way to access embedded images other than svg files. Modifying the templates, when there is information about the serving environment, can change this.

This module uses the Moustache templating system at `Template::Mustache`. More later on how to change this.

# Creating a Renderer
The first step in rendering is to create a renderer.

The renderer needs to take into account the output format, eg., html, incorporate non-default templates (eg., a designer may want to have customised classes in paragraphs or headers). The Pod renderer requires templates for a number of document elements, see TEMPLATES below.

Essentially, a hash of element keys pointing to Mustache strings is provided to the renderer. The `Pod::To::HTML` and `Pod::To::MarkDown` modules in this distribution provide default templates to the `ProcessedPod` class.

The renderer can be customised on-the-fly by modifying the keys of the template hash. For example,

```
    use RenderPod;
    my $renderer .= RenderPod.new;
    $renderer.modify-templates( %(format-b =>
        '<strong class="myStrongClass {{# addClass }}{{ addClass }}{{/ addClass }}">{{{ contents }}}</strong>')
    );
    # The default template is something like
    #       'format-b' => '<strong{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</strong>'
    # the effect of this change is to add myStrongClass to all instances of B<> including any extra classes added by the POD

```
This would be wanted if a different rendering of bold is needed in some source file, or a page component. Bear in mind that for HTML, it is probably better to add another css class to a specific paragraph (for example) using the Pod config metadata. This is picked up by ProcessedPod (as can be seen in the above example where `addClass` is used to add an extra class to the `<strong> ` container.

# Pod Source Configuration
Most Pod source files will begin with `=begin pod`. Any line with `=begin xxx` may have configuration data, eg.

```
    =begin pod :kind<Language> :subkind<Language> :class<Major text>
    ...

```
The first `=begin pod` to be rendered after initiation, or after the `.delete-pod-structure` method, will have its configuration data transferred to the `ProcessedPod` object's `%.pod-config-data` attribute.

When multiple files are rendered, any meta data associated with the pod can be accessed during the Rendering Iteration.

# Templates
The nature of the templates and their interpretation depends on the `rendition` method, which must be over-ridden for a different templating engine.

When a ProcessPod instance is instantiated, a templating object xxxx can be passed via the `:template` parameter, eg.

```
my $p = ProcessedPod.new;
$p.templates(:templates( xxxx ) );

```
If the object is a Hash, then it is considered a Hash of the templates, and verified for completeness.

If the object is a String, then it is considered a `path/filename` to a file containing a raku program that evaluates to a Hash, which is then verified as a Hash of templates.

Each entry in the template Hash can be either (a) a String or (b) a block.

## String Template
For example if `'escaped' =` '{{ contents }}', > is a line in a hash declaration of the templates, then the right hand side is the `Mustache` template for the `escaped` key. The template engine is called with a hash containing String data that are interpolated into the template. `contents` is provided for all keys, but some keys have more complex data.

## Block Templates
`Mustache` by design is not intended to have any logic, although it does allow lambdas. Since the latter are not well documented and some template-specific preprocessing is required, or the default action of the Templating engine needs to be over-ridden, extra functionality is provided.

Instead of a plain text template being associated with a Template Hash key, the key can be associated with a block that can pre-process the data provided to the Mustache engine, or change the template. The block must return a String with a valid Mustache template.

For example,

```
'escaped' => -> %params { %params<contents>.subst-mutate(/\'/, '&39;', :g ); '{{ contents }}' }


```
The block is called with a Hash parameter that is assigned to `%params`. The `contents` key of `%params`) is adjusted because `Mustache` does not escape single-quotes.

## Partials and New Templates
Mustache allows for other templates to be used as partials. Thus it is possible to create new templates that use the templates needed by ProcessedPod and incorporate them in output templates.

For example:

```
$p.modify-templates( %(:newone(
    '<container>{{ contents }}</container>'
    ),
    :format-b('{{> newone }}'))
);


```
Now the pod line `This is some B<boldish text> in a line` will result in

```
<p>This is some <container>boldish text</container> in a line</p>
```
# Change the Templating Engine
In order to change the Templating Engine, a Templater Role needs to be created using the MustacheTemplater role in this distribution as a model. Then a new class similar to ProcessedPod can be created as

```
class NewProcessedPod is GenericPod does myNewTemplater {}
```
The new role may only need to over-ride `method rendition( Str $key, Hash %params --` Str )>.

Assuming that the templating engine is NewTemplateEngine, and that - like Template::Mustache - it is instantiates with `.new`, and has a `.render` method which takes a String template, and Hash of strings to interpolate, and which returns a String, viz `.render( Str $string, Hash %params, :from( %hash-of-templates) --` Str )>.

# Custom Pod and Template
Standard Pod allows for Pod::Blocks to be named and configuration data provided. This allows us to leverage the standard syntax to allow for non-standard blocks and templates.

If a class needs to be added to Pod Block, say a specific paragraph, then the following can be put in a pod file

# para

Paragraph texts

The 'para' template needs to be changed (either on the fly or at instantiation)

para => '<p{{# class }} class="{{ class }}">{{{ contents }}</p>'

A completely new block can be created. For example, the HTML module adds the `Image` custom block by default, and provides the `image` template. (In keeping with other named blocks, _Title_ case may be conventionally used for the block name but _Lower_ case is required for the template. Note the _Upper_ case (all letters) is reserved for descriptors that are added (in HTML) as meta data.

Suppose we wish to have a diagram block with a source and to assign classes to it. We want the HTML container to be `figure`.

In the pod source code, we would have:

```
    =for diagram :src<https://someplace.nice/fabulous.png> :class<float left>
    This is the caption.

```
Note that the `for` takes configuration parameters to be fed to the template, and ends at the first blank line or next `pod` instruction.

Then in the rendering program we need to provide to ProcessedPod the new object name, and the corresponding template. These must be the same name. Thus we would have:

```
    use Pod::To::HTML;
    my Pod::To::HTML $r .= new;
    $r.custom = <diagram>;
    $r.modify-templates( %( diagram => '<figure source="{{ src }}" class="{{ class }}">{{ contents }}</figure>' , ) );

```
# Rendering Strategy
A rendering strategy is required for a complex task consisting of many Pod sources. A rendering strategy has to consider:

*  The pod contained in a single file may be provided as one or more trees of Pod blocks. A pod tree may contain blocks to be referred to in a Table of Contents (TOC), and also it may contain anchors to which other documentation may point. This means that the pod in each separate file will automatically produces its own TOC (a list of headers in the order in which they appear in the pod tree(s), and its own Glossary (a list of terms that are encountered, perhaps multiple times for each term, within the text).

*  When only a single file source is used (such as when a Pod::To::name is called by the compiler), then the content, TOC and glossary have to be rendered together.

*  Multiple pod files will by definition exist in a collection that should be rendered together in a consistent manner. The content from a single source file will be called a **Component**. This will be handled in another module raku-render-collection There have to be the following facilities

*  A strategy to create one or more TOC's for the whole collection that collect and combine all the **Component** TOC's. The intent is to allow for TOCs that are designed and do not follow the alphabetical name of the **Component** source, together with a default alphabetical list.

*  A strategy to create one or more Glossary(ies) from all the **Component** glossaries

# Rendering many Pod Sources
A complete render strategy has to deal with multiple page components.

The following sketches the use of the `Pod::To::HTML` class.

For example, suppose we want to render each POD source file as a separate html file, and combine the global page components separately.

The the `ProcessedPob` object expects a compiled Pod object. One way to do this is use the `Pod::From::Cache` module.

```
    use Pod::To::HTML;
    my $p = Pod::To::HTML.new;
    my %pod-input; # key is the path-name for the output file, value is a Pod::Block
    my @processed;

    #
    # ... populate @pod-input, eg from a document cache, or use EVALFILE on each source
    #

    my $counter = 1; # a counter to illustrate how to change output file name

    for %pod-input.kv -> $nm, $pd {
        with $p { # topicalises the processor
            .name = $nm;
            # change templates on a per file basis before calling process-pod,
            # to be used with care because it forces a recache of the templates, which is slow
            # also, the templates probably need to be returned to normal (not shown here), again requiring a recache
            if $nm ~~ /^ 'Class' /
            {
                .replace-template( %(
                    format-b => '<strong class="myStrongClass {{# addClass }}{{ addClass }}{{/ addClass }}">{{{ contents }}}</strong>'
                    # was 'format-b' => '<strong{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</strong>'
                    # the effect of this change is to add myStrongClass to all instances of B<> including any extra classes added by the POD
                ))
            }
            .process-pod( $pd );
            # change output name from $nm if necessary
            .file-wrap( "{ $counter++ }_$nm" );
            # get the pod structure, delete the information to continue with a new pod tree, retain the cached templates
            @processed.append: $p.delete-pod-structure; # beware, pod-structure also deletes body, toc, glossary, etc contents
        }
    }
    # each instance of @processed will have TOC, Glossary and Footnote arrays that can be combined in some way
    for @processed {
        # each file has been written, but now process the collection page component data and write the files for all the collection
    }
    # code to write global TOC and Glossary html files.

```
# Methods Provided by ProcessedPod
## modify-templates
```
method modify-templates( %new-templates )
```
Allows for templates to be modified or new ones added before or during pod processing.

**Note:** Since the templating engine needs to be reinitialised in order to clear a template cache, it is probably not efficient to modify templates too many times during processing.

`modify-templates` **replaces** the keys in the `%new-templates` hash. It will add new keys to the internal template store.

Example:

```
use PodRender;
my PodRender $p .= new;
$p.templates( 'path/to/newtemplates.raku');
$p.replace-template( %(
                    format-b => '<strong class="myStrongClass {{# addClass }}{{ addClass }}{{/ addClass }}">{{{ contents }}}</strong>'
                    # was 'format-b' => '<strong{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</strong>'
                    # the effect of this change is to add myStrongClass to all instances of B<> including any extra classes added by the POD
                ))

```
## rewrite-target
This method may need to be over-ridden, eg., for MarkDown which uses a different targetting function.

```
method rewrite-target(Str $candidate-name is copy, :$unique --> Str )
```
Rewrites targets (link destinations) to be made unique and to be cannonised depending on the output format. Takes the candidate name and whether it should be unique, returns with the cannonised link name Target names inside the POD file, eg., headers, glossary, footnotes The function is called to cannonise the target name and to ensure - if necessary - that the target name used in the link is unique. The following method uses an algorithm designed to pass the legacy `Pod::To::HTML` tests.

When indexing a unique target is needed even when same entry is repeated When a Heading is a target, the reference must come from the name

```
method rewrite-target(Str $candidate-name is copy, :$unique --> Str) {
        return $!default-top if $candidate-name eq $!default-top;
        # don't rewrite the top

        $candidate-name = $candidate-name.subst(/\s+/, '_', :g);
        if $unique {
            $candidate-name ~= '_0' if $candidate-name (<) $!targets;
            ++$candidate-name while $!targets{$candidate-name};
            # will continue to loop until a unique name is found
        }
        $!targets{$candidate-name}++;
        # now add to targets, no effect if not unique
        $candidate-name
    }

```
## process-pod
```
method process-pod( $pod ) {
```
Process the pod block or tree passed to it, and concatenates it to previous pod tree. Returns a string representation of the tree in the required format

## render-block
```
method render-block( $pod )
```
Renders a pod tree, but probably a block Returns only the pod that was passed

## render-tree
```
method render-tree( $pod )
```
Tenders the whole pod tree Is actually an alias to process-pod

## delete-pod-structure
```
method delete-pod-structure
```
Deletes any previously processed pod, keeping the template engine cache Returns the pod-structure deleted as a Hash, for storage in an array when multiple files are processed.

## file-wrap
```
method file-wrap(:$filename = $.name, :$ext = 'html' )
```
Saves the rendered pod tree as a file, and its document structures, uses source wrap Filename defaults to the name of the pod tree, and ext defaults to html

## source-wrap
```
method source-wrap( --> Str )
```
Renders all of the document structures, and wraps them and the body Uses the source-wrap template

## Individual component renderers
The following are called by `source-wrap` but could be called separately, eg., if a different textual template such as `format-b` should be used inside the component.

*  method render-toc( --> Str )

*  method render-glossary(-->Str)

*  method render-footnotes(--> Str)

*  method render-meta(--> Str)

# Public Class Attributes
```
# provided at instantiation or by attributes on Class instance
has $.front-matter is rw = 'preface'; # Text between =TITLE and first header, this is used to refer for textual placenames
has Str $.name is rw;
has Str $.title is rw = $!name;
has Str $.subtitle is rw = '';
has Str $.path is rw; # should be path of original document, defaults to $.name
has Str $.top is rw = $!default-top; # defaults to top, then becomes target for TITLE
has &.highlighter is rw; # a callable (eg. provided by external program) that converts [html] to highlighted raku code

# document level information
has $.lang is rw = 'en'; # language of pod file

# Output rendering information
has Bool $.no-meta is rw = False; # set to True eliminates meta data rendering
has Bool $.no-footnotes is rw = False; # set to True eliminates rendering of footnotes
has Bool $.no-toc is rw = False; # set to True to exclude TOC even if there are headers
has Bool $.no-glossary is rw = False; # set to True to exclude Glossary even if there are internal anchors

# debugging
has Bool $.debug is rw; # outputs to STDERR information on processing
has Bool $.verbose is rw; # outputs to STDERR more detail about errors.

# Structure to collect links, eg. to test whether they all work
```
# Minimum Template Set
There is a minimum set of templates that must be provided for a Pod file to be rendered. These are:

```
    < raw comment escaped glossary footnotes footer
                format-c block-code format-u para format-b named source-wrap defn dlist-start dlist-end
                output format-l format-x heading title format-n format-i format-k format-p meta
                list subtitle format-r format-t table item notimplemented section toc pod >

```
When the `.templates` method is called, the templates will be checked against this list for completeness. An Exception will be thrown if all the templates are not provided. Extra templates can be included. `Pod::To::HTML` uses this to have partial templates that use the required templates.

The following is the method used to generate the files for Pod::To::HTML (at some point in the development cycle).

The variables not explicitly declared in the subroutine are `our` scoped to give the default texts.

```
method html-templates( :$css-text = $default-css-text ) {
        %(
        # the following are extra for HTML files and are needed by the render (class) method
        # in the source-wrap template.
            'camelia-img' => $camelia-svg,
            'css-text' => $css-text,
            # note that verbatim V<> does not have its own format because it affects what is inside it (see POD documentation)
            :escaped('{{ contents }}'),
            :raw('{{{ contents }}}'),
            'block-code' => q:to/TEMPL/,
                <pre class="pod-block-code{{# addClass }} {{ addClass }}{{/ addClass}}">
                {{# contents }}{{{ contents }}}{{/ contents }}</pre>
                TEMPL
            'comment' => '<!-- {{{ contents }}} -->',
            'dlist-start' => "<dl>\n",
            'defn' => '<dt>{{ term }}</dt><dd>{{{ contents }}}</dd>',
            'dlist-end' => "\n</dl>",
            'format-b' => '<strong{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</strong>',
            'format-c' => '<code{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</code>
            ',
            'format-i' => '<em{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</em>',
            'format-k' => '<kbd{{# addClass }}class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</kbd>
            ',
            'format-l' => '<a href="{{ target }}"{{# addClass }} class="{{ addClass }}"{{/ addClass}}>{{{ contents }}}</a>
            ',
            'format-n' => '<sup><a name="{{ retTarget }}" href="#{{ fnTarget }}">[{{ fnNumber }}]</a></sup>
            ',
            'format-p' => -> %params {
                %params<contents> = %params<contents>.=trans(['<pre>', '</pre>'] => ['&lt;pre&gt;', '&lt;/pre&gt;']);
                '<div{{# addClass }} class="{{ addClass }}"{{/ addClass }}><pre>{{{ contents }}}</pre></div>'
            },
            'format-r' => '<var{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</var>',
            'format-t' => '<samp{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</samp>',
            'format-u' => '<u{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</u>',
            'format-x' => '<a name="{{ target }}"></a>{{# text }}<span class="glossary-entry{{# addClass }} {{ addClass }}{{/ addClass }}">{{{ text }}}</span>{{/ text }} ',
            'heading' => '<h{{# level }}{{ level }}{{/ level }} id="{{ target }}"><a href="#{{ top }}" class="u" title="go to top of document">{{{ text }}}</a></h{{# level }}{{ level }}{{/ level }}>
            ',
            'item' => '<li{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</li>
            ',
            'list' => q:to/TEMPL/,
                <ul>
                    {{# items }}{{{ . }}}{{/ items}}
                </ul>
                TEMPL
            'named' => q:to/TEMPL/,
                <section>
                    <h{{# level }}{{ level }}{{/ level }} id="{{ target }}"><a href="#{{ top }}" class="u" title="go to top of document">{{{ name }}}</a></h{{# level }}{{ level }}{{/ level }}>
                    {{{ contents }}}
                </section>
                TEMPL
            'notimplemented' => '<span class="pod-block-notimplemented">{{{ contents }}}</span>',
            'output' => '<pre class="pod-output">{{{ contents }}}</pre>',
            'para' => '<p{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}</p>',
            'pod' => '<section name="{{ name }}"{{# addClass }} class="{{ addClass }}"{{/ addClass }}>{{{ contents }}}{{{ tail }}}
                </section>',
            'section' => q:to/TEMPL/,
                <section name="{{ name }}">{{{ contents }}}{{{ tail }}}
                </section>
                TEMPL
            'subtitle' => '<div class="subtitle{{# addClass }} {{ addClass }}{{/ addClass }}">{{{ contents }}}</div>',
            'table' => q:to/TEMPL/,
                <table class="pod-table{{# addClass }} {{ addClass }}{{/ addClass }}">
                    {{# caption }}<caption>{{{ caption }}}</caption>{{/ caption }}
                    {{# headers }}<thead>
                        <tr>{{# cells }}<th>{{{ . }}}</th>{{/ cells }}</tr>
                    </thead>{{/ headers }}
                    <tbody>
                        {{# rows }}<tr>{{# cells }}<td>{{{ . }}}</td>{{/ cells }}</tr>{{/ rows }}
                    </tbody>
                </table>
                TEMPL
            'title' => '<h1 class="title{{# addClass }} {{ addClass }}{{/ addClass }}" id="{{ target }}">{{{ text }}}</h1>',
            # templates used by output methods, eg., source-wrap, file-wrap, etc
            'source-wrap' => q:to/TEMPL/,
                <!doctype html>
                <html lang="{{ lang }}">
                    {{> head-block }}
                    <body class="pod">
                        {{> header }}
                        <div class="toc-glossary">
                        {{# toc }}{{{ toc }}}{{/ toc }}
                        {{# glossary }}{{{ glossary }}}{{/ glossary }}
                        </div>
                        <div class="pod-body{{^ toc }} no-toc{{/ toc }}">
                            {{{ body }}}
                        </div>
                        {{# footnotes }}{{{ footnotes }}}{{/ footnotes }}
                        {{> footer }}
                    </body>
                </html>
                TEMPL
            'footnotes' => q:to/TEMPL/,
                <div class="footnotes">
                    <ol>{{# notes }}
                        <li id="{{ fnTarget }}">{{{ text }}}<a class="footnote" href="#{{ retTarget }}"> « Back »</a></li>
                        {{/ notes }}
                    </ol>
                </div>
                TEMPL
            'glossary' => q:to/TEMPL/,
                 <table id="Glossary">
                    <caption>Glossary</caption>
                    <tr><th>Term</th><th>Section Location</th></tr>
                    {{# glossary }}
                    <tr class="glossary-defn-row">
                        <td class="glossary-defn">{{{ text }}}</td><td></td></tr>
                        {{# refs }}<tr class="glossary-place-row"><td></td><td class="glossary-place"><a href="#{{ target }}">{{{ place }}}</a></td></tr>{{/ refs }}
                    {{/ glossary }}
                </table>
                TEMPL
            'meta' => q:to/TEMPL/,
                {{# meta }}
                    <meta name="{{ name }}" value="{{ value }}" />
                {{/ meta }}
                TEMPL
            'toc' => q:to/TEMPL/,
                <table id="TOC">
                    <caption>Table of Contents</caption>
                    {{# toc }}
                    <tr class="toc-level-{{ level }}">
                        <td class="toc-text"><a href="#{{ target }}">{{# counter }}<span class="toc-counter">{{ counter }}</span>{{/ counter }} {{ text }}</a></td>
                    </tr>
                    {{/ toc }}
                </table>
                TEMPL
            'head-block' => q:to/TEMPL/,
                <head>
                    <title>{{ title }}</title>
                    <meta charset="UTF-8" />
                    {{# metadata }}{{{ metadata }}}{{/ metadata }}
                    {{# css }}<link rel="stylesheet" href="{{ css }}">{{/ css }}
                    {{^ css }}{{> css-text }}{{/ css }}
                    {{# head }}{{{ head }}}{{/ head }}
                </head>
                TEMPL
            'head' => '',
            'header' => '<header>{{> camelia-img }}{{ title }}</header>',
            'footer' => '<footer><div>Rendered from <span class="path">{{ path }}{{^ path }}Unknown{{/ path}}</span></div>
                <div>at <span class="time">{{ renderedtime }}{{^ renderedtime }}a moment before time began!?{{/ renderedtime }}</span></div>
                </footer>',
        )
}

```







----
Rendered from RenderPod.pod6 at 2020-07-31T22:16:11Z