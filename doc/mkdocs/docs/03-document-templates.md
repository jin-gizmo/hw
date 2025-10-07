# Document Templates

The source directory for a document template should be created using the docma
CLI:

```bash
docma new <DIRECTORY>
```

The resulting directory is structured thus:

```bare
<DIRECTORY>/
├── config.yaml    ... Mandatory
├── charts/        ... Specification files for charts
├── content/       ... Document content (HTML, PDF, Markdown etc.)
├── data/          ... Data files (e.g CSV / JSONL files)
├── fonts/         ... Font files (e.g. .ttf files)
├── overlays/      ... Overlay content files (Typically HTML or PDF)
├── queries/       ... Query specifications used for charts
└── resources/     ... HTML resources (image files etc.)
```

Only the `config.yaml` file is mandated. While the other components can be
present, or not, as required, and directory structure is arbitrary, it is
**strongly** recommended to adhere to the layout shown above.

> Files and directories in the template source directory matching `.*` are not
> copied into the compiled template.



## Template Configuration File

The document template configuration file, `config.yaml`, is critical to the
setup and operation of a **docma** template. The structure of the configuration
file is validated during template compilation. It contains the following
elements.

| Key            | Type           | Required | Description                                                  |
| -------------- | -------------- | -------- | ------------------------------------------------------------ |
| id             | string         | Yes      | Template identifier. This must be at least 3 chars long, start with an alpha, end with an alphanumeric and contain only alpha-numerics and `+-_=` characters. |
| description    | string         | Yes      | Template description.                                        |
| owner          | string         | Yes      | Template owner.                                              |
| version        | string         | Yes      | Template version. Must be in the form major.minor.patch (e.g. `1.0.0`). |
| documents      | list           | Yes      | A list of [document references](#document-references) to be included in the specified order in the output document. |
| overlays       | object         | No       | Document overlay specifications to enable watermarking / stamping of the output document (PDF only). See [Watermarking](05-docma-template-rendering.md#watermarking) below. |
| imports        | list           | No       | A list of specifications for external files to include during compilation. See [Document Imports](#document-imports) below. |
| parameters     | object         | No       | Contains optional keys `defaults` and `schema`.              |
| -> defaults    | object         | No       | Default values for rendering parameters.                     |
| -> schema      | object         | No       | A JSON Schema for the rendering parameters. See [Docma Parameter Validation](05-docma-template-rendering.md#docma-parameter-validation). |
| options        | object         | No       | Options passed to the WeasyPrint PDF generator. See [WeasyPrint Options](#weasyprint-options). |
| -> stylesheets | list           | No       | A list of CSS style sheet files that will be fed to the PDF generator. See [CSS Style Sheets](#css-style-sheets) below. |
| metadata       | object         | No       | Values to be added to the output document metadata. See [Document Metadata](05-docma-template-rendering.md#document-metadata) below. |
| -> author      | string         | No       | Document author.                                             |
| -> title       | string         | No       | Document title.                                              |
| -> subject     | string         | No       | Document subject.                                            |
| -> keywords    | string \| list | No       | A string of semi-colon separated keywords or a list of keywords for the PDF. |

> Prior to docma v2.0, metadata fields were specified in the PDF convention of
> `/Author` instead of `author`. This is still supported for backward
> compatibility but the naming shown above should now be used. Docma will use
> the appropriate conventions for PDF and HTML when producing output.

### Document References

The `documents` key in the configuration file is a list of component documents
that will be rendered and assembled into the final output document.

Each element in the document list can be either a string containing the name of
a content file or an object containing the following keys:

| Key | Type   | Required | Description                                                                   |
| --- | ------ | -------- | ----------------------------------------------------------------------------- |
| src | string | Yes      | Name of a content file.
| if  | string | No       | A string that will be Jinja rendered with the run-time parameters and evaluated as a truthy value. If the value is true (the default), the document is included. Truthy *true* values are  `true` / `t` / `y` / `yes` and non-zero integers. Truthy *false* values are `false` / `f` / `no` / `n` and zero and empty strings.|

Content files must be of one of the following types:

*   HTML (`*.html` / `*.htm`)
*   PDF (`*.pdf`) (for PDF output documents only)

Content files can (and generally should) be contained within the template source
directory hierarchy. The file is referenced by its path relative to the template
source base directory.

Content files can also be remote and will be loaded dynamically during the
rendering process. This differs from [document imports](#document-imports) which
incorporate the document into the template during template compilation.
For remote content files, any of the forms supported by the importer subsystem
can be used. e.g.

*   `http(s)://host/some/path/...`
*   `s3://bucket/some/path/...`

Unlike imports, dynamically referenced content documents must be in HTML or PDF
format. There is no compilation of other formats to HTML.

> It is *strongly* recommended to include all content files in the template
> itself (e.g using [document imports](#document-imports) for remote files).
> This will be faster and more predictable at run-time as well as
> improving traceability of documents.

For example:

```yaml
documents:
  - content/cover.html
  # Our main contract document template
  - content/contract.html
  # Boilerplate PDF to include
  - content/standard-terms.pdf
  # Reference to a file in S3
  - s3://my-content-bucket/extra-terms.pdf
  # Now a conditional document using evaluated parameters.
  # The "if" condition will evaluate to the string "True" or "False".
  - src: content/even-more-terms.pdf
    if: '{{ contract.term_in_years >= 3 }}'
```

Some of the HTML files may have been compiled from other formats (e.g.
Markdown) during the compilation phase. All references to the file during
rendering must use the HTML file name. So, for example, a file `content/text.md`
in the template source, will be present as `content/text.html` in the compiled
template.

> The original, uncompiled files are also replicated into the template to allow
> later recompilation and for traceability. The uncompiled files are not used in the rendering process.

See [Document Template Content](#document-template-content) for more
information.

### Overlay Documents

> PDF outputs only.

The `overlays` key in the configuration file is a list of documents that are
prepared in the same way as the primary documents. These are used when the final document requires
an overlaid stamp or underlaid watermark. See [Watermarking](05-docma-template-rendering.md#watermarking) for
more information.


### Document Imports

The [configuration file](#template-configuration-file) may contain an `imports`
key to specify a list of external files that will be included within the
compiled template package. The imported file is processed just like a local
file, including compilation of supported non-HTML formats (e.g. Markdown) into
HTML.

Imports are specified as a URL, with the URL scheme determining the means of
access. Imports are currently supported for:

*   AWS S3: `s3://....`

*   Web content: `http(s)://...`

The [content importers interface](09-installation-and-usage.md#content-importers) is extensible. New sources
can be added easily.

Each import specification is either a string or an object, like so:

```yaml
imports:

  # Simple string format. This S3 file will be placed in the template based on
  # the last component of the filename (i.e `myfile.pdf`).
  - s3://my-bucket/some/path/myfile.pdf

  # Object format to allow a file to be imported and renamed in the process.
  # This will copy the file into the template as `content/afile.pdf`.
  - src: s3://my-bucket/some/path/myfile.pdf
    as: content/afile.pdf

  # This Markdown file will be compiled and can be referenced elsewhere in the
  # template as "content/mydoc.html"
  - src: s3://my-bucket/some/path/somedoc.md
    as: content/mydoc.md

  # Import an image
  - src: http://a.url.com/some/image.png
    as: resources/image.png

  # Import a font:
  - src: http://host/my-corporate-font.ttf
    as: fonts/my-corporate-font.ttf
```

> Imported docs are limited to 10MB in size.

### WeasyPrint Options

> PDF outputs only.

WeasyPrint is used for converting HTML to PDF for PDF document production.  It
provides a number of
[options](https://doc.courtbouillon.org/weasyprint/stable/api_reference.html#weasyprint.DEFAULT_OPTIONS)
to control aspects of the PDF production process. These can be specified under
the `options` key of the [template configuration
file](#template-configuration-file).

The following options are set by docma itself. They can be overridden in the
template but it's best not to.

|Option|Value set by docma| Notes |
|-|-|------|
|media|print| |
|optimize\_images|True|This is required to avoid an image loading bug in WeasyPrint.|

### CSS Style Sheets

> PDF outputs only.

The [configuration file](#template-configuration-file) may contain an `options
--> styesheets` key that lists files containing style sheets that will be
applied to _all_ HTML document files when converting them to PDF. Hence, these
files should only contain styles that should be applied everywhere.

In some cases, including for HTML outputs, it will be more appropriate to have
styles defined within the HTML source document to which they relate, or included
from CSS files using the Jinja `include` directive.

### Sample Configuration File

A sample file might look like this:

```yaml
description: Contract of Sale
owner: Cest Moi
version: 1.0.0

# List the primary files containing document content. File names are relative to
# the root of the template.
documents:
  - content/cover.html
  # Our main contract document template
  - content/contract.html
  # Boilerplate PDF to include
  - content/standard-terms.pdf
  # Now a conditional document using evaluated parameters.
  # The "if" condition will evaluate to the string "True" or "False".
  - src: content/extra-terms.pdf
    if: '{{ contract.term_in_years >= 3 }}'

# Bring these files into the package when building the template.
imports:
  - src: s3://my-bucket/common-files/standard-terms.pdf
    as: content/standard-terms.pdf

# Used in the HTML to PDF conversion
options:
  stylesheets:
    - styles.css

parameters:
  # These defaults are deep-merged into any parameters specified at run-time
  # during rendering. The latter will take precedence.
  defaults:
    locale: en_AU
    our_abn: 54321123456
    contract:
      term_in_years: 3
  # JSON Schema used to validate parameters supplied at run-time.
  schema:
    $schema: https://json-schema.org/draft/2020-12/schema
    title: Parameters validation schema
    type: object
    required:
      - locale
      - customer_name
      - customer_abn
      - contract
      - price
    properties:
      locale:
        type: string
        format: locale
      customer_name:
        type: string
        minLength: 1
      customer_abn:
        type: string
        format: au.ABN
      contract:
        type: object
      price:
        type: number
        minimum: 1.00

# This gets Jinja rendered and added as metadata to the output documents.
# PDF / HTML conventions for metadata are respected.
metadata:
  title: Contract of Sale
  subject: '{{ customer_name }}'

```



## Document Template Content

The `documents` and `overlays` keys in the document template
[configuration file](#template-configuration-file) list the files that will be
processed and assembled to produce the final PDF document.

Two types of file are permitted in these lists:

1.  HTML files (`*.html` / `*.htm`)

2.  PDF files (`*.pdf`) (PDF output only).

The HTML files may be either files directly constructed by the template author,
files that have been imported via the `imports` key, or HTML that has been
compiled from other formats (e.g. Markdown) during the template compilation
phase.

For compiled files, the original file suffix indicates the content type and
hence the process used to compile it to HTML format.

The [content compiler](09-installation-and-usage.md#content-compilers) interface is extensible. New file
types can be added easily.

### HTML Files (\*.html, \*.htm)

When producing standalone HTML outputs, normal HTML conventions should be
followed, keeping in mind the limitations of the target rendering environment
(e.g. a variety of email clients).

When producing PDF outputs, the source HTML used in a docma template should be
written explicitly for print, rather than web layout. There are a set of special
HTML constructs available when the target media is print. Effective use of
these is essential to producing nice output. For an excellent short tutorial on
the subject, see [Designing For Print With
CSS](https://www.smashingmagazine.com/2015/01/designing-for-print-with-css/)

HTML source files are copied unchanged to the compiled docma template during the
compilation phase.

HTML files can reference other resources in the compiled template (e.g. images,
style sheets etc.) using URLs in the format `file:filename`. For example

```html
<IMG src="file:resources/logo.png" alt="logo">
```

> The `file:` scheme indicator is essential. The filename is relative to the
> template base directory. Do not use `file://` as that implies a network
> location will follow, which makes no sense for local files.

HTML files may contain [Jinja](https://jinja.palletsprojects.com/en/) markup to
manipulate content during the [rendering phase](05-docma-template-rendering.md#docma-template-rendering).

> Take care when re-purposing HTML content from other systems that may leave
> Jinja detritus behind. This may need to be manually deleted first.

HTML files can also reference dynamic content generators that will be invoked
during the [rendering phase](05-docma-template-rendering.md#docma-template-rendering). This can be used to
include content for charts, QR codes etc. Dynamic content generators are
accessed by referencing a URL with the `docma` scheme.

For example, the following will generate and insert a QR code:

```html
<IMG
  src="docma:qrcode?{{
    {
      'text': 'Hello world!',
      'fg': 'white',
      'bg': '#338888'
    } | urlencode
  }}"
>
```

This is the same thing, more cryptically:

```html
<IMG src="docma:qrcode?text=Hello+world%21&fg=white&bg=%23338888">
```

See [Dynamic Content Generation](05-docma-template-rendering.md#dynamic-content-generation) for more
information.


Important points to note:

*   The [WeasyPrint](https://weasyprint.org) package is designed to convert HTML
    for print to PDF. It does an excellent job, but some constructs take a bit
    of fiddling to get right. It seems to be more aligned to Safari behaviour
    than, say, Chrome, if that helps when previewing template components.

*   HTML produced by some WYSIWYG editors can be a tortured, gnarly mess.
    WeasyPrint may struggle with it. In many cases, it's better to hand-write
    lean, clean HTML using an IDE or an AI crutch of some kind.

### PDF Files (\*.pdf)

> PDF output only.

PDF files in the template are copied to the compiled template unchanged. They
are simply added into the final document composition process as-is. This is
useful for boilerplate content, such as contract terms and conditions.

PDF files are not Jinja rendered during compilation. Once again, they are used
as-is.

### Markdown Files (\*.md)

All Markdown files are converted to HTML during the compilation phase. i.e.
`myfile.md` in the template source becomes `myfile.html` in the compiled
template.

> The HTML variant of the name **must** be used everywhere in the template when
> referencing the file.

Markdown files may contain [Jinja](https://jinja.palletsprojects.com/en/) markup
to manipulate content during the [rendering phase](05-docma-template-rendering.md#docma-template-rendering).

Conversion from Markdown to HTML is done using the Python
[markdown](https://python-markdown.github.io) package with the following
[extensions](https://python-markdown.github.io/extensions/) enabled:

*   extras
*   admonition.

Important points to note:

*   The conversion from Markdown to HTML will *not* add
    `<HTML>...<BODY></BODY></HTML>`
    framing around the result. This is an advantage, as it means the content can
    be included in other documents using Jinja `{% include 'myfile.html' %}`
    directives. If a Markdown originated source file is to be used stand-alone,
    a small HTML wrapper that references the content file may be needed to
    provide the HTML framing, style sheet etc.

*   The Markdown format is particularly suited to longer, textual content.
    It is a lot easier to edit and maintain than HTML, but complex styling is
    more difficult. The Python [markdown](https://python-markdown.github.io)
    package has some non-standard extensions that do help with this.


## Locale in Docma Templates

Prior to version 2.2.0, docma had no particular notion of the region or locale
with which a particular template, or the documents it produces, is associated.
If special formatting was required, it was up to the template designer to handle
that manually.

This applied for elements such as:

* phone numbers
* currencies
* numbers and percentages
* dates and times.

Version 2.2.0 introduces the concept of *locale*. A new suite of [docma provided
Jinja filters](05-docma-template-rendering.md#custom-jinja-filters-provided-by-docma) use locale information
to handle the elements listed above in accordance with locale specific
conventions instead of requiring the template designer to handle everything
manually. For example:

```jinja
{{ 123456 | decimal }} -- Format using locale specific separators etc.
{{ 123456 | AUD }} -- Format however Australian dollars are shown in the current locale
```

See [Docma Jinja Rendering](05-docma-template-rendering.md#docma-jinja-rendering) for more information.

The locale for a template manifests as an additional Jinja rendering parameter,
`locale`, which is expressed in the normal way as a combination of a language
indicator and a 2 character ISO country code. e.g. `en_AU`, `en_CA`, `fr_CA`.
It can be set in the same way as any other rendering parameter, including any,
or all, of the following (from lowest precedence to highest):

*   Including it in the `parameters -> defaults` in the
    [template configuration file](#template-configuration-file).

*   Specifying it on the command line when rendering a template to PDF or HTML
    output.

*   Setting it within a template using `{% set locale="...." %}`.

*   In some [jinja filters](05-docma-template-rendering.md#custom-jinja-filters-provided-by-docma), specifying
    locale as an explicit argument to override the current effective value.

From version 2.2.0, new templates created using [docma
new](09-installation-and-usage.md#creating-a-new-document-template) will include a default value in the
[template configuration file](#template-configuration-file). It's a good idea to
add it to earlier templates, thus:

```yaml
# config.yaml

parameters:
  defaults:
    locale: "en_AU"
```

> If `locale` is not specified using one of the mechanisms described above, it
> will default to whatever random value the underlying platform assumes.
> Good luck with that.


