# Installation and Usage

## TL;DR

First install the [prerequisites](#prerequisites) then ...

```bash
pip install docma

# Optionally, add duckdb and lava support
pip install 'docma[duckdb]'
pip install 'docma[lava]'

# Check docma installed ok
docma --help

# Create our first docma template. This is a working basic template.
docma new my-template

# Compile it
docma compile -i my-template -t my-template.zip

# Render it to PDF.
docma pdf -t my-template.zip -o my-doc.pdf
```



## Prerequisites

### General

Python3.11+ is required.

### Mac

On Mac OS, GTK is required for the HTML to PDF process.

```bash
brew install gtk+
```

If [DuckDB](https://duckdb.org) data sources are used, install the DuckDB CLI.

```bash
brew install duckdb
```

### Linux

On Linux, [Pango](https://www.gtk.org/docs/architecture/pango) is required for
the HTML to PDF process.

If [DuckDB](https://duckdb.org) data sources are used,
[install the DuckDB CLI](https://duckdb.org/docs/installation/?version=stable&environment=cli&platform=linux&download_method=package_manager).
The Python API will be installed automatically when docma is installed.

To build the user guide, [Pandoc](https://pandoc.org) is required. Follow the
[Pandoc installation instructions](https://pandoc.org/installing.html).

### DOS

**Docma** might work on DOS. How would I know? Why would I care?

I guess you could try WSL 2. If you do, please let us know. 


## Installing Docma

### Installing with Pip

Basic install:

```bash
pip install docma
```

This will
install the base **docma** Python package and the docma CLI. This will not
install support for [duckdb](07-data-sources-in-docma.md#data-provider-type-duckdb) or
[lava](07-data-sources-in-docma.md#data-provider-type-lava) data providers.

To install support for the [duckdb](07-data-sources-in-docma.md#data-provider-type-duckdb) data provider:

```bash
pip install 'docma[duckdb]'
```

To install support for the [lava](07-data-sources-in-docma.md#data-provider-type-lava) data provider:

```bash
pip install 'docma[lava]'
```

### Installing from the Repo

Clone the docma repo.

The rest of the setup is handled by the Makefile.

```bash
# Create venv and install the required Python packages
make init
# Activate the virtual environment
source venv/bin/activate
```

To run the **docma** CLI directly from the repo:

```bash
python3 -m docma.cli.docma --help
```

To build docma, use the Makefile.

```bash
# See what we can build ...
make
# ... or ....
make help
```

To build an install bundle:

```bash
make pkg
```


## Docker

The repo includes support for building a docker container on an Amazon Linux
2023 base with **docma** installed. To build the image:

```bash
make docker
```

This will include support for the [duckdb](07-data-sources-in-docma.md#data-provider-type-duckdb) data
provider, but not the [lava](07-data-sources-in-docma.md#data-provider-type-lava) data provider.

> The basic image doesn't add any fonts to the minimal set already available
> in the Amazon Linux 2023 image. To add fonts, build your own image on the
> **docma** base image.


## The Docma CLI

The **docma** CLI provides everything required to compile and render document
templates.

```bash
# Get help
docma --help
```

It supports the following sub-commands.

| Command    | Description                                                       |
|------------|-------------------------------------------------------------------|
| compile    | Compile a source directory into a document template.              |
| html       | Render a document template to PDF.                                |
| html-batch | Render a batch of HTML documents from a single document template. |
| info       | Print information about a document template.                      |
| new        | Create a new docma template source directory.                     |
| pdf        | Render a document template to PDF.                                |
| pdf-batch  | Render a batch of PDF documents from a single document template.  |

Each sub-command has its own help:

```bash
docma compile --help
```

A typical usage sequence might be:

```bash
# First create the source for the document template in its own directory
docma new my-template

# Add content, configuration etc. Then ...

# Compile
docma compile -i my-template -t my-template.zip

# Render to PDF
docma pdf -t my-template.zip -o my-doc.pdf --file parameters.yaml

# Render to HTML
docma html -t my-template.zip -o my-doc.pdf --file parameters.yaml
```


## Creating a New Document Template

To create a new docma template directory:

```bash
docma new <DIRECTORY>
```

This will prompt the user to enter a small number of configuration parameters.
They are *all* mandatory. Do not leave anything blank.

The specified directory will now contain a very simple, but complete, document
template source directory that can be compiled and rendered:

```bash
docma compile -i <DIRECTORY> -t my-template.zip
docma pdf -t my-template.zip -o my-doc.pdf
```


## Docma Python API

The API is quite basic:

```python
from docma import compile_template, render_template_to_pdf

template_src_dir = 'a/b/c'
template_location = 'my-template.zip'  # ... or a directory when experimenting
pdf_location = 'my-doc.pdf'
params = {...}  # A Dict of parameters.

compile_template(template_src_dir, template_location)

pdf = render_template_to_pdf(template_location, params)

# We now have a pypdf PdfWriter object. Do with it what you will. e.g.
pdf.write(pdf_location)
```

> Refer to the API documentation for more information.


## Building the Documentation

Docma comes with this user guide and auto-generated API documentation.

To build the user guide, [Pandoc](https://pandoc.org) is required. For mac OS:

```bash
brew install pandoc
```

To build the documentation:

```bash
make doc
```

The generated documentation is placed in the `dist/doc` directory. On mac OS:

```bash
# Open the user guide
open -a Safari dist/doc/docma-user-guide.site/index.html

# Open the API doc
open -a Safari dist/doc/api/_build/html/index.html
```

By default, the user guide is generated in multi-page HTML, markdown and EPUB
formats. To also generate Microsoft Word (docx) and single page HTML, edit this
line in `doc/Makefile` to add `docx` and `html`:

```make
# FORMATS=md docx epub html site
FORMATS=md site epub
```

To edit the documentation, ensure that a spell check is done as part of the
process using:

```bash
make spell
```

This requires the `aspell` tool. For mac OS:

```bash
brew install aspell
```

## Running Unit Tests

The unit tests require some docker based components (Postgres, web server etc.)
to be up and running. These require a `.env` file containing credentials for
test accounts etc. In the main directory, copy `dot-env-sample` to `.env` and
edit it to add passwords in the indicated spots. The values don't really matter
as the accounts will be created as part of each test session. Even so, **do
not** add `.env` to the repo. It's bad form.

To run the tests:

```bash
# Start the docker components. This will take a while on first invocation as it
# needs to download base images and build some stuff on them.
make up

# Run tests 
make test

# Get a coverage report 
make coverage

# Check the coverage report (on a Mac)
open -a Safari dist/test/htmlcov/index.html

# Stop the docker components when done
make down
```


## Extending Docma

Docma has a number of plugable interfaces to allow extension. The process works
by placing a Python file in the appropriate directory in the code base, unless
otherwise indicated. These are automatically discovered as required.

### Content Importers

Content importers operate during the docma [compile](04-docma-template-compilation.md#docma-template-compilation)
phase. They collect components from external sources and inject them into the
compilation process.

Imported components are referenced via a URL `scheme://....`. The `scheme` is
used to select the importer to be used.

To create a new importer, add a new Python file into `docma/importers`. It will
contain a decorated function that has a signature like so:

```python
@content_importer('http', 'https')
def _(uri: str, max_size: int = 0) -> bytes:
    """Get an object from the web."""
    ...
```

### Content Compilers

Content compilers operate during the docma [compile](04-docma-template-compilation.md#docma-template-compilation)
phase. They transform a source format into HTML. The source format is determined
by the filename suffix.

To create a new compiler, add a new Python file into `docma/compilers`. It will
contain a decorated function that has a signature like so:

```python
@content_compiler('xyz')
def _(src_data: bytes) -> str:
    """Compile xyz source files into HTML."""

    return ...
```

### URL Fetchers

[URL fetchers](05-docma-template-rendering.md#dynamic-content-generation) operate during the docma
[render](05-docma-template-rendering.md#docma-template-rendering) phase.  They provide WeasyPrint with the
means to resolve URLs within the HTML being converted to PDF.

URL fetchers are selected based on the scheme of the URL.

To create a new URL fetcher, add a new Python file into `docma/fetchers`.
For example, to handle URLs of the form `xyz://....`, the new file will have a
function with a signature like so:

```python
@fetcher('xyz')
def _(purl: ParseResult, context: DocmaRenderContext) -> dict[str, Any]:
    """
    Fetch xyz:... URLs for WeasyPrint.

    :param purl:    A parsed URL. See urllib.parse.urlparse().
    :param context: Document rendering context.
    

    :return:        A dict containing the URL content and mime type.
    """

    ...

    return {
        'string': ...,  # This is named `string` but must be a bytestring.
        'mime_type': ...

    }
```

### Content Generators

[Content generators](05-docma-template-rendering.md#dynamic-content-generation) operate during the docma
[render](05-docma-template-rendering.md#docma-template-rendering) phase. They dynamically generate content for
WeasyPrint when a URL in the following form is accessed.

```bare
docma:<generator-name>?<generator-params>
```

They are typically used for generating image content
(charts, QR codes etc.) but they can be used wherever URLs return content to
WeasyPrint.

To create a new content generator, add a new Python file into
`docma/generators`. Start by copying the sample `swatch.py` generator and
modifying as required.

### Data Providers

[Data providers](07-data-sources-in-docma.md#data-sources-in-docma) operate during the docma
[render](05-docma-template-rendering.md#docma-template-rendering) phase.

The data provider handler is selected by the `type` component of a
[data source specification](07-data-sources-in-docma.md#data-source-specifications).

To create a new data provider, add a new Python file into `docma/data_providers`.
Start by copying one of the existing providers and modify as needed.

### Format Checkers

Docma has a number of custom [format
checkers](05-docma-template-rendering.md#format-checkers-provided-by-docma) that serve a dual role as JSON
Schema string formats and custom Jinja tests. These are implemented using a
simple plugin mechanism. Read the docstring at the top of `docma/lib/plugin.py`
before launching into it.

To create a new format checker, add a new Python file into
`docma/plugins/format_checkers`.  Start by copying one of the existing checkers.

Checkers can be grouped together in families (e.g. the `au.*` suite) using
nested Python packages (directories containing `__init__.py`). The discovery and
loading process is automatic.

Each checker is basically a decorated function with a single parameter,
being the string value to be checked, and must return a boolean indicating
whether it conforms to the required format, or not.

It is also possible to have checkers with names generated dynamically at
run-time. Tricky. Don't start here on day one but check out the
`DateFormatResolver` class in `docma/jinja/resolvers.py` if the fever is upon
you.

> Resolvers are *not* automatically discovered.

### Custom Jinja Filters

Docma has a number of custom Jinja
[filters](05-docma-template-rendering.md#custom-jinja-filters-provided-by-docma). These are implemented using
a simple plugin mechanism. Read the docstring at the top of
`docma/lib/plugin.py`  before launching into it.

To create a new format checker, add a new Python file into
`docma/plugins/jinja_filters`.  Start by copying one of the existing filters.

Checkers can be grouped together in families (e.g. the `au.*` suite) using
nested Python packages (directories containing `__init__.py`). The discovery and
loading process is automatic.

It is also possible to have filters with names generated dynamically at
run-time. For example, the [currency filters](05-docma-template-rendering.md#jinja-filter-currency) work this
way. Tricky. Don't start here on day one but check out the
`CurrencyFilterResolver` class in `docma/jinja/resolvers.py` if inspiration
strikes.

> Resolvers are *not* automatically discovered.

### Custom Jinja Tests

Docma does not currently provide any custom Jinja tests, other than custom
[format checkers](05-docma-template-rendering.md#format-checkers-provided-by-docma) which are kept separate
because they server both Jinja and JSON Schema.

Unlike custom [format checkers](05-docma-template-rendering.md#format-checkers-provided-by-docma), custom
tests can be written to accept arguments additional to the value being tested.

All of the scaffolding required to add custom Jinja tests is present. The
mechanism is the same as used for adding [custom jinja
filters](#custom-jinja-filters) except that the required decorator is
`@jtest` instead of `@jfilter` and they should be placed in
`docma/plugins/jinja_tests` instead of `docma/plugins/jinja_filters`. The
discovery and loading process is automatic.

### Custom Jinja Extensions

Docma has a number of custom Jinja
[extensions](05-docma-template-rendering.md#custom-jinja-extensions-provided-by-docma). These are all
contained in the file `docma/jinja/extensions.py`. New ones can be added to this
file but if you think you need to, think again.



