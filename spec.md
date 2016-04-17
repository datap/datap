# datap Language Specification

## Document Information

- Version: 0.1
- Date: 2016-04-17
- License:

## Table of Contents



## Scope

datap is a YAML format to define configurable, modular data processing pipes.

## Definitions

* []: optional elements
* $: replace the following string with an appropriate name
* n*: repeat the element n times
* |: or

## datap Elements

###  Context

A datap **context** is defined in a single YAML document. A YAML document can contain at most one context.

A context spans a tree of the following types of *nodes*:

* tap
* pipe
* junction
* joint
* factory
* warning
* error
* structure

The flow of data is from leafs towards root, and ends at a *tap*. Thus, each sub-tree below a *tap* defines a data processing pipe. We use *upstream* to denote nodes that are in a sub-tree relative to a given node. We use *downstream* to denote nodes that are in the branch towards the tree's root.

## Nodes

### Structure

#### Structure

**Structure** nodes fulfill two purposes:

* they define a hierarchy of other nodes
* they provide a structure to define *variables* and their scope

In terms of processing, structures are of no relevance and data is simply passed through.

```
[pipe|junction|joint|factory|warning|error|structure]
  $structureName:
    [variables]
    n* pipe|junction|joint|factory|warning|error|structure
```

NOTE: as a consequence:

* a structure may never be a leaf
* a structure has no other recognizable type declaration than being a named associative list. Thus, any named associative list inside a structure is itself a structure.

#### Variable

##### Variable Definition

Variables can be defined in any *node* in a *context*.

```
variables:
  n* $variableName: $value
```

For example:

```{YAML}
Closing Prices:
  variables:
    series: Close
    startDate: 2000-01-01
  SPX: *SPX
  DJIA: *DJIA
```

##### Variable Reference

Variables can be referenced from:

* *parameter* default values
* function *arguments*
* other downstream *variable* definitions

They are referenced using the @ prefix.

##### Example

```
AAPL:
  type: tap
  variables:
    #variable reference
    #maxNaRatioDefault must be defined upstream
    maxNaRatio: '@maxNaRatioDefault'
    yahooSymbol: AAPL
    quandlCode: 'YAHOO/AAPL'
  pipe: *QYPipe
```

### Tap

#### Tap

A **tap** defines an entry point to specific data of a context.[^1]

  [^1] Conceptually, you can think of a tap as a function: when you open a tap (call the function), data pours out (data is returned).

A tap can only be defined if there is no other tap upstream.

```
[n* structure]
  $tapname:
    type: tap
    [attributes]
    [parameters]
    [variables]
    pipe|junction|joint|factory|warning|error|structure
```

#### attributes

Attributes can contain information and/or meta data that is not part of the data processing. For example, you can store a long name, description, etc.

NOTE:


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
