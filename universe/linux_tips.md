---
title: linux command line
date: 2020-2-23
tags: linux
---

# Command line 

## shebang
a shebang is the interpreter of this script. like this
``` sh
#!/bin/sh
```

## Redirection

| command | function                       |
|---------|--------------------------------|
| a > b   | a's std output override into b |
| a >> b  | a's std output appand into b   |
| a < b   | b as a's input                 |
| a << b  | here document                  |
| a <<< b | here                           |

### Here document

A block as input
``` sh
echo << token
    text
token

example:
echo << __EOF__
halo
__EOF__

$ halo
```

### Here

a bit like *here document*, a line as input.
``` sh
read ans1 ans2 ans3 <<< "a1 a2 a3"

a1 a2 a3 as input to read process
```

## Useful commands

### Grep

`grep [option] file1 file2...`

The most commonly used
| options  | description                    |
|----------|--------------------------------|
| -i       | --ignore-case                  |
| -v       | --revert-mathch                |
| -c       | --count                        |
| -l       | --file-with-match              |
| -L       | --file-without-math            |
| -n       | --line-number                  |
| -h       | no filename in multi file mode |
| -a/-text | do not ignore binary data      |
| -e       | --regexp                       |
| -f       | specify regexp file            |


### Sed

sed can deal with text file with script-command or script-file.

`sed [options][target-file]`

| options | description |
|---------|-------------|
| -e      |             |
| <++>    | <++>        |
| <++>    | <++>        |
| <++>    | <++>        |
| <++>    | <++>        |
| <++>    | <++>        |
| <++>    | <++>        |

script action

| action | description                                  |
|--------|----------------------------------------------|
| a\     | add a new line or lines next to current line |
| c\     | replace current line with some text          |
| d      | delete line                                  |
| i\     | insert text befor curent line                |
| h      | <++>                                         |
| H      | <++>                                         |
| g      | <++>                                         |
| G      | <++>                                         |
| I      | <++>                                         |
| p      | print line                                   |
| n      | <++>                                         |
| q      | quit sed                                     |
| r      | <++>                                         |
| !      | <++>                                         |
| s      | replace string with other string             |
| <++>   | <++>                                         |
| <++>   | <++>                                         |
| <++>   | <++>                                         |
| <++>   | <++>                                         |
| <++>   | <++>                                         |

replace identifier

| options | description                 |
|---------|-----------------------------|
| g       | global replace in this line |
| p       | print line                  |
| w       | write line into file        |
| x       | <++>                        |
| y       | <++>                        |
| <++>    | <++>                        |



### Cut

| options | description |
|---------|-------------|
| <++>    | <++>        |
| <++>    | <++>        |
| <++>    | <++>        |
| <++>    | <++>        |
| <++>    | <++>        |
| <++>    | <++>        |
| <++>    | <++>        |

## Regular Expressions

### Literals

| pattern | description     |
|---------|-----------------|
| .       | match any char  |
| char    | literaly a char |

### Metacharacters

#### Anchors

| pattern | description                |
|---------|----------------------------|
| ^       | found at the begin of line |
| $       | found at the end of line   |

#### Quantifiers

| pattern | description                                            |
|---------|--------------------------------------------------------|
| ?       | match **an element** zero or one time                  |
| *       | match **an element** zero or more times                |
| +       | match **an element** one or more times                 |
| {}      | match **an element** a specific number of times        |
| {n,}    | match **an element** if it occurs n or more times      |
| {,n}    | match **an element** if it occurs no more than n times |

A quantifiers is after **an element**, such as `.*` means match any char with any lengh equivalent any string.

An element can like this `[A_Z]`and this`[:digit:]`. We can see the pattern before a quantifiers as a whole.



#### Bracket Expressions and Character Classes

- Bracket Expressions
    - specify a set of characters to be match
``` sh
$ grep -h '[bg]zip' test.txt
bzip2
bzip123
gzip
```

- Negation
    - If the first char in a bracket expression is a caret(^), the **remaining char** are taken to be a set of chars that must not be present at the given char position.
``` sh
$ grep -h '[^bg]zip' test.txt
bunzip
gunzip
funzip
```

- Character Classes
    - A set of characters range
``` sh
$ grep -h '[^ABCDEF]' test.txt
or you can
$ grep -h '[^A-F]' test.txt

Here are more expressions
$ grep -h '[A-Fa-z0-9]' test.txt
```

- POSIX Character Class

| char       | description                                                |
|------------|------------------------------------------------------------|
| [:alnum:]  | alphanumeric characters.equivalent to[A-Za-z0-9]           |
| [:word:]   | same as alnum, with the addition of the underscore char(\_) |
| [:alpha:]  | [A-Za-z]                                                   |
| [:blank:]  | space and tab char                                         |
| [:digit:]  | number 0 through 9                                         |
| [:lower:]  | the lowercase letters                                      |
| [:upper:]  | the uppercase letters                                      |
| [:space:]  | [\t\r\n\v\f]                                               |
| [:xdigit:] | chars used to express hexadecimal numbers.[0-9a-fA-F]      |





## Flow control

### if statements

#### File Expressions

| command   | description                          |
|-----------|--------------------------------------|
| f1 -ef f2 | f1 and f2 have the same indoe number |
| f1 -nt f2 | f1 newer than f2                     |
| f1 -ot f2 | f1 older than f2                     |
| -d f1     | f1 exists an is a directory          |
| -e f1     | f1 exists                            |
| -f f1     | f1 is a regular file                 |

#### String Expressions

| command             | description                              |
|---------------------|------------------------------------------|
| string              | string is not null                       |
| -n string           | the lenth of string is greater than zero |
| -z string           | the lenth of string is zero              |
| s1 = s2 or s1 == s2 | equal                                    |
| s1 !=s2             | not equal                                |
| s1 > s2             | s1 sorts after s2                        |
| s1 < s2             | s1 sortf before s2                       |

#### Integer Expressions

| command       | description      |
|---------------|------------------|
| int1 -eq int2 | equal            |
| int1 -ne int2 | not equal        |
| -le           | less or equal    |
| -lt           | less than        |
| -ge           | greater or equal |
| -gt           | greater          |

#### A More Modern Version of test

- **Designed of string**
    - `[[ command ]]`

and here is a new feature:
`string1 =~ regex`. This return true if string1 matched by the extended regular expression

- **Designed of Integer**
    - `(( command ))
    - integer only

### case

``` sh
case word in
    [pattern [| pattern]...) commands ;;]...
esac

case $num in
    0) echo "zero"
        ;;  # ;; break and out of case
    ...
esac
```

#### pattern

| pattern     | description                                      |
|-------------|--------------------------------------------------|
| a)          | matches if word equals a                         |
| [[:alpha:]] | matches if word is a single alphabetic           |
| ???)        | matches if word is exactly three characters long |
| *.txt       | matches if word ends with .txt                   |
| *)          | matches any value.                               |

### for/while/until

- for 
    - `for variable in words; do commands done`
    - `for (( expression1; expression2; expression3 )); do commands done`
- while 
    - `while commands; do commands; done`
    - while commands is true continue
- until 
    - `until commands; do commands; done`
    - while commands is false continue
     


