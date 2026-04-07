---
description: Apply Swift naming conventions and API design guidelines
user_invocable: true
---

# Skill 01 — Swift Naming Conventions & API Design

**Source:** [Swift.org API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)  
**Applies to:** All Swift code

---

## Core Principles

1. **Clarity at the point of use** is the #1 goal. Code is read far more than written.
2. **Clarity trumps brevity.** Never sacrifice readability for shorter code.
3. **Write doc comments for every public declaration.**

## Naming Rules

### Types & Protocols
- Types and protocols: `UpperCamelCase` — `ItemCategory`, `DataFetching`
- Everything else: `lowerCamelCase` — `inputText`, `fetchItems()`
- Acronyms uniformly cased: `utf8Bytes`, `isRepresentableAsASCII`

### Functions & Methods
- Name by **side effects**:
  - No side effects → noun: `distance(to:)`, `sorted()`
  - Side effects → imperative verb: `sort()`, `append(_:)`
- Mutating/nonmutating pairs: `sort()`/`sorted()`, `formUnion()`/`union()`
- Factory methods begin with `make`: `makeIterator()`
- Methods should read as grammatical English: `x.insert(y, at: z)`

### Variables & Parameters
- Name by **role**, not type: `greeting` not `string`, `service` not `dataObject`
- Booleans read as assertions: `isEmpty`, `canSubmit`, `isLoading`
- Compensate for weak types: when a parameter is `Any` or a primitive, precede with a role noun

### Protocols
- Describing "what something is": nouns — `Collection`, `UserProfile`
- Describing capability: `-able`/`-ible`/`-ing` — `Equatable`, `DataFetching`

### Argument Labels
- Include all words needed to avoid ambiguity: `remove(at: position)` not `remove(position)`
- Omit needless type-repeating words: `remove(_ member)` not `removeElement(_ member)`
- Value-preserving conversions omit the first label: `String(23)`
- Narrowing conversions use a descriptive label: `init(truncating:)`

## Mistakes to Avoid

- [ ] Using abbreviations (`idx`, `btn`, `mgr`) — spell it out
- [ ] Using `self.` when not required for disambiguation
- [ ] Force unwrapping (`!`) in production code
- [ ] Naming computed properties that are O(n) without documenting cost
- [ ] Overloading on return type alone
