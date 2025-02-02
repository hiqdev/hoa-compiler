## Reasons to fork
The Hoa project was archived, and no upgrades or patches are neither provided nor accepted by merge requests.

HOA Packages include some code that is Deprecated for PHP 8.0 and PHP 8.1, but we needed these packages to run on modern PHP versions.

## What's changed in from?
The changes mainly affected the return data type hinting in methods declaration, access to uninitialized properties.

## How to use
We've currently forked the following packages, primarily to make hoa/ruler work with PHP 8.1:

<table>
<tr><th>Original package</th><th>Forked package</th></tr>
<tr><td>hoa/ruler</td><td>hiqdev/hoa-ruler</td></tr>
<tr><td>hoa/compiler</td><td>hiqdev/hoa-compiler</td></tr>
<tr><td>hoa/protocol</td><td>hiqdev/hoa-protocol</td></tr>
<tr><td>hoa/iterator</td><td>hiqdev/hoa-iterator</td></tr>
</table>
You can simply replace requirements in composer.json from hoa packages to the corresponding forked packages: there is no need to change something in the codebase.

If you use someone's package, that requires hoa – simply add forks to your project root composer.json: we have marked forks as a replacement, so composer will install them instead of the original packages.

## Versions
We've forked from the latest hoa package versions and bump own versions starting from 1.0.

## Testing

Before running the test suites, the development dependencies must be installed:

```sh
$ composer install
```

Then, to run all the test suites:

```sh
$ vendor/bin/hoa test:run
```

For more information, please read the [contributor
guide](https://hoa-project.net/Literature/Contributor/Guide.html).

## Quick usage

As a quick overview, we will look at the PP language and the LL(k) compiler
compiler.

### The PP language

A grammar is constituted by tokens (the units of a word) and rules (please, see
the documentation for an introduction to the language theory). The PP language
declares tokens with the following construction:

```
%token [source_namespace:]name value [-> destination_namespace]
```

The default namespace is `default`. The value of a token is represented by a
[PCRE](http://pcre.org/). We can skip tokens with the `%skip` construction.

As an example, we will take the *simplified* grammar of the [JSON
language](http://json.org/). The complete grammar is in the
`hoa://Library/Json/Grammar.pp` file. Thus:

```
%skip   space          \s
// Scalars.
%token  true           true
%token  false          false
%token  null           null
// Strings.
%token  quote_         "        -> string
%token  string:string  [^"]+
%token  string:_quote  "        -> default
// Objects.
%token  brace_         {
%token _brace          }
// Arrays.
%token  bracket_       \[
%token _bracket        \]
// Rest.
%token  colon          :
%token  comma          ,
%token  number         \d+

value:
    <true> | <false> | <null> | string() | object() | array() | number()

string:
    ::quote_:: <string> ::_quote::

number:
    <number>

#object:
    ::brace_:: pair() ( ::comma:: pair() )* ::_brace::

#pair:
    string() ::colon:: value()

#array:
    ::bracket_:: value() ( ::comma:: value() )* ::_bracket::
```

We can see the PP constructions:

  * `rule()` to call a rule;
  * `<token>` and `::token::` to declare a token;
  * `|` for a disjunction;
  * `(…)` to group multiple declarations;
  * `e?` to say that `e` is optional;
  * `e+` to say that `e` can appear at least 1 time;
  * `e*` to say that `e` can appear 0 or many times;
  * `e{x,y}` to say that `e` can appear between `x` and `y` times;
  * `#node` to create a node the AST (resulting tree);
  * `token[i]` to unify tokens value between them.

Unification is very useful. For example, if we have a token that expresses a
quote (simple or double), we could have:

```
%token  quote   "|'
%token  handle  \w+

string:
    ::quote:: <handle> ::quote::
```

So, the data `"foo"` and `'foo'` will be valid, but also `"foo'` and `'foo"`! To
avoid this, we can add a new constraint on token value by unifying them, thus:

```
string:
    ::quote[0]:: <handle> ::quote[0]::
```

All `quote[0]` for the rule instance must have the same value. Another example
is the unification of XML tags name.

### LL(k) compiler compiler

The `Hoa\Compiler\Llk\Llk` class provide helpers to manipulate (load or save) a
compiler. The following code will use the previous grammar to create a compiler,
and we will parse a JSON string. If the parsing succeed, it will produce an AST
(stands for Abstract Syntax Tree) we can visit, for example to dump the AST:

```php
// 1. Load grammar.
$compiler = Hoa\Compiler\Llk\Llk::load(new Hoa\File\Read('Json.pp'));

// 2. Parse a data.
$ast = $compiler->parse('{"foo": true, "bar": [null, 42]}');

// 3. Dump the AST.
$dump = new Hoa\Compiler\Visitor\Dump();
echo $dump->visit($ast);

/**
 * Will output:
 *     >  #object
 *     >  >  #pair
 *     >  >  >  token(string, foo)
 *     >  >  >  token(true, true)
 *     >  >  #pair
 *     >  >  >  token(string, bar)
 *     >  >  >  #array
 *     >  >  >  >  token(null, null)
 *     >  >  >  >  token(number, 42)
 */
```

Pretty simple.

### Compiler in CLI

This library proposes a script to parse and apply a visitor on a data with a
specific grammar. Very useful. Moreover, we can use pipe (because
`Hoa\File\Read` —please, see the [`Hoa\File`
library](http://central.hoa-project.net/Resource/Library/File/)— supports `0` as
`stdin`), thus:

```sh
$ echo '[1, [1, [2, 3], 5], 8]' | hoa compiler:pp Json.pp 0 --visitor dump
>  #array
>  >  token(number, 1)
>  >  #array
>  >  >  token(number, 1)
>  >  >  #array
>  >  >  >  token(number, 2)
>  >  >  >  token(number, 3)
>  >  >  token(number, 5)
>  >  token(number, 8)
```

You can apply any visitor classes.

### Errors

Errors are well-presented:

```sh
$ echo '{"foo" true}' | hoa compiler:pp Json.pp 0 --visitor dump
Uncaught exception (Hoa\Compiler\Exception\UnexpectedToken):
Hoa\Compiler\Llk\Parser::parse(): (0) Unexpected token "true" (true) at line 1
and column 8:
{"foo" true}
       ↑
in hoa://Library/Compiler/Llk/Parser.php at line 1
```

### Samplers

Some algorithms are available to generate data based on a grammar. We will give
only one example with the coverage-based generation algorithm that will activate
all branches and tokens in the grammar:

```php
$sampler = new Hoa\Compiler\Llk\Sampler\Coverage(
    // Grammar.
    Hoa\Compiler\Llk\Llk::load(new Hoa\File\Read('Json.pp')),
    // Token sampler.
    new Hoa\Regex\Visitor\Isotropic(new Hoa\Math\Sampler\Random())
);

foreach ($sampler as $i => $data) {
    echo $i, ' => ', $data, "\n";
}

/**
 * Will output:
 *     0 => true
 *     1 => {" )o?bz " : null , " %3W) " : [false, 130    , " 6"   ]  }
 *     2 => [{" ny  " : true } ]
 *     3 => {" Ne;[3 " :[ true , true ] , " th: " : true," C[8} " :   true }
 */
```

## Research papers

  * *Grammar-Based Testing using Realistic Domains in PHP*,
    presented at [A-MOST 2012](https://sites.google.com/site/amost2012/) (Montréal, Canada)
    ([article](https://hoa-project.net/En/Literature/Research/Amost12.pdf),
     [presentation](http://keynote.hoa-project.net/Amost12/EDGB12.pdf),
     [details](https://hoa-project.net/En/Event/Amost12.html)).

## Documentation

The
[hack book of `Hoa\Compiler`](https://central.hoa-project.net/Documentation/Library/Compiler) contains
detailed information about how to use this library and how it works.

To generate the documentation locally, execute the following commands:

```sh
$ composer require --dev hoa/devtools
$ vendor/bin/hoa devtools:documentation --open
```

More documentation can be found on the project's website:
[hoa-project.net](https://hoa-project.net/).

## Getting help

There are mainly two ways to get help:

  * On the [`#hoaproject`](https://webchat.freenode.net/?channels=#hoaproject)
    IRC channel,
  * On the forum at [users.hoa-project.net](https://users.hoa-project.net).

## Contribution

Do you want to contribute? Thanks! A detailed [contributor
guide](https://hoa-project.net/Literature/Contributor/Guide.html) explains
everything you need to know.

## License

Hoa is under the New BSD License (BSD-3-Clause). Please, see
[`LICENSE`](https://hoa-project.net/LICENSE) for details.
