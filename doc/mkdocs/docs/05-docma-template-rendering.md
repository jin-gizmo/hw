# Docma Template Rendering

The document rendering phase combines a compiled docma template with run-time
specified parameters and dynamically generated content to produce a final output
document.

The rendering process is slightly different for PDF and HTML outputs.

## Rendering for PDF Outputs

![](img/render-phase-pdf.svg)

The main steps in the process for PDF production are:

1.  Marshal the [rendering parameters](#rendering-parameters).

2.  [Validate the rendering parameters](#docma-parameter-validation).

3.  Collect the list of documents to be incorporated into the final output PDF.

4.  [Render HTML documents](#docma-jinja-rendering) in the component list using
    Jinja to inject the [rendering parameters](#rendering-parameters).

5.  Convert the HTML documents to PDF using [WeasyPrint](https://weasyprint.org/).
    This process will also generate any [dynamic content](#dynamic-content-generation)
    from specifications embedded in the source HTML.

6.  Assemble all of the components (generated PDFs and any listed static PDFs)
    into a single PDF document.

7.  Add any requested [watermarking or stamping](#watermarking) to the document.

8.  Jinja render any required metadata specified in the
    [template configuration file](03-document-templates.md#template-configuration-file) and add it to
    the PDF.

9.  Optionally, compress the PDF using lossless compression.

>   Depending on the PDF contents, compression may, or may not, help.

## Rendering for HTML Outputs

![](img/render-phase-html.svg)

The main steps in the process for HTML production are:

1.  Marshal the [rendering parameters](#rendering-parameters).

2.  [Validate the rendering parameters](#docma-parameter-validation).

3.  Collect the list of documents to be incorporated into the final output HTML.

4.  [Render HTML documents](#docma-jinja-rendering) in the component list using
    Jinja to inject the [rendering parameters](#rendering-parameters).

5.  Process `<IMG>` tags in the HTML to generate and embed any
    [dynamic content](#dynamic-content-generation) from specifications embedded
    in the source HTML. Static images may also be embedded.

6.  Assemble all of the component HTML documents into a single HTML document.

8.  Jinja render any required metadata specified in the
    [template configuration file](03-document-templates.md#template-configuration-file) and add it to
    the HTML.



## Docma Parameter Validation

Docma supports the use of [JSON Schema](https://json-schema.org) to validate
rendering parameters at run-time. Parameters are validated against a schema
provided in the `parameters->schema` key in the 
[template configuration file](03-document-templates.md#template-configuration-file) prior to generating
the output document. Failing validation will halt the production process.

> Provision, and hence use, of a parameter validation schema is optional, but
> highly recommended to reduce the risk of generating an important
> document incorrectly or with nonsensical values.

All of the normal facilities of  [JSON Schema](https://json-schema.org) are
available, except for external schema referencing with `$ref` directives. Like
the [JSON Schema built-in string
formats](https://json-schema.org/draft/2020-12/draft-bhutton-json-schema-validation-00#rfc.section.7.3),
docma provided [format checkers](#format-checkers-provided-by-docma) can be used
in a schema specification with the `format` attribute of string objects

The following sample schema fragment shows how these are used:

```yaml
type: object
properties:
  customer_email:
    type: string
    format: email  # This is a JSON schema built-in format checker
  customer_abn:
    type: string
    format: au.ABN  # This is a docma provided format checker
  target_consumption:
    type: number
    minimum: 0
  consumption_unit:
    type: string
    format: energy_unit  # This is a docma provided format checker
  start_date:
    type: string
    format: date.dmy  # This is a docma provided format checker
```

See also [Format Checkers Provided by Docma](#format-checkers-provided-by-docma).


## Docma Jinja Rendering

The rendering phase uses Jinja to render HTML content with the parameters
provided at run-time. Other components (e.g.
[query specifications](07-data-sources-in-docma.md#query-specifications)) also use Jinja rendering on some
of their content.

> The docma Jinja subsystem has been refactored somewhat in v2.2.0.

All of the facilities provided by Jinja are available, including parameter
injection, loops, conditional content and use of the `include` directive to
incorporate other content from the document template. Include directives should
use the name of the file relative to the template root. e.g.

```jinja2
{% include 'my-file.html' %}
```

See also [Jinja Rendering Parameters Provided by
Docma](#jinja-rendering-parameters-provided-by-docma).

In addition to standard Jinja facilities, docma also provides a number of extra
[filters](#custom-jinja-filters-provided-by-docma) and
[extensions](#custom-jinja-extensions-provided-by-docma).


### Rendering Parameters

The parameters used by docma during the template rendering process is the union
of the following (from highest to lowest precedence):

1.  [Parameters provided by docma](#jinja-rendering-parameters-provided-by-docma).

2.  Parameters supplied by the user at run-time.

3.  Parameters specified under `parameters->defaults` in the
    [template configuration file](03-document-templates.md#template-configuration-file)

Parameters are any object that can be represented in JSON / standard YAML, which
can include arbitrary combinations of objects, lists and scalar values.

The marshalling process *deep-merges* the parameter trees from each source. Lists
are not merged. One list will replace another if they occur at the same location.

#### Jinja Rendering Parameters Provided by Docma

In addition to user supplied parameters, docma includes the following items
under the `docma` key.

| Key      | Notes | Description                   |
| ---------------------- |--| ----------------------------- |
| calendar | | The Python `calendar` module. |
| data     | | Function to invoke a [docma data provider](07-data-sources-in-docma.md#data-sources-in-docma) and return the data as a list of dictionaries. See [Data Source Specifications for HTML Rendering](07-data-sources-in-docma.md#data-source-specifications-for-html-rendering).|
| datetime | | The Python `datetime` module. |
| format | | The format of the output document to be produced, `PDF` or `HTML`. This can be used, among other things, for format specific content or formatting (e.g. CSS variations). |
| paramstyle | (1) |Corresponds to the DBAPI 2.0 `paramstyle` attribute of the underlying database driver when processing a  [query specification](07-data-sources-in-docma.md#query-specifications). |
| template | | An object containing information about the document template.|
| --> description | | The `description` field from the [template configuration file](03-document-templates.md#template-configuration-file).|
| --> doc\_no |(2)| The document number in the list being included in the final document, starting at 1. |
| --> document |(2)| The path for the source document being processed. This is a pathlib `Path()` instance.|
| --> id   | | The `id` field from the [template configuration file](03-document-templates.md#template-configuration-file).|
| --> overlay\_id |(3) | The ID of the current overlay set being rendered. |
| --> overlay\_path |(3) | The path for the current overlay file being rendered. This is a pathlib `Path()` instance.|
| --> page |(2) | The starting page number for the current document with respect to the final output document. This may be useful for manipulating page numbering in multipart documents. Or not.|
| --> version | | The `version` field from the [template configuration file](03-document-templates.md#template-configuration-file).|
| version  | | The docma version. |

Notes:

1.  The `paramstyle` parameter is only available for use in
    [query specifications](07-data-sources-in-docma.md#query-specifications).
2.  The `template.doc_no`, `template.document` and `template.page` parameters
    are only available when a document file is being rendered (i.e. not when an
    overlay is being rendered). The `template.page` parameter is only available
    for PDF outputs.

3.  The `template.overlay_id` and `template.overlay_path` parameters are only
    available when an overlay file is being rendered.

For example, to insert today's date:

```jinja2
{{ docma.datetime.date.today() | date }} -- The "date" filter will format for the locale
```

To check whether we are producing HTML or PDF:

```jinja
We are producing {{ docma.format }} output using docma version {{ docma.version }}.
```



### Custom Jinja Filters Provided by Docma

> Jinja filter management has changed significantly in docma 2.2. Some filters have been renamed (with backward compatible aliases) and a new, more extensible filter plugin system has been implemented.

In addition to the standard filters provided by Jinja, docma provides a number of additions. These are divided into:

* [Generic filters](#generic-filters)
* [Region / country specific filters](#region-specific-filters).

Note that custom filter names are case insensitive.

#### Generic Filters

Filters marked with * are locale aware.

| Filter Name                    | Description                                                                                                                 |
|--------------------------------| -------------------------------------------------- |
| [compact_decimal](#jinja-filter-compact_decimal) * | Format a number in a compact format. |
| [css_id](#jinja-filter-css_id) | Sanitise a string to be a valid CSS identifier.    |
| [currency](#jinja-filter-currency) * | Format currency. |
| [date](#jinja-filter-date) * | Format a date. |
| [datetime](#jinja-filter-datetime) * | Format a datetime. |
| [decimal](#jinja-filter-decimal) * | Format a number. |
| [dollars](#jinja-filter-dollars) | Legacy. Format currency a value as dollars. |
| [parse_date](#jinja-filter-parse_date) * | Parse a date string into a [datetime.date](https://docs.python.org/3/library/datetime.html#datetime.date) instance. |
| [parse_time](#jinja-filter-parse_time) * | Parse a date string into a [datetime.time](https://docs.python.org/3/library/datetime.html#datetime.time) instance. |
| [percent](#jinja-filter-percent) * | Format a percentage. |
| [phone](#jinja-filter-phone) * | Format a phone number. |
| [require](#jinja-filter-require) | Abort with an error message an expression does not have a truthy value. |
| [sql\_safe](#jinja-filter-sql_safe) | Ensure that a string value is safe to use in SQL and generate an error if not. |
| [time](#jinja-filter-time) * | Format a time value. |
| [timedelta](#jinja-filter-timedelta) * | Format a timedelta value. |

#### Region Specific Filters

| Filter Name  | Description                                      |
| ------------ | -------------------------------------------------- |
| abn | Deprecated. Use [au.abn](#jinja-filter-auabn). |
| acn | Deprecated. Use [au.abn](#jinja-filter-auacn). |
| [au.abn](#jinja-filter-auabn) | Format an Australian Business Number (ABN). |
| [au.acn](#jinja-filter-auacn) | Format an Australian Company Number (ACN). |


#### Jinja Filter: au.abn

Format an Australian Business Number (ABN).

This supersedes the, now deprecated, `abn` filter.

```jinja
{{ '51824753556'  | au.abn }} --> 51 824 753 556 
```

#### Jinja Filter: au.acn

Format an Australian Company Number (ACN). This supersedes the, now deprecated,
`acn` filter.

```jinja
{{ '123456789' | au.acn }} --> 123 456 789
```

#### Jinja Filter: compact_decimal

Format numeric values.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[format_compact_decimal()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_compact_decimal)
API. All of its parameters can be used in the filter to obtain fine-grained control over formatting.

The locale is determined as described in [Locale in Docma
Templates](03-document-templates.md#locale-in-docma-templates).  It can also be specified explicitly by
adding a `locale` argument to the filter.

The filter signature is:

```python
compact_decimal(
    value: str | int | float,
    *args,
    rounding: str = 'half-up',
    default: int | float | str | None = None,
    **kwargs
)
```

| Parameter | Description                                                                                                                                                                                                                                                                                              |
|-|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|value | Filter input value. Numbers and strings containing numbers are accepted.                                                                                                                                                                                                                                |
| *args | Passed to Babel's [format_compact_decimal()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_compact_decimal).                                                                                                                                                                   |
| rounding | How to round the value. This must be one of the rounding modes in Babel's `decimal.ROUND_*`, with the `ROUND_` prefix removed. Case is ignored and hyphens become underscores. Defaults to `half-up` (Excel style rounding), instead of `half-even` (Bankers rounding) which is Python's normal default. |
| default | The default value to use for the filter if the input value is empty (i.e. None or an empty string). If the input value is empty and `default` is a string, it is used as-is as the return value of the filter. If the input value is empty, and `default` is not specified, an error is raised. Otherwise, the default is assumed to be numeric and is used as the input to the filter. **Note:** The `default` parameter does something different to the Jinja standard [default](https://jinja.palletsprojects.com/en/stable/templates/#jinja-filters.default) filter. They are both useful but not interchangeable. |
| **kwargs | Passed to Babel's [format_compact_decimal()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_compact_decimal). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{{ 1234.5678 | compact_decimal }} --> 1K
{{ '1234.5678' | compact_decimal }} --> 1K
```

Locale can be specified explicitly, if required:

```jinja
{{ 1234.5678 | compact decimal(locale='fr_FR') }} --> 1 k
```

#### Jinja Filter: css_id

Sanitise a string to be a valid CSS identifier.

```jinja
{{ 'a/()=*&bcd'' | css_id }} --> abcd
```

#### Jinja Filter: currency

Format a currency value.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[format_currency()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_currency)
API.

It is important to understand that there are two orthogonal aspects to formatting
currency values:

1.  The currency involved, such as Australian dollars (AUD), Euros (EUR) etc.

2.  The locale in which the currency is to be presented.

For example:

*   One Australian dollar would appear in Australia as `$1.00`.
*   One Australian dollar would appear in the US as `A$1.00`
*   One Australian dollar would appear in France as `1,00 AU$`

For the docma filter, the currency (AUD in the example above) can be
specified in one of two ways:

1.  By providing an argument to the currency filter `{{ 1 | currency('AUD') }}`;
    or
2.  Using the currency name itself as an alias for the filter name
    `{{ 1 | AUD }}`. Docma dynamically generates a filter alias for known
    currencies. Case is not significant.

The locale is determined as described in [Locale in Docma
Templates](03-document-templates.md#locale-in-docma-templates).  It can also be specified explicitly by
adding a `locale` argument to the filter.

The filter signature is:

```python
currency(
    value: str | int | float,
    currency: str,
    *args,
    rounding: str = 'half-up',
    default: int | float | str | None = None,
    **kwargs
)

# ... or .... 

<CURRENCY_CODE>(
    value: str | int | float,
    *args,
    rounding: str = 'half-up',
    default: int | float | str | None = None,
    **kwargs
)
```

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| value     | Filter input value. Numbers and strings containing numbers are accepted. Jinja will inject this automatically. |
| currency  | The currency code (e.g `AUD`, `EUR` etc.)                    |
| *args     | Passed to Babel's [format_currency()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_currency). |
| rounding  | How to round the value. This must be one of the rounding modes in Babel's `decimal.ROUND_*`, with the `ROUND_` prefix removed. Case is ignored and hyphens become underscores. Defaults to `half-up` (Excel style rounding), instead of `half-even` (Bankers rounding) which is Python's normal default. |
| default   | The default value to use for the filter if the input value is empty (i.e. None or an empty string but not zero). If the input value is empty and `default` is a string, it is used as-is as the return value of the filter. If the input value is empty, and `default` is not specified, an error is raised. Otherwise, the default is assumed to be numeric and is used as the input to the filter. **Note:** The `default` parameter does something different to the Jinja standard [default](https://jinja.palletsprojects.com/en/stable/templates/#jinja-filters.default) filter. They are both useful but not interchangeable. |
| **kwargs  | Passed to Babel's [format_decimal()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_decimal). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{{ 123 | AUD }} --> $123.00
{{ 123 | currency('AUD') }} --> $123.00
{{ '123' | NZD }} --> NZD123.00 (numeric strings are fine as input)
{{ None | AUD }} --> ERROR!
{{ None | AUD(default=0) }} --> $0.00
{{ None | AUD(default='FREE!')}} --> FREE!
{{ -123 | AUD(format="¤#,###;(¤#)", currency_digits=False)}} --> ($123)
```

Locale can be specified explicitly, if required:

```jinja
{{ 123 | EUR(locale='en_GB') }} --> €123.00
{{ 123 | EUR(locale='fr_FR' }} --> 123,00 €
```

The legacy [dollars](#jinja-filter-dollars) filter can be replicated like so (use whatever dollar currency is appropriate):

```jinja
{{ 1234.5 | dollars }} --> {{ 1234.5 | AUD }} --> $1,234.50
{{ 1234.5 | dollars(0) }} --> {{ AUD(format="¤#,###", currency_digits=False) }} --> $1,235
```


#### Jinja Filter: date

Format date values.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[format_date()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_date)
API. All of its parameters can be used in the filter to obtain fine-grained
control over formatting.

The locale is determined as described in [Locale in Docma
Templates](03-document-templates.md#locale-in-docma-templates).  It can also be specified explicitly by
adding a `locale` argument to the filter.

The filter signature is:

```python
date(value: datetime.date | datetime.datetime, *args, **kwargs)
```

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| value     | Filter input value. This must be a `datetime.date` or `datetime.datetime` instance. |
| *args     | Passed to Babel's [format_date()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_date). |
| **kwargs  | Passed to Babel's [format_date()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_date). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{% set value = docma.datetime.date(2025, 9, 17) %}

{{ value | date }} --> 17 Sept 2025 (medium format is the default)
{{ value | date(format='short') }} --> 17/9/25
{{ value | date(format='long') }} --> 17 September 2025
{{ value | date(format='full') }} --> Wednesday, 17 September 2025
{{ value | date(format='dd/MM/yyyy')}} --> 17/09/2025
```

Locale can be specified explicitly, if required:

```jinja
{{ value | datetime(locale='en_US') }} --> Sep 17, 2025
```

If date strings need to be handled, they will need to be converted to a Python
[datetime.date](https://docs.python.org/3/library/datetime.html#datetime.date) instance
first. For date strings guaranteed to be in ISO 8601 format, Python's standard
`datetime.date.fromisoformat()` is fine. Otherwise, the safest way to do this
for dates containing only numbers (no month names) is to use the
[parse_date](#jinja-filter-parse_date) filter as this is (docma) locale aware,
unlike the Python standard datetime.datetime.strptime().

```jinja
{{ docma.datetime.date.fromisoformat('2025-09-1') | date }} --> 1 Sept 2025
{{ '1/9/2025' | parse_date | date }} --> 1 Sept 2025
{{ '1/9/2025' | parse_date(locale='en_US') | date }} --> 9 Jan 2025
```

> **Here be dragons:** There is no fully reliable way to parse dates containing
> month names in a generic, locale-aware away. Don't be tempted to attempt this
> in a docma template. If you think you need to, you are either solving the problem the wrong way or solving the wrong
> problem.

#### Jinja Filter: datetime

Format datetime values.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[format_datetime()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_datetime)
API. All of its parameters can be used in the filter to obtain fine-grained control over formatting.

The locale is determined as described in [Locale in Docma
Templates](03-document-templates.md#locale-in-docma-templates).  It can also be specified explicitly by
adding a `locale` argument to the filter.

The filter signature is:

```python
datetime(value: datetime.date | datetime.datetime | datetime.time, *args, **kwargs)
```

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| value     | Filter input value. Typically, this would be a `datetime.datetime` instance. While `datetime.date` and `datetime.time` instances are also accepted, they are unlikely to be particularly useful. |
| *args     | Passed to Babel's [format_datetime()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_datetime). |
| **kwargs  | Passed to Babel's [format_datetime()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_datetime). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{% set value = docma.datetime.datetime(2025, 9, 17, 14, 15, 16) %}

{{ value | datetime }} --> 17 Sept 2025, 2:15:16 pm
```

Locale can be specified explicitly, if required:

```jinja
{{ value | datetime(locale='en_US') }} --> Sep 17, 2025, 2:15:16 PM
```

If datetime strings need to be handled, they will need to be converted to a
Python
[datetime.datetime](https://docs.python.org/3/library/datetime.html#datetime.datetime)
instance first. For datetime strings guaranteed to be in ISO 8601 format,
Python's standard `datetime.datetime.fromisoformat()` is fine.

```jinja
{{ docma.datetime.datetime.fromisoformat('2025-09-17T14:15:16') | datetime }}
```

> Avoid using the Python standard `datetime.datetime strptime()` if at all
> possible. This will use the platform locale and cannot handle the docma locale.
> The results can be very unpredictable.

#### Jinja Filter: decimal

Format numeric values.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[format_decimal()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_decimal)
API. All of its parameters can be used in the filter to obtain fine-grained
control over formatting.

The locale is determined as described in [Locale in Docma
Templates](03-document-templates.md#locale-in-docma-templates).  It can also be explicitly specified by
adding a `locale` argument to the filter.

The filter signature is:

```python
decimal(
    value: str | int | float,
    *args,
    rounding: str = 'half-up',
    default: int | float | str | None = None,
    **kwargs
)
```

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| value     | Filter input value. Numbers and strings containing numbers are accepted. |
| *args     | Passed to Babel's [format_decimal()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_decimal). |
| rounding  | How to round the value. This must be one of the rounding modes in Babel's `decimal.ROUND_*`, with the `ROUND_` prefix removed. Case is ignored and hyphens become underscores. Defaults to `half-up` (Excel style rounding), instead of `half-even` (Bankers rounding) which is Python's normal default. |
| default   | The default value to use for the filter if the input value is empty (i.e. None or an empty string, but not zero). If the input value is empty and `default` is a string, it is used as-is as the return value of the filter. If the input value is empty, and `default` is not specified, an error is raised. Otherwise, the default is assumed to be numeric and is used as the input to the filter. **Note:** The `default` parameter does something different to the Jinja standard [default](https://jinja.palletsprojects.com/en/stable/templates/#jinja-filters.default) filter. They are both useful but not interchangeable. |
| **kwargs  | Passed to Babel's [format_decimal()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_decimal). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{{ 1234.5678 | decimal }} --> 1,234.568 (Default is to round to 3 decimal places)
{{ '1234.5678' | decimal }} --> 1,234.568 (numeric strings are fine as input)
{{ None | decimal }} --> ERROR!
{{ None | decimal(default=0) }} --> 0
{{ None | decimal(default='nix')}} → nix
```

Locale can be specified explicitly, if required:

```jinja
{{ 1234.5678 | decimal(locale='fr_FR') }} --> 1 234,568
```

#### Jinja Filter: dollars

Round and format a currency value as dollars.

Banker's half-up, rounding is used (like Excel) instead of the half-even rounding that is Python's normal default.

> This filter is a legacy that is not actually deprecated (yet), but its use is discouraged. Use the [currency](#jinja-filter-currency) filter in preference.

The filter signature is:

```python
dollars(value: str | int | float, precision: int = 2, symbol: str = '$')
```


| Parameter | Description                                      |
| --------- | ------------------------------------------------ |
| value     | A number or numeric string.                      |
| precision | Number of decimal places to show. Defaults to 2. |
| symbol    | The currency symbol to show. Defaults to `$`.    |

Examples:

```jinja
{{ 1234.50 | dollars }} --> $1,234.50
{{ 1234.50 | dollars(0) }} --> $1,235
```

#### Jinja Filter: parse_date

Parse a date string into a [datetime.date](https://docs.python.org/3/library/datetime.html#datetime.date) instance.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[parse_date()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.parse_date)
API.

The filter signature is:

```python
parse_date(value: str, *args, **kwargs) -> datetime.date
```

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| value     | Filter input value. The parser understands component ordering variations by locale but cannot handle month names. Numbers only. |
| *args     | Passed to Babel's [parse_date()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.parse_date). |
| **kwargs  | Passed to Babel's [parse_date()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.parse_date). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{{ '1/9/2025' | parse_date) }} --> datetime.date(2025, 9, 1)
```

Locale can be specified explicitly, if required:

```jinja
{{ '1/9/2025' | parse_date(locale='en_US') }} --> datetime.date(2025, 1, 9)
```

#### Jinja Filter: parse_time

Parse a date string into a [datetime.time](https://docs.python.org/3/library/datetime.html#datetime.time) instance.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[parse_time()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.parse_time)
API.

The filter signature is:

```python
parse_time(value: str, *args, **kwargs) -> datetime.time
```

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| value     | Filter input value.                                          |
| *args     | Passed to Babel's [parse_time()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.parse_time). |
| **kwargs  | Passed to Babel's [parse_time()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.parse_time). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{{ '2:15 pm' | parse_time) }} --> datetime.time(14, 15)
```

Locale can be specified explicitly, if required:

```jinja
{{ '1/9/2025' | parse_date(locale='en_US') }} --> datetime.date(2025, 1, 9)
```

#### Jinja Filter: percent

Format percentage values.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[format_percent()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_percent)
API. All of its parameters can be used in the filter to obtain fine-grained
control over formatting.

The locale is determined as described in [Locale in Docma
Templates](03-document-templates.md#locale-in-docma-templates).  It can also be specified explicitly by
adding a `locale` argument to the filter.

The filter signature is:

```python
percent(
    value: str | int | float,
    *args,
    rounding: str = 'half-up',
    default: int | float | str | None = None,
    **kwargs
)
```

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| value     | Filter input value. Numbers and strings containing numbers are accepted. |
| *args     | Passed to Babel's [format_percent()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_percent). |
| rounding  | How to round the value. This must be one of the rounding modes in Babel's `decimal.ROUND_*`, with the `ROUND_` prefix removed. Case is ignored and hyphens become underscores. Defaults to `half-up` (Excel style rounding), instead of `half-even` (Bankers rounding) which is Python's normal default. |
| default   | The default value to use for the filter if the input value is empty (i.e. None or an empty string). If the input value is empty and `default` is a string, it is used as-is as the return value of the filter. If the input value is empty, and `default` is not specified, an error is raised. Otherwise, the default is assumed to be numeric and is used as the input to the filter. **Note:** The `default` parameter does something different to the Jinja standard [default](https://jinja.palletsprojects.com/en/stable/templates/#jinja-filters.default) filter. They are both useful but not interchangeable. |
| **kwargs  | Passed to Babel's [format_percent()](https://babel.pocoo.org/en/latest/api/numbers.html#babel.numbers.format_percent). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{{ 0.1234 | percent }} --> 12%
{{ '0.1234' | percent }} --> 12%
{{ None | percent }} --> ERROR!
{{ None | percent(default=0) }} --> 0%
{{ None | percent(default='--')}} --> --
```

Locale can be specified explicitly, if required:

```jinja
{{ '0.123' | percent(locale='fr_FR') }} --> 12 %
```

#### Jinja Filter: phone

Format phone numbers. If a number cannot be formatted, the unmodified input is
returned.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter.

The underlying process is implemented using the excellent Python
[phonenumbers](https://github.com/daviddrysdale/python-phonenumbers) package,
which is itself a port of
[Google's libphonenumber library](https://github.com/google/libphonenumber).

Phone number formatting varies substantially internationally. Hence, the filter
needs to determine the relevant region for each phone number. It can do that in
one of 3 ways (highest precedence to lowest)

1.  An international code in the source phone number.

2.  An explicit region code argument to the phone filter (expressed as a
    two-character ISO country code).

3.  By assuming the phone number is associated with the effective locale setting
    For example, a locale setting of `en_AU` would imply the number is part of the
    Australian phone numbering plan.

The filter signature is:

```python
phone(number: str, region: str = None, *, format: str = None)
```
| Parameter | Description |
|-|-|
| number | The phone number input to the filter. Phone numbers are always strings, never integers. Ever. |
| region | The region to which the phone number belongs as a 2 character ISO country code. Ignored if the phone number includes an international code. If not specified, the country code from the current effective locale is used. |
| format | See below. |

Phone numbers can be formatted in different ways. The following values of the `format` parameter are supported:

| Format        | Description                                                                                                                                            |
|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| E164          | E.164 is the standard International Telecommunication Union (ITU) format for worldwide telephone numbers. e.g. `+61491570006`                          |
| INTERNATIONAL | The full international phone number, formatted as per national conventions. e.g. `+61 491 570 006`                                                     |
| NATIONAL      | The national number component of the phone number without the international code component, formatted as per national conventions. e.g. `0491 570 006` |
| RFC3966       | The URI format for phone numbers. This will typically generate one-touch call links in on-line content. e.g. `tel:+61-491-570-006`                     |

If not specified, `NATIONAL` is used if the region for the phone number matches
that for the current locale and `INTERNATIONAL` otherwise.

Examples (assuming locale is set to `en_AU`):

```jinja
{{ '0491 570 006' | phone }} --> 0491 570 006 (Locale will provide "AU" as region)
{{ '+61 491 570 006' | phone }} --> 0491 570 006 (Region comes from the number)
{{ '4155550132' | phone('US') }} --> +1 415-555-0132
{{ '4155550132'| phone('US', format='NATIONAL') }} --> (415) 555-0132
{{ '4155550132'| phone('US', format='RFC3966') }} --> tel:+1-415-555-0132
{{ 'bad-to-the-phone' }} --> bad-to-the-phone (If all else fails, return the input)
```



#### Jinja Filter: require

Abort with an error message if the value is not a truthy value (i.e. a non-empty
string, non-zero integer etc), otherwise return the value.

This is useful for situations where it is better to abort if an expression is
expected to have a value, but doesn't, rather than make assumptions.

```jinja
Dear Bob,

Your flight details have changed and your flight will now depart at
{{ flight_time | require('flight_time must be a non-empty string') }}.

Don't be late.
```

#### Jinja Filter: sql_safe

Ensure that a string value is safe to use in SQL and generate an error if not.

This is primarily for use in query specifications to avoid SQL injection. It has
a puritanical view on safety but will cover most normal requirements.

Examples:

```jinja
SELECT * from {{ table | sql_safe }} ...
```

#### Jinja Filter: time

Format time values.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[format_time()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_time)
API. All of its parameters can be used in the filter to obtain fine-grained
control over formatting.

The locale is determined as described in [Locale in Docma
Templates](03-document-templates.md#locale-in-docma-templates).  It can also be specified explicitly by
adding a `locale` argument to the filter.

The filter signature is:

```python
time(value: datetime.time | datetime.datetime, *args, **kwargs)
```

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| value     | Filter input value. This must be a `datetime.time` or `datetime.datetime` instance. |
| *args     | Passed to Babel's [format_time()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_time). |
| **kwargs  | Passed to Babel's [format_time()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_time). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{% set value = docma.datetime.time(14, 15) %}

{{ value | time }} --> 2:15:00 pm
```

Locale can be specified explicitly, if required:

```jinja
{{ value | datetime(locale='de_DE') }} --> 14:15:00
```

If time strings need to be handled, they will need to be converted to a Python
[datetime.time](https://docs.python.org/3/library/datetime.html#datetime.time)
instance first. The safest way to do this is to use the
[parse_time](#jinja-filter-parse_time) filter as this is (docma) locale aware.

```jinja
{{ '14:15' | parse_time | time }} --> 2:15:00 pm
{{ '2:15 pm' | parse_time | time }} --> 2:15:00 pm
```

#### Jinja Filter: timedelta

Format timedelta values.

This is a [locale-aware](03-document-templates.md#locale-in-docma-templates) filter that provides an
interface to the Babel
[format_timedelta()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_timedelta)
API. All of its parameters can be used in the filter to obtain fine-grained control over formatting.

The locale is determined as described in [Locale in Docma
Templates](03-document-templates.md#locale-in-docma-templates).  It can also be specified explicitly by
adding a `locale` argument to the filter.

The filter signature is:

```python
timedelta(value: datetime.timedelta | datetime.datetime | datetime.time, *args, **kwargs)
```

| Parameter | Description                                                  |
| --------- | ------------------------------------------------------------ |
| value     | Filter input value. This must be a `datetime.timedelta` instance. |
| *args     | Passed to Babel's [format_timedelta()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_timedelta). |
| **kwargs  | Passed to Babel's [format_timedelta()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_timedelta). This includes the option of using the `locale` parameter to specify locale. |

Examples (assuming locale is set to `en_AU`):

```jinja
{% set d1 = docma.datetime.datetime(2025, 9, 17, 14, 15, 16) %}
{% set d2 = docma.datetime.datetime(2025, 9, 19, 14, 15, 16) %}

{{ (d2 - d1) | timedelta }} --> 2 days (default format is 'long')
{{ (d2 - d1) | timedelta(format='narrow') }} --> 2d
{{ (d2 - d1) | timedelta(add_direction=True) }} --> in 2 days
```

Locale can be specified explicitly, if required:

```jinja
{{ (d2 - d1) | timedelta(locale='uk_UA') }} --> '2 дні'
```

> The Babel [format_timedelta()](https://babel.pocoo.org/en/latest/api/dates.html#babel.dates.format_timedelta) rounding process is not particularly intuitive on first appearance but makes sense once you get the hang of it. You may need to experiment with `threshold` and `granularity` arguments to get the desired effect.

### Custom Jinja Extensions Provided by Docma

Docma provides some custom Jinja extensions.
In Jinja, extensions are invoked using the following syntax:

```jinja
{% tag [parameters] %}
```

In addition to the custom extensions described below, docma also provides the
following standard Jinja extensions:

*   [debug](https://jinja.palletsprojects.com/en/stable/extensions/#debug-extension)
*   [loopcontrols](https://jinja.palletsprojects.com/en/stable/extensions/#loop-controls).

#### Jinja Extension: abort

The **abort** extension forces the rendering process to abort with an exception
message. It would typically be used in response to some failed correctness check
where it's preferable to fail document production rather than to proceed in
error.

For example:

```jinja
{% if bad_data %}
    {% abort 'Fatal error - bad data' %}
{% endif %}
```

#### Jinja Extension: dump\_params

The **dump\_params** extension simply dumps the rendering parameters for
debugging purposes. The standard Jinja
[debug](https://jinja.palletsprojects.com/en/stable/extensions/#debug-extension)
extension does something similar (and a bit more) but much less readably.

Typical usage would be:

```html
<PRE>{% dump_params %}</PRE>
```

#### Jinja Extension: global

The **global** extension allows values defined within the Jinja content of a
HTML document to be made available when rendering other components in a
document template.

For example, the following declares the *globals* `a` and `b`.

```jinja
{% global a=1, b='Plugh' %}
```

These can then be accessed, either in the file in which they were declared or in
a different HTML document, or a [query specification](07-data-sources-in-docma.md#query-specifications)
like so:

```jinja
You are at Y{{ globals.a + 1 }}. A hollow voice says "{{ globals.b }}."
```

The result will be:

```bare
Your are at Y2. A hollow voice says "Plugh".
```

Compare this with the standard Jinja `set` operation:

```jinja
{% set a=1 %}
```

The variable `a` can only be accessed in the file in which it is defined, or a
file that includes that file. It cannot be accessed in a different HTML
document, or a [query specification](07-data-sources-in-docma.md#query-specifications)

> **Warning**: It is important to understand that within the docma rendering
> phase, the Jinja rendering of the component HTML documents is *completed* before
> the [generation and injection of dynamic content](#dynamic-content-generation).
> This means that only the *final* value of any global parameter is available
> during the dynamic content generation phase. i.e. It is not possible to use
> globals to pass a loop variable from the Jinja rendering of the HTML into
> the dynamic content generation phase.


### Format Checkers Provided by Docma

Docma includes an extensible set of format checkers. These can be used in two ways:

1.  In [JSON schema specifications](#using-format-checkers-in-json-schema-specifications)
    as `format` specifiers for `string` data elements; and

2.  As [Jinja tests](#using-format-checkers-in-jinja) in content that will be
    Jinja rendered.

The docma provided format checkers are divided into:

* [Generic checkers](#generic-format-checkers)
* [Region / country specific checkers](#region-specific-format-checkers).

All docma provided format checker names are case insensitive.

It is easy to [add new format checkers](09-installation-and-usage.md#format-checkers), as required.

#### Generic Format Checkers


| Test Name        | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| date.dmy         | Date formatted as day/month/year. The separators can be any of `/-_.` or missing (e.g. `31/12/2024`, `31.12.2024`, `31-12-2024`, `31_12_2024`, `31122024`). |
| date.mdy         | Date formatted as month/day/year. The separators can be any of `/-_.` or missing  (e.g. `12/31/2024`, ...). |
| date.ymd         | Date formatted as year/month/day. The separators can be any of `/-_.` or missing (e.g. `2024/12/31`, ...). |
| DD/MM/YYYY       | *JSON Schema use only*. Deprecated. Use `date.dmy` instead.  |
| energy_unit      | An energy unit (e.g. kWh, MVArh).                            |
| locale           | A locale specifier (e.g. `en_AU`, `fr_FR`)                   |
| power_unit       | A power unit (e.g. kW, MVA).                                 |
| semantic_version | A version in the form `major.minor.patch` (e.g. `1.3.0`).    |

#### Region Specific Format Checkers

| Checker Name | Description         |
| ---------------- | -------------------------- |
| ACN           | Deprecated. Use `au.abn` instead.  |
| ABN           | Deprecated. Use `au.abn` instead.   |
| au.ABN           | Australian Business Number.                                  |
| au.ACN           | Australian Company Number.                                   |
| au.MIRN          | Australian energy industry Gas Meter Installation Registration Number. |
| au.NMI           | Australian energy industry National Metering Identifier. |
| MIRN             | Deprecated. Use `au.MIRN` instead. |
| NMI              | Deprecated. Use au.NMI instead. |

#### Using Format Checkers in JSON Schema Specifications

JSON Schema specifications are supported, and strongly recommended, in a number
of docma components, such as as the [template configuration
file](03-document-templates.md#template-configuration-file), and [query
specifications](07-data-sources-in-docma.md#query-specifications). They provide run-time type checking of
important data elements and are an important safety mechanism.

Like the [JSON Schema built-in
string formats](https://json-schema.org/draft/2020-12/draft-bhutton-json-schema-validation-00#rfc.section.7.3),
docma provided format checkers can be used in a schema specification with the
`format` attribute of string objects, like so:

```yaml
type: object
properies:
  prop1:
    type: string
    format: ...  # Use a built in format like "email" or one of docma's format checkers
```

> Examples are given in YAML rather than JSON for readability, and because they
> are specified in YAML in docma.

For example, consider a document template for a contract that requires
parameters for customer email, contract start date, and customer Australian
Business Number (ABN) to be specified: The relevant portion of the [template
configuration file](03-document-templates.md#template-configuration-file) might look like this:

```yaml
parameters:
  schema:
    $schema: https://json-schema.org/draft/2020-12/schema  # Schema for the schema!
    title: Parameters validation schema
    type: object
    required:
      - locale
      - customer_email
      - customer_abn
      - contract_start_date
    properties:
      locale:
        type: string
        format: locale  # This is a docma provided format checker
      customer_email:
        type: string
        format: email  # This is a standard JSON schema format checker
      customer_abn:
        type: string
        format: au.ABN   # This is a docma provided format checker.
      contract_start_date:
        type: string
        format: date.dmy  # This is a docma provided format checker
```

Docma will validate values provided at run-time against this schema.

#### Using Format Checkers in Jinja

In addition to the [standard tests provided by
Jinja](https://jinja.palletsprojects.com/en/stable/templates/#list-of-builtin-tests),
the docma format checkers can also be used as Jinja tests, like so:

```jinja
{% if contract_date is not date.dmy %}
{% abort 'Bad date' %}
{% endif %}
```

When used as Jinja tests, none of the docma format checkers accept an arguments
additional to the value being checked. 

> Docma can also be extended with Jinja tests that can accept additional
> arguments, but these would not also be used in JSON Schema specifications and
> hence would not be considered to be format checkers.


## Dynamic Content Generation

When docma converts HTML into PDF or stand-alone HTML, it needs to resolve all
URLs in the source HTML in things such as `<img src="...">` tags. It does this
via a custom *URL fetcher* that allows content requests to be intercepted and
the resulting content generated dynamically. In this way, docma can generate
dynamic content, such as charts, for inclusion in the final output document.

> There are some differences in this process depending on whether the final
> output is PDF of HTML. See
[Dynamic Content Generation Differences Between PDF and HTML Output](#dynamic-content-generation-differences-between-pdf-and-html-output).

All URLs are constituted thus:

```bare
scheme://netloc/path;parameters?query#fragment
```

Docma determines which custom URL fetcher to apply based on the URL scheme (i.e.
the first part before the colon). The URL fetchers handle a range of non-standard,
docma specific schemes, as well as the standard `http` and `https` schemes.

Docma currently handles the following non-standard schemes:

| Scheme                 | Description                                                                |
|------------------------|----------------------------------------------------------------------------|
| [docma](#scheme-docma) | Interface to docma dynamic content generators of various types.            |
| [file](#scheme-file)   | Interface to access files contained within the compiled document template. |
| [s3](#scheme-s3)       | Interface to access files from AWS S3.                                     |

> The docma URL fetcher interface is easily expandable to handle other schemes.
> See [URL Fetchers](09-installation-and-usage.md#url-fetchers).

### Dynamic Content Generation Differences Between PDF and HTML Output

PDF generation from HTML is performed by WeasyPrint, which will invoke a custom
URL fetcher for *any* URL it needs to access during the conversion process.
This includes, *but is not limited to*, `<IMG>` tags.

For standalone HTML output, the process of invoking a custom URL fetcher is done
by docma itself. It is *only* applied to the `src` attribute of `<IMG>` tags
under specific circumstances. When it is done, the `src` attribute is replaced
in the `<IMG>` tag with the actual content returned by the URL fetcher. i.e.
the data is embedded within the standalone HTML output.

In practice, these differences work naturally, relative to the final viewing
environment for the produced document, static PDF or dynamic HTML.

By default, in HTML outputs, `<IMG>` tags have the content embedded
in place of the `src` attribute in the following circumstances:

*   The `src` URL is not `http(s)://` (i.e. any of the docma custom
    schemes described below); or

*   The `src` URL is `http(s)://`, has no query component `?...`, 
    and the content size is between 100 bytes and 1MB in size.

For the `http(s)://` URLs, it is possible to override the default behaviour by
adding the `data-docma-embed` attribute to the `<IMG>` tag.

For images that are not embedded, it is assumed that the client (e.g. an email
client or web browser) will fetch the images as required at display time.

```html
<!-- Force the image to be embedded -->
<IMG src="http://host/img.png" data-docma-embed="true">

<!-- Prevent the image from being embedded -->
<IMG src="http://host/img.png" data-docma-embed="false">

<!-- This will not be embedded due to size unless we force it -->
<IMG src="http://host/multi-mega-byte-img.png">

<!-- This will not be embedded due to size unless we force it -->
<IMG src="http://host/one-pixel-img.png">

<!-- This will not be embedded due to query component unless we force it -->
<IMG src="http://host/do/something?x=20">

<!-- This will always be embedded and cannot be prevented -->
<IMG src="s3://my-bucket/corporate-logo.png">
```

### Scheme: docma

URLs of the following form are intercepted by docma and used to invoke a dynamic
content generator.

```bare
docma:<generator-name>?<generator-params>
```

Note that for these docma URLs, there is no *netloc* component and hence no `//`
in the URL.

For example, this will generate a QR code:

```html
<IMG style="height: 40px"
    src="docma:qrcode?text=Hello%s20world&fg=white&bg=red">
```

The URL should be properly URL encoded. This can be fiddly, but Jinja can help
here. The example above could also have been written in dictionary format thus:

```html
<IMG style="height: 40px" src=docma:qrcode?{{
  {
    'text': 'Hello world',
    'fg': 'white',
    'bg': 'red'
  } | urlencode
}}">
```

It could also have been written as a sequence of tuples:

```html
<IMG style="height: 40px" src=docma:qrcode?{{
  (
    ('text', 'Hello world'),
    ('fg', 'white'),
    ('bg', 'red')
  ) | urlencode
}}">
```
> The sequence format is **required** if any of the parameters needs to be used
> more than once.

Available content generators are:

| Name | Description |
|--|-----------------|
| [qrcode](#generating-qr-codes) | Generate a QR code. |
| [swatch](#generating-graphic-placeholders-swatches) | Generate a colour swatch as graphic placeholder. |
| [vega](#generating-charts-and-graphs) | Generate a chart based on the [Vega-Lite](https://vega.github.io/vega-lite/) declarative syntax for specifying charts / graphs. |

> The dynamic content generator interface is readily extensible to add new types
> of content. See [Content Generators](09-installation-and-usage.md#content-generators).

#### Generating QR Codes

The QR code dynamic generator accepts the following parameters:

| Parameter | Type | Required | Description |
|-|-|-|------------------------|
| bg | String | No | Background colour of the QR code (e.g. `blue` or `#0000ff`). Default is white. |
| border | Integer | No | Number of boxes thick for the border. Default is the minimum allowed value of 4. |
| box | Integer | No | Number of pixels for each box in the QR code. Default is 10. |
| fg | String | No | Foreground colour of the QR code. Default is black. |
| text | String | Yes | Content to be encoded in the QR code. |

Examples:

```html
<IMG style="height: 40px"
    src="docma:qrcode?text=Hello%s20world&fg=white&bg=red">
```

```html
<IMG style="height: 40px" src=docma:qrcode?{{
  {
    'text': 'Hello world',
    'fg': 'white',
    'bg': 'red'
  } | urlencode
}}">
```

#### Generating Charts and Graphs

Docma supports the [Vega-Lite](https://vega.github.io/vega-lite/) declarative
syntax for specifying charts / graphs. Vega-Lite specifies a mapping between
source data and visual representations of the data. Docma provides mechanisms
for specifying and accessing various data sources and feeding this data through
a Vega-Lite specification to generate charts and graphs.

This is a large topic and more information is provided in
[Charts and Graphs in Docma](06-charts-and-graphs-in-docma.md#charts-and-graphs-in-docma). To whet your appetite,
check out the [Vega-Lite sample gallery](https://vega.github.io/vega-lite/examples/).

This section just summarises the parameters for the `vega` content generator for
reference:

| Parameter | Type | Required | Description |
|-|-|-|------------------------|
| data | String | No | A [docma data source specification](07-data-sources-in-docma.md#data-sources-in-docma). This argument can be repeated if multiple data sources are required.  If not specified, the file referenced by the `spec` parameter must contain all of the required data. |
| format | String | No | Either `svg` (the default) or `png`. Stick to `svg` if at all possible.|
| spec | String | Yes | The name of the file in the compiled document template that contains the Vega-Lite specification for the chart. The contents can be either YAML or JSON. |
| ppi | Integer | No | (`png` format only) Pixels-per-inch resolution of the generated image. Default 72. |
| scale | Float | No | (`png` format only) Scale the chart by the specified factor. Default is 1.0. Generally, it's better to control display size in the HTML but increasing the scale here can improve resolution. |
| params | JSON string | No | A string containing a JSON encoded object containing additional rendering parameters used when rendering the [chart specification](06-charts-and-graphs-in-docma.md#chart-specification-files) and any associated [query specifications](07-data-sources-in-docma.md#query-specifications). |

Examples:

```html
<IMG style="width: 5cm;"
    src="docma:vega?spec=charts/my-chart.yaml&data=...">
```

```html
<IMG style="width: 10cm;" src=docma:vega?{{
  (
    ( 'spec', 'charts/my-chart.yaml' ),
    ( 'data', 'file;data/my-data.csv' ),
    ( 'params', { 'extra_rendering_param': 1234 } | tojson)
  ) | urlencode
}}">
```

#### Generating Graphic Placeholders (Swatches)

The swatch generator produces a simple coloured rectangle with an optional text
message. It's not intended to be useful in final documents, Mondrian
notwithstanding. It has two purposes:

1.  As a simple code sample for dynamic content generators that can be copied
    and modified for new requirements.

2.  As a temporary placeholder when developing the structure of a docma template
    that will be replaced subsequently by a real piece of content (e.g. a chart).


| Parameter | Type | Required | Description |
|-|-|-|------------------------|
| color | String | No | Fill colour of the swatch. Default is a light grey. |
| font | String | No | Font file name for the text. Default is `Arial`. If the specified font is not available, a platform specific default is used.|
| font_size | Integer | No | Font size. Default is 18. |
| height | Integer | Yes | Swatch height in pixels. |
| text | String | No | Text to centre in the swatch. No effort is made to manipulate it to fit. |
| text_color | String | No | Colour for text. Default is black. |
| width | Integer | Yes | Swatch width in pixels. |

> **Colour** or **color**? The code and docma templates stick with `color`,
> because, well, that battle is lost. The user guide uses `colour` in descriptive
> text. Blame Webster for messing it up, not me.

Examples:

```html
<IMG src="docma:swatch?width=150&height=150&color=seagreen">
```

```html
<IMG src="docma:swatch?{{ {
    'width': 150,
    'height': 150,
    'color': '#0080ff',
    'text': 'Hello world',
    'text_color': 'yellow',
    'font_size': 24
    } | urlencode }}"
>
```

### Scheme: file

URLs in HTML files of the form `file:...` are intercepted by docma and the
content is extracted from a file within the compiled document template. As the
file is local to the template, there is no network location so the URL will be
like so:

```html
<IMG src="file:resources/logo.png" alt="logo">
```

> Do not include `//` after `file:`. It will not work.

### Scheme: s3

URLs in HTML files of the form `s3://...` are intercepted by docma and the
content is extracted from AWS S3. A typical usage would be something like:

```html
<IMG src="s3://my-bucket/some/path/logo.png" alt="logo">
```
> Files are limited to 10MB in size.


## Watermarking

**Docma** supports the ability to watermark and stamp PDF documents using
the concept of document *overlays*.

> Overlays are not supported for HTML output documents.

An **overlay** is a PDF document, generated by **docma** that can be used as
either a watermark or a stamp.

A **watermark** is content merged into every page of the final PDF **under** the
main document content.

A **stamp** is content merged into every page of the final PDF **over** the
main document content.

Overlays are defined in the [template configuration file](03-document-templates.md#template-configuration-file)
using the following structure:

```yaml
overlays:
  my-overlay-1:
    # We can have HTML files that will be rendered like other docs
    - a4-portrait.html
    # ... or static PDFs
    - a4-landscape.pdf

  # or ...
  my-overlay-2: a4-portrait.html
```

Each overlay is a named list of documents (or a single document). When **docma**
is requested to add a watermark (or stamp), it is provided with the name of one
or more of the overlays (e.g. `my-overlay-1`).

It will then render each of the files in each overlay list in the same way as the
main document, including rendering with dynamic run-time parameters.

Each page in the main document is then merged with the first page of the first
overlay document in each list that has (approximately) the same page dimensions.

> The process will abort if a matching overlay page cannot be found for a main
> document page.

The presence of the `overlays` section in the configuration file does not itself
enable watermarking / stamping. This has to be explicitly requested.

Watermarking / stamping can be requested using the `--watermark` / `--stamp`
CLI options. If using the Python API, the `watermark` / `stamp` parameters to the
`render_template()` function are used.

It is possible to have both watermarking and stamping used on a single document,
as well as having multiple overlays applied to a single document.

>   A simple grid overlay is provided as part of the basic template created by the
    [docma new](09-installation-and-usage.md#creating-a-new-document-template) command. This can be handy
    when adjusting page layout. To add the grid, the docma CLI rendering command
    would be `docma pdf --stamp grid ...`. Grid size and colour are
    adjustable in the parameter defaults in the template config file.


## Document Metadata

Docma allows the template to control some of the metadata added to the final PDF
or HTML and enforces some values of its own.

PDF and HTML documents have slightly different conventions regarding metadata
naming and formatting. Docma handles these variations.

In HTML, the metadata fields are added into the `<HEAD>` of the final document
in this form:

```html

<meta content="Fred Nurk" name="author"/>
<meta content="A document about stuff" name="title"/>
<meta content="DRAFT, Top-Secret" name="keywords"/>
<meta content="2024-11-21T00:04:38.699978+00:00" name="creation_date"/>
```

In PDF, the meta data fields are used to populate the standard metadata elements
recognised by common PDF readers.

| HTML Naming    | PDF Naming    | Controlled by | Comments                                            |
|----------------|---------------|---------------|-----------------------------------------------------|
| author         | /Author       | Template      | From the `metadata->author` key in `config.yaml`    |
| creation\_date | /CreationDate | Docma         | Document production datetime                        |
| creator        | /Creator      | Docma         | Based on template `id`, `version` and docma version |
| keywords       | /Keywords     | Template      | From the `metadata->keywords` key in `config.yaml`  |
| subject        | /Subject      | Template      | From the `metadata->subject` key in `config.yaml`   |
| title          | /Title        | Template      | From the `metadata->title` key in `config.yaml`     |


## Batch Rendering

Docma supports the ability to generate a batch of output documents from a single
document template using the `pdf-batch` (PDF) and `html-batch` (HTML) sub-commands
of the [docma CLI](09-installation-and-usage.md#the-docma-cli).

The document template needs to anticipate the need for batch rendering by
including some Jinja controlled content that will be varied for each document
produced via document specific parameters. The source for the document specific
batch parameters is a [docma data loader](07-data-sources-in-docma.md#data-sources-in-docma). Data returned
by the data loader is merged in with the fixed rendering parameters, a row at a
time, and docma produces an output document using that combination. The source
data for the batch parameters is specified using a [docma data source
specification](07-data-sources-in-docma.md#data-source-specifications).

> The following describes the process for PDF document batches. The process is
> similar for HTML batches.


![](img/render-batch.svg)

This is how a batch rendering is invoked:

```bash
# Long form arguments
docma pdf-batch --template my-template.zip \
    --file static-params.yaml \
    --data-source-spec 'postgres;pglocal;queries/batch.yaml' \
    --output 'whatever-{{id}}-{{familyname|lower}}.pdf'

# Short form arguments
docma pdf-batch -t my-template.zip \
    -f static-params.yaml \
    -d 'postgres;pglocal;queries/batch.yaml' \
    -o 'whatever-{{id}}-{{familyname|lower}}.pdf'
```

Let's examine this bit by bit.

The docma `pdf-batch` sub-command is invoked specifying the compiled document
template:

```bash
docma pdf-batch --template my-template.zip
```

Rendering parameters are specified exactly as for the single document rendering
process. These parameters are the same for every document in the rendering batch:

```bash
    --file static-params.yaml \
```

The [docma data source specification](07-data-sources-in-docma.md#data-source-specifications) tells docma
how to obtain rows of data to control the batch rendering. Each row is a set of
key/value pairs that will be merged into the static rendering parameters and
used to render one PDF document:

```bash
    --data-source-spec 'postgres;pglocal;queries/batch.yaml' \
```

> The [docma data source specification](07-data-sources-in-docma.md#data-source-specifications)
> is interpreted within the context of the document template.

As docma will be producing a series of PDF documents, it needs a mechanism to
provide each document with a unique name that corresponds to the batch data
entry that was used to produce it. This is done using the `--output` option with
an argument that is Jinja rendered to construct the filename. In this example,
it is assumed that the batch data contains `id` and `familyname` elements and
that these are a unique combination to avoid filename clashes:

```bash
    --output 'whatever-{{id}}-{{familyname|lower}}.pdf'
```

> There are some strict constraints on the filename rendering process for safety
reasons.



