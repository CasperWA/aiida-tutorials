Querying the AiiDA database {#sec:querybuilder}
===========================

TODO:

 * update table 1
 * add task: 
   rank structures by pore volume, surface, pore distribution, elements
 * add task: find good candidate materials for methane storage
     * pick 2 good ones, 2 bad ones and HKUST-1
     * put them inside a group "candidate_structures"
 * add task: consider blocked pockets for the selected candidates

We will now learn how to *query* the AiiDA database using the python interface.
Queries are, in essence, questions to your database and you will use the
answers you obtain in order to identify good candidate materials for methane
storage.

## The AiiDA QueryBuilder

AiiDA provides a querying tool called the *QueryBuilder*.
When working in the `verdi shell` or a jupyter notebook with `%aiida` enabled,
the `QueryBuilder` class has already been imported.

Let's create a new querybuilder for our query:
```python
qb = QueryBuilder()
```

Now we need to say which nodes of the AiiDA graph we are interested in.
All node types of the AiiDA graph derive from the `Node` class, so in order to select the most
generic type of node, simply append `Node`:

```python
qb.append(Node)
```

At this point, we can finish our query by asking for all nodes matching these criteria:

```python
qb.all() # Returns all nodes in the database
```

> **Note**  
> We could also have used `qb.first()` to get the first node.
> Always use tab-completion when in doubt which functions can be used on a python object.

This command will return us all the Nodes directly, which may
not be the most wise thing to do considering that is the biggest family
of AiiDA stored objects that we can query. 
To understand the size of the result, we can type the following command:

```python
qb.count() # Returns an integer, the number of nodes in the database
```

If you are interested to retrieve a subclass of a node, append that
specific subclass instead of Node:

```python
CifData = DataFactory('cif') 
qb = QueryBuilder() # Creating a new QueryBuilder instance 
qb.append(CifData) # Telling the QueryBuilder instance that I want a cif data type 
qb.all() # Asking for all the results!
```

> **Note**  
> Remember to create a new `qb = QueryBuilder()` instance for each new query.


Other data types we will need in the tutorial are:
```
ParameterData = DataFactory('parameter')
NetworkCalculation = CalculationFactory('zeopp.network')
RaspaCalculation = CalculationFactory('raspa')
```

---
### Exercise

Find the number of `CifData` and `NetworkCalculation` nodes in the database.  
Is there one calculation for every structure?

---

| Node & subclasses |  Number in DB
|-------------------| --------------
|       Node        |      4707
|   StructureData   |      621
|   ParameterData   |      1338
|    KpointsData    |      861
|      UpfData      |       99
|  JobCalculation   |      448

*List of some Node subclasses and how many times they occur in our test database.*

> **Note**  
> Advanced users who are familiar with SQL 
> (Structured Query Language) syntax, can inspect the generated SQL query via:
> ```python
> str(qb)
> ```
 
## Projections

So far, we've retrieved node objects but we are interested in the information they contain.
In database language, performing a projection means to extract one or
more specific columns from a table. In AiiDA language it means to
specify which properties of the selected nodes our query should return.

|---------| -------------------------|
|Entity   | Properties               |
|---------| -------------------------|
|Node     | id, uuid, type, label, description, ctime, mtime|
|Computer | id, uuid, name, hostname, description, enabled, transport_type, scheduler_type|
|User     | id, email, first_name, last_name, institution|
|Group    | id, uuid, name, type, time, description|
|---------| -------------------------|

*<center>A selection of entities and some of their properties.</center>*

For example, we might be interested only in type and universally unique
identifier (UUID) of each node:

```python
qb = QueryBuilder() 
qb.append(Node, project=["type", "uuid"]) 
qb.all()
```
> **Note**  
> Please use the `id` keyword in QueryBuilder queries to access the PK of a node.

## Filters

We already know how to filter by the type of node,
but often we would like to filter the results by other node properties.

For example, we might want to select all
the calculations that were launched on a specific date. In database
language, this is called "adding a filter" to a query. A filter is a
boolean operator that returns `True` or `False`. 

|  Operator   |         Datatype         |               Example
|-------------| -------------------------| ------------------------------------
|     ==      |            All           |              `{'==':12}`
|     in      |            All           |    `{'in':['FINISHED', 'PARSING']}`
| >,<,<=,>= |  floats, integers, dates |             `{'>':5.2}`
|    like     |           Chars          |       `{'like':'calculation%'}`
|    ilike    |           Chars          |       `{'ilike':'caLculAtioN%'}`
|     or      |                          |  `{'or':[{'<':5.3}, {'>':6.3}]}` 
|     and     |                          |  `{'and':[{'>':5.3}, {'<':6.3}]}`

*<center>Selection of filter operators.</center>*

If you want to add filters to your query, use the `filters`
and provide a dictionary with the filters you would like to apply.
Let's filter out the structure with label "HKUST1":

```python
qb = QueryBuilder()  # Instantiating a new QueryBuilder 
qb.append(CifData,   # I want structures! 
  project=["ctime"], # I'm interested in creation time! 
  filters={"label": {"==":"HKUST-1"}})  # Only structures with this label
qb.all()
```

There is also the possibility to combine multiple filters on
the same object using the "and" or the "or" keyword in the filter
section:

```python
from datetime import datetime, timedelta 
qb = QueryBuilder() 
qb.append(CifData, 
  project=["uuid"],   # I want to see only the UUID
  filters={ "or":[    # First filter is an or statement 
            { "ctime": {">":datetime.now() - timedelta(days=12) }},
            { "label": "Raspa test" } 
           ]}
) 
qb.all()
```

In the above example we added an "or" keyword between the two filters.
The query return every structure in the database that was created in the
last 12 days or is named "graphene".

**Hints for the exercises:**

-   The operator '>', '<' works with date-type properties with the
    expected behavior.

-   For your date comparisons you will need to create a `datetime`
    object to which you can assign a date of your preference. You will
    have to do the necessary import (`from datetime import datetime`)
    and create an object by giving a specific date. E.g. 
    `datetime(2015, 12, 26)`. For further information, you can consult the Python's
    online documentation.

**Exercises:**

-   Write a query that returns all instances of StructureData that have
    been created after the 1st of January 2016.

-   Write a query that returns all instances of Group whose name starts
    with "tutorial".

