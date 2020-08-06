---
title: Yaml Basic
date: 2020-8-6
tags: yaml, tool
---

[Offical Tutorial](https://yaml.org/spec/1.2/spec.html)

## Basic

`key: value`

Yaml will auto detect data type, but some time that's what we don't want. So you need  **Type cast** :

``` yaml
age: !!str 10
```

- type avaliable
    * str
    * int
    * float
    * bool
    * null
    * ISO date type:`2020-1-1 11:11:22`
    * array


## Array

**Can not use TAB** in yaml. Use indentation to grading and `-` char to sign as a array.

``` yaml
arr:
  - item1
    - item11
  - item2
    - item21
    - item22
```


## Object

**Can not use TAB** in yaml. Use indentation to grading but no `-` char.

``` yaml
person:
  name: ring
  age: 10
```


## Long string value

``` yaml
message:
  a long long long
  long long
  long msg
```

To end with `\n` 

``` yaml
message: >
  a long long long
  long long
  long msg
```

To end with `\n` for each line 

``` yaml
message: |
  a long long long
  long long
  long msg

  #a long long long\nlong long\nlong msg\n
```


## Use Reference

Assign with pointer

``` yaml
father: &father_info
  name: ring

monitor: *father_info

# value of monitor same as value of father
```


## New part

Yaml not allow you have dulplicate key. You can use `---` to statement in a new part

``` yaml
name: ring
---
name: ring
```


## End of File

You can use `...` to tell yaml, this is the end.

