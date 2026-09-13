.. title: Data Exchange Format (1) - Working with CSV, Part 1
.. slug: data-exchange-csv-001
.. date: 2026-09-13T00:00:00.000+08:00
.. tags: DataEngineering,FileFormat
.. link:
.. description:
.. type: text

CSV (**C**\ omma-**S**\ eparated **V**\ alues) files are common to anyone working with tabular data, no matter if those data are called "tables" or "spreadsheets" in conversations or your computer.
If you work in information technology industry, you've likely encountered them repeatedly in class assignments, personal projects, or data-centered tasks in the workplace, from fundamental data engineering to analytics applications.

.. TEASER_END

Because sample CSV files on public platforms usually follow standard formatting rules that are commonly assumed, popular libraries and commercial tools parse them seamlessly.
As the result, most tutorials in data science/analytics often make working with CSVs effortless, and hope the luck holds up throughout the rest of the tutorials, and, without exception, your subsequent career.

In the real world, however, that kind of luck often runs out.
Capricious input make your default settings fail, throwing unexpected errors, and you might even doubt something's corrupted ("Must be their fault!").
These edge cases requires careful handling - especially when you are the one who's responsible for writing up CSVs (through machine, programically).

This series will provide an overview on (perhaps "modern"?) CSV format, covering core configurations needed to process a CSV file accurately, along with reasons behind those settings.
To demonstrate these concepts, I will use Python's built-in ``csv`` module for the code examples. Even if your case relies on diffent languages or tools, the underlying concepts remains the same, and you will find equivalent settings in most of them.

****

===============================
Benefits and Limitations of CSV
===============================

Why use CSV instead of other formats like Excel Workbook or XML?

.. code-block::

    order_id,customer_name,order_date,total_amount,status
    1001,Alice Smith,2026-03-15,129.99,Completed
    1002,Bob Jones,2026-03-16,45.50,Processing
    1003,Charlie Brown,2026-03-16,89.00,Shipped
    1004,Diana Prince,2026-03-17,210.75,Completed

Its simplicity is clear from the example above: column cells are separated by commas (`,`), and row boundaries are defined by line breaks. Such design provides several advantages:

1. Because CSV is plain text, any text editor can open it, and human can also inspect values in the file easily, bringing both portability and readability.
2. Virtually all tools that work with tabular data supports importing and exporting CSV.
3. Both reading (file scanning) and writing (file appending) require minimum overhead. Data can be streamed row-by-row without holding complicated in-memory states.

However, the simplicity also comes with tradeoffs:

1. Compared to bytes representation, plain text representation consumes significantly more disk space and network bandwith (especially for numeric and date/time values).
2. Every cell is stored as raw text, and CSV itself doesn't store any data type information ("Which type of `order_id` column is it? Integer, or string?"). In order to ensure correctness, sender and receiver must agree on the same schema implicitly.
3. CSV can not represent hierarchical or nested data (arrays, records, sets, etc.) natively.


==========================
Header and Data Formatting
==========================
Even when processing files that follow standard convensions, most developers encounter challenge of converting text columns into appropriate data types.
This example code below shows how Python's native ``csv.reader`` function parses our example file:

.. code-block:: python

    from csv import reader

    with open("./example.csv", "r", newline="") as csv_file:
        csv_reader = reader(csv_file)  # Apply default parser settings
        
        for row in csv_reader:                           # For each row,
            for cell in row:                             # and for each column cell,
                print(f"{cell}, of type {type(cell)}.")  # print out stringified value and identify type
            
            print("===")

Output:

.. code-block::

    order_id, of type <class 'str'>
    customer_name, of type <class 'str'>
    order_date, of type <class 'str'>
    total_amount, of type <class 'str'>
    status, of type <class 'str'>
    ===
    1001, of type <class 'str'>
    Alice Smith, of type <class 'str'>
    2026-03-15, of type <class 'str'>
    129.99, of type <class 'str'>
    Completed, of type <class 'str'>
    ===
    1002, of type <class 'str'>
    Bob Jones, of type <class 'str'>
    ...

The output itself points out two common pitfalls when parsing CSV contents:

* Header row is not skipped and it is treated like regular data rows.
* All fields are parsed as strings (`str` objects in Python) type by default.

A header row usually appears at the top of CSV data to define number of columns and corresponding title by order.
Beacuse headers are very common, most tools and libraries provide options to skip header rows, while some even offer utilities using heuristics to detect presence of headers:

.. code-block:: python

    from csv import Sniffer

    with open("./example.csv", "r", newline="") as csv_file:
        snf = Sniffer()
        has_header = snf.has_header(csv_file.read(80))  # Detect header by sampling
        csv_file.seek(0)                                # Move pointer back to start of file

        csv_reader = reader(csv_file)
        
        if has_header:
            _ = next(csv_reader)  # Skip the first row when header row exists

        for row in csv_reader:
            for cell in row:
                print(f"{cell}, of type {type(cell)}.")
            
            print("===")

Output:

.. code-block::

    1001, of type <class 'str'>
    Alice Smith, of type <class 'str'>
    2026-03-15, of type <class 'str'>
    129.99, of type <class 'str'>
    Completed, of type <class 'str'>
    ===
    1002, of type <class 'str'>
    Bob Jones, of type <class 'str'>
    ...


Python offers ``csv.DictReader`` to simplify file ingestion process, and map data rows into dictionaries (Python `dict` objects):

.. code-block:: python

    from csv import DictReader

    with open("./example.csv", "r", newline="") as csv_file:
        csv_reader = DictReader(csv_file)

        for row in csv_reader:
            for key, val in row.items():
                print(f"[{key}] {val}, of type {type(val)}.")
    
            print("===")

Output:

.. code-block::

    [order_id] 1001, of type <class 'str'>.
    [customer_name] Alice Smith, of type <class 'str'>.
    [order_date] 2026-03-15, of type <class 'str'>.
    [total_amount] 129.99, of type <class 'str'>.
    [status] Completed, of type <class 'str'>.
    ===
    [order_id] 1002, of type <class 'str'>.
    [customer_name] Bob Jones, of type <class 'str'>.
    [order_date] 2026-03-16, of type <class 'str'>.
    [total_amount] 45.50, of type <class 'str'>.
    [status] Processing, of type <class 'str'>.
    ===
    ...


As we can see, DictReader offloads lots of boilerplate code for us.
It automatically extracts field names from the first row, skips the header, and pairs cell values with their corresponding column name in a dictionary.
If a CSV file lacks header row, you must pass column names explicitly using ``fieldnames`` parameter.

However, DictReader still does not parse data types automatically, and leaves all field values as plain text. If human can easily infer column types just by inspecting column value representations, why not just rely on smarter tools like Pandas?

.. code-block:: python

    from pandas import read_csv

    table = read_csv("./example.csv")
    table

.. code-block::

    order_id  customer_name  order_date  total_amount      status
    0      1001    Alice Smith  2026-03-15        129.99   Completed
    1      1002      Bob Jones  2026-03-16         45.50  Processing
    2      1003  Charlie Brown  2026-03-16         89.00     Shipped
    3      1004   Diana Prince  2026-03-17        210.75   Completed

At the first glance, the result looks great. Just like ``DictReader``, Pandas gets the header row and label the columns.
However, if we further look into DataFrame metadata:

.. code-block:: python

    table.info()

.. code-block::

    <class 'pandas.DataFrame'>
    RangeIndex: 4 entries, 0 to 3
    Data columns (total 5 columns):
    #   Column         Non-Null Count  Dtype
    ---  ------         --------------  -----
    0   order_id       4 non-null      int64
    1   customer_name  4 non-null      str
    2   order_date     4 non-null      str
    3   total_amount   4 non-null      float64
    4   status         4 non-null      str
    dtypes: float64(1), int64(1), str(3)
    memory usage: 292.0 bytes

We'll notice that While ``order_id`` and ``total_amounts`` are correctly parsed into numeric types, and descriptive fields remain strings, ``order_date`` column is not converted to datetime type as expected.
That does not mean we should abandon Pandas and look for alternative solutions right now. Pandas just need an extra hint to handle these columns automatically:

.. code-block:: python

    table = read_csv("./example.csv", parse_dates=["order_date"])
    table.info()

.. code-block::

    <class 'pandas.DataFrame'>
    RangeIndex: 4 entries, 0 to 3
    Data columns (total 5 columns):
    #   Column         Non-Null Count  Dtype
    ---  ------         --------------  -----
    0   order_id       4 non-null      int64
    1   customer_name  4 non-null      str
    2   order_date     4 non-null      datetime64[us]
    3   total_amount   4 non-null      float64
    4   status         4 non-null      str
    dtypes: datetime64[us](1), float64(1), int64(1), str(2)
    memory usage: 292.0 bytes

Now Pandas correctly converts ``order_date`` column into a proper time data type!
But it is important to understand this kind of automated process based highly relies on Pandas' internal parser to recognize date patterns.
While production datasets should follow `ISO standards <https://en.wikipedia.org/wiki/ISO_8601>`_ (which elimiate ambiguity, guarantee correct sorting, and preserve readability), real world data is full of exceptions. Developers frequently encounter non-standard formats using different unit separators, localized names, and special unit ordering (e.g. 31st Mar, 2023).
Finally, data type parsing depends on establishing serialization and derialization protocol between provider and receiver, which applies to all non-string data types.
