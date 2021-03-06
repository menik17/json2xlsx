=========
json2xlsx
=========
json2xlsx is a tool to generate MS-Excel Spreadsheets from JSON files.

Installation
------------
Code::

    $ cd some_temporary_dir
    $ git clone git://github.com/menik17/json2xlsx.git
    $ cd json2xlsx
    $ python setup.py build
    $ sudo python setup.py install

Note that you may encounter an error while installing pyparsing on which json2xlsx
depends. This is probably because pyparsing 1.x runs only on Python 2.x while
pyparsing 2.x runs only on Python 3.x. Currently, json2xlsx declares in the package
that pyparsing 1.x is required, which means that Python 3.x users must install
json2xlsx from GitHub with manual modificatin to setup.py. I do not use Python 3.x
often, so please let me know a workaround.

Simple Example
--------------
Let's begin with a smallest example::

    $ cat test.json
    {"name": "John", "age": 30, "sex": "male"}
    $ cat test.ts
    table { "name"; "age"; "sex"; }
    $ json2xlsx test.ts -j test.json -o output.xlsx

This will create an Excel Spreadsheet 'output.xlsx' that contains
a table like this:

+-----+-----+-----+
|name | age | sex |
+-----+-----+-----+
|John | 30  | male|
+-----+-----+-----+

Isn't it super-easy? Here, `test.ts` is a script that defines the table.
Let's call it *table script*.
`-j` option specifies an input JSON file. You can specify as many `-j`
as you wish. `-o` gives the name of the output file::

    $ cat test.json
    [{"name": "John", "age": 30,
      "sex": "male"},
     {"name": "Alice", "age": 18,
      "sex": "female"}
    ]

This would give the following result.

+-----+-----+------+
|name | age | sex  |
+-----+-----+------+
|John | 30  | male |
+-----+-----+------+
|Alice| 18  |female|
+-----+-----+------+

Another way is that you give `-l` option to specify that each line in the input
comprises a single JSON object. In this mode, each line must contain strictly
one JSON object::

    $ cat test.json
    {"name": "John", "age": 30, "sex": "male"}
    {"name": "Alice", "age": 18, "sex": "female"}
    $ json2xlsx test.ts -l -j test.json -o output.xlsx

This would give the same table as above.

Multiple Rows
-------------
If you would like to add more than one row, there are two ways to go.
The first one is that you can give an JSON array as input.

ad hoc Query Example
--------------------
When `-` is specified for input table script, the standard input is used.
`--open` is specified, the generated xlsx file is opened immediately.
Those two options are useful when you want to craete a xlsx file with
an ad hoc query like this::

    $ json2xlsx - -j test.json -o output.xlsx --open
    table { "name"; "age"; "sex"; }
    ^D
    (MS Excel pops up immediately)

Renaming Columns
----------------
Keys in a JSON file are often not appropriate for display use.
For example, you may want to use "Full Name (Family, Given)" instead of
a JSON key "name". You can use `as` modifiers to do this::

    table {
        "name" as "Full Name (Family, Given)";
    }

You can use `"\n"` to wrap in a cell::

    table {
        "name" as "Full Name\n(Family, Given)";
    }

Saving a Few Types
------------------
If a string literal does not contain any spaces, symbols or special characters,
the double quotations can be omitted. This table script::

    table { "name"; "age"; "sex"; }

is equivalent to::

    table { name; age; sex; }

Delimiter
---------
You can use `,` instead of `;`::

    table { name; age; sex; }

`,` and `;` are interchangable except for specifying coordinates.

Adding Title to Table
---------------------
You can put the table title between `table` and `{`::

    table "Employee" { name; age; sex; }

This will create a table like this.

+------------------+
|Employee          |
+-----+-----+------+
|name | age | sex  |
+-----+-----+------+
|John | 30  | male |
+-----+-----+------+
|Alice| 18  |female|
+-----+-----+------+

Adding Styles
-------------
You can add styles to columns::

  table "Analysis Summary" border thinbottom {
    "file_caption" as "Sample" width 20 align right;
    "nSeqs" as "# of \nscaffolds" align right halign center number "#,#";
    "Min" color "green" align right;
    "_1st_Qu" as "1st quantile" align right number "#,#";
  }

1. `width` specifies the width of the column. The unit is unknown (I do not know), so please refer to the openpyxl document for details (although even I have not yet found the answer there).
2. `align right`, `align center`, `align left` will align columns (without the heading) as specified.
3. `halign right`, `halign center`, `halign left` will align the heading columns as specified.
4. `color` specifies the color of the cell. See Color class in style.py of openpyxl for the complete list of the preset colors. Please let me know if you need hex-style colors (json2xlsx does not support it yet).
5. `number` gives the number style of the cell. This will be described in details later.
6. `border` adds a border to the cell. Currently, "thinbottom", "thickbottom" and "doublebottom" are the only available options. Please let me know if you find any use case in which you need others (Border class in style.py of openpyxl tells you what kinds of borders are available) and you would like to see it implemented.

Number Style
------------
The number style is presumably an internal string used in MS Excel.
Here are a couple of examples. See NumberFormat class in style.py
of openpyxl for other examples.

+---------------------+----------+-----------------------------------+
| Number Format Style | Example  | Description                       |
+---------------------+----------+-----------------------------------+
| `%`                 |  24%     | Percentage                        |
+---------------------+----------+-----------------------------------+
| `#,##`              | 123,456  | Insert ',' every 3 digits         |
+---------------------+----------+-----------------------------------+
| `0.000`             |  12.345  | Three digits after decimal point  |
+---------------------+----------+-----------------------------------+
| `@`                 |24        | Force text                        |
+---------------------+----------+-----------------------------------+
| `yyyy-mm-dd`        |2013/11/23| Date                              |
+---------------------+----------+-----------------------------------+
| `0.00+E00`          |1.23+E10  | Scientific notation               |
+---------------------+----------+-----------------------------------+

Grouping
--------
You can group multiple columns. An example table script is here::

    table {
        "name";
        group "personal info" {
            "age",
            "sex";
        }
    }

The generated table will look like this.

+-----+---------------+
|     | personal info |
|     +-------+-------+
|name | age   | sex   |
+-----+-------+-------+
|John | 30    | male  |
+-----+-------+-------+

Nesting is allowed.

Multiple Tables, Multiple Sheets
--------------------------------
You can create multiple tables in a sheet::

    # You can write comments here.
    namesheet "Employee List";
    table { "name", "age", "sex"; }
    # equivalent to "-l -j employee1.json" in the command line
    load "employee1.json" linebyline;
    # vskip adds specified number of blank rows.
    vskip 1;
    table { "company", "revenue"; }
    # You can add as many files.
    load "company1.json";
    load "company2.json";
    # Create a new sheet. The first sheet is implicitly created so we did not need it.
    newsheet;
    namesheet "Products";
    table { "product", "code", "price"; }
    load "product1.json";
    load "product2.json";
    # You can add "-o output.xlsx" in the command line, but here we specify it in the script.
    write "output.xlsx";

Adding a comment in a sheet
---------------------------
We often want to add a comment to a spreadsheet::

    table { "name", "age", "sex"; }
    load "employee1.json";
    legend 2, 0 "As of Apr. 2000";

`legend` command takes coordinates and a string, and writes the string in the cell.
The coordinates is a pair of two integers, *row, column*.
It originates at the cell right next to the top right of the table.
Below we show the coordinates.

+-----+---------------+-------+-------+
|     | personal info | (0,0) | (0,1) |
|     +-------+-------+-------+-------+
|name | age   | sex   | (1,0) | (1,1) |
+-----+-------+-------+-------+-------+
|John | 30    | male  | (2,0) | (2,1) |
+-----+-------+-------+-------+-------+

CSV Support
-----------
Comma Separated Values (CSV) is also supported.
Let's see an example::

    table { "name", "age", "sex"; }
    loadcsv "employee1.csv";

Here is the content of employee1.csv::

    "John","30","male"
    "Alice","18","female"

Note that the order of the column must be the same as the column definition in the table.
If you would like to reorder the columns, you can specify the column order::

    table { "sex", "age", "name"; }
    loadcsv "employee1.csv" 2,1,0;

You can use `-1` for a blank column::

    table { "sex", "blank", "name"; }
    loadcsv "employee1.csv" 2,-1,0;

When the first line of the input CSV file is a header, give `withheader`::

    table { "sex", "age", "name"; }
    loadcsv "employee1.csv" 2,1,0 withheader;

then you can skip the first line.

Miscellanous
------------
You can use non-ASCII characters. UTF-8 is the only supported coding.

Changelog
---------
2016/05/26 FIX: work with newer pyparsing/openpyxl packages.
2013/06/05 FIX: attributes did not show up when the table caption is specified.
2013/06/05 ADD: better document on cell styles.
2013/05/24 Initial upload to PyPI

Note
----
Suggestions and comments are welcome.

License
-------
Modified BSD License.

Author
------
Masahiro Kasahara

