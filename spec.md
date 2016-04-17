# datap Language Specification

## Document Information

- Version: 0.1
- Date: 2016-04-17
- License: 

## Scope

datap is a format to define configurable, modular data processing pipes.

## Definitions

[]: optional elements
$: replace the following string with an appropriate name
n*: repeat the element n times
|: or

## datap Elements

###  Context

A datap **context** is defined in a single YAML document. A YAML document can contain at most one context. 

A context spans a tree of the following types of nodes:

* tap
* pipe
* junction
* joint
* factory
* warning
* error
* structure


## Nodes


### Structure

#### Structure

**Structure** nodes define a hierarchy of other nodes, and provide a structure to define and share *variables*.

Structures are ignored in processing pipes, they are treated as simple pass-through joints.

```
$name:
  [variables]
  pipe|junction|joint|factory|warning|error|structure
```

#### Variable

Variables can be defined in any node in a context, and re-used in parameters, arguments, and other variable definitions.

### Tap

#### Tap

A **tap** defines an entry point to specific data of a context. 

Conceptually, you can think of a tap as a function: when you open it, data pours out.

```
$name:
  type: tap
  [parameters]
  [variables]
  pipe|junction|joint|factory|warning|error|structure
```

#### Parameters

A **parameter** allows a user to provide input parameters when calling a tap.

A tap may have 0 to n parameters.

Parameters may have default values.

```
parameters:
  n* $parameterName: [$defaultArgument]
```

#### Examples

```{YAML}
AAPL:
  type: tap
  parameters:
    code: 'YAHOO/AAPL'
    periods: 10
  pipe: *Quandl 
```