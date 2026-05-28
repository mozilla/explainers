# MLS Explainer 

Author: [Anna Weine](https://github.com/Frosne)

Last updated: May 28, 2026

People increasingly use web applications for group communication, collaboration, and shared work. These groups often exchange sensitive information, such as messages, documents, files, comments, tasks, or project state. Users need this information to remain confidential and authentic even as group membership changes over time. When a group adds a new participant or removes an existing participant, future communication should be protected according to the updated membership.

**Building end-to-end protected group communication is difficult.** Applications need to manage cryptographic identities, group membership changes, message protection, state updates, and asynchronous delivery. There are many subtle ways for this design to fail: a missed update, an incorrectly authenticated membership change, unsafe key reuse, or inconsistent group state can silently undermine the protections users believe they have. Some applications will make these mistakes; others may avoid offering strong group protection entirely because the engineering cost and security risk are too high. In both cases, users lose.

These risks do not disappear just because an application uses an existing MLS library. The application still needs to integrate that library correctly with its identity model, storage, delivery system, application state, and user interface. That means some security-sensitive parts of group communication are still left to each web application to assemble and maintain.  A browser-provided API can reduce this burden by making secure group communication available as a common web platform capability, rather than requiring every application to build the same security-sensitive machinery around MLS independently. 

This explainer proposes a “batteries included” continuous group key agreement API based on a subset of the Messaging Layer Security protocol. The proposed API focuses on providing the basic tools necessary to secure group communication in web applications, while keeping group secrets in browser-managed state and minimizing the amount of cryptographic state management that each application needs to implement itself.


## **Alternatives**

There is currently no Web platform API for group secret establishment and end-to-end group communication. Existing security mechanisms generally address two-party or client-server communication, not shared cryptographic group state for dynamic groups. As a result, web applications that want this property usually need to build and integrate it themselves.

The Web platform does provide pieces that can be used to assemble such a system. Applications can use Web Cryptography APIs for lower-level cryptographic operations, JavaScript or WebAssembly for protocol logic, persistence mechanisms such as IndexedDB for local state, and transport APIs to exchange protocol messages between clients.

Applications can also bring an MLS implementation into the web application directly. For example, a Rust implementation such as OpenMLS can be compiled to WebAssembly and used from JavaScript. This makes it possible to run MLS-related logic in a web application today, but only as part of an application-managed design rather than as a common Web platform capability.


## **Outline of a proposed solution**

The `MLS` interface is the client-level entry point for the API. It lets an application create a browser-managed client identity for participating in MLS groups, and exposes group lifecycle operations such as creating a new group or joining an existing group.

`MLSGroupView` represents an MLS group. Applications can use this to decrypt protected messages received from other group members, produce protected application messages to send to the group, and manage group state.


## **Proposed API Usage**

The following examples illustrate how an application could use the proposed API. Method names and return types may change as the API design is refined.

In the following examples, calls on `myApp` represent application-defined operations, such as network delivery or server communication. These are not part of the proposed API.


### Preparing a client

A client first creates a local MLS identity, which includes the credential and key package needed to participate in groups. The application can then store or publish this identity through its own service so that another client can add this client to a group.

```js
const identity = await mls.createIdentity("alice@example");
await myApp.saveIdentity("alice", identity);
```

In this flow, the application receives the values it needs to participate in group setup, but the user agent manages the MLS-related state needed to continue using the identity and credential.


### Creating a group

A client can create a new MLS group from its local identity.

```js
const group = await mls.createGroup(identity);
```

This creates a group where the identified client is the only member. The returned `MLSGroupView` represents this client’s local view of the group. The application can use this object to add members, process messages, protect application data, inspect group state, and export secrets.


### Adding a member

To add another client, the application obtains that client’s key package through an application-defined mechanism:

```js
const bobIdentity = await myApp.getUserDetails("bob");
const commitOutput = await group.add(bobIdentity.keyPackage);
```

All operations on the group need to be synchronized with all other existing members of the group. So any operation that alters the state of a group returns an MLS message.  The application is responsible for distributing these messages to the group.

```js
await myApp.deliverToExistingGroupMembers(commitOutput.commit);
```

Adding a new member to a group also produces a message that needs to be sent to the new member. This message contains all the information necessary to join the group.  This message needs to be sent to the new member. 

```js
if (commitOutput.welcome) {
    await myApp.deliverWelcomeToNewMember("bob", commitOutput.welcome);
}
```


## **Joining a group**

A newly added client joins a group by processing a Welcome message delivered by the application.

```js
const welcome = await myApp.receiveWelcomeFromApplicationServer();
const joinedGroup = await mls.joinGroup(identity, welcome);
```

The `groupJoin()` operation produces an  `MLSGroupView` that represents the new client’s local view of the group. The new member (“bob”) has the same view of the group state as the creator of the group (“alice”).

The API also supports removing members from groups, using a similar pattern.

There are some cases that require the application to mediate outright conflicts — for example, when two members commit simultaneously. In such cases, the delivery service is expected to sequence commits, and members whose commit was not accepted must re-apply their proposals on top of the winning commit.


## **Protecting application messages**

To send a message to the group, the application calls `send()` on the group view.

```js
const protectedMessage = await group.send("hello group");
await myApp.deliverToGroup(protectedMessage);
```

The `send()` operation does not perform network delivery. It produces an MLS-protected application message that the application transports through its own infrastructure. 

Sending an application message does not alter group state. If a member does not receive a message, they simply miss its content — the group state remains consistent for all members.


## **Receiving messages**

When the application receives an MLS message, it passes the message to the relevant group view.

```js
const received = await group.receive(incomingMessage);
```

The result depends on the type of message that was processed. For example, a received application message may produce application plaintext.

```js
if (received.type === "application") { 
    myApp.displayMessage(received.content);
}
```

A received message may update the local group state and produce related output.

```js
if (received.type === "group-update") {
    await myApp.updateLocalConversationState(received);
}
```


## **Additional MLS capabilities**

The examples above focus on the core operations needed by most applications: creating and joining groups, processing group updates, and sending and receiving protected messages. 

Some applications may also need additional MLS capabilities. These can include exporting group-derived secrets for application-specific encryption, processing MLS proposals, supporting different MLS credential types, among others.

The proposed API is intentionally opinionated, exposing the operations that matter for a browser client. The underlying MLS implementation is fully capable, however, and can participate in groups that use the core MLS features. As we learn more about what applications need, the API can be expanded to include additional capabilities.

The following example illustrates secret export, one of the more commonly needed capabilities.


### Exporting an application secret

Some applications need group-derived secret material for application-specific cryptographic operations. For example, a file sharing application might use an exported secret to encrypt a file. Bytes exported from the group are converted to a WebCrypto API AES-256-GCM key.

```js
const context = new TextEncoder().encode("shared-folder-123");
const key = await group.exportSecret("files", context, 32);
const fileKey = await crypto.subtle.importKey(
    "raw", 
    key.secret,     
    { name: "AES-GCM" }, 
    false, 
    ["encrypt", "decrypt"] );
```


## **Caveats, shortcomings, and other drawbacks of design choices, both current and any prior iterations**

MLS can protect message contents and authenticate group state, but delivery services and network observers may still learn information such as message timing, message size, group activity, and delivery patterns depending on the application’s transport design.

Applications are responsible for binding credentials to user accounts and verifying who they're communicating with. Users may not have visibility into whether this binding is done correctly.


## **draft specification**

<https://frosne.github.io/MLS-WebAPI-Proposal/>


## **incubation and/or standardization destination, this can be:**

The eventual standards-track destination remains open.
