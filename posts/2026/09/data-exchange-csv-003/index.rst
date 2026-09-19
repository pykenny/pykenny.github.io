.. title: Data Exchange Format (1) - Working with CSV, Part 3
.. slug: data-exchange-csv-003
.. date: 2026-09-19T00:00:00.000+08:00
.. tags: DataEngineering,FileFormat
.. link:
.. description:
.. type: text

In the last article, we explained how CSV specification defines columns and eliminates ambiguity to correctly identify delimiters and quotes.
In this part, we will focus on how CSV defines row boundaries and cover how CSV parsers handles malformed data.

.. TEASER_END

****

=============================
Line Breaks as Row Boundaries
=============================

As we discussed earlier, a standard CSV row looks like a sentence:

    You'll find that a standard CSV row is written much like a sentence:
    commas separate individual clauses, and line break marks the end of a full statement.

In computer systems, line breaks are non-printable characters.
In text editors, they do not take visual space - just silently move subsequent content to the next line.

According to CSV specification, a line break is defined as a *Carriage Return* plus a *Line Feed*, or referred to as *CRLF*, or ``\r\n`` if you are familiar with common escape sequences that represents hidden symbols in programming languages.
Reason for using two separate characters dates back to mechanical operations of early typewriters and terminals.
In a terminal, a Carriage Return moves the cursor horizontally back to the left margin,
while a Line Feed moves the cursor down vertically by one line.
Using the two control symbols together completes current line, creates a new line directly below it, and sets the cursor to the beginning of the new line.

We covered how CSV removes symbol ambiguity by quoting values that contain delimiter characters and literal quotes, and line breaks are treated in exactly the same way:
if a field contains new lines, it must be enclosed with double-quotes.
Because of this, if your CSV dataset contains multiline content, it can become difficult to read as a row record may span across multiple lines:

So what looks like this in Excel:

+----------+----------------+------------+--------------+------------+-----------------------------------+
| order_id | customer_name  | order_date | total_amount | status     | notes                             |
+==========+================+============+==============+============+===================================+
| 1001     | Alice, Smith   | 2026-03-15 |     1,129.99 | Completed  | Customer requested                |
|          |                |            |              |            | "leave at the front door" option. |
+----------+----------------+------------+--------------+------------+-----------------------------------+
| 1002     | Bob, Jones     | 2026-03-16 |        45.50 | Processing | Marked as "priority               |
|          |                |            |              |            | shipping" by support.             |
+----------+----------------+------------+--------------+------------+-----------------------------------+
| 1003     | Charlie, Brown | 2026-03-16 |    27,400.00 | Shipped    | Standard Delivery                 |
+----------+----------------+------------+--------------+------------+-----------------------------------+
| 1004     | Diana, Prince  | 2026-03-17 |       210.75 | Completed  | Gift message:                     |
|          |                |            |              |            | "Happy Birthday!"                 |
+----------+----------------+------------+--------------+------------+-----------------------------------+

... will look like this in a text editor:

.. code-block::

    order_id,customer_name,order_date,total_amount,status,notes
    1001,"Alice, Smith",2026-03-15,"1,129.99",Completed,"Customer requested
    ""leave at front door"" option."
    1002,"Bob, Jones",2026-03-16,"45.50",Processing,"Marked as ""priority
    shipping"" by support."
    1003,"Charlie, Brown",2026-03-16,"27,400.00",Shipped,Standard delivery
    1004,"Diana, Prince",2026-03-17,"210.75",Completed,"Gift message:
    ""Happy Birthday!"""

The simplified example remains readable because it only has 4 rows, with each row having at most one line break within the context.
For larger datasets with lengthy text content fields, developers should rely on CSV readers or other data inspection tools instead of viewing raw text.


===============================
Newlines That All Look the Same
===============================

While standard requires using CRLF for row boundaries, line breaks inherently bring significant portability issues because definition of newline is platform-specific.
Generally speaking, Windows system uses CRLF, while macOS and Linux use LF.
When a CSV dataset is generated programically and strictly follows standard or a predefined format, parsing it should not be the problem.
But what if the dataset is typed in an editor or even manually modified later on? In these cases, the author might not notice such subtle differences between platforms.

This is a common issue in text processing, and a CSV parser that strictly enforces CRLF (or LF) line break definition might produce unexpected result.
Because of this, while Python's ``csv.writers`` interface provides ``lineterminator`` and ``dialect`` options to configure output format,
``csv.reader`` omits line terminator settings. Instead, it takes any unquoted CR, LF, or combination of the two as a valid line break.

We can prove this design choice by creating the following example:

.. code-block:: python

    from csv import DictReader

    csv_content = (
        "row_1,row_2,row_3,row_4\n"
        "1,2,3,4\r\n"
        "5,6,7,8\r"
        "9,10,11,12\n\r"
        "13,14,15,16\n\r\n\n\n\r\r"
        "17,18,19,20"
    )

    with open("./example_mixed_newlines.csv", "w", newline="") as writer:
        writer.write(csv_content)
    
    with open("./example_mixed_newlines.csv", "r", newline="") as text_reader:
        csv_reader = DictReader(text_reader)

        for row in csv_reader:
            print(row)
            print("===")

Look at the literal string that defines the CSV content, it is clear that each row uses different combination of characters for line separator.
This inconsistency can look more confusing in a text editor:

.. code-block::

    row_1,row_2,row_3,row_4
    1,2,3,4
    5,6,7,8
    9,10,11,12

    13,14,15,16





    17,18,19,20

However, Python's ``csv.reader`` deals with this and slices the records perfectly:

.. code-block::

    {'row_1': '1', 'row_2': '2', 'row_3': '3', 'row_4': '4'}
    ===
    {'row_1': '5', 'row_2': '6', 'row_3': '7', 'row_4': '8'}
    ===
    {'row_1': '9', 'row_2': '10', 'row_3': '11', 'row_4': '12'}
    ===
    {'row_1': '13', 'row_2': '14', 'row_3': '15', 'row_4': '16'}
    ===
    {'row_1': '17', 'row_2': '18', 'row_3': '19', 'row_4': '20'}
    ===


But keep in mind that other CSV tools may have different parser design and configurations,
and you should always proceed with caution it you find data source contains inconsistent line breaks.


=====================================
Python's Special Handling of Newlines
=====================================

You may have noticed that in our examples, ``newline`` parameter is explicitly set when opening the file:

.. code-block:: python

    from csv import DictReader

    with open("./example.csv", "r", newline="") as file_reader:
        csv_reader = DictReader(file_reader)
        # ...

This is due to Python's text I/O design. Setting ``newline=""`` disables Python's `uniersal newline translation <https://docs.python.org/3/glossary.html#term-universal-newlines>`_,
guaranteeing the original newline characters get preserved during
both `reading <https://docs.python.org/3/library/io.html#:~:text=If%20newline%20is%20%27%27%2C%20universal%20newlines%20mode%20is%20enabled%2C%20but%20line%20endings%20are%20returned%20to%20the%20caller%20untranslated.>`_
and `writing <https://docs.python.org/3/library/io.html#:~:text=If%20newline%20is%20%27%27%20or%20%27%5Cn%27%2C%20no%20translation%20takes%20place.>`_.
Because ``csv`` module has its own processing logic, applying this setting prevents file handler from conflicting with CSV processor.


================================
Handling Surrounding Whitespaces
================================

Just like this article, regular English writings often leave a space right after a comma.
Parsing an unquoted CSV dataset preserves these spaces:

.. code-block::
    
    order_id,customer_name,order_date,total_amount,status
     1001, Alice Smith, 2026-03-15, 129.99, Completed
    1002 ,Bob Jones ,2026-03-16 ,45.50 ,Processing 
     1003 , Charlie Brown , 2026-03-16 , 89.00 , Shipped 
    1004,Diana Prince,2026-03-17,210.75,Completed

.. code-block:: python

    {   'customer_name': ' Alice Smith',
        'order_date': ' 2026-03-15',
        'order_id': ' 1001',
        'status': ' Completed',
        'total_amount': ' 129.99'}
    {   'customer_name': 'Bob Jones ',
        'order_date': '2026-03-16 ',
        'order_id': '1002 ',
        'status': 'Processing ',
        'total_amount': '45.50 '}
    {   'customer_name': ' Charlie Brown ',
        'order_date': ' 2026-03-16 ',
        'order_id': ' 1003 ',
        'status': ' Shipped ',
        'total_amount': ' 89.00 '}
    {   'customer_name': 'Diana Prince',
        'order_date': '2026-03-17',
        'order_id': '1004',
        'status': 'Completed',
        'total_amount': '210.75'}

The looks fine and just requires string trim on the fields for value parsing.
But what happens if we try to parse a quoted CSV dataset with spaces surrounding the quotes?

.. code-block::

    order_id,customer_name,order_date,total_amount,status
     1001, "Alice, Smith", 2026-03-15, "1,129.99", Completed
    1002  ,"Bob, Jones" ,2026-03-16 ,"45.50" ,Processing 
     1003 , "Charlie, Brown" , 2026-03-16 , "27,400.00" , Shipped 
    1004,"Diana, Prince",2026-03-17,"210.75",Completed

.. code-block:: python

    {   None: ['129.99"', ' Completed'],
        'customer_name': ' "Alice',
        'order_date': ' Smith"',
        'order_id': ' 1001',
        'status': ' "1',
        'total_amount': ' 2026-03-15'}
    {   'customer_name': 'Bob, Jones ',
        'order_date': '2026-03-16 ',
        'order_id': '1002 ',
        'status': 'Processing ',
        'total_amount': '45.50 '}
    {   None: ['400.00" ', ' Shipped '],
        'customer_name': ' "Charlie',
        'order_date': ' Brown" ',
        'order_id': ' 1003 ',
        'status': ' "27',
        'total_amount': ' 2026-03-16 '}
    {   'customer_name': 'Diana, Prince',
        'order_date': '2026-03-17',
        'order_id': '1004',
        'status': 'Completed',
        'total_amount': '210.75'}

We will cover the ``None`` values shown in the sample output shortly.
Now notice that that parsed result is broken on fields where we deliberately added spaces around the quotes.
The CSV standard does not define how to handle these interleaving whitespaces (or other literal characters), and strictly speaking, there should be no spaces outside a quoted value.
As a result, Python's ``csv`` module has its own rules for handling these edge cases:

* If a field does not start immediately with a quoting character, the parser treats it as an unquoted value. This is why we got broken result for the first and third rows.
* If there are remaining characters (including spaces) following a quoted field, the parser attempts to concate them together.

In Python's ``csv`` module, ``skipinitialspace=True`` settings literally favors human-style formatting, ignores any leading whitespaces before parsing field values to find out presence of an opening quote.
However, we should understand this is used for handling very specific edge case that does not comply with CSV standard, and may fall short of fixing other structural corruptions in the dataset.


============================
Too Many, or Too Few Fields?
============================

In some cases, a corrupted dataset may mistakenly contains rows with more or fewer fields than specified in the schema.
The following example demonstrates one row with extra fields, and another one missing some data:

.. code-block::
    
    order_id,customer_name,order_date,total_amount,status
    1001,Alice Smith,2026-03-15,129.99,Completed,3.0,Standard Rate
    1003,Charlie Brown,89.00

Standard ``csv.reader`` and ``csv.writer`` do not care about this kind of misalignment.
They simply parse or export the fields without checking if column count matches the number of columns in underlying schema:

.. code-block:: python

    ['order_id', 'customer_name', 'order_date', 'total_amount', 'status']
    [   '1001',
        'Alice Smith',
        '2026-03-15',
        '129.99',
        'Completed',
        '3.0',
        'Standard Rate']
    ['1003', 'Charlie Brown', '89.00']

On the contrary, ``csv.DictReader`` and ``csv.DictWriter`` inherently keep predefined column list, allowing them to handle imcompatities between input and schema:

.. code-block:: python

    {   None: ['3.0', 'Standard Rate'],
        'customer_name': 'Alice Smith',
        'order_date': '2026-03-15',
        'order_id': '1001',
        'status': 'Completed',
        'total_amount': '129.99'}
    {   'customer_name': 'Charlie Brown',
        'order_date': '89.00',
        'order_id': '1003',
        'status': None,
        'total_amount': None}

Looking at the ``csv.DictReader`` output above, and we can notice that by default it uses ``None`` key to store extra field values, and sets missing column values to ``None``.
This behavior can be configured through ``restkey`` and ``restval`` parameters, respectively. Similarly, ``csv.DictWriter`` offers ``restval`` parameter to fill in missing column values, and uses ``extrasaction`` option to decide how to handle dictionary keys not defined in column names.

Once again, these options presents common irregularities found across different CSV data sources.
The need for such features emerges from manual data entry errors, poorly implemented serialization logic, or custom formatting rules that deviate from the original CSV design.
These factors makes CSV a simple and flexible, yet notoriously challenging format for data representation and exchange.
