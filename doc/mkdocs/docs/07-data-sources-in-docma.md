# Data Sources in Docma

Docma can access data files and live data sources during the rendering phase.
This is done by the *data provider* subsystem.
The returned data can be used in the following ways:

*   Source data for [charts and graphs](#data-source-specifications-for-charts).

*   Injection into the Jinja rendering process for
    [HTML content](#data-source-specifications-for-html-rendering) (e.g. for
    tables or other variable content).

Data providers return their data as a list of objects, one row of data per
object.

> Be careful with dataset sizes. This interface is not designed for very large
> amounts of data. Do as much data preparation / reduction outside of docma as
> possible (e.g. via database queries to generate just the essential data).

## Data Source Specifications

Docma uses the concept of **data source specifications** to control the process
of obtaining the data and what to do with it. They contain the following
components.

| Component | Description |
|-|-----------------------|
| type | The [data provider type](#data-provider-types) (e.g. `file` if the data comes from a file). This controls the connection / access mechanism.|
| location | Where to find the data. For a file based source it would be the path to the file. For a database provider, it would point to the connection information for the database.|
| query | The file name in the document template containing a [query specification](#query-specifications) that defines a query to execute on the data provider. This is required for database-like sources. It is not used for some data provider types. |
| target | For charts, the position in the Vega-Lite specification where the data will be attached. This is a dot separated dictionary key sequence pointing into the chart specification. If not provided, this defaults to `data.values`, which is the primary data location for a Vega-Lite specification. |

### Data Source Specifications for Charts

The HTML to include a chart is of the form:

```html
<IMG src="docma:vega?spec=charts/my-chart.yaml&data=...">
```

The value of the `data` parameter is a docma
[data source specification](#data-source-specifications)
expressed in string form, like so:

```bare
data=type;location[;query[;target]
```
> Why jam all this together into a single URL parameter rather than have
> separate parameters for each component? The reason is
> because a Vega-Lite chart can have multiple data sources and hence multiple
> instances of the `data` parameter.

It can be fiddly combining all these components in a URL in a readable way. The
recommended approach is to use Jinja to assemble all the pieces and handle the
gory details of URL encoding, like so:

```html
<IMG style="width: 10cm;" src=docma:vega?{{
  (
    ( 'spec', 'charts/my-chart.yaml' ),
    ( 'data', 'file;data/my-data.csv' ),
    ( 'data', 'file;data/more-data.csv;;datasets.more_data' ),
    (
      'data', (
        'postgres', 'pgdb01', 'queries/usage-by-day.yaml', 'datasets.usage_by_day'
      ) | join(';'),
    ),
  ) | urlencode
}}">
```

In this example, three data sets are specified:

1.  The first one is extracted from a local CSV file and attached to the chart
    specification at the default location of `data.values` (i.e the `values`
    object under the `data` object is replaced with our CSV data).

2.  The second one is extracted from a local CSV file and attached to the chart
    specification as the `datasets.more_data` object in the specification. Note
    that the unused `query` component must be provided as an empty string to ensure the
    `target` component is correctly placed.

2.  The third one is extracted by running a query against a Postgres database
    and attached to the chart specification as the `datasets.usage_by_day`
    object in the specification.

### Data Source Specifications for HTML Rendering

A data source specification can be invoked directly in Jinja content within a
document that is to be rendered. 

This is done using the `docma.data()` function provided in the [run-time
rendering parameters](05-docma-template-rendering.md#jinja-rendering-parameters-provided-by-docma). It accepts
three arguments corresponding to the first three components of a [data source
specification](#data-source-specifications):

*   type 
*   location
*   query (optional).

The `docma.data()` function also accepts an optional `params` argument which is
a dictionary of additional parameter values that will be merged into the Jinja
rendering parameters when rendering the
[query specification](#query-specifications).

See also [Jinja Rendering Parameters Provided by
Docma](05-docma-template-rendering.md#jinja-rendering-parameters-provided-by-docma).

For example, the following document content invokes the
[postgres](#data-provider-type-postgres) data provider to run a query on the
Custard Appreciation Society membership records and present the data in a table.

```html
<TABLE>
  <THEAD>
  <TR>
    <TH>Custard Type</TH>
    <TH>Bid Price</TH>
  </TR>
  </THEAD>
  <TBODY>
  {% for row in docma.data('postgres', 'pgdb01', 'queries/custard-price.yaml') %}
    <TR>
      <TD>{{ row.favouritecustard }}</TD>
      <TD>{{ row.price | dollars(2) }}</TD>
    </TR>
  {% endfor %}
  </TBODY>
</TABLE>
</BODY>
</HTML>
```

If a query only returns a single row, that would be referenced like so:

```jinja
{{ docma.data(...)[0] }}
```

... or ...

```jinja
{{ docma.data(...) | first }}
```


## Query Specifications

Some data providers require a query to be specified to extract the data. In
docma, this is done using a **query specification**.

A query specification is a YAML formatted file in the document template. It is
referenced as the third element of a
[data source specification](#data-source-specifications). These should be placed
in the `queries` directory in the template.

Docma thus externalises all database queries into a single, visible collection
rather than embedding them in random places within the template document
components.

A query specification file contains the DML of the query to be executed as well
as information on how to handle query parameters. It contains the following
keys:

| Key | Type |Required | Description|
|---|---|---|--------------------------|
| description | String | Yes | A description for human consumption. Not used by docma. |
| options | Object | No | Query control options.|
|--> fold_headers | Boolean | No | Convert all headers to lowercase (prior to row validation). This is sometimes necessary as different database drivers can handle the case treatment of headers in different ways. The default is `false`.|
|--> row_limit | Integer | No | Abort if the query returns more than the specified number of rows. This is a safety mechanism. The default is 0, meaning no limit is applied. |
| parameters | List | No | A list of [query parameter specification](#query-parameters) objects. If empty, the query has no parameters (which would be unusual). The parameter values are Jinja rendered using the run-time rendering parameters. |
| query | String | Yes | The [query text](#query-text). This will be Jinja rendered using the run-time rendering parameters. |
| schema | Object | No | A JSON Schema specification for each row of data returned by the query. See [Query Schemas](#query-schemas) below. |

Here's a sample:

```yaml
description: Extract Custard Appreciation Society membership records

query: >-
  SELECT * FROM "{{ db.schema | sql_safe }}".custard
  WHERE favouritecustard=%s
    AND custardbidprice > %s
    AND custardjedi=%s;
  SORT BY surname;

parameters:
  - name: favouritecustard
    value: '{{ db.favouritecustard }}'
  - name: custardbidprice
    value: '{{ db.custardbidprice }}'
    type: decimal
  - name: custardjedi
    value: '{{ db.custardjedi }}'
    type: boolean

options:
  row_limit: 20
  fold_headers: true
```

### Query Text

The query itself is pretty vanilla (ha!) SQL with some notable exceptions.

**Firstly,** the query text will be Jinja rendered with the docma run-time rendering
parameters. This makes it easy to do things such as switching schemas without
having to alter the document template. (This is close to impossible in some
popular analytics platforms that shall remain nameless.)

> It is probably best not to embed data source specification references within a
> query specification (i.e. recursive calls to the data provider subsystem).
> Don't cross the streams.

Care is required to avoid SQL injection risks. In the example above, the schema
is quoted and also filtered using the docma specific
[sql_safe](05-docma-template-rendering.md#custom-jinja-filters-provided-by-docma) Jinja filter. This filter
will abort if the value contains something unsafe.

**Secondly,** the values for query parameters are replaced with placeholders. The
actual values are determined at run-time from the parameter specifications. The
placeholder is database driver specific unfortunately, based on the
[paramstyle](https://peps.python.org/pep-0249/#paramstyle)
it uses. The `pg8000` driver used for Postgres uses `%s` style, whereas DuckDB
uses `?`. It's not my fault.

> **DO NOT** attempt to use Jinja to format query parameters into the SQL text
> itself. This is seriously unsafe. Use
> [query parameter specifications](#query-parameters).

**Thirdly,** when the data is to be used in a [Vega-Lite
chart](06-charts-and-graphs-in-docma.md#charts-and-graphs-in-docma), all of the data returned by the query needs
to be JSON serialisable. i.e.  Python types such as `datetime`, `Decimal` etc
will be a problem. The query should type cast everything to types that can be
JSON serialised. For example:

```sql
-- This will not work ...
SELECT date_of_birth as dob, height_in_cm as height
FROM people;

-- This will work (Postgres syntax) ...
SELECT date_of_birth::text as dob, height_in_cm::float as height
FROM people;
```

### Query Parameters

Query parameters are specified as a list of query parameter specification objects.
These contain the following keys:

| Key | Type |Required | Description |
|---|---|---|-------------------------------------------------------------------|
|name|String|Yes| The parameter name. This is used for database drivers that support `named` and `pyformat` [paramstyles](https://peps.python.org/pep-0249/#paramstyle). It is mandatory for all parameters for maintainability. |
|value|String|Yes| The parameter value. In many cases this will be a Jinja value injection construct.|
|type|String|No| A type indicator. Docma uses this to cast the value to the specified type. Only the following are supported: `str` / `string`, `int` / `integer`, `float`, `decimal`, `bool` / `boolean`. The default is `string`. Alternatively, cast string values within the DML. |

These are supplied to the query at run-time using the DBAPI 2.0 driver's
query parameter mechanism to avoid SQL injection risks.

### Query Schemas

In some situations, it may be important to validate that the data returned by a
query meets certain conditions and to abort document production if it does
not. This can be achieved by including a `schema` object in the [query 
specification](#query-specifications).

> The data preparation process should take proper care to ensure valid data.
> Query Schemas are the last line of defence against bad data appearing in
> documents.

The `schema` object is a JSON Schema specification that is used to validate each
row of data. Validation failures will abort the process. Note that all rows are
returned as objects with keys based on the column names in the query.

For example, consider the following query specification:

```yaml
description: Get company information.

query: >-
  SELECT name, age_in_years, abn
  FROM companies;

# This schema will validate each row. We don't have to
# validate every attribute in a row. Just the ones we're
# worried about.
schema:
  type: object  # It's always one object per row of data
  properties:
    age_in_years:
      type: number
      minimum: 0
      maximum: 200
    abn:
      type: string
      format: au.ABN  # This is a docma provided format checker
```

In addition to the standard format specifiers supported by JSON Schema, the
[format checkers provided by
docma](05-docma-template-rendering.md#format-checkers-provided-by-docma) are available for `string` objects.


## Data Provider Types

> The data provider interface is readily extensible to add new data sources.
> See [Data Providers](09-installation-and-usage.md#data-providers).


### Data Provider Type: duckdb

Docma can read data from a local file containing a [DuckDB](https://duckdb.org)
database.

The DuckDB data provider is a useful mechanism for handling data extracts with
docma. Docma, running on a DuckDB data extract can be quite fast, even with
moderately large datasets.

> Why DuckDB rather than, for example, SQLite? DuckDB has a much more complete
> SQL implementation than SQLite, and one which is much closer to Postgres. It
> also has a very powerful and flexible mechanism for accessing data from other
> sources (files in various formats, AWS S3 etc.). And it goes like the clappers.

The `type` component of the [data source specification](#data-sources-in-docma)
is `duckdb`.

The `location` component is the name of a local file (not a template file)
containing the database.

The `query` component is the name of a
[query specification](#query-specifications) file.

#### Examples

This example shows a DuckDB database being used to supply data to a chart (using Jinja tuple notation):

```html
<IMG
    style="width: 10cm;"
    src="docma:vega?{{ (
        ( 'spec', 'charts/dog-woof-power-chart.yaml' ),
        ( 'data', 'duckdb;/tmp/demo/dogs.db;queries/woof-power.yaml' ),
    ) | urlencode }}"
>
```

This example shows a DuckDB database being used to populate an HTML table:

```html
<TABLE>
  <THEAD>
  <TR>
    <TH>Dog</TH>
    <TH class="Woof">Woof</TH>
  </TR>
  </THEAD>
  <TBODY>
  {% for row in docma.data('duckdb', '/tmp/demo/dogs.db', 'queries/woof-power.yaml') %}
  <TR>
    <TD class="Dog">{{ row.Dog }}</TD>
    <TD class="Woof">{{ row.Woof }}</TD>
  </TR>
  {% endfor %}
  </TBODY>
</TABLE>
```


### Data Provider Type: file

Docma can read data from static files contained within the compiled document
template.


The `type` component of the [data source specification](#data-sources-in-docma)
is `file`.

The `location` component is the name of the file, relative to the root of the
template. Handling is determined based on the file suffix. The following formats
are supported:

|File Suffix| Description |
|-|--------------|
|csv|A CSV file with a header line. The excel dialect is assumed.|
|jsonl|A file containing one JSON formatted object per line.|

The `query` component is not used.

#### Examples

This example shows a CSV file being used to supply data to a chart (using Jinja tuple notation):

```html
<IMG
    style="width: 10cm; justify-self: start;"
    src="docma:vega?{{ (
          ( 'spec', 'charts/woof.yaml'),
          ( 'data', 'file;data/dogs.csv'),
        ) | urlencode }}"
>
```

This example shows a CSV file being used to populate an HTML table:

```html
<TABLE>
  <THEAD>
  <TR>
    <TH>Dog</TH>
    <TH class="Woof">Woof</TH>
  </TR>
  </THEAD>
  <TBODY>
  {% for row in docma.data('file', 'data/dogs.csv', 'queries/woof-power.yaml') %}
  <TR>
    <TD class="Dog">{{ row.Dog }}</TD>
    <TD class="Woof">{{ row.Woof }}</TD>
  </TR>
  {% endfor %}
  </TBODY>
</TABLE>
```

In the example above, the csv file (`data/dogs.csv`) might look something like this:

```bare
Dog,Woof
Bolliver,28
Rin Tin Tin,55
Fido,43
Kipper,91
Zoltan,81
Pluto,53
Scooby,24
Cerberus,87
Snoopy,52
```



### Data Provider Type: lava

If the lava package is [installed](09-installation-and-usage.md#installing-docma), docma can use the lava
connection subsystem to read data from a database. Lava provides support for
connecting to a range of database types, including Postgres, Redshift, SQL
Server, MySQL and Oracle. It also manages all of the connection details,
credentials etc.

The `type` component of the data source specification is `lava`.

The `location` component is a lava connection ID for a database.

The `query` component is the name of a
[query specification](#query-specifications) file.

The lava realm must be specified during rendering, either by setting the
`LAVA_REALM` environment variable, or via the `--realm` argument to the CLI.

> The [query text](#query-text) must use a
> [paramstyle](https://peps.python.org/pep-0249/#paramstyle) that matches the
> underlying driver being used by lava. Refer to the lava user guide for more
> information.

#### Examples

This example shows a lava database connector (ID=`redshift/prod`) being used to supply data to a chart (using Jinja tuple notation):

```html
<IMG
    style="width: 10cm;"
    src="docma:vega?{{ (
        ( 'spec', 'charts/dog-woof-power-chart.yaml' ),
        ( 'data', 'lava;redshift/prod;queries/woof-power.yaml' ),
    ) | urlencode }}"
>
```

This example shows a lava database connector (ID=`redshift/prod`) being used to populate an HTML table:

```html
<TABLE>
  <THEAD>
  <TR>
    <TH>Dog</TH>
    <TH class="Woof">Woof</TH>
  </TR>
  </THEAD>
  <TBODY>
  {% for row in docma.data('lava', 'redshift/prod', 'queries/woof-power.yaml') %}
  <TR>
    <TD class="Dog">{{ row.Dog }}</TD>
    <TD class="Woof">{{ row.Woof }}</TD>
  </TR>
  {% endfor %}
  </TBODY>
</TABLE>
```


### Data Provider Type: params

Docma can extract data from a list of objects in the rendering parameters.


The `type` component of the [data source specification](#data-sources-in-docma)
is `params`.

The `location`  component is a dot separated key sequence
to select a data list within the parameters. Each element of the list must be
an object (not a string).

The `query` component is not used.

#### Examples

Consider the following rendering parameters:

```yaml
param1: value1
param2: value2

data:
  custard:
    prices:
      - type: lumpy
        price: 1.53
      - type: baked
        price: 2.84
      - type: runny
        price: 3.50
```

A data source specification of `params;data.custard.prices` would return the
following data rows:

```json
{ "type": "lumpy", "price": 1.53 }
{ "type": "baked", "price": 2.84 }
{ "type": "runny", "price": 3.5 }
```

This example shows rendering parameters being used to supply data to a chart (using Jinja tuple notation):

```html
<IMG
    style="width: 10cm;"
    src="docma:vega?{{ (
        ( 'spec', 'charts/custard-prices-chart.yaml' ),
        ( 'data', 'params;data.custard.prices' ),
    ) | urlencode }}"
>
```

This example shows rendering parameters being used to populate an HTML table:

```html
<TABLE>
  <THEAD>
  <TR>
    <TH>Custard Type</TH>
    <TH class="price">Price</TH>
  </TR>
  </THEAD>
  <TBODY>
  {% for row in docma.data('params', 'data.custard.prices') %}
  <TR>
    <TD class="type">{{ row.type }}</TD>
    <TD class="price">{{ row.price }}</TD>
  </TR>
  {% endfor %}
  </TBODY>
</TABLE>
```


### Data Provider Type: postgres

Docma can read data from a Postgres database.

The `type` component of the [data source specification](#data-sources-in-docma)
is `postgres`.

The `location` component is an alpha-numeric label for the database. This is
used to determine connection details from environment variables or the contents
of a `.env` file.

If the `location` component is `xyz`, then docma will read the following values
from a `.env` file to connect to the database.

| Name | Description |
|-|--------------|
|XYZ\_USER|Database user name|
|XYZ\_PASSWORD|Password. Exactly one of `XYZ_PASSWORD` and `XYZ_PASSWORD_PARAM` must be specified.|
|XYZ\_PASSWORD\_PARAM|AWS SSM parameter containing the password.|
|XYZ\_HOST|Host name|
|XYZ\_PORT|Port number|
|XYZ\_DATABASE|Database name|
|XYZ\_SSL|A truthy value specifying if SSL should be enforced (default `no`)|

Values can also be overridden by environment variables with the same names as
above, prefixed with `DOCMA_`. e.g. `DOCMA_XYZ_USER`.

> Take care to ensure the `.env` file is excluded from any GIT repo. A redacted
> sample is provided in the `test` directory.

The `query` component is the name of a
[query specification](#query-specifications) file.

#### Examples

This example shows a Postgres database being used to supply data to a chart (using Jinja tuple notation):

```html
<IMG
    style="width: 10cm;"
    src="docma:vega?{{ (
        ( 'spec', 'charts/dog-woof-power-chart.yaml' ),
        ( 'data', 'postgres;prod01;queries/woof-power.yaml' ),
    ) | urlencode }}"
>
```

This example shows a Postgres database being used to populate an HTML table:

```html
<TABLE>
  <THEAD>
  <TR>
    <TH>Dog</TH>
    <TH class="Woof">Woof</TH>
  </TR>
  </THEAD>
  <TBODY>
  {% for row in docma.data('postgres', 'prod01', 'queries/woof-power.yaml') %}
  <TR>
    <TD class="Dog">{{ row.Dog }}</TD>
    <TD class="Woof">{{ row.Woof }}</TD>
  </TR>
  {% endfor %}
  </TBODY>
</TABLE>
```


