---
layout: post
title: "Airscript – A Programming Language Designed to Deal with Sensitive Data"
date: 2023-03-16 14:01:04 -0500
categories: blog
---

Authors: [Rishi](https://www.linkedin.com/in/rishivijayv/), [Brian](https://www.linkedin.com/in/brian-shih/), [Vishnu](https://www.linkedin.com/in/srivishnur/)

*The original blog link no longer exists after Airkit was acquired*

When developers build applications that deal with sensitive data, such as a user’s Social Security Number (SSN), they often need to abide by data regulations such as  [PII](https://www.comparitech.com/net-admin/what-is-pii-compliance/) or [PCI](https://www.nerdwallet.com/article/small-business/pci-compliance). These regulations have two main requirements:

1. The ability to respond to data requests: The system needs to be able to respond to requests about what sensitive data the system holds at any moment.
2. Compliance auditing: The system needs to have a detailed log about when/how the sensitive data is collected and when/where the sensitive data is transmitted to a third-party system.

Building an application that abides by these requirements requires a lot of manual work and overhead. It’s also error-prone to data leakages and missed audit events. At Airkit, we wanted to make it super easy for developers to build compliant applications that deal with sensitive data. To achieve that, we built our own application framework as well as our own programming language, Airscript, to enable developers to build such applications. Let’s look at the language and runtime primitives we baked into Airscript.



### Data tags

With Airscript, developers can attach data tags to runtime values. For example, to mark a piece of data as sensitive, developers can use the SET_TAG(value, PII) function.

What’s unique about Airscript’s runtime is that it automatically tracks the data tags across functions.

Let’s look at a code snippet:

```
SET_TAG(pii_var, PII);
LET a = UPPER_CASE(pii_var);
IS_PII(a) // true
```

In this example, we first set the pii_var variable to contain the PII tag. Then we pass the pii_var to an UPPER_CASE function that converts a string to upper case letters. What’s interesting is that the output of the function, a, is still tagged as PII. 

In the following example, we concatenate a PII text and a PCI text. The output of the expression is a value tagged as both PCI and PII, since the combined string contains substrings that are PII and PCI.

```
LET pii_var = SET_TAG("foo", PII);
LET pci_var = SET_TAG("bar", PCI);
LET a = pii_var + pci_var
```

If a runtime value is copied to another variable, the data tags will also be copied. In the following [LET-IN](https://support.airkit.com/reference/let-in) expression, we create two alias variables - a and b  and assign them to pii_var and pci_var respectively. The output of an expression is an object containing two key-value pairs, “foo” which is mapped to data tagged as PII and “bar”, which is mapped to data tagged as PCI. 

```
LET 
  a = pii_var, 
  b = pci_var
IN
  {"foo": a, "bar": b }
```



### Metadata Boxing

In programming languages, boxing is the process of converting a primitive type to an object type. In Java, you may choose to convert the primitive int into a boxed Integer. The exact meaning and behavior of boxing depend on the language you’re using. We developed a special form of boxing called metadata boxing for Airscript.

The core idea behind metadata boxing is that the runtime not only boxes all primitive data types, it also attaches a bag of metadata to each boxed object, called [data tags](https://support.airkit.com/docs/data-masking-and-auditing). The data tags of each boxed value can be used to store arbitrary metadata about the underlying data. In Airscript’s case, the boxed value stores the sensitivity of the data (e.g. PII, PCI, HIPAA). By having the data tags attached to the raw value, the tags will automatically be propagated alongside the data across function calls.

Airscript is compiled into JSON and interpreted by a runtime built with Typescript. Airscript has primitive types such as number, string, boolean, date, and null. But, the runtime doesn’t use Javascript primitives as the runtime representation for Airscript’s primitive data types. Instead, the runtime uses a boxed data type called AirValue. Each Airscript primitive data type has corresponding AirValue constructors which instantiate the boxed objects. These AirValue constructors create AirValue instances.

For example, instead of using Javascript’s primitive number as the runtime value, the runtime constructs a boxed object with the AirNumber constructor. Similarly, there are other constructors such as AirBoolean, AirString, AirRecord, AirList, AirNull that are used to create runtime values for other primitive data types. 

To develop a better understanding of what this means, let’s take a look at the following expression

```typescript
1 + 3
```

After being compiled, the Airscript expression is evaluated by the following code:

```typescript
AirNumber(1).plus(AirNumber(3))
```

Here, the primitive Airscript numbers 1 and 3 are both instantiated into instances of AirNumber. Note that because we don’t use Javascript primitives, we couldn’t use Javascript built-in operators like +. Instead, built our own methods that work on AirValue instances. 

Let’s look at a more complex example - Let-In.

```
LET 
  units = a + b,
  unit_price = c[0],
IN {
  "total_price": units * unit_price * 1.08,
  "units": units
}
```

Under the hood, the above Airscript expression is compiled and evaluated as follows (simplified version)

```typescript
(function(units: AirNumber, unit_price: AirNumber) {
   return AirRecord({
       "total_price": units.times(unit_price).times(AirNumber(1.08))
       "units": units
   })
})(a.plus(b), c.get(0))
```

This example is quite complex, but the takeaway is that functions in the Airscript runtime take in AirValue instances as inputs. The function body is also implemented with the API AirValue exposes. This essentially means that Airscript compiles down into lower-level AirValue methods.

Boxing Airscript primitives come with trade-offs. Primitive Javascript data types are stored on the stack while objects are stored in the heap. As a result, boxing can be slower and more memory-intensive. It also requires us to build a suite of methods for each AirValue instance to implement Airscript’s evaluator.  But, having end-to-end control over the runtime data structure unlocks data tagging, which we will cover in the next section. 



### Data Tagging Internals

Earlier, we mentioned that each AirValue instance has a bag of metadata, otherwise known as data tags. Data tagging is the process of adding a tag to an AirValue instance’s tags. 

Each AirValue instance has a setTag method to add a tag to an AirValue instance. It also has a getTag method to retrieve the tag’s value.

```typescript
const tagged = airValue.setTag("tagName", "tagValue")
const tagValue = tagged.getTag("tagName") // returns "tagValue"
```

To tag data as sensitive, developers can just call setTag  as follows

```typescript
const tagged = airValue.setTag("PII", ...)
const tagged = airValue.setTag("PCI", ...)
const tagged = airValue.setTag("HIPAA", ...)
```

Earlier, we demonstrated that AirValue instances have methods like plus, times, get, etc. All AirValue methods are implemented in a way that preserves the data tags of the AirValue instances. To showcase this, let’s look at the get method in action:

```typescript
const airValue = AirRecord({ "foo": "bar" })
const tagged = airValue.setTag("PII", "abc123")
const childAirValue = tagged.get("foo") // AirString
childAirValue.getTag("PII") // abc123
```

In this code snippet, we created an AirRecord with the value { “foo”: “bar”}. We then tag the entire object as PII. If we access the foo property, the child AirValue will also have the PII tag with the same tag value. In this example, ”abc123” is just an arbitrary string to demonstrate that a tag’s value is also propagated to its child.

Similarly, if you concatenate two AirString instances with the `concat` method, the resulting AirString instance combines the tags of the two respective AirValue instances.

```typescript
const piiString = AirString("foo").setTag("PII", 123)
const pciString = AirString("bar").setTag("HIPAA", 456)
const combined = piiString.concat(pciString) // "foobar"
combined.getTag("PII") // 123
combined.getTag("PCI") // 456
```

In the example above, we created two AirString instances and tag them as PII and HIPAA, respectively. We then concatenate the two AirString instances to yield the combined AirString instance. The combined AirString is both PII and PCI. 

In case you’re curious, here is the simplified implementation of the concat method for AirString. 

```typescript
class AirString {
    ...
    
    public concat(input: AirString): AirString {
        const rawJS = this.getRawJS() + input.getRawJS()
        const mergedTags = mergeTags(this.getTags(), input.getTags())
        return AirString(input).setTags(mergedTags)
    }
}
```

We can see here that concat merges the tags of the two AirString instances. The returned AirString instance contains the combined tags. 

Now that we have a deeper understanding of how AirValue’s APIs preserve data tags, let’s take a look at how Airscript’s evaluator uses the AirValue API. 



### More Airscript expressions!

In the following expression, assume piiVar is a variable that holds PII data and pciVar is a variable that holds PCI data.

```
"My address is {{UPPERCASE(piiVar)}} and my primary doctor is {{pciVar}}"
```

The Airscript evaluator takes the expression above and returns an AirString that is both PII and PCI. Text Interpolation is Airscript’s implementation of [Javascript’s Template String](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals). It breaks the expression down into a list of expressions that form the string: My address is , UPPERCASE(piiVar), and my primary doctor is, and pciVar. The runtime evaluates each subexpression and feeds them to the TextInterpolation evaluator. 

Here is the simplified version of Text Interpolation’s evaluator:

```typescript
function TextInterpolation(sections: AirString[]): AirString {
   return sections.reduce((acc, section) => acc.concat(section, new AirString(""))
}
```

We can see that the evaluator simply loops over the sub-text and calls the concat method that we covered in the previous section to stitch them together. Since concat retains the tags of the AirString it deals with, Text Interpolation’s evaluator will also retain the tags of its subcomponents.

UPPERCASE is a builtin Airscript function and here is a simplified version of it

```typescript
function UpperCase(airValue: AirString): AirString {
  const rawJS = airValue.getRawJS()
  const upperCased = rawJS.toLocaleUpperCase()
  const tags = airValue.getTags()
  return AirString(upperCased).setTags(tags)
}
```

As we can see here, UpperCase takes in an AirString. It extracts the rawJS with the method getRawJS. It then creates a new AirString instance with the tags from the original AirString. This is generally how built-in functions ensure that the output of a function retains the tags in a way that semantically makes sense.

These examples serve as a summary of how Airscript’s runtime works. Expressions are compiled down into lower-level AirValue methods that manipulate AirValue instances. These lower-level AirValue methods have mechanisms to ensure that tags are retained and combined. Because of this, developers who write Airscript don’t need to worry about how sensitive data flows through the runtime. The runtime will automatically propagate data tags associated with each runtime value. 



### Data Lineage

Developers want to know the lifecycle of sensitive data across the application runtime. They are interested in learning how the data was captured, how the data was transformed, and whether or not the data was sent to third-party services. 

Metadata boxing already allows us to track sensitive data across application runtime. However, we still need a way to group the audit events based on which data the audit event is related to. In Airkit’s runtime, we assign a unique tagId to each piece of sensitive data. Each time an audit event is emitted, we attach the tagId to the audit event. This way, the audit events can be grouped together by the tagId. 

The concept of tagId is inspired by distributed tracing. Distributed tracing uses traceId and spanId to trace how task execution propagates across microservices. Instead of traceId and spanId, we use tagId and parentTagIds to track how sensitive data flows across the runtime and external services. So how is tagId attached to AirValue instances? The setTag method we covered earlier automatically returns a tagged AirValue instance with a unique tagId.

The following two figures are two audit logs with the same tagId , a “Create” event when sensitive data flows into the runtime and a “Read” event when the sensitive data leaves the application runtime to a third-party service. Note that the two audit events have the same sensitiveDataTagId. 

<img src="/assets/data_tagging/sensitive_data_read.png" alt="typecheck" width="80%"/>

<img src="/assets/data_tagging/sensitive_data_create.png" alt="typecheck" width="80%"/>

### Combining sensitive data

Sometimes, we need to combine tags. For example, developers may choose to concatenate PII AirString with HIPAA AirString. In this situation, we would like the new AirString instance to somehow have a pointer to have the two tagIds that correspond to the AirValue instances it was derived from. We introduce parentTagIds, an array of tagIds from which the new AirValue instance was derived. We call tags created as a result of merging AirValue instances a “derived tag”. Let’s look at how we can generate derived tags with the following code snippet.

```javascript
const piiString1 = AirString("foo").setTag(PII, ...)
const tag1 = piiString1.getTag(PII) // { tagId: "uuid1", parentTagIds: [] }

const piiString2 = AirString("bar").setTag(PII, ...)
const tag2 = piiString2.getTag(PII) // { tagId: "uuid2", parentTagIds: [] }

const combinedString = piiString1.concat(piiString2)
const tag3 = combinedString.getTag(PII)
// { tagId: "uuid3", parentTagIds: ["uuid1", "uuid2"] }
```

As we can see here, piiString1 and piiString2 are given unique tagIds. When we concatenate the two AirString instances, the combined AirString has parentTagIds, which point to the original data that it was derived from, which is piiString1 and piiString2.

The following audit event is emitted when a PII data that is created by merging two other PII data leaves the application runtime. Highlighted is the parentTagIds that are part of the audit event.

<img src="/assets/data_tagging/parent_tag_ids.png" alt="typecheck" width="80%"/>

### Emitting Audit events

One thing we haven’t covered is how the Airscript runtime automatically emits audit events when sensitive data leaves the application runtime. 

AirValue is the runtime representation of the primitive data types in Airscript. However, when the data leaves the platform (i.e. the developer makes a POST request), the AirValue instance needs to be unwrapped into raw Javascript value to be serialized and sent across the network. This is the perfect place to automatically emit audit events because unless data is flowing to external services across the network, there is no need to unwrap AirValue instances into raw JS value.

### Storage

Airkit's application framework comes with its own state store. Developers can store variables by performing a `SET_VARIABLE(variable, value)` call such as:

```
SET_VARIABLE(session.foo, { "hello": "world" })
```

Storing application states poses a new challenge for tracking sensitive data. When the application state contains sensitive data, we need to make sure the sensitivity of the data is persisted along with the data. This way, when the persisted state is accessed by a concurrent client, the instantiated AirValue instances have the correct sensitivity tag in them.

To achieve this, we defined a new data format to store the application state.

 For example, suppose we have the following raw data, where we assume SSN  is tagged as PII.

```json
{
  "SSN": "123456":
  "pets": ["dog", "cat"]
}
```

Airscript’s runtime instead stores it as follows

```json
{
  "$$type": "object",
  "tags": {},
  "properties": {
    "ssn": {
      "$$type": "string",
      "tags": {
        "PII": { "tagId": "uuid3", "parentTagIds": ["uuid1", "uuid2"] }
      },
      "value": "123456"
    },
    "arr": {
      "$$type": "array",
      "tags": {},
      "value": [
        { "$$type": "string", "tags": {}, "value": "dog" },
        { "$$type": "string", "tags": {}, "value": "cat" }
      ]
    }
  }
}
```

This data format stores the tags of the data alongside the value. Each value has a $$type to specify the runtime value type and a tags key-value pair to store metadata about the value, such as its sensitivity. This is the same data format Airscript runtime uses to serialize/deserialize application state when it is being sent between servers and clients under the hood. 

### Conclusion

Programming language design is driven by the core values of the programming language. For example, one of Rust’s core values is memory safety. As a result, developers of Rust built ownership to govern how Rust program manages memory and enforce programmers to write memory-safe code. Because one of Airscript’s core values is to make it easy for developers to build compliant apps, we built metadata boxing as a first-class runtime primitive to track sensitive data flowing across the runtime and automatically emit audit events. 

Having end-to-end control over a programming language is filled with possibilities. It allows us to tailor the language for certain use cases, thus gaining a competitive advantage over more generic programming languages.

Special thanks to Cam Kennedy for his huge contributions on the data tagging project!
