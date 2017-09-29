# PL/JSON

**PL/JSON** provides packages and APIs for dealing with JSON formatted data within PL/SQL code.  
General information about JSON is available at http://www.json.org

## A demo of things you can do with PL/JSON
```
declare
  obj pljson;
  list pljson_list;
begin
  
  obj := pljson('
    {
      "a": null,
      "b": 12.243,
      "c": 2e-3,
      "d": [true, false, "abdc", [1,2,3]],
      "e": [3, {"e2":3}],
      "f": {
        "f2":true
      }
    }');
  obj.print;
  -- equivalent to print
  dbms_output.put_line(obj.to_char);
  
  -- print compact way
  obj.print(false);
  -- equivalent to print compact way
  dbms_output.put_line(obj.to_char(false));
  
  -- add to json object
  obj.put('g', 'a little string');
  -- remove from json object
  obj.remove('g');
  
  -- count of direct members in json object
  dbms_output.put_line(obj.count);
  
  -- test if an element exists
  if not obj.exist('json is good') then
    obj.put('json is good', 'Yes!');
    if obj.exist('json is good') then
      obj.print;
      dbms_output.put_line(':-)');
    end if;
  end if;
  
  -- you can build lists (arrays) too
  -- however notice that we have to use the 'to_json_value' function on json objects
  list := pljson_list(); --fresh list;
  list.append(pljson('{"lazy construction": true}').to_json_value);
  list.append(pljson_list('[1,2,3,4,5]'));
  list.print;
    -- empty list and nested lists are supported
  list := pljson_list('[1,2,3,[3, []]]');
  list.print;
  -- count of direct members in json list
  dbms_output.put_line(list.count);
  
  -- you can also put json object or json lists as values
  obj.put('nested json', pljson('{"lazy construction": true}'));
  obj.put('an array', pljson_list('[1,2,3,4,5]'));
  obj.print;
  
  -- support for dates
  obj.put('a date', pljson_ext.to_json_value(to_date('2017-10-21', 'YYYY-MM-DD')));
  -- and convert it back
  dbms_output.put_line(pljson_ext.to_date(obj.get('a date')));
  
  obj := pljson(
    '{
      "a" : true,
      "b" : [1,2,"3"],
      "c" : {
        "d" : [["array of array"], null, { "e": 7913 }]
      }
    }');
  
  -- get elements using a json path expression
  -- pljson supports a simple dot path expression and '[n]' for arrays
  -- it never raises an exception (null is returned instead)
  -- arrays are 1-indexed 
  -- the empty string as path returns the entire json object
  -- can 'get_string', 'get_number', etc.
  dbms_output.put_line(pljson_ext.get_number(obj, 'c.d[3].e'));
  
  -- all pljson_... objects are copies
  -- so modification in place is difficult
  -- but put with path can do it
  pljson_ext.put(obj, 'c.d[3].e', 123);
  obj.print;
  
  -- if you provide an invalid path then an error is raised
  -- you can, however, specify a path that doesn't exists but should be created
  -- arrays are 1-indexed.
  -- gaps will be filled with json null(s)
  obj := pljson();
  pljson_ext.put(obj, 'a[2].data.value[1][2].myarray', pljson_list('[1,2,3]'));
  obj.print;
  -- fill the holes
  pljson_ext.put(obj, 'a[1]', 'filler1');
  pljson_ext.put(obj, 'a[2].data.value[1][1]', 'filler2');
  obj.print;
  -- replace larger structures:
  pljson_ext.put(obj, 'a[2].data', 7913);
  obj.print;
  
  obj := pljson(
    '{
      "a" : true,
      "b" : [1,2,"3"],
      "c" : {
        "d" : [["array of array"], null, { "e": 7913 }]
      }
    }');
  obj.print;
  
  -- remove element
  pljson_ext.remove(obj, 'c.d[3].e');
  obj.print;
  -- remove array of array
  pljson_ext.remove(obj, 'c.d[1]');
  obj.print;
  -- remove null element
  pljson_ext.remove(obj, 'c.d[1]');
  obj.print;
  
  -- you can ignore check for duplicate keys
  obj := pljson();
  -- enables fast construction without checks for duplicate keys
  obj.check_duplicate(false);
  for i in 1 .. 10 loop
    obj.put('a'||i, i);
  end loop;
  obj.put('a'||5, 'tada');
  obj.print;
  obj.check_duplicate(true);
  -- fix possible duplicates but does not preserve order
  obj.remove_duplicates();
  obj.print;
  
  -- create json objects and lists from sql statements
  list := pljson_dyn.executeList('select * from tab');
  list.print;
  obj := pljson_dyn.executeObject('select * from tab');
  obj.print;
end;
/
```

### View json data as table (also works for json strings stored in table)
```
select * from table(pljson_table.json_table(
  '[
    { "id": 0, "displayname": "Back",  "qty": 5, "extras": [ { "xid": 1, "xtra": "extra_1" }, { "xid": 21, "xtra": "extra_21" } ] },
    { "id": 2, "displayname": "Front", "qty": 2, "extras": [ { "xid": 9, "xtra": "extra_9" }, { "xid": 90, "xtra": "extra_90" } ] },
    { "id": 3, "displayname": "Middle", "qty": 9, "extras": [ { "xid": 5, "xtra": "extra_5" }, { "xid": 20, "xtra": "extra_20" } ] }
  ]',
  pljson_varray('[*].id', '[*].displayname', '[*].qty', '[*].extras[*].xid', '[*].extras[*].xtra'),
  pljson_varray('id', 'displayname', 'qty', 'xid', 'xtra'),
  table_mode => 'nested'
));
```
returns

|ID | DISPLAYNAME|QTY|XID|XTRA|
|:---|:---|:---|:---|:---|
| 0	| Back	| 5	| 1	| extra_1 |
| 0	| Back	| 5	| 21|	extra_21|
| 2	| Front	| 2	| 9 |	extra_9 |
| 2	| Front	| 2	| 90|	extra_90|
| 3	| Middle| 9	| 5 |	extra_5 |
| 3 |	Middle| 9	| 20|	extra_20|

###### and many other (automatic support for Double numbers or Oracle numbers, base64 encode/decode, XML to json, etc.)

## Install

1.  Download the latest release -- https://github.com/pljson/pljson/archive/develop.zip
2.  Extract the zip file
3.  Use `sql*plus`, or something capable of running `sql*plus` scripts, to
    run the `install.sql` script
4.  To test the implementation, run the `/testsuite/testall.sql` script

**NOTICE:**

all pljson types and packages start with 'PLJSON'  
in earlier releases they started with 'JSON' but this conflicted with new native json support in Oracle 12c  
so they were renamed to start with PLJSON but in installation we create synonyms that start with JSON  
(e.g. JSON_LIST is synonym for PLJSON_LIST)

these synonyms can be dropped without affecting the software  
they are there only for backward compatibility with earlier versions of PLJSON

most of the examples use the old naming starting with 'JSON'  
these work with the synonyms but you are advised when you try the examples  
and in your code to use 'PLJSON_...' instead of 'JSON_...'

## Documentation

Documentation **will** be in **docs**.  
API documentation **will** be located at http://pljson.github.io/pljson/api.

Currently the documentation is not ready.  
We advise that you look at  
the examples within the **examples** folder  
or the tests within the **testsuite** folder  
which are very educating and serve as a kind of tutorial too.

## Project folders and files

**install.sql** install the pljson packages and types in your schema

**uininstall.sql** completely uninstall packages and types

**src**  source code in PL/SQL, it is accessed by the install and uninstall scripts

**examples** useful examples to learn how to use pljson

**testsuite** a set of testsuites to verify installation, just run **testall.sql**

**testsuite-utplsql** the same set of testsuites but utilizing the utplsql framework (which you must install separately), just run ut_testall.sql

**docs** the documentation (not ready yet)

site_src  source for the documentation

build-apidocs.sh, build-apidocs.bat, metalsmith.json, package.json are necessary for creating the documentation

## Project State (last updated 2017-09-29)

This is the 'develop' branch and the only active branch currently. You should install this.  
Currently all bugs found so far have been fixed.  
There have been a lot of speed improvements in many areas and a lot of new functionality introduced.

The project is active and maintained.  
Any new bug will be fixed as soon as possible.  
New feature requests are welcome and usually most are honored.

Only the documentation is missing in order to have a complete product.  
We hope that soon this gap will be addressed.  
Until then you can see the examples for guidance on how to get started and even progress further with the testsuite code.  
The api is easy and simple and you can get a quick view of it by looking at the package and type specifications.  
Don't be afraid to experiment and ask questions.

## Contributing

Please follow the [contributing guidelines](CONTRIBUTING.md) to submit fixes or new features.

## License

[MIT License](LICENSE)
