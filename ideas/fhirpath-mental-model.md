# FHIRPath: A Node-Based Processing Model

## Introduction

FHIRPath is a path-based navigation and extraction language designed for FHIR resources. 
At first glance, expressions like `Patient.name.where(use = 'official').given` might seem like simple property access. 
But understanding how FHIRPath really works may be quite tricky without proper mental model.
This is clear from ongoing community discussions, where even experienced users debate how certain expressions should behave:
* [What should it do](https://chat.fhir.org/#narrow/channel/179266-fhirpath/topic/what.20should.20it.20do.3F/with/529563311) 
* [Can we chain iif from left side](https://chat.fhir.org/#narrow/channel/179266-fhirpath/topic/Can.20we.20chain.20iif.20from.20left.20side.3F/with/529625685)

To cut through this complexity, this guide will help you build an accurate mental model of how FHIRPath works by introducing "stream processing" model and walk 
through the language by examples.


## Meet Our Patient: Sarah Smith

To make the concepts concrete, we’ll use a single Patient resource as our running example throughout this guide:

```json
{
  "resourceType": "Patient",
  "id": "sarah-smith",
  "active": true,
  "name": [
    {
      "use": "official",
      "family": "Smith",
      "given": ["Sarah", "Jane"]
    },
    {
      "use": "nickname",
      "given": ["SJ"]
    }
  ],
  "gender": "female",
  "birthDate": "1985-08-15",
  "address": [
    {
      "use": "home",
      "city": "Boston",
      "state": "MA",
      "postalCode": "02101"
    }
  ]
}
```

We’ll keep coming back to this patient — Sarah Smith — as we explore how different nodes handle real data step by step.


## Input, Context, and Arguments

Before diving into nodes, it helps to know the three things every node works with. 

### What is Input?

A key rule in FHIRPath is:
all input is treated as a collection — even if it looks like a single value.

It can be:

- empty ({ }, meaning “no value” or “unknown”)
- a singleton ([1])
- ordered ([1, 2, 3] vs [3, 2, 1])
- non-unique ([1, 1, 2])
- mixed-type (children() can return different resource types)

Example:

['Sarah'] and ['Sarah', 'Jane'] are both collections, just with different lengths.

### What is Context?

Context is the “workspace” in which evaluation happens. Unlike input, which changes as you move through the data, context often carries useful constants or shortcuts. 

It contains:
- **Variables**: Named values available during evaluation
- **Special variables**: like `$this`, `$index`, and `$total`
- **Standard FHIR variables**: such as `%context`, `%resource`
- **Environment**: External data like terminology servers

Sometimes FHIRPath also provides extra variables for special use cases — think of these as bonus tools:

- Questionnaires (FHIR SDC): When filling out a long medical form, some answers may be automatically calculated from others. To make this possible, FHIR gives the calculation access to the entire form as an external variable, not just the current question.
- Subscriptions: When the system needs to detect changes (for example, when a patient’s record is updated), it provides both the new version of the resource (the current input) and the previous version in a variable called %previous. This allows you to compare before-and-after and act only if something has changed.

Some of the variables are especially important to understand:

- **`$this`**: Initially set to the input. It can be temporarily changed by some function nodes (select, where, etc)
- **`%context`**: The anchor to the very first input of the FHIRPath expression — it always points back to where evaluation began, no matter how deep you navigate. It is especially important when your expression drills into deeply nested structures but you still need to refer back to the original input resource.
- **`%resource`**: The resource containing the current focus. It may change during evaluation when crossing resource boundaries such as domainresource.contained, or bundle.entry.resource.
- **`%rootResource`**: The root resource, useful when working inside nested resources.

#### Example: Evaluating name.given

Let’s see context in action with our patient (we’ll use Sarah Smith throughout this guide):

```
Expression: name.given
Input: [Patient(Sarah Smith)]
Initial Context: {
  %context: [Patient(Sarah Smith)],
  %resource: [Patient(Sarah Smith)],
  %rootResource: [Patient(Sarah Smith)],
  $this: [Patient(Sarah Smith)]
}
Output: ['Sarah', 'Jane', 'SJ']
```

Here, name.given drills into the Patient’s names and collects all the given names. Notice how the context variables let you still reference the whole Patient, even though the input is now deeper inside the resource.


### What is Argument?

Arguments are the extra details you give to a function node so it knows exactly what to do. Each argument is itself another node to be evaluated. 

Example 1:

```
substring(0, 2)
```

substring is the function. The numbers 0 and 2 are argument nodes. They are evaluated first, then the function applies its logic: “start at position 0, take 2 characters.”

Example 2:

```
where(use = 'official')
```

Here the argument is a condition (use = 'official'). For each item in the input, the condition is evaluated. Only items where it returns true are kept.

## Data Flow as Collections 

Alongside context, we also need to understand how data flows. As we noted earlier, every node works with collections as input. Now let’s unpack what that really means.

A collection can take many forms:

- **It can be empty** `{ }`, representing no value/unknown
- **It can be ordered**: `[1, 2, 3]` vs `[3, 2, 1]`
- **It can contain duplicates**: `[1, 1, 2]` is valid
- **It can be singleton**: `[1]` is a collection of one element
- **It can be typed**: each element has a type
- **It can be mixed**: each element may have different type (`children()` returns mixed collection)

### The Empty Collection

The empty collection deserves special mention. { } is used to:
- Represent missing or unknown values
- Propagates through most operations, unless the function/operation indicates that an exception should be thrown with no input (not too many of these).
- Acts as "unknown" in three-valued logic. Sometimes it means the result is unknown, sometimes it forces an empty result, depending on the operator or function in use.


#### Starting with a Patient resource

Let’s see this in action with our patient:
```
Expression: name.given
Input: [Patient(Sarah Smith)]
Initial Context: {
  %context: [Patient(Sarah Smith)],
  %resource: [Patient(Sarah Smith)],
  %rootResource: [Patient(Sarah Smith)]
  $this: [Patient(Sarah Smith)]
}
Output: ['Sarah', 'Jane', 'SJ']
```
Here’s what’s happening step by step:

- Input: the entire Patient resource is the starting point.
- Context: special variables are initialized to reference that Patient.
- Expression: name.given drills down into the Patient’s names.
- Output: the result is a list of given names — Sarah, Jane, SJ.

This is your first complete cycle of input → context → expression → output.
Keep this structure in mind, because the same pattern repeats no matter how complex the FHIRPath expression becomes.


## The Core Mental Model: Everything is a Processing Node

Once you know about input, context, and arguments, the next step is to see how they come together inside nodes. In FHIRPath, every part of an expression is a node — no matter if it’s a simple value, a function call, or an operator.
The key insight that makes FHIRPath intuitive is this: 
**every part of a FHIRPath expression is a processing node with the same interface**.

```javascript
node(context, input, args) -> { output, context }
```
Nodes don’t work in isolation: they are connected in a chain, passing along both the data being processed and the surrounding context.

This simple idea of a chain of nodes carrying both data and context is the foundation for everything that follows.

### The Universal Node Interface

The important thing to keep in mind is that nodes always operate in the same pattern:
they take input, apply their logic, and hand the result — together with the current context — over to the next node.

```
                 arguments (other nodes)
                        │
                        ▼
                 ┌─────────────────┐
input         ──►│                 │──► output
(collection)     │   Processing    │    (collection)
                 │      Node       │
context       ──►│                 │──► context
(variables)      └─────────────────┘    (possibly modified)
```

Every node:
- **Receives** a collection as input
- **Receives** a context containing variables and environment
- **Has arguments** which are themselves nodes
- **Orchestrates** how and when its argument nodes are evaluated
- **Processes** the results according to its logic
- **Produces** a collection as output
- **Produces** a context (usually unchanged, but some nodes modify it)


### How Nodes Connect

A FHIRPath expression is a tree of connected nodes. This connection is what turns simple pieces into a meaningful expression. **Input** and **context** flow through these nodes:


Expression: Patient.name.given

Visually, this expression forms a tree:

```
                    ┌─────┐
                    │  .  │ (dot node - root)
                    └──┬──┘
                       │
                ┌──────┴──────────┐
                │                 │
            ┌───┴──┐          ┌───┴───┐
            │  .   │          │ given │
            └───┬──┘          └───────┘
                │
        ┌───────┴────────┐
        │                │
    ┌───┴────┐      ┌───┴──┐
    │Patient │      │ name │
    └────────┘      └──────┘
```

Here’s how the evaluation flows step by step:
1. Patient (identifier) - extracts Patient resource from the input
2. . (dot) - passes that Patient along to the next node
3. name (identifier) - extracts name field(s) from the Patient
4. . (dot) - passes those names to the next node
5. given (identifier) - extracts the given names

In this way, the nodes connect like links in a chain. Each one does a small job, and together they navigate through the resource to reach exactly the data you’re after.

### Context Propagation Patterns

Context doesn’t always behave the same way as it moves through an expression: sometimes it flows through unchanged, sometimes it’s temporarily modified, and sometimes it’s permanently altered.

There are three main patterns:

1. **Pass-through nodes** are the most common. They leave the context untouched, simply passing it from one node to the next.
   - Simple navigation: `name`, `given`
   - Simple functions: `first()`, `count()`
   - Operators: `+`, `=`, `and`

2. **Temporary context nodes** introduce extra variables while they do their work, but restore the original context afterwards.
   - `where()`: adds `$this` for each item
   - `select()`: adds `$this` for each item
   - Context is restored after processing

3. **Context-modifying nodes** change the context permanently.
   - `defineVariable()`: adds a new variable
   - Modified context flows to all subsequent nodes

In practice, implementations often clone and modify context to keep things isolated. For example, a variable defined inside a select() shouldn’t “leak” outside of it.


### Node Evaluation Control

A critical idea is that **nodes decide how their arguments' are evaluated**. This is what makes one type of node different from another.

1. **Simple nodes** (literals, identifiers) are the most straightforward. They don't have arguments at all, so there's nothing to control.
2. **Operator nodes** always evaluate both of their arguments in parallel, using the same input and context.
3. **Function nodes** can control:
   - **Whether** an argument is evaluated at all — for example, `iif()` only evaluates one branch that matches the condition.
   - **How many times** the argument is evaluated — `where()` evaluates once per item.
   - **With what context** the argument runs — `where()` and `select()` adds `$this`.
   - **In what order** arguments are evaluated — most functions go left to right.

In other words, while all nodes share the same input–process–output pattern, the evaluation strategy they apply to their arguments can vary significantly. Understanding these differences is key to predicting how a FHIRPath expression will behave.


### The Ambiguity of `Patient`

When you see `Patient` in FHIRPath, remember that it can mean different things. The same command can have different results, depending on the data used.

```fhirpath
Patient.name  // What does "Patient" mean here?
```

At first glance this looks simple, but it could mean two very different things:
1. **Field navigation** - looking for a field named "Patient"
2. **Type filter** - filtering to only Patient resources

Which interpretation applies depends on the input data and the data model.

Examples of the ambiguity:
```fhirpath
// Scenario 1: Input has a Patient field
// Input: {Patient: {name: [{given: ["John"]}]}}
Patient.name  → [{given: ["John"]}]  // Field navigation

// Scenario 2: Different resource type
// Input: {resourceType: "Practitioner", id: "123", name: [...]}
Patient.name  → { }  // Empty - neither field nor type match
```
This kind of ambiguity is one reason FHIRPath can be challenging, and why understanding input and context is so important.


## Node Types in FHIRPath

Now that we’ve covered the core mental model, let’s explore the main categories of nodes in FHIRPath. Each type has its own evaluation behavior and role in building expressions.

### Basic Navigation Nodes

Now let’s look at the simplest kind of nodes in FHIRPath: navigation nodes.
These are the parts of an expression that simply move you deeper into a resource by following its fields — almost like opening nested boxes inside a form.

### Identifier Nodes

An identifier node is the most direct kind of navigation. It just says: “Take the current data and show me the part with this name.”
For example, name applied to our Patient doesn’t do anything clever — it just grabs whatever is in the name field.

Processing our Patient:

```
Type: Identifier
Node: name
```

Processing our Patient:
```
Input:    [Patient(Sarah Smith)]
Context:  {initial context}
     ↓
    name
     ↓
Output:   [HumanName{use:'official'...}, HumanName{use:'nickname'...}]
Context:  {initial context} (unchanged)
```

Key points:
- Takes each item in input collection
- Extracts the 'name' field
- Flattens results into single collection and filters out empty values
- Context passes through unchanged

### The Dot Operator: Sequential Processing and Context Threading

The dot (`.`) is special - it connects nodes sequentially AND threads context. Think of it as saying: “Now that I’ve got this, go deeper and look at that.”

The dot operator does TWO critical jobs:
1. **Passes along the data**: Takes left's output as right's input
2. **Carries the context**: Passes left's context to right

This dual role is why variables work properly when you chain expressions with dots:

```fhirpath
// Variables propagate through dots
defineVariable('x', 5).name.select(%x + 1)  // Works!

// Without dots, context doesn't flow
defineVariable('x', 5) | name.select(%x + 1)  // %x is not available here
```

Here’s what the dot is actually doing:

1. Evaluates left side with original input/context
2. Takes left's output as right's input
3. **Passes left's output context to right** (crucial for variables!)
4. Returns right's output and context

### Building Navigation Chains

Now that we know the dot connects steps together, let’s see how chaining works in practice. A navigation chain is just a series of dots, each one moving a little deeper into the resource. Step by step, you drill down from the Patient to the exact detail you want.

Let's trace `name.family` (starting from a Patient resource):

Dot operator evaluates left side with original input/context, 
takes left's output as right's input and passes left's output context to right.
Returns right's output and context.

```
Step 1: name
Input:  [Patient(Sarah Smith)]
Output: [HumanName{official}, HumanName{nickname}]

Step 2: family (with dot)
Input:  [HumanName{official}, HumanName{nickname}]
Output: ['Smith'] 
```

### Navigation Examples with Our Patient

Now that we know how navigation chains work, let’s see them in action with Sarah Smith’s Patient resource. Each expression starts from the Patient and drills down step by step.

```fhirpath
// Starting from Patient resource
name           → [HumanName{official}, HumanName{nickname}]
name.use       → ['official', 'nickname']
name.given     → ['Sarah', 'Jane', 'SJ']
name.family    → ['Smith']  // Empty from nickname filtered out
active         → [true]
birthDate      → ['1985-08-15']

// Navigating deeper
address.city         → ['Boston']
address.state        → ['MA']
address.postalCode   → ['02101']
```
These examples show the power of chaining:
- name.given walks from the Patient → name array → given names.
- address.city walks from the Patient → address array → city field.

The same pattern applies no matter how deep you go — dots simply pass along the results, letting you explore the resource like following a path through folders on your computer.


## Simple Function Nodes

Functions are special nodes that do something with the data you’ve reached. Unlike navigation (which just follows fields deeper), functions can transform, count, or reshape the data.
   
### Functions Without Arguments

These functions don’t need any extra input. They just work on what you already have.

#### `first()` - Get First Element

```
Node: first()
Type: Function
```

```
Input:    ['Sarah', 'Jane', 'SJ']
Context:  {initial}
     ↓
  first()
     ↓
Output:   ['Sarah']
Context:  {initial} (unchanged)
```

This always takes the first element. If the list is empty, the result is empty too.

#### `count()` - Count Elements


```
name.count()

Input:    [Name{official}, Name{nickname}]
Output:   [2]
Type:     Integer
```

It just counts how many elements are in the collection.

### Functions With Arguments

These funtions need extra details (arguments) to know what to do.

#### `substring()` - Extract Text

```
Node: substring(start [, length])
Type: Function
```

Example:
```
name.family.substring(0, 2)

Input:    ['Smith']
Process:  substring from position 0, length 2
Output:   ['Sm']
```

```
substring(0, 2) orchestration:
1. Evaluate arg1: 0 → [0]
2. Evaluate arg2: 2 → [2]
3. Apply substring using [0] and [2]
```

How `substring` orchestrates its arguments:
1. **Evaluates arguments once** with parent `$this` as input (TODO: think more about this!!!!)
2. **Uses results** to perform `substring` operation

You can pass expressions as arguments:
```fhirpath
name.family.substring(1, name.family.length() - 2)
// Orchestration:
// 1. Evaluate arg1: 1 → [1]
// 2. Evaluate arg2: name.family.length() - 2 → [3]
// 3. Apply substring(1, 3) to input
//or
```

For example, this expression is probably not what you want:

```fhirpath
Patient.name.family.substring(1, length() - 2)
//or even this expression in Patient context:
'123456'.substring(1, length() - 2)
```
Here, `length()` is evaluated with `%context` as input (the Patient resource), not the string.
So this becomes equivalent to: `Patient.name.family.substring(1, Patient.length() - 2)`.
or even `'123456'.substring(1, Patient.length() - 2)`.

You probably need something like this:

```fhirpath
name.family.select(substring(1, length() - 2))
// which means:
name.family.select($this.substring(1, $this.length() - 2))
```
Here, `select` will set `$this` to each family string.

See [discussion](https://chat.fhir.org/#narrow/channel/179266-fhirpath/topic/what.20should.20it.20do.3F/with/529563311)

## Literal and Operator Nodes
Now that we’ve seen navigation and simple functions, let’s look at two other building blocks: literals and operators.


### Literal Nodes

Literals are constant values you place directly in an expression. It doesn’t care about the input — it always produces the same output

```
Node: 'official'
Type: String literal
```

Processing:
```
Input:    [anything]      // ignored!
Context:  {any context}   // passed through
     ↓
 'official'
     ↓  
Output:   ['official']
Context:  {any context}   // unchanged
```

FHIRPath supports several types of literals:

```fhirpath
'official'     // String
42             // Integer  
3.14           // Decimal
true           // Boolean
@2023-01-15    // Date
```

### Operator Nodes: Parallel Evaluation

Operators combine or compare values. Unlike the dot `(.)` operator, which passes output step by step, most operators evaluate their arguments in parallel with the same input and context.

#### Equality Operator

```
Node: =
Type: Binary operator
```

Let’s check if a given name equals the family name:

```
                    Input: [Name{given:['Sarah','Jane'], family:'Smith'}]
                    Context: {initial}
                         ↓
                    ┌────┴────┐
                    │    =    │
                    └────┬────┘
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
        given.first()             family
              ↓                     ↓
         ['Sarah']              ['Smith']
              └──────────┬──────────┘
                         │
                         = (comparison)
                         ↓
                     [false]
```

Key insight: Both sides get the SAME input (the Name object) and context!

### Operator Examples

With our Patient's official name:

```fhirpath
// Input: [Name{use:'official', family:'Smith', given:['Sarah','Jane']}]

// Both sides of = get the same Name input
given.first() = family     → [false]  // 'Sarah' != 'Smith'
given.last() = family      → [false]  // 'Jane' != 'Smith'

// More examples
use = 'official'           → [true]
use != 'nickname'          → [true]
given.count() > 1          → [true]
family = 'Smith'           → [true]

// Complex: still parallel evaluation
given.first() = given.last()    → [false]  // 'Sarah' != 'Jane'
                                // Both sides evaluated with same input!
```

### Operator Precedence

FHIRPath has 13 precedence levels (highest to lowest):

1. `.` (navigation)
2. `[]` (indexing) 
3. unary `+` and `-`
4. `*`, `/`, `div`, `mod`
5. `+`, `-`, `&` (string concatenation)
6. `is`, `as` (type operations)
7. `|` (union)
8. `>`, `<`, `>=`, `<=`
9. `=`, `~`, `!=`, `!~`
10. `in`, `contains`
11. `and`
12. `xor`, `or`
13. `implies`

Example showing precedence:
```fhirpath
// Parentheses show default grouping
name.use = 'official' and name.given.count() > 1
// Evaluates as:
((name.use) = 'official') and ((name.given.count()) > 1)
```

## Iterator Function Nodes: Working with Collections

Iterator functions are special: instead of working on the collection as a whole, they loop through each item one by one, adding temporary variables like `$this` and `$index` while they process. After the loop, the original context is restored.
This is what makes expressions like `where()` and `select()` so powerful.

### The `where()` Function

`where()` filters a collection based on a condition:

```
Node: where(condition)
Type: Iterator function
```

How `name.where(use = 'official')` works:

```
Input: [Name{use:'official'...}, Name{use:'nickname'...}]
Context: {initial}

The where node orchestrates its argument (use = 'official'):
- Evaluates it MULTIPLE times (once per input item)
- With DIFFERENT context each time (adds $this, $index)
- Controls the evaluation loop

For each item:
  Item 1: Name{use:'official'...}
    where creates context: {initial + $this: Name{official}, $index: 0}
    where evaluates argument: use = 'official'
    Result: [true] → Include this item
    
  Item 2: Name{use:'nickname'...}  
    where creates context: {initial + $this: Name{nickname}, $index: 1}
    where evaluates argument: use = 'official'
    Result: [false] → Exclude this item

Output: [Name{use:'official'...}]
Context: {initial}  // Original context restored!
```

Key insights:
- **where controls the loop** - the argument doesn't know it's in a loop
- **where modifies context** before each evaluation
- `$this` refers to the current item being processed
- `$index` is the current position (0-based)
- Original context is restored after processing
- Only items where condition returns true are included

### Using `$this` in where()

```fhirpath
// Find names with more than one given name
name.where($this.given.count() > 1)
// Result: [Name{given:['Sarah','Jane']...}]

// Find names where family starts with 'S'
name.where(family.substring(0,1) = 'S')
// Result: [Name{family:'Smith'...}]

// Complex condition
name.where(use = 'official' and given.exists())
// Result: [Name{use:'official', given:['Sarah','Jane']...}]
```

### The `select()` Function

`select()` transforms each item in a collection:

```
Node: select(expression)
Type: Iterator function
```

Example: `name.select(given.first() + ' ' + family)`

```
Input: [Name{given:['Sarah','Jane'], family:'Smith'}, 
        Name{given:['SJ'], family:{ }}]

For each item:
  Item 1: Name{official}
    Temporary context: {initial + $this: Name{official}}
    Evaluate: given.first() + ' ' + family
    Result: ['Sarah Smith']
    
  Item 2: Name{nickname}
    Temporary context: {initial + $this: Name{nickname}}
    Evaluate: given.first() + ' ' + family
    Result: ['SJ']  // + with empty gives 'SJ'

Output: ['Sarah Smith', 'SJ']
Context: {initial}  // Restored
```

### More Iterator Functions

#### `exists()` - Check if any match

```fhirpath
name.exists(use = 'official')
// Process each name, return true if ANY match
// Result: [true]
```

#### `all()` - Check if all match

```fhirpath
name.all(given.exists())
// Process each name, return true only if ALL have given names
// Result: [false]  // nickname has no given
```

#### `distinct()` - Remove duplicates

```fhirpath
name.use.distinct()
// Input: ['official', 'nickname']
// Output: ['official', 'nickname']  // already unique
```

### Boolean Logic and Empty Collections

FHIRPath uses three-valued logic where empty represents "unknown":

```fhirpath
// With our Patient
active and gender.exists()     → [true] (true and true)
active and deceased            → { }    (true and empty = unknown)
active or deceased             → [true] (true or empty = true)

// Truth tables
true and { }   → { }     // unknown
false and { }  → [false] // definitely false
{ } and { }    → { }     // unknown

true or { }    → [true]  // definitely true  
false or { }   → { }     // unknown
{ } or { }     → { }     // unknown

not(true)      → [false]
not(false)     → [true]
not({ })       → { }     // unknown
```

## Control Flow Nodes

### The `iif()` Function: Conditional Logic

`iif()` evaluates conditions and returns different values:

```
Node: iif(condition, trueResult, falseResult)
Type: Conditional function
```

Key behavior: **iif orchestrates conditional evaluation!**

Example:
```fhirpath
iif(name.count() > 1, 'Multiple names', 'Single name')

How iif orchestrates its three arguments:
1. ALWAYS evaluates first argument (condition): name.count() > 1 → [true]
2. Decides which branch to evaluate based on result
3. Since true, evaluates ONLY second argument: 'Multiple names'
4. NEVER evaluates third argument!
5. Returns: ['Multiple names']
```

This orchestration is different from operators:
```fhirpath
// Operator: evaluates BOTH sides
true and defineVariable('x', 5)  // Both sides evaluated!

// iif: evaluates only needed branch  
iif(true, defineVariable('x', 5), defineVariable('x', 10))  // Only first branch!
```

This lazy evaluation is crucial for:
```fhirpath
// Safe division - avoids division by zero
iif(count() > 0, total / count(), 0)

// Conditional context modification  
iif(gender = 'female', 
    defineVariable('title', 'Ms.'),
    defineVariable('title', 'Mr.'))
```

Context propagation with iif:
```fhirpath
Patient
  .iif(gender = 'female',
    defineVariable('prefix', 'Ms.'),
    defineVariable('prefix', 'Mr.'))
  .name  // The dot here receives the modified context!
  .select(%prefix + ' ' + given.first() + ' ' + family)

// The context flows: iif → dot → name → dot → select
// Result: ['Ms. Sarah Smith']
```

Chaining iif - input becomes $this:
```fhirpath
Patient.name
  .where(use = 'official')
  .iif(given.count() > 1,        // $this is the official Name
    given.join(' '),              // Multiple given names
    given.first())                // Single given name
  
// Step by step:
// 1. Patient.name → [Name{official}, Name{nickname}]
// 2. where(...) → [Name{official}]
// 3. iif gets Name{official} as input
// 4. Inside iif, $this = Name{official}
// 5. Evaluates: given.count() > 1 → true
// 6. Returns: given.join(' ') → ['Sarah Jane']
```

### The `defineVariable()` Function

`defineVariable()` permanently adds a variable to the context:

```
Node: defineVariable(name, value)
Type: Context modifier
```

Example flow:
```fhirpath
Patient
  .defineVariable('patientName', name.where(use='official').given.first())
  .address
  .select(%patientName + ' lives in ' + city)
```

Step by step:
```
1. Start with Patient
   Context: {initial}
   
2. defineVariable('patientName', ...)
   Evaluate expression: name.where(use='official').given.first() → ['Sarah']
   Context becomes: {initial + %patientName: ['Sarah']}
   Output: [Patient] (passes through input)
   
3. The dot before address is crucial!
   - Left (defineVariable) output: [Patient]
   - Left context: {initial + %patientName: ['Sarah']}
   - The dot passes BOTH to the right side
   
4. address
   Input: [Patient] (from dot)
   Context: {initial + %patientName: ['Sarah']} (from dot!)
   Output: [Address{city:'Boston'...}]
   
5. select(...)
   Context still has %patientName thanks to dot propagation!
   Result: ['Sarah lives in Boston']
```

**Key insight**: The dot operator is what allows variables defined on the left to be available on the right. Without the dot, context wouldn't propagate!

Key points:
- Variable name must start with `%`
- Variable is available to all subsequent nodes
- Context modification is permanent
- Input passes through unchanged

Chaining defineVariable - input becomes $this:
```fhirpath
Patient.name
  .where(use = 'official')
  .defineVariable('fullName', given.join(' ') + ' ' + family)
  .given  // Still operating on the Name, not Patient!

// Step by step:
// 1. Patient.name → [Name{official}, Name{nickname}]
// 2. where(...) → [Name{official}]
// 3. defineVariable gets Name{official} as input
// 4. Inside defineVariable expression, $this = Name{official}
// 5. Evaluates: given.join(' ') + ' ' + family → ['Sarah Jane Smith']
// 6. Context: adds %fullName: ['Sarah Jane Smith']
// 7. Output: [Name{official}] (passes through!)
// 8. given operates on Name{official} → ['Sarah', 'Jane']
```

Using both together:
```fhirpath
Patient
  .defineVariable('patientId', id)
  .name
  .where(use = 'official')
  .iif(exists(),
    defineVariable('hasOfficial', true),
    defineVariable('hasOfficial', false))
  .select(%patientId + ': ' + iif(%hasOfficial, given.join(' '), 'No official name'))

// Shows how:
// - defineVariable modifies context permanently
// - iif receives whatever was piped to it
// - Both can access $this from their position in the chain
```

### Combining Control Flow

Complex example using both:
```fhirpath
Patient
  .defineVariable('ageInYears', 
    today().year - birthDate.substring(0,4).toInteger())
  .defineVariable('ageGroup',
    iif(%ageInYears >= 65, 'Senior',
      iif(%ageInYears >= 18, 'Adult', 'Minor')))
  .name
  .select(given.first() + ' is ' + %ageGroup)

// Result: ['Sarah is Adult']
```

## Putting It All Together

Now let's see how all these concepts work in real scenarios.

### Example 1: Finding Contact Information

Task: "Find the home phone number for our patient's official name"

```fhirpath
Patient
  .defineVariable('officialName', 
    name.where(use = 'official').select(given.first() + ' ' + family).first())
  .telecom
  .where(system = 'phone' and use = 'home')
  .select(%officialName + ': ' + value)

// Result: ['Sarah Smith: 555-1234']
```

Breaking it down:
1. Start with Patient
2. Store the official name for later use
3. Navigate to telecom array
4. Filter for home phone
5. Combine with stored name

### Example 2: Age-Based Logic

Task: "Determine if patient needs pediatric or adult care"

```fhirpath
Patient
  .defineVariable('age', today().year - birthDate.substring(0,4).toInteger())
  .iif(%age < 18,
    'Pediatric patient: ' + name.given.first(),
    'Adult patient: ' + name.given.first() + ' (Age: ' + %age.toString() + ')')

// Result: ['Adult patient: Sarah (Age: 39)']
```

### Example 3: Complex Clinical Query

Task: "Find all official names with their cities, but only for active patients"

```fhirpath
Patient
  .where(active = true)
  .defineVariable('patient', $this)
  .name
  .where(use = 'official')
  .select(
    given.join(' ') + ' ' + family + 
    ' from ' + %patient.address.where(use = 'home').city.first()
  )

// Result: ['Sarah Jane Smith from Boston']
```

### Common Patterns

#### Pattern 1: Safe Navigation
```fhirpath
// Handle missing data gracefully
name.where(use = 'official').given.first()
  .iif(exists(), $this, 'Unknown')
```

#### Pattern 2: Aggregation
```fhirpath
// Count specific items
name.where(given.count() > 1).count()
```

#### Pattern 3: Complex Filtering
```fhirpath
// Multi-condition filtering
address.where(
  state = 'MA' and 
  city.exists() and
  postalCode.matches('^021')
)
```

#### Pattern 4: Data Transformation
```fhirpath
// Build new structures
select({
  fullName: given.join(' ') + ' ' + family,
  isOfficial: use = 'official'
})
```

## Summary: The Mental Model

1. **Everything is a node** that processes collections AND orchestrates evaluation
2. **Nodes control their arguments**:
   - Simple functions: evaluate arguments once
   - Iterators: evaluate arguments multiple times with modified context
   - Conditionals: evaluate only needed arguments
   - Operators: evaluate all arguments in parallel
3. **Context flows** through the expression tree via dots
4. **Dots create pipelines** for both data AND context
5. **Iterator functions** temporarily modify context with `$this`
6. **Empty collections** represent unknown/missing values
7. **Control flow nodes** enable complex logic through orchestration

### Evaluation Patterns Summary

| Node Type | Evaluation Strategy | Context Behavior |
|-----------|-------------------|------------------|
| **Literals** | No arguments | Pass through |
| **Identifiers** | No arguments | Pass through |
| **Operators** | All arguments in parallel | Pass through |
| **Simple Functions** | Arguments once | Pass through |
| **Iterators** | Arguments multiple times | Temporary $this/$index |
| **Conditionals** | Selected arguments only | From evaluated branch |
| **Context Modifiers** | Arguments once | Permanently modified |

With this mental model, you can:
- Read any FHIRPath expression by tracing data flow
- Understand HOW nodes control evaluation
- Debug by following input→output transformations
- Build complex queries by composing simple nodes
- Predict which code executes and when

Remember: FHIRPath nodes don't just transform data—they orchestrate the entire evaluation process. Master these concepts, and you've mastered FHIRPath!
