# MLS Explainer 

Author: [Anna Weine](https://github.com/Frosne)
Last updated: May 19, 2026

People increasingly use web applications for group communication, collaboration, and shared work. These groups often exchange sensitive information, such as messages, documents, files, comments, tasks, or project state. Users need this information to remain confidential and authentic even as group membership changes over time. When a group adds a new participant or removes an existing participant, future communication should be protected according to the updated membership.

Building end-to-end protected group communication on the Web is difficult. Applications need to manage cryptographic identities, group membership changes, message protection, state updates, and asynchronous delivery. Mistakes in these areas can weaken the security guarantees that users expect.

Today, each web application that wants end-to-end protected group communication usually needs to implement or integrate complex group cryptography itself. That includes not only encryption and decryption, but also cryptographic identity, group membership changes, key updates, message validation, state persistence, and error handling.

This duplicates security-sensitive code across applications. It also makes security properties harder to evaluate: different applications may use different libraries, storage models, update behavior, and failure handling. Users have little visibility into whether group changes are handled correctly, whether removed participants lose access to future protected communication, or whether long-term key material is being managed safely.

Users should be able to rely on secure group communication in web applications without depending on every application to independently implement complex group cryptographic state management correctly.


## **Methodology for approaching & evaluating solutions**

Mozilla’s Web Vision identifies openness, agency, and safety as core values for the Web. For this problem, those values suggest that an appropriate solution should make end-to-end encrypted group communication easier to build correctly on the Web, without requiring each application to independently implement complex cryptographic state management.

A solution should be evaluated against the following criteria:

- User safety. Users should be able to rely on confidentiality and authenticity for group communication without depending on each application getting complex cryptographic implementation details right.

- Openness. The solution should not require applications to use a particular server, delivery service, account system, or application policy.

- User and developer agency. Developers should be able to build secure group features without reimplementing the full MLS protocol, and users should be able to benefit from secure group communication in ordinary web applications.

- Narrow scope. The solution should expose a carefully scoped capability rather than giving applications unnecessary access to sensitive cryptographic material.

- Web platform fit. The solution should work within existing Web platform boundaries, including secure contexts, origins, storage policies, and private browsing.


## **Prior proposals that attempt to solve the problem**

Several existing technologies and approaches address parts of this problem, but none provide a common Web platform API for managing end-to-end encrypted and authenticated group communication as group membership changes.

Existing technologies solve some of these pieces. Messaging Layer Security defines the underlying protocol for secure group state. Web Cryptography APIs expose lower-level cryptographic primitives. JavaScript libraries and WebAssembly modules can implement MLS or MLS-related logic inside an application. Server-side systems can manage accounts, permissions, and delivery. Transport APIs can move protocol messages between clients.

JavaScript and WebAssembly implementations are especially important today because they allow applications to experiment with group communication without waiting for a new Web platform API. For example, a Rust MLS implementation such as OpenMLS can be compiled to WebAssembly and used from a web application. This can provide a practical path for prototyping and deployment, while still using the Web as the application platform. However, these implementations remain part of the application: they do not by themselves create a browser-enforced trust boundary between application code and the user's private key material or group state.

Moreover, JavaScript and WebAssembly implementations remain application-managed. Each application must choose an implementation, ship it to users, keep it updated, integrate it with browser APIs, provide randomness and time sources where needed, and ensure that key material and group state are handled safely. These responsibilities are part of the motivation for exploring whether the Web platform should provide a common MLS API.

Existing encrypted messaging applications also demonstrate that end-to-end protected group communication can be deployed successfully in real products. For example, Signal provides end-to-end encrypted messaging and calling as part of its own application and service. These systems are valuable prior art, but they are application-specific: they do not provide a general Web platform capability that other web applications can use for secure group creation, membership changes, and message protection.

**Limitations in existing proposals**

Each of the existing approaches address important parts of secure group communication, but they leave significant work to each application.

The MLS protocol defines the cryptographic protocol for secure groups, but it does not define how MLS should be exposed to web applications. It does not provide JavaScript interfaces, a browser storage model, origin scoping, integration with private browsing, site data clearing behaviour, or Web platform error handling.

The Web Cryptography API exposes low-level cryptographic primitives, but applications using the API still need to design or integrate their own group membership logic, state transitions, key lifecycle, and storage behaviour.

As mentioned above, JavaScript and WebAssembly implementations can provide practical group cryptography today, including MLS implementations. However, this approach still keeps the cryptographic implementation inside the application’s own trust boundary. It does not create a browser-enforced boundary between the application code and sensitive user information. Each application must choose an implementation, ship it to users, keep it updated, decide which information to store, and ensure that the key material is handled safely. 

In general, application-specific end-to-end encryption systems can be tailored to a particular product, but they do not provide a shared Web platform capability. Their security properties, membership-change rules and behaviours, storage behaviour, and failure handling can vary significantly across applications.

Application-specific encrypted messengers can provide strong security properties within their own products. However, they do not solve the Web platform problem for other applications. A collaborative editor, project management tool, file sharing app, or web-based community platform cannot use Signal’s application-specific group system as a browser API. Those applications still need to implement or integrate their own secure group machinery.

Server-managed access control systems remain useful for accounts, permissions, moderation, and abuse prevention. However, they do not by itself provide end-to-end protection. If the servers can access plaintext data, users must still trust the service providers with the confidentiality of that data. 

Transport APIs can move messages between clients and servers, but they do not define secure group behaviour nor do they determine how groups state evolves, how the messages are validated, etc. 

As a result, web applications that want secure end-to-end group features still need to assemble their own combination of group cryptography, state management, storage behaviour, message validation. This increases implementation burden and makes it harder for users, developers, and reviewers to understand whether a group feature provides consistent security properties.


## **Motivation for this explainer**

The Web already provides applications with primitives for networking, storage, workers and cryptography. However, secure group communication requires these pieces to be combined with careful cryptographic state management. Today, this combination is left almost entirely to the applications itself. 

This creates a gap between what users need and what the Web platform makes easy to build. A common Web platform capability for secure groups could reduce duplicated security-sensitive implementation work. Instead of each application independently implementing group cryptographic state management, the user agent could provide a shared interface for the parts that are common across applications. 

Such an interface could also give browsers a clearer role in protecting sensitive user state. For example, the user agent could enforce secure-context restrictions, origin scoping, storage lifetime rules, and deletion behaviour. It could also avoid exposing long-term private key material directly to application code unless explicitly required by design. 

At the same time, the Web platform should not take over application-specific decisions. Applications should remain responsible for account identity, authorisation policy, abuse prevention, moderation, user interface, and message transport. These responsibilities are not common to the applications, and can significantly vary between products and implementations. 

This explainer explores whether a Web API based on Messaging Layer Security can provide this common group-security machinery. 


## **Outline of a proposed solution**

The proposal is to expose MLS through a browser-provided API built around two main objects: MLS and MLSGroupView.

MLS is the client-level entry point. It lets an application create an MLS identity, create credentials, generate key packages, create a new group, join an existing group from a Welcome message, retrieve an existing group view, and delete local MLS state.

MLSGroupView represents one local client’s view of one MLS group. It lets application code change group membership, process MLS messages, produce protected application messages, inspect group state, handle pending proposals or commits, and derive exported secrets.

By letting the user agent manage MLS state, the API reduces the amount of security-sensitive code that each application must implement independently. It also gives the Web platform a consistent place to apply security boundaries around MLS private key material and group state. A standardized API would make the common browser-facing behavior testable across user agents, rather than requiring each application to solve API-level interoperability through its own choice of libraries, storage behavior, and integration code.


## **Usage, examples, sample code and a prose summary of how it clearly solves the problem(s)**

The following examples illustrate how an application could use the proposed API. The examples use the current draft API shape. Method names and return types may change as the API design is refined.


### Preparing a client

A client first creates a local MLS identity state, creates a credential, and generates a key package. The application can then publish the key package through its own service so that another client can add this client to a group.

    const mls = new MLS();
    const clientId = await mls.generateIdentity();
    const credential = await mls.generateCredential("alice@example");
    const keyPackage = await mls.generateKeyPackage(clientId, credential);
    // Application-defined
    await publishKeyPackageToApplicationServer(keyPackage);

In this flow, the application receives the values it needs to participate in group setup, but the user agent manages the MLS-related state needed to continue using the identity and credential.

\



### Creating a group

A client can create a new MLS group from its local identity and credential.

    const group = await mls.groupCreate(clientId, credential);

The returned `MLSGroupView` represents this client’s local view of the group. The application can use this object to add members, process messages, protect application data, inspect group state, and export secrets.


### Adding a member

To add another client, the application obtains that client’s key package through an application-defined mechanism:

    const bobKeyPackage = await fetchKeyPackageFromServer("bob");

    const commitOutput = await group.add(bobKeyPackage);

The group operation produces protocol data that the application must deliver.

    await deliverToExistingGroupMembers(commitOutput.commit);

    if (commitOutput.welcome) {
     await deliverWelcomeToNewMember("bob", commitOutput.welcome);
    }

The user agent is responsible for producing the MLS protocol output and updating local group state according to the API’s rules, the application is responsible for delivery.

\



## **Joining a group**

A newly added client joins a group by processing a Welcome message delivered by the application.

    const welcome = await receiveWelcomeFromApplicationServer();

    const joinedGroup = await mls.groupJoin(clientId, welcome);

After joining, the returned `MLSGroupView` represents the new client’s local view of the group.


## **Protecting application messages**

To send a message to the group, the application calls `send()` on the group view.

    const protectedMessage = await group.send("hello group");
    // Application defined
    await deliverToGroup(protectedMessage);

The `send()` operation does not perform network delivery. It produces an MLS-protected application message that the application transports through its own infrastructure.


## **Processing received messages**

When the application receives an MLS message, it passes the message to the relevant group view.

    const received = await group.receive(incomingMessage);

The result depends on the type of message that was processed. For example, a received application message may produce application plaintext.

    if (received.type === "application-message-plaintext") {
     const plaintext = received.content;
     displayMessage(plaintext);
    }

A received commit may update the local group state and produce commit-related output.

    if (received.type === "commit-processed") {
     await updateLocalConversationState(received);
    }

The exact message validation behavior, error behavior, and state transition behavior need to be defined by the draft specification.


## **Exporting an application secret**

Some applications need group-derived secret material for application-specific encryption. For example, a file sharing application might use an exported secret to protect a file key.

    const context = new TextEncoder().encode("shared-folder-123");
    const exported = await group.exportSecret(
     "file-encryption",
     context,
     32
    );
    const fileKey = exported.secret;

Applications need to use exporter labels and contexts carefully. Different application purposes should use distinct labels and context values.

These examples show how the common cryptographic parts of secure group communication can be provided by the Web platform, while application-specific policy and delivery remain with the application.


## **Caveats, shortcomings, and other drawbacks of design choices, both current and any prior iterations**

The proposal does not, by itself, provide metadata privacy. MLS can protect message contents and authenticate group state, but delivery services and network observers may still learn information such as message timing, message size, group activity, and delivery patterns depending on the application’s transport design.

MLS credentials don't prove real-world identity. Applications are still responsible for binding credentials to user accounts and verifying who they're communicating with. Users may not have visibility into whether this binding is done correctly.

This proposal should be evaluated through prototyping and review before standardization. In particular, prototypes should test whether the API is expressive enough for real group applications while still keeping the browser-facing surface small and understandable.


## **draft specification**

<https://frosne.github.io/MLS-WebAPI-Standard/> 


## **incubation and/or standardization destination, this can be:**

The eventual standards-track destination remains open.
