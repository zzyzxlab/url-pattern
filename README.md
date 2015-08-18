# url-pattern

[![NPM Package](https://img.shields.io/npm/v/url-pattern.svg?style=flat)](https://www.npmjs.org/package/url-pattern)
[![Build Status](https://travis-ci.org/snd/url-pattern.svg?branch=master)](https://travis-ci.org/snd/url-pattern/branches)
[![coverage-97%](http://img.shields.io/badge/coverage-97%-brightgreen.svg?style=flat)](https://github.com/snd/url-pattern/coverage/lcov-report/lib/url-pattern.js.html)
[![Downloads per Month](https://img.shields.io/npm/dm/url-pattern.svg?style=flat)](https://www.npmjs.org/package/url-pattern)
[![Dependencies](https://david-dm.org/snd/url-pattern.svg)](https://david-dm.org/snd/url-pattern)

**easier than regex string matching for urls, domains, filepaths and other strings.  
can capture named parts of strings and conveniently returns them as objects.  
can do the reverse and generate strings given a pattern and such an object.  
great for routing !**

**[the newest version 0.10 introduces breaking changes !](CHANGELOG.md#010)**  
[see the changelog](CHANGELOG.md#010)

> This is a great little library -- thanks!  
> [michael](https://github.com/snd/url-pattern/pull/7)

- [match patterns against strings and extract values](#match-pattern-against-string)
- [generate strings from patterns and values](#stringifying-patterns)
- very fast matching as each pattern is compiled into a regex exactly once
- supports Node.js, [AMD](http://requirejs.org/docs/whyamd.html) and browsers
- [implemented in less than 500 lines of simple, readable, maintainable code](src/url-pattern.coffee)
- [![Build Status](https://travis-ci.org/snd/url-pattern.svg?branch=master)](https://travis-ci.org/snd/url-pattern/branches) [huge test suite](test)
- [![Downloads per Month](https://img.shields.io/npm/dm/url-pattern.svg?style=flat)](https://www.npmjs.org/package/url-pattern) downloads per month
- [![Dependencies](https://david-dm.org/snd/url-pattern.svg)](https://david-dm.org/snd/url-pattern) zero dependencies
- npm package: `npm install url-pattern`
- bower package: `bower install url-pattern`
- [optional segments](#optional-segments-wildcards-and-escaping)
- [escaping](#optional-segments-wildcards-and-escaping)
- [wildcards](#optional-segments-wildcards-and-escaping)
- [customizable pattern syntax](#customizing-the-pattern-syntax)
- pattern parser implemented using elegant, modular, highly testable [parser combinators](https://en.wikipedia.org/wiki/Parser_combinator) from [pcom](https://github.com/snd/pcom)

[check out **passage** if you are looking for simple composable routing that builds on top of url-pattern](https://github.com/snd/passage)

```
npm install url-pattern
```

```
bower install url-pattern
```

``` javascript
> var UrlPattern = require('url-pattern');
```

[lib/url-pattern.js](lib/url-pattern.js) supports [AMD](http://requirejs.org/docs/whyamd.html).  
if [AMD](http://requirejs.org/docs/whyamd.html) is not available it sets the global variable `UrlPattern`.

``` javascript
> var pattern = new UrlPattern('/api/users/:id');

> pattern.match('/api/users/10');
{id: '10'}

> pattern.match('/api/products/5');
null
```
``` javascript
> var pattern = new UrlPattern('/v:major(.:minor)/*');

> pattern.match('/v1.2/');
{major: '1', minor: '2', _: ''}

> pattern.match('/v2/users');
{major: '2', _: 'users'}

> pattern.match('/v/');
null
```
``` javascript
> var pattern = new UrlPattern('(http(s)\\://)(:subdomain.):domain.:tld(/*)')

> pattern.match('google.de');
{domain: 'google', tld: 'de'}

> pattern.match('https://www.google.com');
{subdomain: 'www', domain: 'google', tld: 'com'}

> pattern.match('http://mail.google.com/mail');
{subdomain: 'mail', domain: 'google', tld: 'com', _: 'mail'}

> pattern.match('google');
null
```

### make pattern from string

```javascript
> var pattern = new UrlPattern('/api/users/:id');
```

a `pattern` is immutable after construction.  
none of its methods changes its state.  
that makes it easier to reason about.

### match pattern against string

match returns the extracted segments:

```javascript
> pattern.match('/api/users/10');
{id: '10'}
```

or `null` if there was no match:

``` javascript
> pattern.match('/api/products/5');
null
```

patterns are compiled into regexes which makes `.match()` superfast.

### named segments

`:id` (in the example above) is a named segment:

a named segment starts with `:` followed by the **name**.  
the **name** must be at least one character in the regex character set `a-zA-Z0-9`.

when matching, a named segment consumes all characters in the regex character set
`a-zA-Z0-9-_~ %`.
a named segment match stops at `/`, `.`, ... but not at `_`, `-`, ` `, `%`...

[you can change these character sets. click here to see how.](#customizing-the-pattern-syntax)

if a named segment **name** occurs more than once in the pattern string,
then the multiple results are stored in an array on the returned object:

```javascript
> var pattern = new UrlPattern('/api/users/:ids/posts/:ids');
> pattern.match('/api/users/10/posts/5');
{ids: ['10', '5']}
```

### optional segments, wildcards and escaping

to make part of a pattern optional just wrap it in `(` and `)`:

```javascript
> var pattern = new UrlPattern(
  '(http(s)\\://)(:subdomain.):domain.:tld(/*)'
);
```

note that `\\` escapes the `:` in `http(s)\\://`.
you can use `\\` to escape `(`, `)`, `:` and `*` which have special meaning within
url-pattern.

optional named segments are stored in the corresponding property only if they are present in the source string:

```javascript
> pattern.match('google.de');
{domain: 'google', tld: 'de'}
```

```javascript
> pattern.match('https://www.google.com');
{subdomain: 'www', domain: 'google', tld: 'com'}
```

`*` in patterns are wildcards and match anything.
wildcard matches are collected in the `_` property:

```javascript
> pattern.match('http://mail.google.com/mail');
{subdomain: 'mail', domain: 'google', tld: 'com', _: 'mail'}
```

if there is only one wildcard then `_` contains the matching string.
otherwise `_` contains an array of matching strings.

[look at the tests for additional examples of `.match`](test/match-fixtures.coffee)

### make pattern from regex

```javascript
> var pattern = new UrlPattern(/^\/api\/(.*)$/);
```

if the pattern was created from a regex an array of the captured groups is returned on a match:

```javascript
> pattern.match('/api/users');
['users']

> pattern.match('/apiii/test');
null
```

when making a pattern from a regex
you can pass an array of keys as the second argument.
returns objects on match with each key mapped to a captured value:

```javascript
> var pattern = new UrlPattern(
  /^\/api\/([^\/]+)(?:\/(\d+))?$/,
  ['resource', 'id']
);

> pattern.match('/api/users');
{resource: 'users'}

> pattern.match('/api/users/5');
{resource: 'users', id: '5'}

> pattern.match('/api/users/foo');
null
```

### stringifying patterns

```javascript
> var pattern = new UrlPattern('/api/users/:id');

> pattern.stringify({id: 10})
'/api/users/10'
```

optional segments are only included in the output if they contain named segments
and/or wildcards and values for those are provided:

```javascript
> var pattern = new UrlPattern('/api/users(/:id)');

> pattern.stringify()
'/api/users'

> pattern.stringify({id: 10})
'/api/users/10'
```

wildcards (key = `_`), deeply nested optional groups and multiple value arrays should stringify as expected.

an error is thrown if a value that is not in an optional group is not provided.

an error is thrown if an optional segment contains multiple
params and not all of them are provided.
*one provided value for an optional segment
makes all values in that optional segment required.*

[look at the tests for additional examples of `.stringify`](test/stringify-fixtures.coffee)

### customizing the pattern syntax

finally we can completely change pattern-parsing and regex-compilation to suit our needs:

```javascript
> var options = {};
```

let's change the char used for escaping (default `\\`):

```javascript
> options.escapeChar = '!';
```

let's change the char used to start a named segment (default `:`):

```javascript
> options.segmentNameStartChar = '$';
```

let's change the set of chars allowed in named segment names (default `a-zA-Z0-9`)
to also include `_` and `-`:

```javascript
> options.segmentNameCharset = 'a-zA-Z0-9_-';
```

let's change the set of chars allowed in named segment values
(default `a-zA-Z0-9_- %`) to not allow non-alphanumeric chars:

```javascript
> options.segmentValueCharset = 'a-zA-Z0-9';
```

let's change the chars used to surround an optional segment (default `(` and `)`):

```javascript
> options.optionalSegmentStartChar = '[';
> options.optionalSegmentEndChar = ']';
```

let's change the char used to denote a wildcard (default `*`):

```javascript
> options.wildcardChar = '?';
```

pass options as the second argument to the constructor:

```javascript
> var pattern = new UrlPattern(
  '[http[s]!://][$sub_domain.]$domain.$toplevel-domain[/?]',
  options
);
```

then match:

```javascript
> pattern.match('http://mail.google.com/mail');
{
  sub_domain: 'mail',
  domain: 'google',
  'toplevel-domain': 'com',
  _: 'mail'
}
```

### contribution

**TLDR:  
bugfixes, issues and discussion are always welcome.  
kindly ask before implementing new features.**

i will happily merge pull requests that fix bugs with reasonable code.

i will only merge pull requests that modify/add functionality
if the changes align with my goals for this package,
are well written, documented and tested.

**communicate !**  
write an issue to start a discussion before writing code that may or may not get merged.

## [license: MIT](LICENSE)
