.. currentmodule:: psycopg

.. _static-typing:

Static Typing
=============

Psycopg source code is annotated according to :pep:`0484` type hints and is
checked using the current version of Mypy_ in ``--strict`` mode.

If your application is checked using Mypy too you can make use of Psycopg
types to validate the correct use of Psycopg objects and of the data returned
by the database.

.. _Mypy: http://mypy-lang.org/


Generic types
-------------

Psycopg `Connection` and `Cursor` objects are `~typing.Generic` objects and
support a `!Row` parameter which is the type of the records returned.

By default methods such as `Cursor.fetchall()` return normal tuples of unknown
size and content. As such, the `connect()` function returns an object of type
`!psycopg.Connection[Tuple[Any, ...]]` and `Connection.cursor()` returns an
object of type `!psycopg.Cursor[Tuple[Any, ...]]`. If you are writing generic
plumbing code it might be practical to use annotations such as
`!Connection[Any]` and `!Cursor[Any]`.

.. code:: python

   conn = psycopg.connect() # type is psycopg.Connection[Tuple[Any, ...]]

   cur = conn.cursor()      # type is psycopg.Cursor[Tuple[Any, ...]]

   rec = cur.fetchone()     # type is Optional[Tuple[Any, ...]]

   recs = cur.fetchall()    # type is List[Tuple[Any, ...]]


.. _row-factory-static:

Type of rows returned
---------------------

If you want to use connections and cursors returning your data as different
types, for instance as dictionaries, you can use the `!row_factory` argument
of the `~Connection.connect()` and the `~Connection.cursor()` method, which
will control what type of record is returned by the fetch methods of the
cursors and annotate the returned objects accordingly. See
:ref:`row-factories` for more details.

.. code:: python

   dconn = psycopg.connect(row_factory=dict_row)
   # dconn type is psycopg.Connection[Dict[str, Any]]

   dcur = conn.cursor(row_factory=dict_row)
   dcur = dconn.cursor()
   # dcur type is psycopg.Cursor[Dict[str, Any]] in both cases

   drec = dcur.fetchone()
   # drec type is Optional[Dict[str, Any]]


.. _example-pydantic:

Example: returning records as Pydantic models
---------------------------------------------

Using Pydantic_ it is possible to enforce static typing at runtime. Using a
Pydantic model factory the code can be checked statically using Mypy and
querying the database will raise an exception if the rows returned is not
compatible with the model.

.. _Pydantic: https://pydantic-docs.helpmanual.io/

The following example can be checked with ``mypy --strict`` without reporting
any issue. Pydantic will also raise a runtime error in case the
`!Person` is used with a query that returns incompatible data.

.. code:: python

    from datetime import date
    from typing import Optional

    import psycopg
    from psycopg.rows import class_row
    from pydantic import BaseModel

    class Person(BaseModel):
        id: int
        first_name: str
        last_name: str
        dob: Optional[date]

    def fetch_person(id: int) -> Person:
        with psycopg.connect() as conn:
            with conn.cursor(row_factory=class_row(Person)) as cur:
                cur.execute(
                    """
                    SELECT id, first_name, last_name, dob
                    FROM (VALUES
                        (1, 'John', 'Doe', '2000-01-01'::date),
                        (2, 'Jane', 'White', NULL)
                    ) AS data (id, first_name, last_name, dob)
                    WHERE id = %(id)s;
                    """,
                    {"id": id},
                )
                obj = cur.fetchone()

                # reveal_type(obj) would return 'Optional[Person]' here

                if not obj:
                    raise KeyError(f"person {id} not found")

                # reveal_type(obj) would return 'Person' here

                return obj

    for id in [1, 2]:
        p = fetch_person(id)
        if p.dob:
            print(f"{p.first_name} was born in {p.dob.year}")
        else:
            print(f"Who knows when {p.first_name} was born")
