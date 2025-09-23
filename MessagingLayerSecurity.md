# Explainer: Web API for Messaging Layer Security (MLS)

[Benjamin Beurdouche](mailto:beurdouche@mozilla.com), [Anna Weine](https://github.com/Frosne)

Last updated: September 10, 2025

### User problems to be solved
<!-- Anna: If the users of MLS are messenger users who'd benefit from MLS, the users of the WebAPI are the
companies that will develop in top of the standard -->

The Messaging Layer Security (MLS) standard ([RFC 9420](https://www.rfc-editor.org/rfc/rfc9420.html)) addresses the challenges of scalable, efficient, and standardized end-to-end encryption for group messaging, while ensuring critical security properties such as forward secrecy (FS), post-compromise security (PCS) for groups with more than two participants. 

The standard has already attracted significant attention, with several major <!-- References needed? --> vendors expressing interest in implementing it within their products. Providing a single, standardized, and interoperable approach would offer substantial benefits to messaging application developers as well as the organisations willing to deploy MLS-based communication facilities, enabling them to deliver native, secure messaging support on the web efficiently and reliably. Once supported across all major browsers, developers would be able to write the integration code once and have it work consistently across platforms, significantly reducing development complexity and improving reliability. <!-- Replace one reliability with something else -->


### Methodology for approaching & evaluating solutions

All vendors require different functionalities to achieve their objectives. Prior to evaluating potential approaches, it is necessary to collect functional requirements from prospective implementers. This process enables a clear distinction between core and optional functionalities.

Then, the proposed methodology for assessing potential solutions would be based on the following criteria:

1) Completeness (Modularity??): Priority is given to solutions that provide comprehensive coverage of core functionalities, while optionally supporting additional functions.

2) Simplicity: The ease of implementation and adoption of the solution. <!-- without prior knowledge? -->

3) Interoperability: The ability of the solution to function seamlessly with other potential implementations.

4) Security. MLS uses advanced cryptographical techniques, which are not trivial to implement correctly. A robust API should provide an clear abstraction over these features <!-- components? --> that will allow a secure implementation. 


### Prior/Existing features and/or proposals (if any) that attempt to solve the problem(s)

Several existing initiatives aim to address the stated user problem.

<!-- I am specifically not talking about RFC 9420 here -->

1) The Messaging Layer Security (MLS) Architecture document published as [RFC 9750](https://datatracker.ietf.org/doc/rfc9750) defines the architectural framework for employing MLS within a general secure group messaging infrastructure.

2) Wire has recently advertised the [support and adoption](https://wire.com/hubfs/Whitepapers/Wire%20MLS%20White%20Paper.pdf) of MLS. 

3) Amazon Web Services (AWS) Labs has released an [implementation](https://github.com/awslabs/mls-rs) of the MLS protocol. 

4) Messaging Layer Security over ActivityPub [draft report](https://swicg.github.io/activitypub-e2ee/mls) describes the interfaces between MLS and ActivityPub.

### Flaws or limitations in existing features/proposals that prevent solving the problem(s)

None of the aforementioned features directly address the challenge of establishing a single, unified standard.

### Motivation for this explainer, why we think we can do better than the status quo or other proposals



### Problem statement

For the longest time, due to the lack of technology, pairwise channels were used
to communicate between two principals (actors of a protocol) and a lot of
additional complexities had to be paid by the Applications on top of the
pairwise channels to build Groups. In particular changing the set of
participants in groups securely was impractical without a significant amount of
expertise and security review.

In most cases those custom group protocols were not well (if at all) specified
nor formally analyzed. Most of the security guarantees provided by strong
pairwise channels are broken by applications in non-trivial ways because those
typically don’t provide group agreement (everyone has the same view of the group
and its secrets) or even the correct notion of authentication or secrecy.
Performance was also a concern as the computational complexity of creating
groups or sending messages based on pairwise channels is quite high (O(N^2)) and
incompatible with large dynamic groups.

We aim at solving this issue in the Web Platform by following the WebRTC example
and provide a way for Applications to build evolving groups of participants.
This enables applications for **secure group communication** including
video-conferencing (SFrameTransform for WebRTC, Media Over QUIC,...), secure
messaging (RCS, MIMI), encrypted cloud storage, and more.

### Outline of a proposed solution

Messaging Layer Security (MLS) (RFC 9420) provides convenient and secure
***Continuous Group Key Agreement (CGKA) for dynamic groups***, ranging from one
to hundreds of thousands of participants.

The MLS API provides applications with two main capabilities:

* Group management: applications can manage the participation of different
  entities, including the current browser, in groups.

* Secure messaging: once
  created, a group generates a shared secret that can be used to provide
  end-to-end protected messaging with group participants.

 This explainer outlines the goals, design, and potential use cases of the API.

### Goals

1. **Provide a Continuous Group Key Agreement**: We want to enable Web and
Internet applications to use MLS as a way to build and manage groups using its
secure group key establishment directly from the browser, in particular for
End-to-End Encrypted applications.

2. **State-of-the-Art Security**: Manipulate sensitive values and cryptographic
secrets of the protocol natively instead of exposing them to the web
application. Shipping an up-to-date MLS library which removes the need for
application developers to have a dedicated evaluation and software lifecycle for
this critical component. Similar to TLS, QUIC or WebRTC, ensure to provide a
high quality implementation to users.

3. **Privacy and avoid user tracking**: Prevent user tracking by using
traditional techniques of partitioning the MLS state by origin. This will ensure
that strong identifiers like public keys are not reused across sites.

4. **Ensure Performance**: Avoid back and forth between JavaScript, C++, storage
and cryptographic computations. Handle everything natively within browser native
code.

### Non-Goals

1. **Expose all possible MLS functionality**: The MLS protocol does not
currently have a “standard” API as it can be used in many different ways. We
will provide a significant subset of all these possibilities starting with safe
defaults and will listen for feedback in case extensions are requested by
developers.

2. **Guarantee full interoperability across applications**: We cannot force that
different applications are interoperable, especially in a federated
context. However if service providers decide to agree on interoperability at the
application level by defining coherent MLS group and client configurations, the
MLS layer messages will be interoperable and applications will be able to build
a shared group.

### How do I enable the experiment ?

If you are not already part of an Origin Trial, navigate to **\`about:config\`**.

For Firefox 135: set **\`security.mls.enabled\`** to **\`true\`**.

For Firefox 136+: set **\`dom.origin-trials.mls.state\`** to **\`1\`**.

### Terminology

A few key concepts of the MLS protocol that are necessary to understand the API.

**Identities and Credentials**

The client is a participant in the protocol which is owned by users. Each
client, whether it is part of a group or not, has a unique identity or client
identifier which is a handle to a signature keypair. Each client can be
associated with a credential which is a way to reflect additional information
related to the identity.  The default API offers a facility to create
credentials containing arbitrary strings called basic credentials but X509 or a
verifiable credential can be generated externally and be used in the API.

**Key Package**

In order for users to add clients to a group, those need to
produce some encryption public key and some metadata about the ciphersuite and
extension they support for that client. This information is called a key
package. By default the API contains a function to generate those key packages
with the default parameters and extensions supported by the implementation in
Firefox.

**Groups and GroupViews**

Since the goal of MLS is to build and manage groups, the API is centered about
the notion of group and in particular of the GroupView object which represents
the view of a group for a specific client. The reason for this is that it is
allowed to have multiple clients in a group as long as they have different
client identifiers. Though it is an advanced feature, if multiple clients are
used in the same group, each client is able to manage its view of the group and
may process messages at different times. Each time the group evolves, a sequence
number called epoch is incremented.

**Proposals and Commits**

Group operations such as adding or removing users can be either proposed by
generating a proposal or the change can be made directly by generating a commit
output. Each commit output contains some information, when adding a user, it
contains a commit message, which is the information that needs to be transmitted
to existing members of the group, and a welcome message which is the information
that needs to be transmitted to the newly added client.

### Basic usage

```javascript
let mls = new MLS()

// Create identities for alice and bob
let a = await mls.generateIdentity()
let b = await mls.generateIdentity()

// Create credentials for alice and bob
let ac = await mls.generateCredential("alice")
let bc = await mls.generateCredential("bob")

// Create key packages for alice and bob
let akp = await mls.generateKeyPackage(a, ac)
let bkp = await mls.generateKeyPackage(b, bc)

// Bob creates a group
let gb = await mls.groupCreate(b, bc)

// Bob adds Alice to the group by using the Key Package
// (this generates a commit output which contains the welcome necessary
// to alice and the commit to update current members of the group)
let co1 = await gb.add(akp)

// Bob uses the generated commit to update its group and immediately
// sends a message to the group (alice, bob)
let bgide1 = await gb.receive(co1.commit)
let ctx1 = await gb.send("Hello Alice!")

// Alice uses the welcome to join the group and receives Bob's message
let ga = await mls.groupJoin(a, co1.welcome)
let pt1 = await ga.receive(ctx1)
```

Note that, to ease testing, the code above runs both participants
but each browser will typically run one participant.

### Prototype WebIDL Definition

```javascript

//
// Type definitions
//

// The MLSMessageType is a way to distinguished returned values.

enum MLSObjectType {
  "group-epoch",
  "group-identifier",
  "group-info",
  "client-identifier",
  "credential-basic",
  "key-package",
  "proposal",
  "commit-output",
  "commit-processed",
  "welcome",
  "exporter-output",
  "exporter-label",
  "exporter-context",
  "application-message-ciphertext",
  "application-message-plaintext",
};

dictionary MLSBytes {
  required MLSObjectType type;
  required Uint8Array content;
};

// The MLSGroupMember struct is a non-standard MLS object constructed as a way
// to return the link between a Client Identifier and a Credential. This is
// typically returned inside a list of members when calling the GroupMembers
// function.

dictionary MLSGroupMember {
  required Uint8Array clientId;
  required Uint8Array credential;
};

// The MLSGroupDetails struct is a non-standard MLS object which reflects the
// Membership for a Group at a certain Epoch. The Application can always request
// the membership of the current Group and store that information for future
// purposes.

dictionary MLSGroupDetails {
  required MLSObjectType type;
  required Uint8Array groupId;
  required Uint8Array groupEpoch;
  required sequence<MLSGroupMember> members;
};

// The MLSCommitOutput is a non-standard MLS object that is returned by the
// underneath library after performing an MLS Group Operation. It contains the
// commit (for existing members), the welcome (for eventual new members), the
// identifier of the client that performed the operation for convenience, and
// additional information for eventual use by advanced configuration. As of now,
// those advanced parameters will typically be empty.

dictionary MLSCommitOutput {
  required MLSObjectType type;
  required Uint8Array groupId;
  required Uint8Array commit;
  Uint8Array welcome;
  Uint8Array groupInfo;
  Uint8Array ratchetTree;
  Uint8Array clientId;
};

// The MLSExporterOutput is a non-standard MLS object that contains information
// that was used to generate an exporter value as well as the value of the exporter
// itself.

dictionary MLSExporterOutput {
  required MLSObjectType type;
  required Uint8Array groupId;
  required Uint8Array groupEpoch;
  required Uint8Array label;
  required Uint8Array context;
  required Uint8Array secret;
};

// The MLSReceived is a non-standard MLS protocol object which contains the type
// of the message processed by the decryption function (MLS.receive).

dictionary MLSReceived {
  required MLSObjectType type;
  required Uint8Array groupId;
  Uint8Array groupEpoch;
  Uint8Array content;
  MLSCommitOutput commitOutput;
};

// Type aliases

typedef MLSBytes MLSClientId;
typedef MLSBytes MLSGroupId;
typedef MLSBytes MLSCredential;
typedef MLSBytes MLSKeyPackage;
typedef MLSBytes MLSProposal;
typedef (MLSBytes or Uint8Array) MLSBytesOrUint8Array;
typedef (Uint8Array or UTF8String) Uint8ArrayOrUTF8String;
typedef (MLSBytes or Uint8Array or UTF8String) MLSBytesOrUint8ArrayOrUTF8String;

//
// API
//

// In all these exposed functions, the principal is derived by the DOM code to
// perform storage partitioning based on Origin+OriginSuffix.

[Pref="security.mls.enabled",
SecureContext,
Exposed=(Window,Worker)]

interface MLS {

// The deleteStateForOrigin function will delete all databases for the current
// Origin (Host+OriginSuffix).

  [Throws]
  Promise<undefined> deleteState();

// The generateIdentity function will generate a signature keypair based on the
// default ciphersuite. What is returned is a Client Identifier (the hash of the
// public key) so that public keys are not exposed to the users. There is currently
// no API which returns the private signature key.

  [Throws]
  Promise<MLSClientId> generateIdentity();

// The generateCredentialBasic function allows to generate a single "basic" MLS
// credential which are necessary for building KeyPackages. Other types of
// credentials can be generated externally. (eg. this is internally a Uint8Array
// which can internally be constructed from a string "alice")

[Throws]
  Promise<MLSCredential> generateCredential(Uint8ArrayOrString credentialContent);

// The generateKeyPackage function is leveraging a Signature keypair represented
// by a client identifier handler, as well as an credential to generate and sign
// KeyPackages. These KeyPackages are public values that can be distributed to
// other users and used to add clients to groups.

 [Throws]
  Promise<MLSKeyPackage> generateKeyPackage(MLSBytesOrUint8Array clientId, MLSBytesOrUint8Array credential);

// The groupCreate function will create a Group View for a specific group and
// client identifier based on the identity and credential.

  [Throws]
  Promise<MLSGroupView> groupCreate(MLSBytesOrUint8Array clientId, MLSBytesOrUint8Array credential);

// The groupGet function is a constructor for an MLSGroupView object

  [Throws]
  Promise<MLSGroupView?> groupGet(MLSBytesOrUint8Array groupId, MLSBytesOrUint8Array clientId);

// The groupJoin function processes a Welcome message for a specific Client to
// Join a Group.

  [Throws]
  Promise<MLSGroupView> groupJoin(MLSBytesOrUint8Array clientId, MLSBytesOrUint8Array welcome);
};

[Pref="security.mls.enabled",
  SecureContext,
  Exposed=(Window,Worker)]
interface MLSGroupView {
  // The group identifier is constant across views of the same group
  [Throws]
  readonly attribute Uint8Array groupId;
  // The client identifier is what makes it a view of the group
  [Throws]
  readonly attribute Uint8Array clientId;

  // This function will delete the state of the database for the client
  // which is represented by this view
  [Throws]
  Promise<undefined> deleteState();

  // Add a client to the group using a key package
  [Throws]
  Promise<MLSCommitOutput> add(MLSBytesOrUint8Array keyPackage);

  // Propose to Add a client to the group
  // (this generates a proposal which when received will generate the
  // commit output)
  [Throws]
  Promise<MLSProposal> proposeAdd(MLSBytesOrUint8Array keyPackage);

  // Remove a client from the group
  // (For security reason self removing MUST be proposed instead!)
  [Throws]
  Promise<MLSCommitOutput> remove(MLSBytesOrUint8Array remClientId);

  // Propose to Remove a client (possibly self) from the group
  [Throws]
  Promise<MLSProposal> proposeRemove(MLSBytesOrUint8Array remClientId);

  // Removes all users from the group apart from the client in the view
  // (This could be named removeAll but self, after the commit is
  // received the caller will be alone in the group and can call
  // delete().)
  [Throws]
  Promise<MLSCommitOutput> close();

  // Get information about the current state of the group including
  // identifier, epoch and current membership
  [Throws]
  Promise<MLSGroupInfo> details();

  // Encrypt a message to the group in the current epoch
  [Throws]
  Promise<MLSBytes> send(MLSBytesOrUint8ArrayOrString message);

  // Receive a message for the the current group and client identifiers
  [Throws]
  Promise<MLSReceived> receive(MLSBytesOrUint8Array message);

  // When a commit is generated but lost and cannot be received, the
  // state can still move to the new epoch instead of calling receive on
  // the commit message because we keep the pending commit in the state
  [Throws]
  Promise<MLSReceived> applyPendingCommit();

  // Derive a secret for an external Application
  [Throws]
  Promise<MLSExporterOutput> exportSecret(MLSBytesOrUint8ArrayOrString label, MLSBytesOrUint8Array context, unsigned long long length);

};

```

### A more advanced example…

```javascript
// !!!
// < Insert the code for the basic example here... >
// !!!

// Charlie creates identity keypair, credential and key package
let c = await mls.generateIdentity()
let cc = await mls.generateCredential("charlie")
let ckp = await mls.generateKeyPackage(c, cc)

// Alice adds Charlie to the group
let co2 = await ga.add(ckp)

// Alice and Bob process the commit which is adding Charlie
let agide2 = await ga.receive(co2.commit)
let bgide2 = await gb.receive(co2.commit)

// Charlie joins the group and sends a message
let gc = await mls.groupJoin(c, co2.welcome)
let ctx2 = await gc.send("Hi all")

// Alice and Bob receive charlie's message
let apt2 = await ga.receive(ctx2)
let bpt2 = await gb.receive(ctx2)

// Charlie removes Bob from the group
let co3 = await gc.remove(b)

// Each member receives the commit removing Bob
let agide3 = await ga.receive(co3.commit)
let bgide3 = await gb.receive(co3.commit)
let cgide3 = await gc.receive(co3.commit)

// Alice proposes to be removed from the group
let p4 = await ga.proposeRemove(a)

// Charlie processes the proposal and generates a commit output
let co4 = await gc.receive(p4)

// Alice and Charlie process the commit to remove Alice
let agide4 = await ga.receive(co4.commitOutput.commit)
let cgide4 = await gc.receive(co4.commitOutput.commit)

// Generate identities, credentials and key packages for
// delphine and ernest
let d = await mls.generateIdentity()
let e = await mls.generateIdentity()
let dc = await mls.generateCredential("delphine")
let ec = await mls.generateCredential("ernest")
let dkp = await mls.generateKeyPackage(d, dc)
let ekp = await mls.generateKeyPackage(e, ec)

// Charlie adds Delphine to the Group and process the commit
let co5 = await gc.add(dkp)
let cgide5 = await gc.receive(co5.commit)

// Delphine uses the welcome to join the group
let gd = await mls.groupJoin(d, co5.welcome)

// Delphine proposes to add Ernest to the group
let p6 = await gd.proposeAdd(ekp)

// Charlie generate a commit output for Delphine's proposa to add Ernest
let rec6 = await gc.receive(p6)

// Charlie and Delphine process the commit to add Ernest
let cgide6 = await gc.receive(rec6.commitOutput.commit)
let dgide6 = await gd.receive(rec6.commitOutput.commit)

// Ernest joins the group
let ge = await mls.groupJoin(e, rec6.commitOutput.welcome)

// Create exporter labels and context ("context" in ASCII)
const context_bytes = new Uint8Array([99, 111, 110, 116, 101, 120, 116]); const exporter_len = 32;

// Ernest generates an exporter secret
let exporterOutput = await ge.exportSecret(
	"label", context_bytes, exporter_len);

// This is the shared secret for the group at this epoch
let secret = exporterOutput.secret;

// Display group information for all group views
console.log(await gc.details())
console.log(await gd.details())
console.log(await ge.details())
```

### Notes and additional information

**Exporting secrets for your application**

While sending and receiving MLS application messages through the API is very
convenient, applications sometimes need to generate secrets for their own
use. These symmetric secrets can be generated via the export\_secret API
function using a specific label and context. Generating such secrets is safe and
does not expose anything about the internal secret state of the protocol as this
is done through a KDF. Note that the secret is specific to a specific epoch of
the group.

**Partitioning and storage in the browser**

As of now, the MLS state is stored encrypted at rest in the profile of the user,
similar to credentials so that it can later be migrated. The key for each
database is currently located in the same directory along the encrypted
database.  The state is partitioned by Origin (origin+originAttributeSuffix) and
this was checked with multiple teams, including Privacy and Storage.

**Notes on deleting MLS public and secret state**

The MLS state for the origin (including secrets) can be deleted by the
application by calling the delete state function of the API from within this
origin. The user can also clean the state using the UI to “clear all site data”
(by clicking on the lock icon) or by calling the API via the console if they
don’t wish to clear their cookies at the same time.

**Notes on security for large groups**

It is often asked why anyone would think a thousand participant group could be
honest. The formal security guarantees are the same at a scale of two
participants and for large groups: the cryptographic properties are that \- when
\- the members of the group are honest then during that period of time you
benefit from great Forward Secrecy and Post-Compromise-Security properties.

### Ongoing work

- Alignment with other platforms such as Android
- Web Application Integrity, Consistency and Transparency effort
- Interaction with WebRTC \+ SFrame / MLS
- Get feedback to be in a good shape before submitting to standardization

### Next steps

- We will have to decide about an improved UX and UI for the long state storage,
  and the permissions to add and clear state.
- We expect to provide **custom proposals** and **external join** features.
- We also will likely extend the API to take lists of users to Add and Remove to
     reduce the number of messages to send to other participants.
- We anticipate that there will be a need for the application and users to
  annotate identifiers from other users in a way that cannot be modified by
  applications (e.g. to record that a user identity has been authenticated out
  of bound).
- Consider and discuss the idea of migrating the management of MLS cryptographic
  keys to the Credential Manager.

### Conclusion

The experimental Web API for Messaging Layer Security aims to provide a robust,
privacy-preserving solution for secure group communication on the web and the
internet. As this is not a web-standard, the feature has to be enabled in
about:config.

In parallel of this effort, Mozilla is partnering with major actors of the
industry to bring more guarantees to users when running security and privacy
critical sites through Web Application integrity, consistency and transparency
mechanisms which will help

By leveraging MLS and aligning with open standards, Mozilla seeks to enhance
user security and foster innovation within the web platform community.
