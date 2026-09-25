.. title: Data Exchange Format (1) - Working with CSV, Part 4
.. slug: data-exchange-csv-004
.. date: 2026-09-25T00:00:00.000+08:00
.. tags: DataEngineering,FileFormat
.. link:
.. description:
.. type: text

So far, we have covered standard CSV format, its variations, and common configurations in Python's ``csv`` module.
Now you'd understand what a standard CSV looks like, how the specification defines data boundaries and escapes special characters.
In this part, we will explore few advanced topics related to format variations and additional settings in Python's ``csv`` module.

.. TEASER_END

****

============================
Introducing Escape Character
============================

Though not defined in the original RFC specification, there is an alternative approach to tell special characters in field values without relying on double-quoting.
Similar to how we write string literals in programming languages, this approach defines an excape character to label special characters within field value.
A classic, well-known example of this is MySQL's data export utility (specifically the ``SELECT INTO OUTFILE`` statement).
Many database systems take a similar approach, using backslash ("``\``") as most programming lanagues do.

However, unlike C, Java, or Python, where special characters are represented by predefined sequences like ``"This is an apple.\nThis is an oragne.\nWe call them \"fruits\"."``,
this CSV variation simply prepends escape character right before the raw text:

.. code-block::

    order_id,customer_name,order_date,total_amount,status,notes
    1001,Alice\, Smith,2026-03-15,1\,129.99,Completed,Customer requested\
    "leave at front door" option.
    1002,Bob\, Jones,2026-03-16,45.50,Processing,Marked as "priority\
    shipping" by support.
    1003,Charlie\, Brown,2026-03-16,27\,400.00,Shipped,Standard delivery
    1004,Diana\, Prince,2026-03-17,210.75,Completed,Gift message:\
    "Happy Birthday!"

In the example above, we modified one of our previous quoted examples, replacing bouding quotes with backslash-escaped commas and line breaks.
To escape a backslash itself in field value, simply double it, just as how we escape quotation marks in standard CSVs (using two consecutive backslashes ``\\``).

When using Python's ``csv`` module, developers must disable quoting to switch escaping mechanism:

.. code-block:: python

    from pprint import pprint
    from csv import DictReader, QUOTE_NONE

    with open("./.locals/data-exchange-csv/example.csv", "r", newline="") as file_reader:
        csv_reader = DictReader(file_reader, escapechar="\\", quoting=QUOTE_NONE)
        #                                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        
        for row in csv_reader:
            pprint(row, indent=4)

.. code-block:: python

    {   'customer_name': 'Alice, Smith',
        'notes': 'Customer requested\n"leave at front door" option.',
        'order_date': '2026-03-15',
        'order_id': '1001',
        'status': 'Completed',
        'total_amount': '1,129.99'}
    {   'customer_name': 'Bob, Jones',
        'notes': 'Marked as "priority\nshipping" by support.',
        'order_date': '2026-03-16',
        'order_id': '1002',
        'status': 'Processing',
        'total_amount': '45.50'}
    ...

Disabling quoting introduces significant breaking change, making its ouput incompatible with standard CSV format.
Even if it is not the default behavior, most modern systems support standard CSV formatting, like using MySQL's ``ENCLOSED BY`` clause to enable double-quote escaping.
To reduce complexity, when using CSV as data exchange format, developers should consider sticking to an unified standard across data pipeline whenever possible.

================
Encoding and BOM
================

CSV standards do not specify text encoding, and simply assumes writer and reader are using the same codec.
Developers should be aware of this potential compatibility issue, particularlly when relying on `default text I/O settings <https://docs.python.org/3.11/library/functions.html#open:~:text=In%20text%20mode%2C%20if%20encoding%20is%20not%20specified%20the%20encoding%20used%20is%20platform%2Ddependent>`_
in the Python runtime. If it is impossible to determine text encoding from serialization protocol, you may need to use third-party solutions like `chardet <https://github.com/chardet/chardet>`_ to detect the underlying file encoding.

Another common issue with UTF encoding arises from attempts to open up CSV files in Microsoft Excel.
While Excel's data import feature allows user to select encoding codec,
opening UTF-8 encoded CSV directly requires `Byte Order Mark (BOM) <https://en.wikipedia.org/wiki/Byte_order_mark>`_ at the head of file.
This happens frequently when the source file is generated from a Unix-like operating system, where BOM is often omitted from UTF-8 encoded text files.
Because UTF-8 has no byte order variations (unlike UTF-16 and UTF-32, which strictly requires BOM to identify `endianness <https://en.wikipedia.org/wiki/Endianness>`_),
generally it is safe to decode content without header.

Although one could manually prepending BOM bytes, we can choose ``utf-8-sig`` encoding as a cleaner solution when reading and writing CSV files.
It guarantees BOM is included in the exported file, and ensures reader will skip BOM header properly if it is present.


===============================
Guessing and Detecting Dialects
===============================

Internally, Python's ``csv`` module offers three predefined dialects for common use cases.
These dialects are aliased and can be passed directly to reader's or writer's ``dialect`` parameter:

* ``csv.excel / "excel"``: The default setting. It follows Microsoft Excel's export format, which is fully compliant with RFC CSV standard.
* ``csv.excel_tab / "excel-tab"``: Similar to ``csv.excel``, but uses tab character (``"\t"``) as delimiter that deals with Tab-Separated Values (TSV) files.
* ``csv.unix_dialect / "unix"``: Follows UNIX system conventions: Using LF (``"\n"``) for line breaks, and double-quoting all field values.

As mentioned earlier, not all CSV datasets adhere to these common dialects.
Although manually viewing raw text helps quickly identify the general syntax, one would likely to automate this process when ingesting various data sources.
To solve this, Python provides ``csv.Sniffer`` class to offer format detection utility,
and popular libraries like `Pandas <https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html#:~:text=sep%3DNone%20detects%20the%20separator%20from%20the%20first%20valid%20row%20of%20the%20file%20with%20Python%E2%80%99s%20builtin%20sniffer%20tool%2C%20csv.Sniffer>`_
also use ``csv.Sniffer`` under the hood when speicial characters are not explicitly provided.

So, how does the sniffer work? Scanning the whole dataset to determine its own format is rarely a good idea, so typically this is done by looking for patterns from smaller file sample.
That's why ``Sniffer.sniff()`` method requires user to pass a string of sample text for examination.
How to sample from the file is totally up to developer, which heavily depends on file size, schema, and data properties.
We will not cover algorithm internals, but if you are interested, `here <https://github.com/python/cpython/blob/dbcae904c68b211b13b21b002628bbae27e66587/Lib/csv.py#L278>`_
are details about the pipeline that infers formats using regular expression pattern matching.

There are also other, and more sophisticated approaches based on real world case studies, such as `CleverCSV <https://clevercsv.readthedocs.io/en/latest/index.html>`_.
But again, we want to focus on practical usage instead of research papers. We will save it for an advanced topic in the future.

Column schema detection is another and even broader topic.
CleverCSV included, popular Python libraries that process tabular data such as Pandas, Polars, and PyArrow use different approaches for data type inference from raw strings.
Just as we discussed earlier, it usually relies on underlying serialization protocol, and frequently requires human inspection for accuracy.


==========================================
NULL Logic and Python Object Serialization
==========================================

While Python's ``csv.reader`` does not automatically deserialize field values, ``csv.writer`` must serialize
all non-string objects passed to it. Internally it pass these values to the built-in ``str()`` function to
convert them into string representations, which finally appear in the exported CSV fields.

However, there is one notable exception: nulls.

In Python, stringifying ``None``, either by calling ``str()`` or using string interpolation syntax (e.g. f-string), results in non-empty string "``None``".
While some protocols prefer other terms like "NULL", these conventions are highly system-specific.
Moreover, figuring out how to distinguish literal strings from actual null values introduces more headaches.

Both programming languages and database systems have dedicated symbols for undefined values, but CVS format itself does not define null data type.
How null values are represented and recognized highly depends on serialization protocol.

In most Python applications, and especially database client libraries implementing Python's database API (DB-API),
nulls are transformed into Python ``None`` objects.
Because of this, plus the fact that both ``None`` and empty string are "falsy" values in Python (evaluated to ``False`` when used in boolean expressions),
``csv.writer`` converts ``None`` to an empty string.
Other libraries may offer customizable behavior on processing null values, for instance, `reader <https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html#:~:text=optional%20Additional%20strings%20to%20recognize%20as%20NA/NaN.>`_ and `writer <https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_csv.html#:~:text=Missing%20data%20representation.>`_ functions in Pandas.
