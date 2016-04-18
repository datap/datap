# datap Language Specification

> “If I could do it all again, I'd be a plumber.”
>
> -- <cite>Albert Einstein</cite>

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Preliminary Remarks](#preliminary-remarks)
	- [Document Version](#document-version)
	- [Scope](#scope)
	- [Syntax Definitions](#syntax-definitions)
- [datap Elements](#datap-elements)
	- [*context*](#context)
	- [*variables*](#variables)
		- [Variable Definition](#variable-definition)
		- [Variable Reference](#variable-reference)
		- [Special Variable References](#special-variable-references)
			- [*@inflow*](#inflow)
			- [*@inflowfun*](#inflowfun)
			- [*@context*](#context)
			- [Macro references](#macro-references)
	- [*attributes*](#attributes)
	- [Joints](#joints)
		- [*structure*](#structure)
		- [*tap*](#tap)
			- [*parameters*](#parameters)
			- [Examples](#examples)
		- [*processor*](#processor)
			- [*function*](#function)
			- [*arguments*](#arguments)
			- [Example](#example)
		- [*error* and *warning*](#error-and-warning)
		- [*factory*](#factory)
		- [*pipe*](#pipe)
		- [*junction*](#junction)

<!-- /TOC -->
## Preliminary Remarks

### Document Version

- Version: 0.1
- Date: 2016-04-17
- License:

### Scope

datap is a YAML format to define configurable, modular data processing pipes. These configurations can be used to acquire, pre-process, quality-assure, and merge data.

In practice, each datap setup will consist of the following elements:

1. one or more datap configuration files
2. one or more code libraries, doing the actual units of work
3. a datap interpreter in the programming language of your choice. The interpreter parses the configuration file, and maps processing steps (1) to actual library functions (2)

This document is about the first part only.

### Syntax Definitions

In this document, the datap syntax is described using the following conventions:

* `>`: a reference to a specific datap element
* `[]`: optional elements
* `$`: replace the following string with an appropriate name
* `n*`: repeat the element n times
* `|`: or

## datap Syntax

### *context*

A datap [`>context`](#context) is defined in a single YAML document. A YAML document can contain at most one [`>context`](#context).

A [`>context`](#context) spans a tree whose nodes are each one the following types of *joints*:

* tap: entry point to data, can have parameters
* [`>structure`](#structure): organise taps into hierarchies
* flow control:
	* [`>pipe`](#pipe): combine joints serially
	* [`>junction`](#junction): combine multiple joints into one
* data processing:
	* [`>processor`](#processor): unit of work (data acquisition and pre-processing)
	* [`>factory`](#factory): functional programming construct
* error handling:
	* [`>warning`](#warning)
	* [`>error`](#error)

The flow of data is from leafs towards the root, and ends at a [`>tap`](#tap). Thus, each sub-tree below a [`>tap`](#tap) defines the processing steps of a [`>tap`](#tap). We use the term *upstream* to denote joints that are in a sub-tree relative to a given joint. We use *downstream* to denote joints that are in the joint's ancestry.

### *variables*

#### Variable Definition

Variables can be defined in a [`>structure`](#structure), [`>tap`](#tap), [`>pipe`](#pipe), and [`>junction`](#junction) in a given [`>context`](#context).

A [`>variables`](#variables) section is an *associative list*, called "variables". Each variable is an entry in that list, with the *key* defining the variable *name*, and the *value* defining the variable *value*:

```
>structure|tap|pipe|junction
  variables:
    n* $variableName: $value
```

The names of *[special variable references](#special-variable-references)* cannot be used as variable name (inflow, inflowfun, context)

Example:

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

* [`>parameters`](#parameters) default values
* function [`>arguments`](#arguments)
* other downstream [`>variables`](#variables)

Variable references have an *@* prefix:

```
>parameters|arguments|variables:
  $name: @$variableReferenceName
```
Or, for unnamed [`>arguments`](#arguments):
```
>arguments:
  - @$variableReferenceName
```

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

* *@inflow*
* *@inflowfun*
* *@context*

They are *reserved words* and cannot be used as variable name.

##### *@inflow*

The *@inflow* reference refers to the output of the upstream joints. For a *pipe*, there is a single upstream joint. For a *junction*, there can be more than one. In that case, the *@inflow* refers to the set of upstream outputs.

```{YAML}
MinLength:
  type: error
  function: MinLength
  arguments:
    timeseries: '@inflow'
    minLength: 10
```

##### *@inflowfun*

The *@inflowfun* reference refers to the upstream joints. This is particularly useful in connection with *[factory](#factory)* joints.

```{YAML}
Cache:
  type: factory
  function: Cache
  arguments:
    f: '@inflowfun'
    timeout: 3600
```

##### *@context*

The *@context* reference refers to its surrounding *[context](#context)*.

It is useful to source data from within a *context*, and to re-use it as an input into another tap.

For example:
```{YAML}
Tap:
  type: processor
  function: Tap
  arguments:
    context: '@context'
    tapPath: 'Closing Prices/Indices/SPX'
```

##### Macro references

A *macro* is a custom function that is interpreted by the *datap interpreter*, and whose return value is substituted into the macro reference dynamically at call-time of the tap.

```
@$macroName(n* $parameterName[,])
```

For example, the datapR interpreter provides a macro *Today*, taking no arguments. Here, it is used to make sure that the *default argument* for the *endDate* parameter of the *Ones* tap is set to today, dynamically at call-time of the tap:

```{YAML}
Ones:
  type: tap
  parameters:
    startDate: 2000-01-01
    endDate: '@Today()'
```

### *attributes*

Attributes can contain information and/or meta data that is not part of the datap processing. For example, you can store a long name, description, etc.

```
>pipe|junction|processor|factory|warning|error|structure
  n* $attributeName: $value
```

> Consequentially, attributes are any key value pair for which the key name is not "parameters" or "variables". Also, an attribute cannot be a named associative list, otherwise it would be interpreted as a structure.

### Joints

Joints are the building blocks of any datap configuration, as explained in the *[Context](#context)* section.

#### *structure*

*Structure* joints fulfil two purposes:

* they define a hierarchy of other joints, especially *(taps)[#Tap]*
* they provide a scope to *(variables)[#Variables]*

In terms of processing, structures are of no relevance.

```
[pipe|junction|processor|factory|warning|error|structure]
  $structureName:
    [variables]
    n* pipe|junction|processor|factory|warning|error|structure
```

> Consequentially:
>
> * a structure may never be a leaf
> * a structure has no other recognizable type declaration than being a named associative list. Thus, any named associative list inside a structure is itself a structure.

#### *tap*

A *tap* defines an entry point to specific data, within a context.

>  Conceptually, you can think of a tap as a function: when you open a tap (think "call the function"), data pours out (think: "data is returned as an output").

```
[n* structure]
  $tapname:
    type: tap
    [attributes]
    [parameters]
    [variables]
    pipe|junction|processor|factory|warning|error|structure
```

> There are only structures downstream from a tap.
> There are no other taps upstream from a tap.

##### *parameters*

A *parameter* allows a user to provide an argument when calling a tap.

A tap may have 0 to n parameters.

Parameters may have default values.

```
tap
  parameters:
    n* $parameterName: [$defaultArgument]
```
##### Examples

```{YAML}
AAPL: #tap name
  type: tap
  #attributes
  description: Apple Inc. Stock
  used by: chris
  #parameters
  parameters:
    startDate: 2000-01-01
    endDate: @Today()
    includeWeekends:
  #upstream
  pipe: *Quandl
```

#### *processor*

A *processor* defines a unit of work, such as data acquisition and pre-processing.

```
>pipe
  type: processor
  [>attributes]
  >function
  [>arguments]
```

##### *function*

A *function* is a directive to the datap interpreter how a *processor*, *error*, or *warning* is mapped to an actual function.

```
>processor|error|warning
  function: $functionName
```

> Without an interpreter and a code library, the functionName has no semantic meaning. It is just a name!

##### *arguments*

The *arguments* section define what arguments will be passed to a function.

The arguments can be named or unnamed (ordered):

```
>processor|error|warning
  arguments:
    n* - $argument | n* $parameterName: $argument
```

##### Example

Named arguments:

```{YAML}
DownloadQuandl:
  type: processor
  function: Quandl::Quandl
  arguments:
    code: '@quandlCode'
    type: xts
```

Unnamed arguments:

```{YAML}
DownloadQuandl:
  type: processor
  function: Quandl::Quandl
  arguments:
    - '@quandlCode'
    - xts
```

#### *error* and *warning*

*Error* and *warning* joints allow testing the results of the upstream *processor* steps.

Error and warning are pass-through: after testing, they pass the @inflow and @inflowfun downstream.

An *error* condition is a directive to the interpreter to stop execution and display an error message.
A *warning* condition is a directive to continue execution and display a warning message.

```
>pipe
  type: error|warning
  >function
  [>arguments]
```

Example:

```{YAML}
MinLength:
  type: error
  function: MinLength
  arguments:
    timeseries: '@inflow'
    minLength: 10
```


#### *factory*

*factory* adds functional programming elements to datap.

A *factory* is similar to a *processor*. The difference is that

1. a factory's *function* is executed only once, at *context* **creation time** (and not at *tap* **call time**)
2. the result of the *function* is expected to be itself a function. That function will then be invoked at *tap* call time.

```
>pipe
  type: factory
  [>attributes]
  >function
  [>arguments]
```

Example:

```{YAML}
Cache:
  type: factory
  function: Cache
  arguments:
    f: '@inflowfun'
    timeout: 3600
```

> The *function* of the upstream joint is passed into the Cache function as its *f* argument. Cache is expected to be a function factory that returns as
> an output a memoised version of @inflowfun.


#### *pipe*

A *pipe* joint lets you arrange a number of upstream joints sequentially.

```
>tap|pipe|junction
  $pipeName:
    type: pipe
    [>attributes]
    [>variables]
    n* >pipe|junction|processor|factory|warning|error
```

For example, the following pipe first checks if the number of NAs in a series is below an inacceptable threshold (NA Ratio), then it backfills missing values (Fill NAs):

```{YAML}
NA handling: &NaHandling
  type: pipe
  Fill NAs:
    type: processor
    function: zoo::na.locf
    arguments:
      object: '@inflow'
  NA Ratio:
    type: warning
    function: NaRatio
    arguments:
      timeseries: '@inflow'
      variable: '@series'
      maxRatio: '@maxNaRatio'
```

#### *junction*

A *junction* merges multiple upstream joints into a single stream.

Unlike the *pipe*, the *junction* has a *function*, which is a directive how to merge the upstream pipes.

```
>tap|pipe|junction
  $junctionName:
    type: junction
    [>attributes]
    [>variables]
    >function
    [>arguments]
    n* >pipe
```
