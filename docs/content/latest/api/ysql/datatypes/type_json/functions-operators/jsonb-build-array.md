---
title: jsonb_build_array()
linkTitle: jsonb_build_array()
summary: jsonb_build_array() and json_build_array()
description: jsonb_build_array() and json_build_array()
menu:
  latest:
    identifier: jsonb-build-array
    parent: functions-operators
    weight: 90
isTocNested: true
showAsideToc: true
---

Here is the signature for the `jsonb` variant:

```
input value        VARIADIC "any"
return value       jsonb
```

These two functions take an arbitrary number of actual arguments of mixed SQL data types. (The PostgreSQL documentation uses the term _"[variadic](https://en.wikipedia.org/wiki/Variadic_function)"_ to characterize this.) The data type of each argument must have a direct JSON equivalent or allow implicit conversion to such an equivalent. Use this _ysqlsh_ script to create the required type `t` and then to execute the `assert`.

```postgresql
create type t as (a int, b text);

do $body$
declare
  v_17 constant int := 17;
  v_dog constant text := 'dog';
  v_true constant boolean := true;
  v_t constant t := (17::int, 'cat'::text);

  j constant jsonb := jsonb_build_array(v_17, v_dog, v_true, v_t);
  expected_j constant jsonb := '[17, "dog", true, {"a": 17, "b": "cat"}]';
begin
  assert
    j = expected_j,
  'unexpected';
end;
$body$;
```

Using `jsonb_build_array()` is straightforward when you know the structure of your target JSON value statically, and just discover the values dynamically. However, it doesn't accommodate the case that you discover the desired structure dynamically.

The following _ysqlsh_ script shows a feasible general workaround for this use case. The helper function `f()` generates the variadic argument list as the text representation of a comma-separated list of manifest constants of various data types. Then it invokes `jsonb_build_array()` dynamically. Obviously this brings a performance cost. But you might not have an alternative.

```postgresql
create function f(variadic_array_elements in text) returns jsonb
  immutable
  language plpgsql
as $body$
declare
  stmt text := '
    select jsonb_build_array('||variadic_array_elements||')';
  j jsonb;
begin
  execute stmt into j;
  return j;
end;
$body$;

do $body$
declare
  v_17 constant int := 17;
  v_dog constant text := 'dog';
  v_true constant boolean := true;
  v_t constant t := (17::int, 'cat'::text);

  j constant jsonb := jsonb_build_array(v_17, v_dog, v_true, v_t);
  expected_j constant jsonb := f(
    $$
      17::integer,
      'dog'::text,
      true::boolean,
      (17::int, 'cat'::text)::t
    $$
    );
begin
  assert
    j = expected_j,
  'unexpected';
end;
$body$;
```