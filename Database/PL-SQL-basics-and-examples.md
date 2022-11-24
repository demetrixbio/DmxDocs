[[_TOC_]]

## Overview
The design goals of PL/pgSQL (which is the Posgresql-specific version of PL/SQL) were to create a loadable procedural language that:
- can be used to create functions and trigger procedures
- adds control structures to the SQL language
- can perform complex computations
- inherits all user-defined types, functions, and operators
- can be defined to be trusted by the server
- is easy to use

## Advantages
The issue with regular SQL is that every SQL statement must be executed individually by the database server.
With PL/pgSQL you can **group** a block of computation and a series of queries inside the database server, thus having the power of a procedural language and the ease of use of SQL, but with considerable savings of client/server communication overhead:
- Extra round trips between client and server are eliminated
- Intermediate results that the client does not need do not have to be transferred between server and client
- Multiple rounds of query parsing can be avoided

## Basics
Functions written in PL/pgSQL are defined to the server by executing CREATE FUNCTION commands. Such a command would normally look like, say,

```
CREATE FUNCTION somefunc(integer, text) RETURNS integer
AS 'function body text'
LANGUAGE plpgsql;
```

PL/pgSQL is a block-structured language. The complete text of a function body must be a block. A block is defined as:

```
[ <<label>> ]
[ DECLARE
    declarations ]
BEGIN
    statements
END [ label ];
```

Label is only needed in cases when you are planning to reuse your block of code.

One of the use cases for 'DECLARE' part is being able to define some variables for the scope of your code block and use them in the statements after BEGIN. This can help to eliminate unwanted repeated joins or unnecessary repeated subqueries, which you probably would have to do in plain SQL to achieve your goals. 
Below is a simple example. This code is from a LIMS migration. It adds a new equipment instance with its parent instance and equipment type (ontology tag) specified.

```
DO $$
    DECLARE
        u_id INTEGER := (SELECT id_user from onto.namespace WHERE name = 'equipment_type');
        r_id INTEGER := (SELECT id from material.equipment WHERE name = 'root');        
    BEGIN
        INSERT INTO material.equipment (id_equipment_type, name, parent, id_user)
        VALUES (get_tag('LC Detectors', 'equipment_type'), 'Aviation', r_id, u_id);        

    END;
$$ LANGUAGE plpgsql;
```


And here is an example of PL/pgSQL functions that retrieve a component type tag id and a unit of measure tag id, respectively:

```
-- Get Component type
CREATE OR REPLACE FUNCTION get_ct(tag_name varchar(256)) RETURNS int AS $$
DECLARE p_id int;
BEGIN
    SELECT tag.id 
    FROM onto.tag 
    left join onto.namespace ON tag.id_namespace = namespace.id
    where tag.name = tag_name and namespace.name = 'component_type' INTO p_id;
    RETURN p_id;
END;
$$ LANGUAGE plpgsql;

-- Get Unit of measure
CREATE OR REPLACE FUNCTION get_uom(tag_name varchar(256)) RETURNS int AS $$
DECLARE p_id int;
BEGIN
    SELECT tag.id
    FROM onto.tag
             left join onto.namespace ON tag.id_namespace = namespace.id
    where tag.name = tag_name and namespace.name = 'unit_of_measure' INTO p_id;
    RETURN p_id;
END;
$$ LANGUAGE plpgsql;
```

These two functions can be further used the following way when adding new point structures into the DB:

```
DO $$
DECLARE
  ns_user_id INTEGER;
BEGIN   
    ns_user_id := (SELECT id_user from onto.namespace where name = 'component_type');    
    INSERT INTO measure.point_structure
        (name,component_types,component_units,id_user)
      VALUES       
        ('rpm',ARRAY[get_ct('spin')],ARRAY[get_uom('rpm')],ns_user_id),
        ('mNm',ARRAY[get_ct('torque')],ARRAY[get_uom('mNm')],ns_user_id)
      ON CONFLICT (name) DO NOTHING;
END $$;
```

If you would like to learn more and in depth, refer to the [official PL/pgSQL documentation](https://www.postgresql.org/docs/9.2/plpgsql-overview.html)