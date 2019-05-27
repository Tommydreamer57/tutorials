
# How Functions and Composite Types Changed the Way I Use PostgreSQL

Having learned learned just enough SQL to create some tables and perform the most simple CRUD operations during my time at a coding bootcamp, I was blown away when I learned about composite types in Postgres.

I originally learned that an SQL query always returns a list, so if you want to return a JSON-like structure from your database, you either have to perform multiple SQL queries and stitch the results together in Node (or whatever other language you use), or you have to use something like GraphQL, that automatically generates the SQL you need and structures it in the format you specify.

However, using composite types (a feature that only exists in Postgres, if I am not mistaken), you can not only return a JSON-like structure from a query, but you can also accept JSON-like data as an input to an operation.

Please note that I'm not talking about STORING data in a composite type (which has its own much more limited use-case), I'm rather talking about using composite types as INPUTS and OUTPUTS of a query.

## Sample Schema

My go-to example-schema is one that represents classes in a school.

Here's a simple schema to capture classes, students, teachers, and enrollments:
(Note how students and teachers are both stored in the table "people")

### Tables

```SQL
CREATE TABLE people (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50)
);

CREATE TABLE classes (
    id SERIAL PRIMARY KEY,
    subject VARCHAR(50),
    teacher_id INTEGER REFERENCES people
);

CREATE TABLE student_class_enrollments (
    student_id INTEGER REFERENCES people,
    cid INTEGER REFERENCES classes,
    UNIQUE (student_id, cid)
);
```

### Sample Data

Let's create a class just so we have something to query:

```SQL
-- create a teacher and some students
INSERT INTO people (
    first_name,
    last_name
)
VALUES
('Tommy', 'Lowry'), -- id 1
('Student', 'One'), -- id 2
('Student', 'Two'), -- id 3
('Student', 'Three'); -- id 4

-- create a class
INSERT INTO classes (
    subject,
    teacher_id
)
VALUES
('Postgres Composite Types', 1);

-- enroll students in the class
INSERT INTO student_class_enrollments (
    student_id,
    cid
)
VALUES
(2, 1),
(3, 1),
(4, 1);
```

## Querying the Old Way

Now if I wanted to retreive all the information about a class, I formerly would have written a query something like this:

```SQL
SELECT * FROM classes c
LEFT JOIN people t ON t.id = c.teacher_id
LEFT JOIN student_class_enrollments e ON e.cid = c.id
LEFT JOIN people s ON s.id = e.student_id
```

This would give me data something like:

| class id | subject | teacher id | teacher first name | teacher last name | student id | student first name | student last name |
|----------|---------|------------|--------------------|-------------------|------------|--------------------|-------------------|
| 1        | Post... | 1          | Thomas             | Lowry             | 1          | Student            | One               |
| 1        | Post... | 1          | Thomas             | Lowry             | 2          | Student            | Two               |
| 1        | Post... | 1          | Thomas             | Lowry             | 3          | Student            | Three             |

See how we have duplication of data? Not only are the class and teacher represented once for each student, but our list lacks any representation of our data's depth - class, teacher, and student are all lumped together in each node of the list instead of being organized into a heirarchy.

## Composite Types

Let's see how composite types can help us to format this data in a more reasonable structure.

**Note**

If you're not familiar with composite types, a composite type is a `type` that is 'composed' of other types. Composite types can be composed of built-in SQL types as well as other composite types. You can read more about them in PostgreSQL's [official documentation of composite types](https://www.postgresql.org/docs/10/rowtypes.html).

To show you how this works, let's create some composite types that mirror our data structure:

```SQL
CREATE TYPE person AS (
    id INTEGER,
    first_name VARCHAR(50),
    last_name VARCHAR(50)
);

CREATE TYPE class AS (
    id INTEGER,
    subject VARCHAR(50),
    teacher PERSON,
    students PERSON[]
);
```

Notice how this structure somewhat resembles our tables, with a few key differences:

1. In place of a one-to-many relationship (a single foreign key), we simply nest a teacher within a class.
2. In place of a many-to-many relationship (a join table), we simply nest an array of students within a class.

## Function Signature

Now we're ready to create a function that will output an entire class in the structure that we've defined using our composite types.

There are a few steps to this, so let's start with a function signature that takes in a class id and returns a class:

```SQL
CREATE OR REPLACE FUNCTION read_entire_class(cid INTEGER)
RETURNS class AS $$
DECLARE
BEGIN
END;
$$ LANGUAGE plpgsql;
```

## Variables

Now we'll need to define the variables we will use in order to retreive each key/node to build our composite class:
(This'll make more sense once we actually use them)

```SQL
CREATE OR REPLACE FUNCTION read_entire_class(cid INTEGER)
RETURNS class AS $$
DECLARE
    subject TEXT;
    teacher PERSON;
    students PERSON[];
BEGIN
END;
$$ LANGUAGE plpgsql;
```

## Constructing a Composite Value / Explicit Type Casting

Next, before we get into the difficult stuff, let's add our return statement. Here we'll use the `ROW()` function, along with explicit type casting (`::class`) to put our class together and return it. This might seem rather abstract at first, but under the hood, that's all a composite type is: a single row of field names and corresponding data types.

```SQL
CREATE OR REPLACE FUNCTION read_entire_class(cid INTEGER)
RETURNS class AS $$
DECLARE
    subject TEXT;
    teacher PERSON;
    students PERSON[];
BEGIN

    RETURN ROW(     -- ROW() function to create composite
        id,         -- these must be in correct order
        subject,
        teacher,
        students
    )::class;       -- explicit type casting
END;
$$ LANGUAGE plpgsql;
```

Note that the order of the values used is important and must always be the same as in the declaration of the type. If you put them in the wrong order and the types don't match up, it'll throw an error when you run the function. Worse, if you put them in the wrong order and the types are compatible, you'll be silently mismatching keys in your query until you recognize that something's off.

According to the docs, you actually can leave the `ROW` keyword out, as long as the composite type has more than one field. To do this, you remove `ROW` and leave the parentheses. You may also be able to leave out the explicit type-cast, since SQL will automatically cast the RETURN into the function's declared return type.

Now you should be able to create the function and run it - however you'll get an empty class every time.

## Filling the Composite Value With Query Results

All that's left now is to fill the data structure with query results from the database.

We'll take this one step at a time, assuming minimal knowledge of the language.

### Subject

First, let's get the subject:

```SQL
CREATE OR REPLACE FUNCTION read_entire_class(cid INTEGER)
RETURNS class AS $$
DECLARE
    subject TEXT;
    teacher PERSON;
    students PERSON[];
BEGIN

    SELECT c.subject FROM classes c   -- selects `subject` from classes
    INTO subject                      -- assigns query result to variable `subject`
    WHERE c.id = cid;

    RETURN ROW(
        cid,
        subject,
        teacher,
        students
    )::class;
END;
$$ LANGUAGE plpgsql;
```

`SELECT ... INTO` will assign the result of the query to whatever variable is specified, so that we can reference the result later on.

### Teacher

Now, let's get the teacher:

```SQL
CREATE OR REPLACE FUNCTION read_entire_class(cid INTEGER)
RETURNS class AS $$
DECLARE
    subject TEXT;
    tid INTEGER; -- add teacher id variable
    teacher PERSON;
    students PERSON[];
BEGIN

    SELECT c.subject FROM classes c
    INTO subject
    WHERE c.id = cid;

    -- EITHER --------
    SELECT t.id, t.first_name, t.last_name FROM people t
    INTO teacher
    WHERE t.id IN (                             -- subquery to select correct teacher
        SELECT c.teacher_id FROM classes c
        WHERE c.id = cid
    );
    -- OR --------
    SELECT c.teacher_id FROM classes c          -- extra query to get `teacher_id` instead of using subquery
    INTO tid
    WHERE c.id = cid;

    SELECT t.id, t.first_name, t.last_name FROM people t
    INTO teacher
    WHERE t.id = tid;
    -- END OR ----

    RETURN ROW(
        cid,
        subject,
        teacher,
        students
    )::class;
END;
$$ LANGUAGE plpgsql;
```

There's not much new here, just a potential subquery to get the teacher of the class. I'll keep the subquery since it's more concise.

### Students

Now we just need to get the students. This one is quite a bit more difficult, since we're dealing with a list of students. Here's how you do it:

```SQL
CREATE OR REPLACE FUNCTION read_entire_class(cid INTEGER)
RETURNS class AS $$
DECLARE
    subject TEXT;
    teacher PERSON;
    students PERSON[];
BEGIN

    SELECT c.subject FROM classes c
    INTO subject
    WHERE c.id = cid;

    SELECT t.id, t.first_name, t.last_name FROM people t
    INTO teacher
    WHERE t.id IN (
        SELECT c.teacher_id FROM classes c
        WHERE c.id = cid
    );

    SELECT ARRAY_AGG(ROW(s.id, s.first_name, s.last_name)::person) FROM people s
    INTO students
    WHERE s.id IN (
        SELECT e.student_id FROM student_class_enrollments e
        WHERE e.class_id = cid
    );

    RETURN ROW(
        cid,
        subject,
        teacher,
        students
    )::class;
END;
$$ LANGUAGE plpgsql;
```

`ARRAY_AGG()` takes the results of the query and converts them into an array, so they are compatible with the array type of the students variable. `ROW(...)::person` converts each student into a composite person.

## Done!

Now we have a function that requires nothing more than a class id to bundle up a class with its teacher and students into a nice, nested, JSON-like structure.

In order to invoke the function, run `SELECT * FROM read_entire_class(1)` and look at the beautifully nested results!
