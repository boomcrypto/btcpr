# Bitcoin Payment Request API 0.1  Draft 01 June 2024

## Abstract

This specification standardizes an API to allow merchants (i.e. web sites selling physical or digital goods) and peers (for P2P payments) to utilize one or more payment methods with minimal integration. User agents (e.g., browsers) facilitate the payment flow between merchant/user and payer. 

## Status of This Document

This section describes the status of this document at the time of its publication. A list of current  publications and the latest revision of this technical report can be found in the BTCPR technical reports index at https://github.com/boomcrypto/BTCPR

Boom published Payment Request API 0.1 as a Proposed Recommendation in June 2024.  At this time, a team of volunteers is evaluating the proposal to determine whether the BTC Payment Request API 0.1  should advance to SIP status or if additional changes are required. In the meantime, the Editors have continued to update this specification with new features (see change log below). 

## Changes since last publication

New proposal

## Table of Contents
Abstract
Status of This Document
Changes since last publication
1. Introduction
    1.1 Goals and scope
2. Examples of usage
    2.1 Declaring multiple ways of paying
    2.2 Describing what is being paid for
    2.3 Conditional modifications to payment request
    2.4 Constructing a PaymentRequest
    2.5 POSTing payment response back to a server
    2.6 Using with cross-origin iframes
3. PaymentRequest interface
    3.1 Constructor
    3.2 id attribute
    3.3 show() method
    3.4 abort() method
    3.5 canMakePayment() method
    3.6 onpaymentmethodchange attribute
    3.7 Internal Slots
4. PaymentMethodData dictionary
5. PaymentCurrencyAmount dictionary
    5.1 Validity checkers
6. Payment details dictionaries
    6.1 PaymentDetailsBase dictionary
    6.2 PaymentDetailsInit dictionary
    6.3 PaymentDetailsUpdate dictionary
7. PaymentDetailsModifier dictionary
8. PaymentItem dictionary
9. PaymentCompleteDetails dictionary
10. PaymentComplete enum
11. PaymentResponse interface
    11.1 retry() method
        11.1.1 PaymentValidationErrors dictionary
    11.2 methodName attribute
    11.3 details attribute
    11.4 requestId attribute
    11.5 complete() method
    11.6 Internal Slots
12. Permissions Policy integration
13. Events
    13.1 Summary
    13.2 PaymentMethodChangeEvent interface
        13.2.1 methodDetails attribute
        13.2.2 methodName attribute
        13.2.3 PaymentMethodChangeEventInit dictionary
    13.3 PaymentRequestUpdateEvent interface
        13.3.1 Constructor
        13.3.2 updateWith() method
        13.3.3 Internal Slots
        13.3.4 PaymentRequestUpdateEventInit dictionary
14. Algorithms
    14.1 Can make payment algorithm
    14.2 Payment method changed algorithm
    14.3 PaymentRequest updated algorithm
    14.4 User accepts the payment request algorithm
    14.5 User aborts the payment request algorithm
    14.6 Update a PaymentRequest's details algorithm
        14.6.1 Abort the update
15. Privacy and Security Considerations
    15.1 User protections with show() method
    15.2 Secure contexts
    15.3 Cross-origin payment requests
    15.4 Encryption of data fields
    15.5 How user agents match payment handlers
    15.6 Data usage
    15.7 Exposing user information
    15.8 canMakePayment() protections
    15.9 User activation requirement
16. Accessibility Considerations
17. Dependencies
18. Conformance
A. IDL Index
B. Acknowledgements
C. Changelog
D. References
    D.1 Normative references
    D.2 Informative references

1. Introduction

This section is non-normative.

This specification describes an API that allows user agents (e.g., browsers) to act as an intermediary between three parties in a transaction:

The payee: the merchant that runs an online store, or other party that requests to be paid.

The payer: the party that makes a purchase at that online store, or makes a P2P payment, and who authenticates and authorizes payment as required.

The payment method: the means that the payer uses to pay the payee (e.g., a crypto payment). The payment method provider establishes the ecosystem to support that payment method. A payment method defines:

1. A chain id (see SLIP-44)
1. An identifier (e.g. a smart contract principal and name)
1. An optional additional data type
1. Steps to validate payment method data - Algorithmic steps that specify how a payment method validates the data member of the PaymentMethodData, after it is converted to the payment method's additional data type. If not specified for a given payment method, no validation is done.

3The details of how to fulfill a payment request for a given payment method is an implementation detail of a payment handler, which is an application or service that handles requests for payment. Concretely, a payment handler defines:

1. Steps to check if a payment can be made: How a payment handler determines whether it, or the user, can potentially "make a payment" is also an implementation detail of a payment handler.
2. Steps to respond to a payment request: Steps that return an object or dictionary that a merchant uses to process or validate the transaction. The structure of this object is specific to each payment method and chain id.

1.1 Goals and scope

Allow the user agent to act as intermediary between a merchant or user (in P2P scenarios), user, and payment method provider.
Enable user agents to streamline the user's payment experience by taking into account user preferences, merchant information, security considerations, and other factors.
Standardize (to the extent that it makes sense) the communication flow between a merchant or user, user agent, and payment method provider.
Enable a payment method provider to bring more secure payment transactions to the web.

The following are out of scope for this specification:

- Create a new payment method.

- Integrate directly with payment processors.

2. Examples of usage

This section is non-normative.

In order to use the API, the developer needs to provide and keep track of a number of key pieces of information. These bits of information are passed to the PaymentRequest constructor as arguments, and subsequently used to update the payment request being displayed to the user. Namely, these bits of information are:

The methodData: A sequence of PaymentMethodDatas that represents the payment methods that the site or user supports (e.g., "we support Bitcoin, Stacks, etc.").
The details: The details of the transaction, as a PaymentDetailsInit dictionary. This includes total cost, and optionally a list of goods or services being purchased. Additionally, it can optionally include "modifiers" to how payments are made. For example, "if you pay with a token belonging to network X, it incurs a US$3.00 processing fee".

Once a PaymentRequest is constructed, it's presented to the end user via the show() method. The show() returns a promise that, once the user confirms request for payment, results in a PaymentResponse.

2.1 Declaring multiple ways of paying

When constructing a new PaymentRequest, a merchant or user uses the first argument (methodData) to list the different ways a user can pay for things (e.g. Bitcoin, Lightning, SIP-10 tokens, etc.). More specifically, the methodData sequence contains PaymentMethodData dictionaries containing the payment method identifiers for the payment methods that the merchant or user accepts and any associated payment method specific data (e.g., which credit card networks are supported).

EXAMPLE 1: The `methodData` argument

```javascript
const methodData = [
  {
    supportedMethods: "btc",
    "chain-id": "0",
    data: {
      address: "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa",
      amount: "10000", // sats
    },
  },
  {
    supportedMethods: "lightning",
    "chain-id": "1", 
    data: {
      invoice: "lnbc1pvjluezsp5...",
    },
  },
  {
    supportedMethods: "stx",
    "chain-id": "5757",
    data: {
      principal: "SP1234567890",
      name: "stacks-token",
      amount: "100", // mSTX
    },
  },
  {
	  supportedMethods: "sip-10",
	  "chain-id": "5757",
	  data: {
	    "contract-principal": "SP123456789",
	    "contract-name": "sBTC",
	    amount: "1000" // min denomination
	  }
  }
];
```

2.2 Describing what is being paid for

When constructing a new PaymentRequest, a merchant or user uses the second argument of the constructor (details) to provide the details of the transaction that the user is being asked to complete. This includes the total of the order and, optionally, some line items that can provide a detailed breakdown of what is being paid for.

EXAMPLE 2: The `details` argument
```javascript
const details = {
  id: "super-store-order-123-12312",
  displayItems: [
    {
      label: "Sub-total",
      amount: { currency: "sBTC", value: "5500" },
    },
    {
      label: "Value-Added Tax (VAT)",
      amount: { currency: "sBTC", value: "500" },
    },
    {
      label: "Standard shipping",
      amount: { currency: "sBTC", value: "500" },
    },
  ],
  total: {
    label: "Total due",
    amount: { currency: "sBTC", value: "6500" },
  },
};

```

2.3 Conditional modifications to payment request

Here we see how to add a processing fee for using a card on a particular network. Notice that it requires recalculating the total.

EXAMPLE 3: Modifying payment request based on card type

```javascript

// Certain cards incur a $3.00 processing fee.
const btcFee = {
  label: "Bitcoin transaction fee",
  amount: { currency: "BTC", value: "30000" },
};

// Modifiers apply when the user chooses to pay with
// btc.
const modifiers = [
  {
    additionalDisplayItems: [btcFee],
    supportedMethods: "btc",
    total: {
      label: "Total due",
      amount: { currency: "BTC", value: "36500" },
    },
    data: {
      supportedNetworks: networks,
    },
  },
];

Object.assign(details, { modifiers });

```

2.4 Constructing a PaymentRequest

Having gathered all the prerequisite bits of information, we can now construct a PaymentRequest and request that the browser present it to the user:

EXAMPLE 4: Constructing a `PaymentRequest`

```javascript
async function doPaymentRequest() {
  try {
    const request = new PaymentRequest(methodData, details, options);
    const response = await request.show();
    await validateResponse(response);
  } catch (err) {
    // AbortError, SecurityError
    console.error(err);
  }
}

async function validateResponse(response) {
  try {
    const errors = await checkAllValuesAreGood(response);
    if (errors.length) {
      await response.retry(errors);
      return validateResponse(response);
    }
    await response.complete("success");
  } catch (err) {
    // Something went wrong...
    await response.complete("fail");
  }
}

// Must be called as a result of a click
// or some explicit user action.
doPaymentRequest();

```

2.5 POSTing payment response back to a server

It's expected that data in a PaymentResponse will be POSTed back to a server for processing. To make this as easy as possible, PaymentResponse can use the default toJSON steps (i.e., .toJSON()) to serializes the object directly into JSON. This makes it trivial to POST the resulting JSON back to a server using the Fetch Standard:

EXAMPLE 5: POSTing with `fetch()`

```javascript
async function doPaymentRequest() {
  const payRequest = new PaymentRequest(methodData, details, options);
  const payResponse = await payRequest.show();
  let result = "";
  try {
    const httpResponse = await fetch("/process-payment", {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: payResponse.toJSON(),
    });
    result = httpResponse.ok ? "success" : "fail";
  } catch (err) {
    console.error(err);
    result = "fail";
  }
  await payResponse.complete(result);
}
doPaymentRequest();

```

2.6 Using with cross-origin iframes

To indicate that a cross-origin iframe is allowed to invoke the payment request API, the allow attribute along with the "payment" keyword can be specified on the iframe element.

EXAMPLE 6: Using Payment Request API with cross-origin iframes

```javascript
<iframe src="https://example.com/checkout" allow="payment"></iframe>

```

3. PaymentRequest interface

```javascript
[SecureContext, Exposed=Window]
interface PaymentRequest : EventTarget {
  constructor(sequence<PaymentMethodData> methodData, PaymentDetailsInit details);
  [NewObject] Promise<PaymentResponse> show(optional Promise<PaymentDetailsUpdate> detailsPromise);
  [NewObject] Promise<void> abort();
  [NewObject] Promise<boolean> canMakePayment();
  readonly attribute DOMString id;
  attribute EventHandler onpaymentmethodchange;
};

```

3.1 Constructor

The PaymentRequest is constructed using the supplied sequence of PaymentMethodData methodData including any payment method specific data, and the PaymentDetailsInit details.

The PaymentRequest(methodData, details) constructor MUST act as follows:

1. If this's relevant global object's associated Document is not allowed to use the "payment" permission, then throw a "SecurityError" DOMException.
2. Establish the request's id: If details.id is missing, add an id member to details and set its value to a UUID [RFC4122].
3. Let serializedMethodData be an empty list.
4. Process payment methods: If the length of the methodData sequence is zero, then throw a TypeError, optionally informing the developer that at least one payment method is required.
5. Let seenPMIs be the empty set.
6. For each paymentMethod of methodData: a. Run the steps to validate a payment method identifier with paymentMethod.supportedMethods. If it returns false, then throw a RangeError exception. Optionally, inform the developer that the payment method identifier is invalid. b. Let pmi be the result of parsing paymentMethod.supportedMethods with basic URL parser: If failure, set pmi to paymentMethod.supportedMethods. c. If seenPMIs contains pmi throw a RangeError DOMException optionally informing the developer that this payment method identifier is a duplicate. d. Append pmi to seenPMIs. e. If the data member of paymentMethod is missing, let serializedData be null. Otherwise, let serializedData be the result of serialize paymentMethod.data into a JSON string. Rethrow any exceptions. f. If serializedData is not null, and if the specification that defines the paymentMethod.supportedMethods specifies an additional data type: i. Let object be the result of JSON-parsing serializedData. ii. Let idl be the result of converting object to an IDL value of the additional data type. Rethrow any exceptions. iii. Run the steps to validate payment method data, if any, from the specification that defines the paymentMethod.supportedMethods on object. Rethrow any exceptions. g. Add the tuple (paymentMethod.supportedMethods, serializedData) to serializedMethodData.
7. Process the total: Check and canonicalize total amount details.total.amount. Rethrow any exceptions.
8. If the displayItems member of details is present, then for each item in details.displayItems: Check and canonicalize amount item.amount. Rethrow any exceptions.
9. Let serializedModifierData be an empty list. Process payment details modifiers: a. Let modifiers be an empty sequence. If the modifiers member of details is present, then: Set modifiers to details.modifiers. b. For each modifier of modifiers: i. If the total member of modifier is present, then: Check and canonicalize total amount modifier.total.amount. Rethrow any exceptions. ii. If the additionalDisplayItems member of modifier is present, then for each item of modifier.additionalDisplayItems: Check and canonicalize amount item.amount. Rethrow any exceptions. iii. If the data member of modifier is missing, let serializedData be null. Otherwise, let serializedData be the result of serialize modifier.data into a JSON string. Rethrow any exceptions. iv. Add the tuple (modifier.supportedMethods, serializedData) to serializedModifierData. Remove the data member of modifier, if it is present. Set details.modifiers to modifiers.
10. Let request be a new PaymentRequest. Set request.[[handler]] to null. Set request.[[state]] to "created". Set request.[[updating]] to false. Set request.[[details]] to details. Set request.[[serializedModifierData]] to serializedModifierData. Set request.[[serializedMethodData]] to serializedMethodData. Set request.[[response]] to null. Return request.

3.2 id attribute

When getting, the id attribute returns this PaymentRequest's [[details]].id.

3.3 show() method

The show() method is called when a developer wants to begin user interaction for the payment request. The show() method returns a Promise that will be resolved when the user accepts the payment request. Some kind of user interface will be presented to the user to facilitate the payment request after the show() method returns.

```webidl
[NewObject] Promise<PaymentResponse> show(optional Promise<PaymentDetailsUpdate> detailsPromise);
```

3.4 abort() method

The abort() method is called if a developer wishes to tell the user agent to abort the payment request and to tear down any user interface that might be shown.

```webidl
[NewObject] Promise<void> abort();
```

3.5 canMakePayment() method

The canMakePayment() method can be used by the developer to determine if the user agent has support for one of the desired payment methods.

```webidl
[NewObject] Promise<boolean> canMakePayment();
```

3.6 onpaymentmethodchange attribute

A PaymentRequest's onpaymentmethodchange attribute is an EventHandler for a PaymentMethodChangeEvent named "paymentmethodchange".

3.7 Internal Slots

Instances of PaymentRequest are created with the internal slots in the following table:

|Internal Slot|Description (non-normative)|
|---|---|
|[[serializedMethodData]]|The methodData supplied to the constructor, but represented as tuples containing supported methods and a string or null for data (instead of the original object form).|
|[[serializedModifierData]]|A list containing the serialized string form of each data member for each corresponding item in the sequence [[details]].modifier, or null if no such member was present.|
|[[details]]|The current PaymentDetailsBase for the payment request initially supplied to the constructor and then updated with calls to updateWith(). Note that all data members of PaymentDetailsModifier instances contained in the modifiers member will be removed, as they are instead stored in serialized form in the [[serializedModifierData]] internal slot.|
|[[state]]|The current state of the payment request, which transitions from: "created", "interactive", "closed".|
|[[updating]]|True if there is a pending updateWith() call to update the payment request and false otherwise.|
|[[acceptPromise]]|The pending Promise created during show() that will be resolved if the user accepts the payment request.|
|[[response]]|Null, or the PaymentResponse instantiated by this PaymentRequest.|
|[[handler]]|The Payment Handler associated with this PaymentRequest. Initialized to null.|

4. PaymentMethodData dictionary

```webidl
dictionary PaymentMethodData {
  required DOMString supportedMethods;
  object data;
};
```

5. PaymentCurrencyAmount dictionary

```webidl
dictionary PaymentCurrencyAmount {
  required DOMString currency;
  required DOMString value;
};
```

5.1 Validity checkers

A JavaScript string is a valid decimal monetary value if it consists of the following code points in the given order:

Optionally, a single U+002D (-), to indicate that the amount is negative. One or more code points in the range U+0030 (0) to U+0039 (9). Optionally, a single U+002E (.) followed by one or more code points in the range U+0030 (0) to U+0039 (9).

```javascript
^-?[0-9]+(\.[0-9]+)?$
```

To check and canonicalize amount given a PaymentCurrencyAmount amount, run the following steps:

If the result of IsWellFormedCurrencyCode(amount.currency) is false, then throw a RangeError exception, optionally informing the developer that the currency is invalid. If amount.value is not a valid decimal monetary value, throw a TypeError, optionally informing the developer that the currency is invalid. Set amount.currency to the result of ASCII uppercase amount.currency.

6. Payment details dictionaries

6.1 PaymentDetailsBase dictionary

```webidl
dictionary PaymentDetailsBase {
  sequence<PaymentItem> displayItems;
  sequence<PaymentDetailsModifier> modifiers;
};
```

6.2 PaymentDetailsInit dictionary

```webidl
dictionary PaymentDetailsInit : PaymentDetailsBase {
  DOMString id;
  required PaymentItem total;
};
```

6.3 PaymentDetailsUpdate dictionary

```webidl
dictionary PaymentDetailsUpdate : PaymentDetailsBase {
  PaymentItem total;
  object paymentMethodErrors;
};
```

7. PaymentDetailsModifier dictionary

```webidl
dictionary PaymentDetailsModifier {
  required DOMString supportedMethods;
  PaymentItem total;
  sequence<PaymentItem> additionalDisplayItems;
  object data;
};
```

8. PaymentItem dictionary

```webidl
dictionary PaymentItem {
  required DOMString label;
  required PaymentCurrencyAmount amount;
  boolean pending = false;
};
```

9. PaymentCompleteDetails dictionary

```webidl
dictionary PaymentCompleteDetails {
  object? data = null;
};
```

10. PaymentComplete enum

```webidl
enum PaymentComplete {
  "fail",
  "success",
  "unknown"
};
```


11. PaymentResponse interface

```webidl
[SecureContext, Exposed=Window]
interface PaymentResponse : EventTarget {
  [Default] object toJSON();
  readonly attribute DOMString requestId;
  readonly attribute DOMString methodName;
  readonly attribute object details;
  [NewObject] Promise<void> complete(optional PaymentComplete result = "unknown", optional PaymentCompleteDetails details = {});
  [NewObject] Promise<void> retry(optional PaymentValidationErrors errorFields = {});
};
```

11.1 retry() method

```webidl
[NewObject] Promise<void> retry(optional PaymentValidationErrors errorFields = {});
```

11.1.1 PaymentValidationErrors dictionary

```webidl
dictionary PaymentValidationErrors {
  DOMString error;
  object paymentMethod;
};
```

methodName attribute

The payment method identifier for the payment method that the user selected to fulfill the transaction.

11.3 details attribute

An object or dictionary generated by a payment method that a merchant or user can use to process or validate a transaction (depending on the payment method).

11.4 requestId attribute

The corresponding payment request id that spawned this payment response.

11.5 complete() method

```webidl
[NewObject] Promise<void> complete(optional PaymentComplete result = "unknown", optional PaymentCompleteDetails details = {});
```

11.6 Internal Slots

Instances of PaymentResponse are created with the internal slots in the following table:

|Internal Slot|Description (non-normative)|
|---|---|
|[[complete]]|Is true if the request for payment has completed, or false otherwise.|
|[[request]]|The PaymentRequest instance that instantiated this PaymentResponse.|
|[[retryPromise]]|Null, or a Promise that resolves when a user accepts the payment request.|

12. Permissions Policy integration

This specification defines a policy-controlled feature identified by the string "payment" [permissions-policy]. Its default allowlist is 'self'.

13. Events

13.1 Summary

This section is non-normative.

|Event name|Interface|Dispatched when…|Target|
|---|---|---|---|
|paymentmethodchange|PaymentMethodChangeEvent|The user chooses a different payment method|PaymentRequest|

13.2 PaymentMethodChangeEvent interface

```webidl
[SecureContext, Exposed=Window]
interface PaymentMethodChangeEvent : PaymentRequestUpdateEvent {
  constructor(DOMString type, optional PaymentMethodChangeEventInit eventInitDict = {});
  readonly attribute DOMString methodName;
  readonly attribute object? methodDetails;
};
```

13.2.1 methodDetails attribute

When getting, returns the value it was initialized with. See methodDetails member of PaymentMethodChangeEventInit for more information.

13.2.2 methodName attribute

When getting, returns the value it was initialized with. See methodName member of PaymentMethodChangeEventInit for more information.

13.2.3 PaymentMethodChangeEventInit dictionary

```webidl
dictionary PaymentMethodChangeEventInit : PaymentRequestUpdateEventInit {
  DOMString methodName = "";
  object? methodDetails = null;
};
```

13.3 PaymentRequestUpdateEvent interface

```webidl
[SecureContext, Exposed=Window]
interface PaymentRequestUpdateEvent : Event {
  constructor(DOMString type, optional PaymentRequestUpdateEventInit eventInitDict = {});
  undefined updateWith(Promise<PaymentDetailsUpdate> detailsPromise);
};
```

13.3.1 Constructor

The PaymentRequestUpdateEvent's constructor(type, eventInitDict) MUST act as follows:

```webidl
constructor(DOMString type, optional PaymentRequestUpdateEventInit eventInitDict = {});
```

13.3.2 updateWith() method

```webidl
[NewObject] Promise<void> updateWith(Promise<PaymentDetailsUpdate> detailsPromise);
```

13.3.3 Internal Slots

Instances of PaymentRequestUpdateEvent are created with the internal slots in the following table:

|Internal Slot|Description (non-normative)|
|---|---|
|[[waitForUpdate]]|A boolean indicating whether an updateWith()-initiated update is currently in progress.|

13.3.4 PaymentRequestUpdateEventInit dictionary

```webidl
dictionary PaymentRequestUpdateEventInit : EventInit {};

```

14. Algorithms

When the internal slot [[state]] of a PaymentRequest object is set to "interactive", the user agent will trigger the following algorithms based on user interaction.

14.1 Can make payment algorithm

The can make payment algorithm checks if the user agent supports making payment with the payment methods with which the PaymentRequest was constructed.

```javascript
let request = new PaymentRequest(methodData, details);
request.canMakePayment().then((result) => {
  if (result) {
    // User agent supports the payment methods
  } else {
    // User agent does not support the payment methods
  }
});
```

14.2 Payment method changed algorithm

A payment handler MAY run the payment method changed algorithm when the user changes payment method with methodDetails, which is a dictionary or an object or null, and a methodName, which is a DOMString that represents the payment method identifier of the payment handler the user is interacting with.

14.3 PaymentRequest updated algorithm

The PaymentRequest updated algorithm is run by other algorithms above to fire an event to indicate that a user has made a change to a PaymentRequest called request with an event name of name:

14.4 User accepts the payment request algorithm

The user accepts the payment request algorithm runs when the user accepts the payment request and confirms that they want to pay. It MUST queue a task on the user interaction task source to perform the following steps:

14.5 User aborts the payment request algorithm

The user aborts the payment request algorithm runs when the user aborts the payment request through the currently interactive user interface. It MUST queue a task on the user interaction task source to perform the following steps:

14.6 Update a PaymentRequest's details algorithm

The update a PaymentRequest's details algorithm takes a PaymentDetailsUpdate detailsPromise, a PaymentRequest request, and pmi that is either a DOMString or null (a payment method identifier). The steps are conditional on the detailsPromise settling. If detailsPromise never settles then the payment request is blocked. The user agent SHOULD provide the user with a means to abort a payment request. Implementations MAY choose to implement a timeout for pending updates if detailsPromise doesn't settle in a reasonable amount of time. In the case where a timeout occurs, or the user manually aborts, or the payment handler decides to abort this particular payment, the user agent MUST run the user aborts the payment request algorithm.

14.6.1 Abort the update

To abort the update with a PaymentRequest request and exception exception:

15. Privacy and Security Considerations

15.1 User protections with show() method

This section is non-normative.

15.2 Secure contexts

This section is non-normative.

15.3 Cross-origin payment requests

This section is non-normative.

15.4 Encryption of data fields

This section is non-normative.

15.5 How user agents match payment handlers

This section is non-normative.

15.6 Data usage

Payment method owners establish the privacy policies for how user data collected for the payment method may be used.

15.7 Exposing user information

The user agent MUST NOT share information about the user with a developer without user consent.

15.8 canMakePayment() protections

The canMakePayment() method provides feature detection for different payment methods.

15.9 User activation requirement

If the user agent does not require user activation as part of the show() method, some additional security mitigations should be considered.

16. Accessibility Considerations

This section is non-normative.

17. Dependencies

This specification relies on several other underlying specifications.

18. Conformance

As well as sections marked as non-normative, all authoring guidelines, diagrams, examples, and notes in this specification are non-normative. Everything else in this specification is normative.

A. IDL Index
```webidl
[SecureContext, Exposed=Window]
interface PaymentRequest : EventTarget {
  constructor(sequence<PaymentMethodData> methodData, PaymentDetailsInit details);
  [NewObject] Promise<PaymentResponse> show(optional Promise<PaymentDetailsUpdate> detailsPromise);
  [NewObject] Promise<void> abort();
  [NewObject] Promise<boolean> canMakePayment();
  readonly attribute DOMString id;
  attribute EventHandler onpaymentmethodchange;
};

dictionary PaymentMethodData {
  required DOMString supportedMethods;
  object data;
};

dictionary PaymentCurrencyAmount {
  required DOMString currency;
  required DOMString value;
};

dictionary PaymentDetailsBase {
  sequence<PaymentItem> displayItems;
  sequence<PaymentDetailsModifier> modifiers;
};

dictionary PaymentDetailsInit : PaymentDetailsBase {
  DOMString id;
  required PaymentItem total;
};

dictionary PaymentDetailsUpdate : PaymentDetailsBase {
  PaymentItem total;
  object paymentMethodErrors;
};

dictionary PaymentDetailsModifier {
  required DOMString supportedMethods;
  PaymentItem total;
  sequence<PaymentItem> additionalDisplayItems;
  object data;
};

dictionary PaymentItem {
  required DOMString label;
  required PaymentCurrencyAmount amount;
  boolean pending = false;
};

dictionary PaymentCompleteDetails {
  object? data = null;
};

enum PaymentComplete {
  "fail",
  "success",
  "unknown"
};

[SecureContext, Exposed=Window]
interface PaymentResponse : EventTarget {
  [Default] object toJSON();
  readonly attribute DOMString requestId;
  readonly attribute DOMString methodName;
  readonly attribute object details;
  [NewObject] Promise<void> complete(optional PaymentComplete result = "unknown", optional PaymentCompleteDetails details = {});
  [NewObject] Promise<void> retry(optional PaymentValidationErrors errorFields = {});
};

dictionary PaymentValidationErrors {
  DOMString error;
  object paymentMethod;
};

[SecureContext, Exposed=Window]
interface PaymentMethodChangeEvent : PaymentRequestUpdateEvent {
  constructor(DOMString type, optional PaymentMethodChangeEventInit eventInitDict = {});
  readonly attribute DOMString methodName;
  readonly attribute object? methodDetails;
};

dictionary PaymentMethodChangeEventInit : PaymentRequestUpdateEventInit {
  DOMString methodName = "";
  object? methodDetails = null;
};

[SecureContext, Exposed=Window]
interface PaymentRequestUpdateEvent : Event {
  constructor(DOMString type, optional PaymentRequestUpdateEventInit eventInitDict = {});
  undefined updateWith(Promise<PaymentDetailsUpdate> detailsPromise);
};

dictionary PaymentRequestUpdateEventInit : EventInit {};

