# Mudder.js

## Background

**Requirement** A function that, given two strings, returns one or more strings lexicographically between them (see [Java’s `compareTo` docs](http://docs.oracle.com/javase/8/docs/api/java/lang/String.html#compareTo-java.lang.String-) for a cogent summary of lexicographical ordering).

That is, if `c = lexmid(a, b, 1)` a string, then `a ≶ c ≶ b`.

Similarly, if `cs = lexmid(a, b, N)` is an `N>1`-element array of strings, `a ≶ cs[0] ≶ cs[1]  ≶ … ≶ cs[N-1] ≶ b`.

**Use Case** Reliably ordering (or ranking) entries in a database. Such entries may be reordered (shuffled). New entries might be inserted between existing ones.

[My StackOverflow question](http://stackoverflow.com/q/39125091/500207) links to six other questions about this topic, all lacking convincing solutions. Some try to encode order using floating-point numbers, but a pair of numbers can only be subdivided so many times before losing precision. One “fix” to this is to renormalize all entries’ spacings periodically, but some NoSQL databases lack atomic operations and cannot modify multiple entries in one go. Such databases would have to stop accepting new writes, update each entry with a normalized rank number, then resume accepting writes.

Since many databases are happy to sort entries using stringy fields, let’s just use strings instead of numbers. This library aids in the creation of new strings that lexicographically sort between two other strings.

**Desiderata** I’d like to be able to insert thousands of documents between adjacent ones, so `lexmid()` must never return strings which can’t be “subdivided” further. But memory isn’t free, so shorter strings are preferred.

**Prior art** [@m69’s algorithm](http://stackoverflow.com/a/38927158/500207) is perfect: you give it two alphabetic strings containing just `a-z`, and you get back a short alphabetic string that’s “roughly half-way” between them.

I asked how to get `N` evenly-spaced strings ex nihilo, i.e., not between any two strings. [@m69’s clever suggestion](http://stackoverflow.com/questions/38923376/return-a-new-string-that-sorts-between-two-given-strings/38927158#comment65638725_38927158) was, for strings allowed to use `B` distinct characters, and `B^(m-1) < N < B^m`, evenly distribute `N` integers from 2 to `B^m - 2` (or some suitable start and end), and write them as radix-`B`.

This works! Here’s a quick example, generating 25 hexadcimal (`B=16`) strings:
~~~js
var N = 25; // How many strings to generate. Governs how long the strings are.
var B = 16; // Radix, or how many characters to use, < N

// Left and right margins
var start = 2;
var places = Math.ceil(Math.log(N) / Math.log(B)); // max length for N strings
var end = Math.pow(B, places) - 2;

// N integers between `start` and `end`
var ns = Array.from(Array(N), (_, i) => start + Math.round(i / N * end));

// JavaScript's toString can't pad numbers to a fixed length, so:
var leftpad = (str, desiredLen, padChar) =>
    padChar.repeat(desiredLen - str.length) + str;

var strings = ns.map(n => leftpad(n.toString(B), places, '0'));
console.log(strings);
// > [ '02',
// >  '0c',
// >  '16',
// >  '20',
// >  '2b',
// >  '35',
// >  '3f',
// >  '49',
// >  '53',
// >  '5d',
// >  '68',
// >  '72',
// >  '7c',
// >  '86',
// >  '90',
// >  '9a',
// >  'a5',
// >  'af',
// >  'b9',
// >  'c3',
// >  'cd',
// >  'd7',
// >  'e2',
// >  'ec',
// >  'f6' ]
~~~

This uses JavaScript’s `Number.prototype.toString` which works for bases up to `B=36`, but no more. A desire to use more than thirty-six characters led to [a discussion about representing integers in base-62](http://stackoverflow.com/a/2557508/500207), where @DanielVassallo showed a custom `toString`.

Meanwhile, [numbase](https://www.npmjs.com/package/numbase) supports arbitrary-radix interconversion, and, how delightful, lets you specify the universe of characters to use:
```js
// From https://www.npmjs.com/package/numbase#examples // no-hydrogen
// Setup an instance with custom base string
base = new NumBase('中国上海市徐汇区');
// Encode an integer, use default radix 8
base.encode(19901230); // returns '国国海区上徐市徐汇'
// Decode a string, with default radix 8
base.decode('国国海区上徐市徐汇'); // returns '19901230'
```

Finally, [@Eclipse’s observation](http://stackoverflow.com/a/2510928/500207) really elucidated the connection between strings and numbers, and explaining the mathematical vein that @m69’s algorithm was ad hocly tapping. @Eclipse suggested converting a string to a number and then *treating the result as a fraction between 0 and 1*. That is, just place a radix-point before the first digit (in the given base) and perform arithmetic on it. (In this document, I use “radix” and “base” interchangeably.)

**Innovations** Mudder.js (this dependency-free JavaScript/ES2015 library) is a generalization of @m69’s algorithm. It operates on strings containing *arbitrary substrings* instead of just lowercase `a-z` characters: your strings can contain, e.g., 日本語 characters or 🔥 emoji. (Like @m69’s algorithm, you do have to specify upfront the universe of stringy symbols to operate on.)

You can ask Mudder.js for `N ≥ 1` strings that sort between two input strings. These strings will be as short as possible.

These are possible because Mudder.js converts strings to non-decimal-radix (non-base-10), arbitrary-precision fractional numbers between 0 and 1. Having obtained numeric representations of strings, it’s straightforward to compute their average, or midpoint, `(a + b) / 2`, or even `N` intermediate points `a + (b - a) / N * i` for `i` going from 1 to `N - 1`, using the long addition and long division you learned in primary school. (By avoiding native floating-point, Mudder.js can handle arbitrarily-long strings, and generalizes @Eclipse’s suggestion.)

Because `numbase` made it look so fun, as a bonus, Mudder.js can convert regular JavaScript integers to strings. You may specify a multi-character string for each digit. Therefore, should the gastronome in you invent a ternary (radix-3) numerical system based on today’s meals, with 0=🍌🍳☕️, 1=🍱, and 2=🍣🍮, Mudder.js can help you rewrite (42)<sub>10</sub>, that is, 42 in our everyday base 10, as (🍱🍱🍣🍮🍌🍳☕️)<sub>breakfast, lunch, and dinner</sub>.

**This document** This document is a Markdown file that I edit in Atom. It contains code blocks that I can run in Node.js via Hydrogen, an Atom plugin that talks to Node over the Jupyter protocol (formerly IPython Notebook). I don’t have a terminal window open: all development, including scratch work and test snippets, happens in this Markdown file and goes through Hydrogen.

This workflow allows this document to be a heavily-edited diary of writing the library. You, the reader, can see not just the final code but also the experimentation, design choices and decisions and alternatives, opportunities for improvement, references, and asides.

The result of evaluating all code blocks is included at the bottom of each code block (using custom Atom [plugins](https://github.com/fasiha/atom-papyrus-sedge)).

Furthermore, all source code files and configuration files included in this repository are derived from code blocks in *this* Markdown file. This is done by another plugin that goes through this document and pipes code blocks to external files.

In this way, this document is a primitive (or futuristic?) riff on literate programming, the approach discovered and celebrated by Donald Knuth.

Besides reading this document passively on GitHub or npmjs.org, you will eventually be able to read it as an interactive, live-coding webapp, where each code block is editable and executable in your browser. This in turn makes it a riff on [Alan Kay’s vision](http://blog.klipse.tech/javascript/2016/06/20/blog-javascript.html) for interactive programming environments for *readers* as well as writers.

## Plan
Since we must convert strings to arbitrary-radix digits, and back again, this library includes enhanced versions of

- [`Number.prototype.toString`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/toString) which converts JavaScript integers to strings for bases between base-2 (binary) and base-36 (alphanumeric),
- [`parseInt`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseInt) which inverts this operation by converting a string of digits in some base between 2 and 36 to a JavaScript number.

Specifically, we need versions of these functions that operate on bases >36, and that let the user specify the strings used to denote each digit in the arbitrary-radix numerical system. (Recall that I use “base” and “radix” interchangeably.)

We will create these library functions in the next section.

Once we can represent arbitrary strings as equivalent numbers, we will describe the very specific positional number system that lets us find lexicographic mid-points easily. This positional system involves mapping a given string’s numeric representation to a rational number between 0 and 1, and in this system, the lexicographic midpoint between two strings is the simple mean (average) between their two numbers.

This sounds fancy, but again, it’s quite pedestrian. We’ll implement long addition and long division (the two steps required to calculate the mean of two numbers) in a subsequent section.

Finally, with these preliminaries out of the way, we’ll implement the functions to help us paper over all this machinery and that just give us strings lexicographically between two other strings.

## Symbol tables, numbers, and digits
Let us remind ourselves what `toString` and `parseInt` do:
~~~js
console.log([ (200).toString(2), parseInt('11001000', 2) ])
console.log([ (200).toString(36), parseInt('5K', 36) ])
// > [ '11001000', 200 ]
// > [ '5k', 200 ]
// > undefined
~~~
(200)<sub>10</sub> = (1100 1000)<sub>2</sub> = (5K)<sub>36</sub>. One underlying number, many different representations. Each of these is a positional number system with a different base: base-10 is our everyday decimal system, base-2 is the binary system our computers operate on, and base-36 is an uncommon but valid alphanumeric system.

Recall from grade school that this way of writing numbers, as (digit 1, digit 2, digit 3)<sub>base</sub>, means each digit is a multiple of `Math.pow(B, i)` where `i=0` for the right-most digit (in the ones place), and going up for each digit to its left.
~~~js
var dec = 2 * 100 + // 100 = Math.pow(10, 2)
          0 * 10 +  // 10 = Math.pow(10, 1)
          0 * 1;    // 1 = Math.pow(10, 0)

var bin = 1 * 128 + // 128 = Math.pow(2, 7)
          1 * 64 +  // 64 = Math.pow(2, 6)
          0 * 32 +  // 32 = Math.pow(2, 5)
          0 * 16 +  // 16 = Math.pow(2, 4)
          1 * 8 +   // 8 = Math.pow(2, 3)
          0 * 4 +   // 4 = Math.pow(2, 2)
          0 * 2 +   // 2 = Math.pow(2, 1)
          0 * 1;    // 1 = Math.pow(2, 0)

var aln = 5 * 36 + // 36 = Math.pow(36, 1)
          20 * 1;  // 1 = Math.pow(36, 0)

console.log(dec === bin && dec === aln ? 'All same!' : 'all NOT same?');
// > All same!
~~~
That last example and its use of `K` as a digit might seem strange, but for bases >10, people just use letters instead of numbers. `A=10`, `F=15`, `K=20`, and `Z=36` using this convention.

Both these functions operate on what we’ll call a *symbol table*: a mapping between stringy symbols and the numbers from 0 to one less than the maximum base. Here’s the symbol table underlying `parseInt` and `Number.prototype.toString`, with stringy symbols on the left and numbers on the right:

- `0` ⇔ 0
- `1` ⇔ 1
- `2` ⇔ 2
- ⋮
- `8` ⇔ 8
- `9` ⇔ 9
- `a` ⇔ 10
- `A` ⇒ 10
- ⋮
- `z` ⇔ 35
- `Z` ⇒ 35

(Aside: `parseInt` accepts uppercase letters, treating them as lowercase. `Number.prototype.toString` outputs only lowercase letters. Therefore, uppercase letters above have a right-arrow, instead of bidirectional.)

For both the broader problem of lexicographically interior strings, as well as the sub-problem of converting between numbers and strings, we want to specify our own symbol tables. Here are a few ways we’d like to handle, in order of increasing complexity and flexibility:

1. **a string** Great for those simple use-cases: the string is `split` into individual characters, and each character is the symbol for its index number. Such a symbol table can handle bases as high as the number of characters in the input string. Example: `new SymbolTable('abcd')`.
1. **an array of strings** To specify multi-character symbols such as emoji (which `String.split` will butcher), or whole words. Quaternary (radix-4) Roman-numeral example: `new SymbolTable('_,I,II,III'.split(','))`.
1. **an array of strings, _plus_ a map of stringy symbols to numbers** This would let us specify fully-generic symbol tables like `parseInt`’s, where both `'F'` and `'f'` correspond to 15. The array uniquely sends numbers to strings, and the map sends ≥1 strings to numbers. The quaternary Roman-numeral example capable of ingesting lower-case letters:
~~~js
new SymbolTable('_,I,II,III'.split(','), new Map([  // no-hydrogen
                  [ '_', 0 ],                // zero
                  [ 'I', 1 ], [ 'i', 1 ],    // 1, lower AND upper case!
                  [ 'II', 2 ], [ 'ii', 2 ],  // 2
                  [ 'III', 3 ], [ 'iii', 3 ] // 3
                ]));
~~~

Let’s resist the temptation to be avant-garde: let’s agree that, to be valid, a symbol table must include symbols for *all* numbers between 0 and some maximum, with none skipped. `B` (for “base”) unique numbers lets the symbol table define number systems between radix-2 (binary) up to radix-`B`. JavaScript’s implicit symbol table handles `B≤36`, but as the examples above show, we don’t have to be restricted to base-36.

(Aside: radix-36 doesn’t seem to have a fancy name like the ancient Sumerians’ radix-60 “sexagesimal” system so I call it “alphanumeric”.)

(Aside²: While Sumerian and Babylonian scribes no doubt had astounding skills, they didn’t keep track of *sixty* unique symbols. Not even *fifty-nine*, since they lacked zero. Just two: “Y” for one and “&lt;” for ten. So 𒐘 was four and 𒐏 forty, so forty-four might be Unicodized as 𒐏𒐘?)

We will indulge the postmodern in one way: we’ll allow symbol tables that are no lexicographically-sorted. That is, the number systems we define are allowed to flout the conventions of lexicographical ordering, in which case interior strings produced by Mudder.js won’t sort. I can’t think of a case where this would be actually useful, instead of just playful, so if you think this should be banned, get in touch, but for now, caveat emptor.

### Some prefix problems

The discussion of the Roman numeral system reminds me of a subtle but important point. If `i`, `ii`, and `iii` are all valid symbols, how on earth can we tell if  (iii)<sub>Roman quaternary</sub> is

- (3)<sub>4</sub> = (3)<sub>10</sub>,
- (12)<sub>4</sub> = (6)<sub>10</sub>,
- (21)<sub>4</sub> = (9)<sub>10</sub>, or
- (111)<sub>4</sub> = (21)<sub>10</sub>?

We can’t. We cannot parse strings like `iii`, not without punctuation like spaces which splits a string into an array individual symbols.

At this stage one might recall reading about Huffman coding, or Dr El Gamal’s lecture on [prefix codes](https://en.wikipedia.org/wiki/Prefix_code) in information theory class. In a nutshell, a set of strings has the prefix property, or is prefix-free, if no string starts with another string—if no set member is *prefixed* by another set member.

I’ve decided to allow Mudder.js to parse raw strings only if the symbol table is prefix-free. If it is *not* prefix-free, then Mudder.js’s version of `parseInt` will throw an exception if fed a string—you must pass it an array, having resolved the ambiguity yourself, using punctuation perhaps.

#### Code to detect the prefix property
So, let’s write a dumb way to decide if a set or array of stringy symbols constitutes a prefix code. If any symbol is a prefix of another symbol (other than itself of course), the symbol table **isn’t** prefix-free, and we don’t have a prefix code.
~~~js
function isPrefixCode(strings) {
  // Note: we skip checking for prefixness if two symbols are equal to each
  // other. This implies that repeated symbols in the input are *silently
  // ignored*!
  for (const i of strings) {
    for (const j of strings) {
      if (j === i) { // [🍅]
        continue;
      }
      if (i.startsWith(j)) {
        return false;
      }
    }
  }
  return true;
}
~~~
As with most mundane-seeming things, there’s some subtlety here. Do you see how, at `[🍅]` above, we skip comparing the same strings?—that part’s not tricky, that’s absolutely needed. But because of this, if the input set contains *repeats*, this function will implicitly treat those repeats as the *same* symbol.
~~~js
console.log(isPrefixCode('a,b,b,b,b,b'.split(',')) ? 'prefix code!'
                                                   : 'NOT PREFIX CODE 😷');
// > prefix code!
~~~
One alternative might be to throw an exception upon detecting repeat symbols. Or: instead of comparing the strings themselves at `[🍅]`, compare indexes—this will declare sets with repeats as non-prefix-free, but that would imply that there was some sense in treating `'b'` and `'b'` as different numbers.

So the design decision here is that `isPrefixFree` ignores repeated symbols in its calculation, and assumes repeats are somehow dealt with downstream. Please write if this is the wrong decision.

Making sure it works:
~~~js
console.log(isPrefixCode('a,b,c'.split(',')));
console.log(isPrefixCode('a,b,bc'.split(',')));
// > true
// > false
~~~

#### A faster `isPrefixCode`

But wait! This nested-loop has quadratic runtime, with `N*N` string equality checks and nearly `N*N` `startsWith()`s, for an `N`-element input. Can’t this be recast as an `N*log(N)` operation, by first *sorting* the stringy symbols lexicographically (`N*log(N)` runtime), and then looping through once to check `startsWith`? Try this:
~~~js
function isPrefixCodeLogLinear(strings) {
  strings = Array.from(strings).sort(); // set->array or array->copy
  for (const [i, curr] of strings.entries()) {
    const prev = strings[i - 1]; // undefined for first iteration
    if (prev === curr) {         // Skip repeated entries, match quadratic API
      continue;
    }
    if (curr.startsWith(prev)) { // str.startsWith(undefined) always false
      return false;
    };
  }
  return true;
}
~~~
It was a bit of wild intuition to try this, but it has been confirmed to work: proof by internet, courtesy of [@KWillets on Computer Science StackExchange](http://cs.stackexchange.com/a/63313/8216).

To see why this works, recall that in [lexicographical ordering](http://docs.oracle.com/javase/8/docs/api/java/lang/String.html#compareTo-java.lang.String-), `'abc' < 'abc+anything else'`. The only way for a string to sort between a string `s` and `s + anotherString` is to be prefixed by `s` itself. This guarantees that prefixes sort adjacent to prefixeds.

(Aside: note that we’re use a lexicographical sort here just to find prefixes—our underlying symbol tables are allowed to be postmodern and lexicographically shuffled.)

But is it faster? Let’s test it on a big set of random numbers, which should be prefix-free so neither algorithm bails early:
~~~js
test = Array.from(Array(1000), () => '' + Math.random());
console.time('quad');
isPrefixCode(test);
console.timeEnd('quad');

console.time('log');
isPrefixCodeLogLinear(test);
console.timeEnd('log');
// > quad: 103.818ms
// > log: 1.758ms
~~~
Yes indeed, the log-linear approach using a sort is maybe ~100× faster than the quadratic approach using a double-loop. So let’s use the faster one:
~~~js
isPrefixCode = isPrefixCodeLogLinear;
~~~

### Symbol table object constructor
With this out of the way, let’s write our `SymbolTable` object. Recall from the examples above that it should take

- a string or an array, which uniquely maps integers to strings—since an array element can contain only a single string!—and
- optionally a map (literally a `Map`, or an object) from strings to numbers (many-to-one acceptable).

If a string→number map is provided in addition to a `B`-length array, this map ought to be checked to ensure that its values include `B` numbers from 0 to `B - 1`.

The symbol table should also remember if it’s prefix-free. If it is, parsing strings to numbers is doable. If not, strings must be split into an array of sub-strings first (using out-of-band, non-numeric punctuation, perhaps).

Without further ado:
~~~js
/* Constructor:
symbolsArr is a string (split into an array) or an array. In either case, it
maps numbers (array indexes) to stringy symbols. Its length defines the max
radix the symbol table can handle.

symbolsMap is optional, but goes the other way, so it can be an object or Map.
Its keys are stringy symbols and its values are numbers. If omitted, the
implied map goes from the indexes of symbolsArr to the symbols.

When symbolsMap is provided, its values are checked to ensure that each number
from 0 to max radix minus one is present. If you had a symbol as an entry in
symbolsArr, then number->string would use that symbol, but the resulting
string couldn't be parsed because that symbol wasn't in symbolMap.
*/
function SymbolTable(symbolsArr, symbolsMap) {
  'use strict'; // [⛈]
  if (typeof this === 'undefined') {
    throw new TypeError('constructor called as a function')
  };

  // Condition the input `symbolsArr`
  if (typeof symbolsArr === 'string') {
    symbolsArr = symbolsArr.split('');
  } else if (!Array.isArray(symbolsArr)) {
    throw new TypeError('symbolsArr must be string or array');
  }

  // Condition the second input, `symbolsMap`. If no symbolsMap passed in, make
  // it by inverting symbolsArr. If it's an object (and not a Map), convert its
  // own-properties to a Map.
  if (typeof symbolsMap === 'undefined') {
    symbolsMap = new Map(symbolsArr.map((str, idx) => [str, idx]));
  } else if (symbolsMap instanceof Object && !(symbolsMap instanceof Map)) {
    symbolsMap = new Map(Object.entries(symbolsMap)); // [🌪]
  } else {
    throw new TypeError('symbolsMap can be omitted, a Map, or an Object');
  }

  // Ensure that each integer from 0 to `symbolsArr.length - 1` is a value in
  // `symbolsMap`
  let symbolsValuesSet = new Set(symbolsMap.values());
  for (let i = 0; i < symbolsArr.length; i++) {
    if (!symbolsValuesSet.has(i)) {
      throw new RangeError(symbolsArr.length + ' symbols given but ' + i +
                           ' not found in symbol table');
    }
  }

  this.num2sym = symbolsArr;
  this.sym2num = symbolsMap;
  this.maxBase = this.num2sym.length;
  this.isPrefixCode = isPrefixCode(symbolsArr);
}
~~~

A programmatic note: around `[⛈]` we’re making sure that forgetting `new` when calling `SymbolTable` will throw an exception. It’s a simple solution to the [JavaScript constructor problem](http://raganwald.com/2014/07/09/javascript-constructor-problem.html#solution-kill-it-with-fire)

A microscopic programmatic note: at `[🌪]` we use an ES2017 function, [`Object.entries`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/entries). Node (my development environment in Atom) currently doesn’t support this, so here’s a polyfill that I need to run during development. (If you’re reading this in as an interactive document in a browser that supports `Object.entries`, you won’t have to do anything like this. If you’re using a browser that doesn’t support this, write to me and I’ll use a browser-enabled polyfill.)
~~~js
var entries = require('object.entries');
if (!Object.entries) {
  entries.shim();
}
~~~

Now.

Let’s make sure the constructor at least works:
~~~js
var binary = new SymbolTable('01');
var meals = new SymbolTable('🍌🍳☕️,🍱,🍣🍮'.split(','));
var romanQuat =
    new SymbolTable('_,I,II,III'.split(','),
                    {_ : 0, I : 1, i : 1, II : 2, ii : 2, III : 3, iii : 3});
console.log('Binary', binary);
console.log('Meals', meals);
console.log('Roman quaternary', romanQuat);
// > Binary SymbolTable {
// >  num2sym: [ '0', '1' ],
// >  sym2num: Map { '0' => 0, '1' => 1 },
// >  maxBase: 2,
// >  isPrefixCode: true }
// > Meals SymbolTable {
// >  num2sym: [ '🍌🍳☕️', '🍱', '🍣🍮' ],
// >  sym2num: Map { '🍌🍳☕️' => 0, '🍱' => 1, '🍣🍮' => 2 },
// >  maxBase: 3,
// >  isPrefixCode: true }
// > Roman quaternary SymbolTable {
// >  num2sym: [ '_', 'I', 'II', 'III' ],
// >  sym2num:
// >   Map {
// >     '_' => 0,
// >     'I' => 1,
// >     'i' => 1,
// >     'II' => 2,
// >     'ii' => 2,
// >     'III' => 3,
// >     'iii' => 3 },
// >  maxBase: 4,
// >  isPrefixCode: false }
~~~
A quick note: the quaternary Roman-numeral symbol table is indeed marked as a non-prefix-code.

### Conversion functions: numbers ↔︎ digits ↔︎ strings
We need four converters, two for numbers ↔︎ digits and two more for digits ↔︎ strings. (By numbers, I always mean positive integers in this document.) Let’s write those functions, and it should become clear what role “digits” play in this whole story.

Recall how, when we write “123”, we mean “1 * 100 + 2 * 10 + 3 * 1”. This is how positional number systems work.

To get this breakdown for any given number in base `B`, we repeatedly divide the integer by `B` and peel off the remainder each time to be a digit, giving you its digits from left to right. Here’s the idea in code:
~~~js
SymbolTable.prototype.numberToDigits = function(num, base) {
  base = base || this.maxBase;
  let digits = [];
  while (num >= 1) {
    digits.push(num % base);
    num = Math.floor(num / base);
  }
  return digits.length ? digits.reverse() : [ 0 ];
};
~~~
There’s a bit of incidental complexity here. In current JavaScript engines, [`push`ing](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/push) scalars to the end of an array is usually much faster than [`unshift`ing](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/unshift) scalars to its beginning. In my case:
~~~js
var v1 = [], v2 = [];
console.time('push');
for (let i = 0; i < 1e5; i++) {
  v1.push(i % 7907);
}
console.timeEnd('push');

console.time('unshift');
for (let i = 0; i < 1e5; i++) {
  v2.unshift(i % 7907);
}
console.timeEnd('unshift');
// > push: 5.277ms
// > unshift: 3051.876ms
~~~
So `SymbolTable.prototype.numberToDigits` calculates the left-most digit first and moves right, but `push`ing them onto the array leaves it reversed. So it reverses its final answer. It also has a special case that checks for 0.

Let’s make sure it works:
~~~js
var decimal = new SymbolTable('0123456789');
console.log(decimal.numberToDigits(123));
console.log(decimal.numberToDigits(0));
// > [ 1, 2, 3 ]
// > [ 0 ]
~~~
Let’s also make sure we don’t have any decimal/base-10 chauvinism:
~~~js
console.log(decimal.numberToDigits(123, 2), (123).toString(2));

var hex = new SymbolTable('0123456789abcdef');
console.log(hex.numberToDigits(123), (123).toString(16));
// > [ 1, 1, 1, 1, 0, 1, 1 ] '1111011'
// > [ 7, 11 ] '7b'
~~~
Note that each digit has to be `< B`, due to the modulo operation.

This makes me want to implement digits→string to get `Number.prototype.toString`-like functionality:
~~~js
SymbolTable.prototype.digitsToString = function(digits) {
  return digits.map(n => this.num2sym[n]).join('');
};
~~~
This function doesn’t is independent of what base to operate on. It’s just blindly replacing numbers with strings using the one-to-one `SymbolTable.num2sym` array.

Confirming it works by going from number→digits→string:
~~~js
console.log(decimal.digitsToString(decimal.numberToDigits(123)));
console.log(hex.digitsToString(hex.numberToDigits(123)));
// > 123
// > 7b
~~~

Let’s just work backwards from strings to digits. We’ll build a big regular expressions to peel off each symbol if the symbol table is prefix-free. If it’s not, the “string” must actually be an array.
~~~js
SymbolTable.prototype.stringToDigits = function(string) {
  if (!this.isPrefixCode && typeof string === 'string') {
    throw new TypeError(
        'parsing string without prefix code is unsupported. Pass in array of stringy symbols?');
  }
  if (typeof string === 'string') {
    const re = new RegExp('(' + this.num2sym.join('|') + ')', 'g');
    string = string.match(re);
  }
  return string.map(symbol => this.sym2num.get(symbol));
};
~~~
Again, this operation is independent of the base. It’s just a table lookup, and involves no arithmetic.
~~~js
console.log(decimal.stringToDigits('123'));
console.log(decimal.stringToDigits('123'.split('')));
// > [ 1, 2, 3 ]
// > [ 1, 2, 3 ]
~~~

Finally, we achieve `parseInt`-parity with the digits→number converter. Each element in the digits array is multiplied by a power of base `B` and summed. In code:
~~~js
SymbolTable.prototype.digitsToNumber = function(digits, base) {
  base = base || this.maxBase;
  let currBase = 1;
  return digits.reduceRight((accum, curr) => {
    let ret = accum + curr * currBase;
    currBase *= base;
    return ret;
  }, 0);
};
~~~
A programmatic note: I used `Array.prototype.reduceRight` to loop from the *end* of `digits` to the beginning and avoid manual management of the index-to-power relationship. Also, this let me replace an expensive `Math.pow` call each iteration with a cheap multiply.

Let’s test it, both with 123 = 0x7B (hexadecimal base-16 numbers are commonly prefixed by `0x`):
~~~js
console.log(decimal.digitsToNumber([1, 2, 3]), hex.digitsToNumber([7, 11]));
// > 123 123
~~~
We can trivially write non-stop number↔︎string functions:
~~~js
SymbolTable.prototype.numberToString = function(num, base) {
  return this.digitsToString(this.numberToDigits(num, base));
};
SymbolTable.prototype.stringToNumber = function(num, base) {
  return this.digitsToNumber(this.stringToDigits(num), base);
};
~~~
With these, `SymbolTable` is `parseInt` and `Number.prototype.toString` super-charged.

Now for some silly fun.
~~~js
var oda = new SymbolTable('天下布武');
var meals = new SymbolTable('🍌🍳☕️,🍱,🍣🍮'.split(','));
var base62 = new SymbolTable(
    '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz');
var kangxi = `一丨丶丿乙亅二亠人儿入八冂冖冫几凵刀力勹匕匚匸十卜卩厂厶又口囗土士
              夂夊夕大女子宀寸小尢尸屮山川工己巾干幺广廴廾弋弓彐彡彳心戈戶手支攴
              文斗斤方无日曰月木欠止歹殳毋比毛氏气水火爪父爻爿片牙牛犬玄玉瓜瓦甘
              生用田疋疒癶白皮皿目矛矢石示禸禾穴立竹米糸缶网羊羽老而耒耳聿肉臣自
              至臼舌舛舟艮色艸虍虫血行衣襾見角言谷豆豕豸貝赤走足身車辛辰辵邑酉釆
              里金長門阜隶隹雨青非面革韋韭音頁風飛食首香馬骨高髟鬥鬯鬲鬼魚鳥鹵鹿
              麥麻黃黍黑黹黽鼎鼓鼠鼻齊齒龍龜龠`.replace(/\s/g, '');
var rad = new SymbolTable(kangxi);

var v = [ 0, 1, 9, 10, 35, 36, 37, 61, 62, 63, 1945 ];
var v2 = [ 2e3, 2e4, 2e5, 2e6, 2e7, 2e8, 2e9 ];
console.log(v.map(x => [x, rad.numberToString(x), base62.numberToString(x),
                        oda.numberToString(x), meals.numberToString(x)]
                           .join(' ')));
console.log(v2.map(
    x => [x, rad.numberToString(x), base62.numberToString(x)].join(' ')));
// > [ '0 一 0 天 🍌🍳☕️',
// >  '1 丨 1 下 🍱',
// >  '9 儿 9 布下 🍱🍌🍳☕️🍌🍳☕️',
// >  '10 入 A 布布 🍱🍌🍳☕️🍱',
// >  '35 夕 Z 布天武 🍱🍌🍳☕️🍣🍮🍣🍮',
// >  '36 大 a 布下天 🍱🍱🍌🍳☕️🍌🍳☕️',
// >  '37 女 b 布下下 🍱🍱🍌🍳☕️🍱',
// >  '61 戈 z 武武下 🍣🍮🍌🍳☕️🍣🍮🍱',
// >  '62 戶 10 武武布 🍣🍮🍌🍳☕️🍣🍮🍣🍮',
// >  '63 手 11 武武武 🍣🍮🍱🍌🍳☕️🍌🍳☕️',
// >  '1945 儿勹 VN 下武布下布下 🍣🍮🍣🍮🍌🍳☕️🍌🍳☕️🍌🍳☕️🍌🍳☕️🍱' ]
// > [ '2000 儿木 WG',
// >  '20000 犬甘 5Ca',
// >  '200000 乙殳老 q1o',
// >  '2000000 尸行隶 8OI4',
// >  '20000000 丶人貝黑 1Luue',
// >  '200000000 匕父小玄 DXB8S',
// >  '2000000000 黽几黃水 2BLnMW' ]
~~~
Here, I made a few whimsical number systems to rewrite various interesting numbers in:

- a quaternary radix-4 number system based on Oda Nobunaga’s motto circa late-1560s, <ruby>天下<rp>(</rp><rt>tenka</rt><rp>)</rp>布武<rp>（</rp><rt>fubu</rt><rp>)</rp></ruby> (with 天下 meaning ‘all under heaven’ and 布武 roughly meaning ‘military order’).
- The epicurean ternary radix-3 number system using the day’s meals: 🍌🍳☕️ for breakfast, 🍱 for lunch, and 🍣🍮 for dinner.
- Base-62, using `0-9A-Za-z`, which is actually quite reasonable for database keys.
- a radix-214 number system using all 214 radicals of Chinese as promulgated in the 1716 Dictionary ordered by the Kangxi Emperor. Two billion, instead of ten digits in base-10, is rendered using just four radicals: 黽几黃水, traditionally meaning *frog, table, yellow, water*. Perhaps the next Joshua Foer will fashion this into a memory system for memory championships.

## Arithmetic on digits
In the previous section, we added methods to the `SymbolTable` object to convert positive integers ↔︎ digits ↔︎ strings, using the stringy symbols contained in the object, and a given radix `B`. By “digits” we meant an array of plain JavaScript numbers between 0 and `B - 1`. From this digits array you can:

- recover the number by multiplying sucessive digits with successive powers of `B` and summing, so `[1, 2, 3]` is 123 in base-10 but in hexadecimal base-16, 0x123 is 291;
- create a long string by mapping each digit to a unique stringy symbol, which is independent of base: `[1, 2, 3]` ↔︎ `'123'` using our Arabic symbols or `'一二三'` using Chinese symbols.

Now.

Here’s the way to get strings *between* two given strings.

1. Convert both strings to digits.
2. Instead of treating the sequence of digits as a number with the radix-point to the *right* of the last digit, let’s pretend that the radix point was to the *left of the first digit*. This gives you two numbers both between 0 and 1.
3. Still using the digits array, calculate their average. This new array of digits is readily mapped to a string that will be lexicographically “between” the original two strings.

This might seem confusing! Arbitrary! Over-complicated! But I think every piece of this scheme is necessary and as simple as possible to achieve the desiderata at the top of this document.

**Stupid example** Consider the base `B = 10` decimal system, and two strings, `'2'` and `'44'`. These strings map to digits `[2]` and `[4, 4]` respectively (and also to the numbers 2 and 44—stupid example to get you comfortable).

Instead of these two strings representing integers with the radix-point (decimal point) after them, shift your perspective so that the radix-point is on the left:

1. Not 2, but 0.2 = 2 ÷ 10.
1. Not 44, but 0.44 = 44 ÷ 100.

In other words, pretend both numbers have been divided by base `B` until they first drop below 1.

Why do this? Just look at the mean of these fractions: `(0.2 + 0.44) / 2 = 0.32`. Move the decimal point in 0.32 to the end to get an integer, 32. Map 32 to a string: `'32'`. Look at that: `'2' < '32' < '44'`.

The mean of two numbers comes from splitting the interval between them into two pieces. You can get `N` numbers by splitting the interval into `N + 1` pieces: `0.2 + (0.44 - 0.2) / (N + 1) * i` for `i` running from 1 to `N`. For `N = 5`, these are:
~~~js
var N = 5, a = 0.2, b = 0.44;
var intermediates =
    Array.from(Array(N), (_, i) => a + (b - a) / (N + 1) * (i + 1));
console.log(intermediates);
// > [ 0.24000000000000002, 0.28, 0.32, 0.36, 0.4 ]
~~~
Ignoring floating-point-precision problems, `'2' < '24' < '28' < '32' < '36' < '40' < '44'`.

The same idea works for bases other than `B = 10`. It’s just that adding and dividing becomes a *bit* more complicated in non-decimal bases. Let’s write routines that add two digits arrays, and that can divide a digit array by a scalar, both in base-`B`.

### Adding digits arrays
Let’s re-create the scene. We started with two strings. We use a symbol table containing `B` entries to convert the strings to two arrays of digits, each element of which is a regular JavaScript integer between 0 and `B - 1`. *We are going to pretend that the digits have a radix-point before the first element* and we want to *add* the two sets of digits.

Recall long addition from your youth. You add two base-10 decimal numbers by

1. lining up the decimal point,
1. adding the two rightmost numbers (filling in zeros if one number has fewer decimal places than the other),
1. then moving left,
1. taking care to carry the tens place of a sum if it was ≥10.

**Example base-10** Let’s add 0.12 + 0.456:
```
  0.12
+ 0.456
  -----
  0.576     ⟸ found 6 first, then 7, then finally 5.
```

An example complicated by carries: 0.12 + 0.999:
```
  [1 1]     ⟸ carries
  0.12
+ 0.999
  -----
  1.119    ⟸ found 9 first, then to the left
```

**Example base-16** Here’s a hexadecimal example: 0x0.12 + 0x0.9ab. Recall that the “0x” in the beginning tells you the following number is base-16, its digits going from 0 to ‘f’=15, so `0x1 + 0x9 = 0xa`, and `0xf + 0x1 = 0x10`. Other than that, it’s the same long-addition algorithm:
```
  0.12
+ 0.9ab
  -----
  0.acb    ⟸ found 0xb first, then 0xc, then finally 0xa
```
Let’s check that this is right:
~~~js
console.log((0x12 / 0x100 + 0x9ab / 0x1000).toString(16));
// > 0.acb
~~~

Here’s an example with carries, 0x0.12 + 0x0.ffd:
```
 [1 1]     ⟸ hexadecimal carries
  0.12
+ 0.ffd
  -----
  1.11d    ⟸ found 0xd first, then to the left
```
Checking this too:
~~~js
console.log((0x12 / 0x100 + 0xffd / 0x1000).toString(16));
// > 1.11d
~~~
The carry digit will be either 0 or 1. Why? Because the biggest carry would come from adding the biggest digits: `(B - 1) + (B - 1) = (1) * B + (B - 2) * 1` which would be written with two digits, 1 and `B - 2`. In hexadecimal, this means `0xf + 0xf = 0x1e = 16 + 14 = 30 ✓`. So if you had a column in long-addition of 0xf, 0xf, and a carry of 0x1, the sum will be 0x1f, and you’d write ‘f’ underneath the line and carry that ‘1’ to the column to the left.

**Code** Thinking about code to long-add two arrays of digits, assuming the radix-point to the left of the first element of both, and where each digit is a JavaScript integer between 0 and `B - 1`, I wanted to get three things right: (1) determining when a carry happens—when the sum of two elements was `≥B`; (2) tracking the carry as it moved leftwards; and (3) handling arrays of different lengths.

How do I want to deal with arrays of differing lengths? In the examples above, when a column lacked a number from one of the summands, we pretended it was zero. One option could be to pad a shorter digits array with zeros. But that’s just equivalent to *copying* the trailing elements of the longer array to the result array. My plan is to *make a copy* of the longer array, then update its elements with the result of adding each digit from the shorter array. Because we have to work from the *ends* of both arrays to the beginning, we’ll use `Array.prototype.reduceRight` again:
~~~js
function longAdd(long, short, base) {
  if (long.length < short.length) {
    [long, short] = [ short, long ];
  }
  let carry = false, sum = long.slice(); // `sum` = copy of `long`
  short.reduceRight((_, shorti, i) => {
    const result = shorti + long[i] + carry;
    carry = result >= base;
    sum[i] = carry ? result - base : result;
  }, null);
  return {sum, carry};
};
~~~
Programming note: I used `reduceRight`, a very functional-programming-y technique, in a very mutable way above, essentially as a `for`-loop, except `reduceRight` keeps track of the indexing starting at the end of arrays.

I use a single boolean to indicate whether there’s a carry digit. It’s returned, along with a new array of digits representing the sum. Just like long-addition by hand, the radix-point is to the left of the `sum` array of digits—but, again just like long-addition by hand, if the final `carry` is true, there’s a “1” to the left of that radix point!

An example will help clear this up. Again, I’d like to emphasize that, for purposes of this base-`B` long-addition, an array of digits, like `[1, 2, 3]` or `[4]`, represents the number (0.123)<sub>`B`</sub> or (0.4)<sub>`B`</sub> respectively. In decimal base-10, (0.123)<sub>10</sub> + (0.4)<sub>10</sub> = (0.523)<sub>10</sub>. We also check (0.123)<sub>5</sub> + (0.4)<sub>5</sub> in quinary, base-5:
~~~js
console.log([
  longAdd([ 1, 2, 3 ], [ 4 ], 10),
  longAdd([ 4 ], [ 1, 2, 3 ], 10),
]);
console.log([
  longAdd([ 1, 2, 3 ], [ 4 ], 5), longAdd([ 4 ], [ 1, 2, 3 ], 5),
  (parseInt('123', 5) / parseInt('1000', 5) + 4 / parseInt('10', 5))
      .toString(5)
      .slice(0, 10)
]);
// > [ { sum: [ 5, 2, 3 ], carry: false },
// >  { sum: [ 5, 2, 3 ], carry: false } ]
// > [ { sum: [ 0, 2, 3 ], carry: true },
// >  { sum: [ 0, 2, 3 ], carry: true },
// >  '1.02300000' ]
~~~
Minor programming note: addition in JavaScript floating-point and converting to base-5 using `Number.prototype.toString` for the check above results in a very long string due to numerical issues, so I truncate the result to 10 characters. Nonetheless, we confirm that (0.123)<sub>5</sub> + (0.4)<sub>5</sub> = (1.023)<sub>5</sub>.

## Misc…
Please don’t write code like the above, with chained `map`–`reduce`–`findIndex` insanity and quadratic searches—I’ve been thinking about these things for a bit and just wanted to throw something together. Here’s a more annotated version of both `toEmoji` and `fromEmoji`:

First, let’s make a symbol table `Map` (ES2015 hash table) to go from symbols to numbers ≤`B=3`. This lets us avoid the horrible `findIndex` in `fromEmoi`.
~~~js
 var arrToSymbolMap = symbols => new Map(symbols.map((str, idx) => [str, idx]))
                                     .set('array', symbols)
                                     .set('base', symbols.length);
 console.log(arrToSymbolMap('🍌🍳☕️,🍱,🍣🍮'.split(',')));
 ~~~
The map includes a key `'array'` with value of the initial array to serve as the opposite, a mapping from numbers to symbols.

(Aside: we could have been very modern and used ES2015 `Symbol.for('array')` instead of the string `'array'` as the key.)

With this `Map` representing the symbol table, and helper functions `replaceAll` and `symbolMapToRegexp`…
~~~js
// no-hydrogen
function lexdist(a,b) {
  const minlen = Math.min(a.length, b.length);
  for (let i = 0; i < minlen; i++){
    if (a[i] !== b[i]) {
      return a[i] - b[i];
    }
  }
  return a.length - b.length;
}

var mealMap = arrToSymbolMap('🍌🍳☕️,🍱,🍣🍮'.split(','));

var num2numDigitsInBase = (num, b) =>
    Math.max(1, Math.ceil(Math.log(num + 1) / Math.log(b)));

var num2digits = (num, base) => {
  var numDigits = num2numDigitsInBase(num, base);
  var digits = Array(numDigits);
  for (let i = numDigits - 1; i >= 0; i--) {
    digits[i] = num % base;
    num = Math.floor(num / base);
  }
  return digits;
};
num2digits(3 * 3 * 3 * 3, 2);

var digits2string = (digits, smap) =>
    digits.map(n => smap.get('array')[n]).join('');

digits2string(num2digits(3 * 3 * 3 * 3, 2), arrToSymbolMap('ab'.split('')));

var string2digits = (str, smap) => {
  var re = new RegExp('(' + smap.get('array').join('|') + ')', 'g');
  return str.match(re).map(symbol => smap.get(symbol));
};

var base62 = arrToSymbolMap(
    "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz".split(''));
var az = arrToSymbolMap("abcdefghijklmnopqrstuvwxyz".split(''));
digits2string(string2digits('abcakljafs', az), az);

string2digits('baaa', az);
string2digits('bcde', az);

var digits2num = (digits, smap) => {
  var base = smap.get('base');
  return digits.reduce((prev, curr, i) =>
                           prev + curr * Math.pow(base, digits.length - i - 1),
                       0);
};
digits2num([ 1, 25 ], az);


function longSub(big, small, base) {
  var dist = lexdist(big, small);
  if (dist < 0) {
    [big, small] = [ small, big ];
  } else if (dist === 0) {
    return [ 0 ];
  }
}



function longAdd(a, b, base) {
  // sum starts out as copy of longer
  const sum = a.length < b.length ? b.slice() : a.slice();
  // short is a reference to the shorter
  const short = !(a.length < b.length) ? b : a;

  let carry = 0;
  for (let idx = short.length - 1; idx >= 0; idx--) {
    let tmp = sum[idx] + short[idx] + carry;
    if (tmp >= base) {
      sum[idx] = tmp - base;
      carry = 1;
    } else {
      sum[idx] = tmp;
      carry = 0;
    }
  }
  return {sum : sum, overflow : carry};
}

function longDiv(numeratorArr, den, base) {
  return numeratorArr.reduce((prev, curr) => {
    let newNum = curr + prev.rem * base;
    return {
      div : prev.div.concat(Math.floor(newNum / den)),
      rem : newNum % den,
      den
    };
  }, {div : [], rem : 0, den});
}

function longMean(a, b, base) {
  const {sum, overflow} = longAdd(b, a, base);

  let {div : mean, rem} = longDiv(sum, 2, base);
  if (rem) {
    mean.push(Math.ceil(base / 2));
  }
  if (overflow) {
    mean[0] += Math.floor(base / 2);
  }

  return mean;
}



var Big = require('big.js');
digits2big = (digits, base) =>
    digits.reduce((prev, curr, i) => prev.plus(
                      Big(curr).times(Big(base).pow(digits.length - i - 1))),
                  Big(0));
digits2big(string2digits('aba', az), az.get('base'));

var big2digits1 = (num, base) => {
  var numDigits =
      Math.ceil(num.c.length / Math.log10(base)) + 1; // 'emmy'+'ammy'
  var digits = Array.from(Array(numDigits), x => 0);
  for (let i = numDigits - 1; i >= 0; i--) {
    digits[i] = Number(num.mod(base));
    num = num.div(base).round(null, 0);
  }
  return digits;
};
 var big2digits = (num, base) => {
  if (num.cmp(Big(0)) === 0) {
    return [ 0 ];
  }
  var digits = [];
  while (num >= 1) {
    digits.unshift(Number(num.mod(base)));
    num = num.div(base).round(null, 0);
  }
  return digits;
};
num2digits(0, 2);
num2digits(3 * 3 * 3 * 3, 2);
big2digits(Big(3 * 3 * 3 * 3), 2)
big2digits(Big(5.5), 2);
big2digits(Big(5), 2);
big2digits(Big(1), 26);
big2digits(Big(0), 26);

var zeros = n => Array.from(Array(Math.max(0, n)), _ => 0);


var doStrings = (s1, s2, smap, approximate) => {
  var d1, d2;
  if (s1) {
    d1 = string2digits(s1, smap);
  } else {
    d1 = [ 0 ];
  }
  if (s2) {
    d2 = string2digits(s2, smap);
  } else {
    d2 = zeros(d1.length);
    d2.unshift(1);
    d1.unshift(0);
  }
  var maxLen = Math.max(d1.length, d2.length);
  if (d2.length < maxLen) {
    d2.push(...zeros(maxLen - d2.length));
  } else if (d1.length < maxLen) {
    d1.push(...zeros(maxLen - d1.length));
  }
  var base = smap.get('base');
  var b1 = digits2big(d1, base);
  var b2 = digits2big(d2, base);
  var mean = b1.plus(b2).div(2);

  var round = mean.round(null, 0);
  var remainder = mean.minus(round);

  var whole = big2digits(round, base);
  whole.unshift(...zeros(maxLen - (s2 ? 0 : 1) - whole.length));
  var withremainder = whole.concat(Number(remainder) > 0 ? Math.ceil(base / 2)
                                                         : []); // ceil for 2

  if (approximate) {
    if (lexdist(d1, d2) > 0) {
      [d2, d1] = [ d1, d2 ];
    }
    for (var i = 0; i < d1.length; i++) {
      if (d1[i] < withremainder[i]) {
        return digits2string(withremainder.slice(0, i+1), smap);
      }
    }
    for (; i < withremainder.length; i++) {
      if (withremainder[i]) {
        break;
      }
    }
    return digits2string(withremainder.slice(0, i+1), smap);
  }

  return digits2string(withremainder, smap);; // replace trailing 0s
};
doStrings('b', 'bd', az)
doStrings('ba', 'b', az)
doStrings('cat', 'doggie', az)
doStrings('doggie', 'cat', az,true)
doStrings('ammy', 'emmy', az)
doStrings('aammy', 'aally', az)
doStrings('emmy', 'ally', az)
doStrings('bazi', 'ally', az)
doStrings('b','azz',az);
doStrings('a','b',az);
doStrings(null,'b',az);
doStrings('z','b',az);
doStrings('b', null, az)
doStrings(null, null, az)
doStrings('db', 'cz', az)
doStrings('asd', 'asdb', az,true);

string2digits('wqe', az)


function doLong(s1, s2, smap, approximate) {
  var d1, d2;
  if (s1) {
    d1 = string2digits(s1, smap);
  } else {
    d1 = [ 0 ];
  }
  if (s2) {
    d2 = string2digits(s2, smap);
  } else {
    d2 = [ 1 ];
    d1.unshift(0);
  }

  var mean = longMean(d1, d2, smap.get('base'));

  while(mean[mean.length-1]===0) {
    mean.pop();
  }
  if (mean.length===0){
    throw new RangeError("Couldn't find non-empty midpoint. Did you ask for "+
    "midpoint between two empty or zero-symbol strings?")
  }

  if (approximate) {
    if (lexdist(d1, d2) > 0) {
      [d2, d1] = [ d1, d2 ];
    }
    for (var i = 0; i < mean.length; i++) {
      if ((i < d1.length && d1[i] < mean[i]) || (i >= d1.length && mean[i])) {
        break;
      }
    }
    return digits2string(mean.slice(s2 ? 0 : 1, i + 1), smap);
  }
  if (!s2) {
    mean.shift();
  }
  return digits2string(mean, smap);
}
var nums = arrToSymbolMap('0123456789'.split(''));
var bin = arrToSymbolMap('01'.split(''));
var base36 = arrToSymbolMap('0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ'.split(''));
doStrings('95', '9501', nums,true)
doLong('89', '91', nums)
doLong('89', '91', nums, true)
doLong('95', null, nums,!true)
doLong('b', null, az)
doStrings('b', null, az)
doLong('0','0',nums)

console.log(Array.from(Array(1000)).reduce((prev,curr)=>doLong('A', prev, base62, true), 'y').length)
doLong('b','y', base62)


function subdivPow2(a, b, smap, n) {
  var sols = [a , b];
  for (let i=0; i<n;i++){
    var tmp = [];
    for (let j=0;j<sols.length-1;j++) {
      tmp.push(doLong(sols[j], sols[j+1], smap, true));
    }
    sols = sols.concat(tmp)
    sols.sort();
  }
  return sols;
}

var tern = arrToSymbolMap('012'.split(''));

subdivPow2('1','11',tern, 9)
doLong('1','1000002',tern,true)
([1,2,3].concat([10,20,30])).sort()

function range0f(n, f) { return Array.from(Array(n), (_, i) => f(i)); }
function range(n) { return Array.from(Array(n), (_, i) => i); }
function empty(n) { return Array.from(Array(n)); }
function zeros(n) { return Array.from(Array(n), () => 0); }
function longAddRem(a, b, base) {
  if (a.den !== b.den) {
    throw new Error(
        'unimplemented: adding fractions of different denominators');
  }
  var res = longAdd(a.div, b.div, base);
  if (res.overflow) {
    throw new Error('unsupported: overflow add');
  }
  var remscale = Math.pow(base, Math.abs(a.div.length - b.div.length));
  var rem = a.div.length < b.div.length ? remscale * a.rem + b.rem
                                        : remscale * b.rem + a.rem;
  while (rem >= a.den) {
    rem -= a.den;
    var tmp = zeros(res.sum.length);
    tmp[tmp.length - 1] = 1;
    res = longAdd(res.sum, tmp, base);
    if (res.overflow) {
      throw new Error('unsupported: overflow add');
    }
  }
  return {div : res.sum, rem, den : a.den};
}
console.log(longAddRem({div : [ 4, 5 ], rem : 7, den : 12},
                       {div : [ 1 ], rem : 7, den : 12}, 10))
.45+(7/12)*1e-2 + .1+(7/12)*1e-1


console.log(longAddRem({div: [4,5], rem:7, den:12}, {div:[1,0], rem:7, den:12}, 10))
.45+(7/12)*1e-2 + .10+(7/12)*1e-2

console.log(longAddRem({div: [4,5,6,7,8], rem:1, den:10}, {div:[1], rem:7, den:10}, 10))
.45678+1/10*1e-5 + .1+7/10*1e-1

function roundQuotientRemainder(sol, base) {
  var places = Math.ceil(Math.log(sol.den) / Math.log(base));
  var scale = Math.pow(10, places);
  var rem = Math.round(sol.rem * scale / sol.den);
  var remDigits = num2digits(rem, base);
  return sol.div.concat(zeros(places - remDigits.length)).concat(remDigits);
}
.1 + .1/19
roundQuotientRemainder({div: [1], rem: 1, den:19}, 10)

function subdivLinear(a,b,smap, n) {
  var base = smap.get('base');
  var aN = longDiv(a, n, base);
  var bN = longDiv(b, n, base);
  var as = empty(n-1);
  var bs = empty(n-1);
  as[0] = aN;
  bs[0] = bN;
  for (var i = 1; i< n-1; i++) {
    as[i] = longAddRem(aN, as[i-1], base);
    bs[i] = longAddRem(bN, bs[i-1], base);
  }
  as.reverse();
  var res = empty(n-1);
  for (i = 0; i < n-1; i++) {
    res[i] = longAddRem(as[i], bs[i], base);
  }
  return res;
}
300/19
[3,20,101].map(x=>1+Math.ceil(Math.log10(x)))

subdivLinear([1], [2], nums, 19)

range(19).map(i=>.1 + .1/19*i).map(x=>Math.round(x*10000)/10000)
subdivLinear([1], [2], nums, 19).map(x=>roundQuotientRemainder(x, nums.get('base')))
subdivLinear([9], [1], nums, 4)

subdivLinear([2], [4, 4], nums, 4)
subdivLinear([2, 0], [4, 4], nums, 10)
subdivLinear([2], [4, 4], nums, 10).map(x=>roundQuotientRemainder(x, nums.get('base')))
subdivLinear([2, 0], [4, 4], nums, 10).map(x=>roundQuotientRemainder(x, nums.get('base')))

longDiv([2], 6, 10)
longDiv([2, 0], 6, 10)
longDiv([2,0, 0], 6, 10)

'2' < '24'
'20' < '24'
'24' < '28'
'28' < '32'
'32' < '36'
'36' < '40'
'40' < '44'
'36' < '4'
'4' < '44'
// A symbol map might contain an 'escape hatch' symbol, i.e., one that is only
// used to find midpoints between equal strings. Such an escape hatch symbol
// would be one that is not in the standard symbol list. Using this escape hatch
// would effectively increase the base of this string, and indeed all strings,
// so it would be added to the symbol list, and future midpoints ought to use
// it---if the escape hatch didn't become a regular symbol, how could the
// midpoint system translate a string containing it to a number?

'qwe' < 'qwea'
// the problem with 'ba' is that there’s no string that can go between it and 'b'. This kind of implies that they’re the same string, given a-z symbols. I mean, if two integers have no integer between them, they're the same too right? It just so happens that, in lexicographic distance, sure 'b' < 'ba', but that doesn't change the underlying fact---integer 5 and 005 don't stop being the same even though their lexicographic distance is different.

~~~

##References

Cuneiform: http://it.stlawu.edu/~dmelvill/mesomath/Numbers.html and https://en.wikipedia.org/wiki/Sexagesimal#Babylonian_mathematics and Cuneiform Composite from http://oracc.museum.upenn.edu/doc/help/visitingoracc/fonts/index.html


## Unused functions
~~~js
// no-hydrogen
function rightpad(arr, finalLength, val) {
  const padlen = Math.max(0, finalLength - arr.length);
  return arr.concat(Array(padlen).fill(val || 0));
}

function longAddxxxxxx(a, b, base) {
  if (a.length < b.length) {
    a = rightpad(a, b.length);
  } else if (b.length < a.length) {
    b = rightpad(b, a.length);
  }
  const res = a.reduceRight((accum, ai, i) => {
    const sum = b[i] + ai + accum.carry;
    return {
      sum : accum.sum.concat(sum < base ? sum : sum - base),
      carry : sum >= base
    };
  }, {sum : [], carry : 0});
  res.sum.reverse();
  return res;
}

function longAddxxx(a, b, base) {
  // sum starts out as copy of longer
  const sum = a.length < b.length ? b.slice() : a.slice();
  // short is a reference to the shorter
  const short = !(a.length < b.length) ? b : a;

  let carry = 0;
  for (let idx = short.length - 1; idx >= 0; idx--) {
    let tmp = sum[idx] + short[idx] + carry;
    if (tmp >= base) {
      sum[idx] = tmp - base;
      carry = 1;
    } else {
      sum[idx] = tmp;
      carry = 0;
    }
  }
  return {sum, overflow : carry};
};

~~~
