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

### Context

A datap *context* is defined in a single YAML document. A YAML document can contain at most one *context*.

A *context* spans a tree whose nodes are each one the following types of *joints*:

* tap
* pipe
* junction
* processor
* factory
* warning
* error
* structure

The flow of data is from leafs towards the root, and ends at a *tap*. Thus, each sub-tree below a *tap* defines a data processing pipe. We use the term *upstream* to denote joints that are in a sub-tree relative to a given joint. We use *downstream* to denote joints that are in the joint's ancestry.

### Variables

#### Variable

Variables can be defined in any *joint* in a given *context*. A variables section is an associative list, called "variables". Each variable is an entry in that list, with the key defining the variable name, and the value defining the variable value:

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

#### Variable Reference

Variables can be referenced from:

* *parameter* default values
* function *arguments*
* other upstream *variable* definitions

```
parameters|arguments|variables:
  $
```

Variable references have an @ prefix.

For example:

```{YAML}
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

#### Special Variable References

The following variable references can be used without defining the variables downstream:

* @inflow
* @inflowfun
* @context

##### @inflow

The @inflow reference refers to the output of the upstream joint.


### Joints

### Structure

#### Structure

**Structure** joints fulfill two purposes:

* they define a hierarchy of other joints
* they provide a structure to define *variables* and their scope

In terms of processing, structures are of no relevance and data is simply passed through.

```
[pipe|junction|processor|factory|warning|error|structure]
  $structureName:
    [variables]
    n* pipe|junction|processor|factory|warning|error|structure
```

NOTE: as a consequence:

* a structure may never be a leaf
* a structure has no other recognizable type declaration than being a named associative list. Thus, any named associative list inside a structure is itself a structure.



### Tap

#### Tap

A **tap** defines an entry point to specific data of a context.

>  Conceptually, you can think of a tap as a function: when you open a tap (think "call the function"), data pours out (think: "data is returned as an ouput").

```
[n* structure]
  $tapname:
    type: tap
    [attributes]
    [parameters]
    [variables]
    pipe|junction|processor|factory|warning|error|structure
```

> There are only structures downstream from a tap. There are no other taps upstream from a tap.

#### Attributes

Attributes can contain information and/or meta data that is not part of the datap processing. For example, you can store a long name, description, etc.

```
pipe|junction|processor|factory|warning|error|structure
  n* $attributeName: $value
```

> Consequentially, attributes are any key value pair for which the key name is not parameters or variables. Also, an attribute cannot be a named associative list, otherwise they would be interpreted as a structure.

#### Parameters

A **parameter** allows a user to provide an argument when calling a tap.

A tap may have 0 to n parameters.

Parameters may have default values.

```
tap
  parameters:
    n* $parameterName: [$defaultArgument]
```

#### Examples

```{YAML}
AAPL: #tap name
  type: tap
  #attributes:
  description: Apple Inc. Stock
  used by: chris
  #parameters
  parameters:
    startDate: 2000-01-01
    includeWeekends: FALSE
  #downstream
  pipe: *Quandl
```

### Pipe

A **pipe** element lets you arrange a number of upstream joints sequentially.

```
pipe|junction|processor|factory|warning|error|structure
  $pipeName:
    type: pipe
    [attributes]
    [variables]
    n* pipe|junction|processor|factory|warning|error|structure
```

#### Examples

```{YAML}

```
