# Regular Expression Linear-Time Flag for ECMAScript

## Status

Stage: 0


## Authors

Aleksandr Abashkin


## Summary

The `/l` flag enables [linear-time matching](https://arxiv.org/pdf/2311.17620) using a finite automaton engine that explores all possible matches simultaneously, eliminating catastrophic backtracking.
A read-only `Boolean` property `RegExp.prototype.linearTime` is provided to indicate whether a regular expression was constructed with `/l`.

## Motivation

Regular expressions in ECMAScript are currently implemented using backtracking engines. While very fast for most patterns, some patterns with nested quantifiers, alternations, or backreferences may trigger catastrophic backtracking, causing exponential runtime `(O(2^n))`. This behavior has been widely documented as a source of [Regular Expression Denial of Service (ReDoS)](https://en.wikipedia.org/wiki/ReDoS) vulnerabilities when processing untrusted input.

Example of a vulnerable pattern:

```js
/(a+)*b/.exec('aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa'); // Exponential runtime
```

Here, the nested quantifiers `(a+)*` combined with the absence of `b` in the input can cause the engine to explore an exponential number of backtracking paths.

Existing mitigations are inadequate:

- **Linters and static analysis tools** may detect obvious risky patterns, but cannot prevent exponential backtracking at runtime.
- **Try/catch or timeouts** do not help: the `RegEx` engine runs synchronously, so the code cannot catch a hanging match in time.
- **Third-party libraries** can provide safety but may break feature compatibility or require extra dependencies.
- **Worker threads** isolate long-running regex, but do not reduce its execution time; they only prevent blocking the main thread, and require complex coordination.

ECMAScript currently has no language-level guarantee of safe linear-time regular expression matching for all input.


## Proposed solution

The `/l` flag is opt-in and provides a guarantee of linear-time matching (`O(pattern.length × input.length)`).
Regular expressions without the flag continue to use the standard backtracking engine, ensuring full backward compatibility.

When the `/l` flag is used:

1. The regular expression MUST be executed using an algorithm that guarantees linear time complexity relative to the input length.

2. If the pattern contains features that cannot be implemented with such an algorithm (e.g., backreferences, lookahead, lookbehind), a `SyntaxError` is thrown at construction time.

3. This validation occurs at construction, not at execution time, ensuring that developers can safely detect unsupported patterns during development.

This design ensures that:
- All `/l` regular expressions are safe from ReDoS attacks.
- Behavior is consistent across all conforming implementations.
- Developers have a simple way to write secure regular expressions.
- Existing code continues to work unchanged.


## Use cases

**Literal notation**

```js
const pattern = /abc+/l;
const match = re.exec('abccc');

console.log(pattern.linearTime); // true
```

**Constructor notation**

```js
const pattern = new RegExp('abc+', 'l');
console.log(pattern.linearTime); // true
```

Unsupported pattern triggers `SyntaxError`:

```js
try {
  const pattern = /(a+)*b/l; // SyntaxError: pattern unsupported for linear matching
}
catch (error) {
  console.log('Pattern not allowed with /l');
}
```

## Syntax

```
/l // Linear-time flag
```

- Applies only to newly constructed regular expressions.
- Patterns using unsupported features (backreferences, lookaheads, lookbehinds) throw at construction.
- Existing regular expressions without `/l` continue using the standard backtracking engine.
- `RegExp.prototype.linearTime` is read-only and reflects whether `/l` was used.


## Comparison

Some programming languages provide linear-time regular expression engines with built-in protection against catastrophic backtracking:

* Go – linear-time engine is part of the standard library. All regexes run in O(pattern × input), making ReDoS impossible.
* C# – standard library supports a non-backtracking mode for most patterns (`RegexOptions.NonBacktracking`), preventing exponential backtracking.


## Implementations

V8 [provides](https://v8.dev/blog/non-backtracking-regexp) an experimental linear-time regular expression engine accessible via command-line flags (`--enable-experimental-regexp-engine`). This engine supports most JavaScript RegExp features except backreferences and lookarounds and enforces linear-time guarantees for all `/l` patterns.

```shell
node -e '/(a*)*b/l.test("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaac")' --enable-experimental-regexp-engine
```

This demonstrates that support for linear-time regex execution can be included in the standard.

## Does this proposal affect ECMAScript lexing?

No. The `/l` flag only affects user-constructed regular expressions. ECMAScript’s built-in lexer uses fixed patterns for tokenization, none of which rely on the `/l` flag or unsupported features. Therefore, existing lexing behavior remains unchanged: all code that parsed correctly before this proposal will continue to parse correctly afterward.


## Q&A

**Q**: Why is the proposal this way?<br/>
**A**: The `/l` flag provides a simple, opt-in way to guarantee linear-time regular expression matching. Without it, developers must rely on external libraries or manual pattern restrictions, which is error-prone and inconsistent across platforms.

**Q**: Why does this need to be built-in, instead of being implemented in JavaScript?<br/>
**A**: Linear-time guarantees for all patterns cannot be reliably implemented in user-space using standard JavaScript `RegExp`. Polyfills can only check patterns or reimplement engines partially, often with significant performance overhead. A native implementation ensures consistent behavior and performance across browsers and Node.js.


## Specification

[RegExp.prototype.linearTime](https://monolithed.github.io/proposal-regular-expression-linear-time-flag/)
