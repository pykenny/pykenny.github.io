.. title: Data Exchange Format (1) - Working with CSV, Part 2
.. slug: data-exchange-csv-002
.. date: 2026-09-17T00:00:00.000+08:00
.. tags: DataEngineering,FileFormat
.. link:
.. description:
.. type: text

In previous article, we talked about how the CSV format represents tabular data, its simplicity, and its limitation on data types support.
For now, our example still looks similar to regular CSV tutorials because we worked with standard, simplified data.
We will dive into complex cases in future articles, and this post will focus on CSV structure: how the format accuratly defines table cells/columns.

.. TEASER_END

****

==========================================
Is Your CSV Really Comma-Separated Values?
==========================================

Let's review our sample data:

.. code-block::

    order_id,customer_name,order_date,total_amount,status
    1001,Alice Smith,2026-03-15,129.99,Completed
    1002,Bob Jones,2026-03-16,45.50,Processing
    1003,Charlie Brown,2026-03-16,89.00,Shipped
    1004,Diana Prince,2026-03-17,210.75,Completed

You'll find that a standard CSV row is written much like a sentence: commas separate individual clauses, and line break marks the end of a full statement.
Hence, developers are often tempted to build up a minimum CSV parser using basic string splitting:

.. code-block:: python

    def parse_csv(file: str, encoding: None | str = None) -> list[list[str]]:
        result = []

        with open(file, "r", encoding="utf-8") as file_reader:
            for line in file_reader:
               result.append(line.rstrip().split(","))  # Try remove linebreak and split substrings with comma character
    
        return result

However, this approach assumes fields are always separated by commas, and fails if a dataseet uses other symbol instead.
In CSV, the "comma" that separates data values and determines column boundaries is called *delimiter*.
While comma is the default delimiter in most applications, it can be any single character.
Different convensions may favor alternative characters:

* Microsoft Excel and Google Sheets support Tab-Separated Values (TSV) format, which uses tab character (``\t``) as delimiter.
* Database export pipelines frequently use pipe character (``|``) because it provides visual separator similar to vertical gridlines in spreadsheets.

In Python's ``csv`` module, delimiter can be customized using ``delimiter`` parameter on both reader and writer constructors:

.. code-block:: python

    from csv import DictReader

    with open("./example.csv", "r", newline="") as csv_file:
        csv_reader = DictReader(csv_file, delimiter="|")
        #                                 ^^^^^^^^^^^^^

        for row in csv_reader:
            ...  # Process rows

===================
Delimiter Collision
===================

Our example looks as clean as a poem. But what happens when descriptive fields contain some commas, or when our numeric values includes thousands separators?

.. code-block::

    order_id,customer_name,order_date,total_amount,status
    1001,Alice, Smith,2026-03-15,1,129.99,Completed
    1002,Bob, Jones,2026-03-16,45.50,Processing
    1003,Charlie, Brown,2026-03-16,27,400.00,Shipped
    1004,Diana, Prince,2026-03-17,210.75,Completed

If we process the above data using string-split logic in our ``parse_csv()`` implementation, we will end up with seven columns instead of five defined in header:

.. code-block:: python

    # Split result of the first row
    ['1001', 'Alice', ' Smith', '2026-03-15', '1', '129.99', 'Completed']

Because delimiters are characters as well, inevitably they will appear in field values. This scenario is known as `delimiter collision <https://en.wikipedia.org/wiki/Delimiter#Delimiter_collision>`_.
Intuitively, developers may try to avoid this by choosing unusual characters as delimiter, or even using non-printable characters (such as Bell Control ``\a``, which regular users rarely type).
However, merely betting on probability is highly discouraged in production.
As a standardized data format, CSV already provides solution so that developers do not have to rely on guesswork or luck.

===============
"Quoted" Values
===============

To distinguish delimiters from value content while preserving human readability, CSV standard adopts approach similar to how we define string literal in programming languages: enclosing fields containing special characters in quotes.
Standard convention uses double quotes (``""``) to wrap up a field value, and quotes are strictly required whenever a delimiter appears within the content:

.. code-block::

    order_id,customer_name,order_date,total_amount,status
    1001,"Alice, Smith",2026-03-15,"1,129.99",Completed
    1002,"Bob, Jones",2026-03-16,"45.50",Processing
    1003,"Charlie, Brown",2026-03-16,"27,400.00",Shipped
    1004,"Diana, Prince",2026-03-17,"210.75",Completed

Take a look at the corrected dataset above, Python's CSV parser can now separate the fields correctly, and we can easily identify column boundaries visually as well.
Now it looks even more like a poem with quoted verses. Just as delimiters, Python's ``csv`` module allows customizing quoting character with ``quotechar`` parameter:

.. code-block:: python

    from csv import DictReader

    with open("./example_quoted.csv", "r", newline="") as csv_file:
        csv_reader = DictReader(csv_file, quotechar='"')
        #                                 ^^^^^^^^^^^^^

        for row in csv_reader:
            for key, val in row.items():
                print(f"[{key}]: {val}")
            
            print("===")

.. code-block::

    [order_id]: 1001
    [customer_name]: Alice, Smith
    [order_date]: 2026-03-15
    [total_amount]: 1,129.99
    [status]: Completed
    ===
    [order_id]: 1002
    [customer_name]: Bob, Jones
    [order_date]: 2026-03-16
    [total_amount]: 45.50
    [status]: Processing
    ===
    ...


=======================
"""Quotes""" in Quotes"
=======================

These enclosing marks are called **double quotes** in the CSV standard.
But as we solve delimiter collision, this introduces a new problem: how do we include quote symbols (or other characters selected for quoting) inside of field value?

.. code-block::

    order_id,customer_name,order_date,total_amount,status,notes
    1001,"Alice, Smith",2026-03-15,"1,129.99",Completed,"Customer requested "leave at front door" option."
    1002,"Bob, Jones",2026-03-16,"45.50",Processing,"Marked as "priority shipping" by support."
    1003,"Charlie, Brown",2026-03-16,"27,400.00",Shipped,Standard delivery
    1004,"Diana, Prince",2026-03-17,"210.75",Completed,"Gift message: "Happy Birthday!""

Once again, we get collision issue on identifying quoting characters between value boundary and literal ones. And once again, we can't rely on luck by choosing another unusaul character to prevent collisions!
So how do CSV standards handle this?

The solution is intuitive: using preceding mark to represent a literal quote mark within a quoted field. That is to say, using two consecutive quoting characters instead (``""``).
Is this idea robust enough? Theoretically, under only two situations do consecutive quotes appear in a CSV field:

* A quoted empty string: ``""``. Because there are no characters between the quotes, and the field is surrounded by delimiters or line breaks, parser faces no ambiguity.
* Quote symbol within a quoted string (e.g. ``"""This is a book"""``). When following the standards, stripping outer quotes from the raw string leaves pairs of consecutive quotes. Parser can then translate each pair into a single literal quote.

Below is corrected version of our previous sample dataset, with literal quote symbols doubled to eliminate ambiguities:

.. code-block::

    order_id,customer_name,order_date,total_amount,status,notes
    1001,"Alice, Smith",2026-03-15,"1,129.99",Completed,"Customer requested ""leave at front door"" option."
    1002,"Bob, Jones",2026-03-16,"45.50",Processing,"Marked as ""priority shipping"" by support."
    1003,"Charlie, Brown",2026-03-16,"27,400.00",Shipped,Standard delivery
    1004,"Diana, Prince",2026-03-17,"210.75",Completed,"Gift message: ""Happy Birthday!"""

Running Python's CSV reader on this file using default configuration should give you the expected outcome:

.. code-block::

    [order_id]: 1001
    [customer_name]: Alice, Smith
    [order_date]: 2026-03-15
    [total_amount]: 1,129.99
    [status]: Completed
    [notes]: Customer requested "leave at front door" option.
    ===
    [order_id]: 1002
    [customer_name]: Bob, Jones
    [order_date]: 2026-03-16
    [total_amount]: 45.50
    [status]: Processing
    [notes]: Marked as "priority shipping" by support.
    ===
    ...
