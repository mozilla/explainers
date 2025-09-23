# Explainer: Web API for Messaging Layer Security (MLS)

[Benjamin Beurdouche](mailto:beurdouche@mozilla.com), [Anna Weine](https://github.com/Frosne)

Last updated: September 23, 2025

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

We believe that a single, standardized Web API can significantly improve implementation quality by providing an interface that is both resistant to misuse and easy to implement. We also consider that focusing on a limited set of core functions would create a simpler, more approachable <!-- if more approacheable interface exists as a notion -->, and less error-prone interface. 

### Outline of a proposed solution

Our goal is to provide a simplified MLS API for Web applications.

This includes basic functions for group management, such as adding and removing members. For groups, both secure messaging using the internal MLS key schedule and exporting of keying material for more advanced applications is possible.

### Usage, examples, sample code and a prose summary of how it clearly solves the problem(s)

The key notion of any messaging protocol is a client. Client is an agent that uses this protocol to establish shared cryptographic state with other clients. The client does not have to be a part of a group yet. 

Each client could be seen as a public indentity ("Alice", for example), a public encryption key, and a public signature key. Client credentials is a way to prove that a current member owns a specific identity (by proving the owning of the public key). The client identifier and public key -- along with any external credentials -- are often bundled into what is called a key package that is used to add clients to groups.


```javascript
let mls = new MLS()

// Create identity for alice
let alice = await mls.generateIdentity()

// Create credential for alice
let alice_credential = await mls.generateCredential(alice)

// Bind the credential and indentity to one key package for alice
let alice_key_package = await mls.generateKeyPackage(alice, alice_credential)

TODO: Include some code examples of client setup here. Highlight the important attributes on the client objects.
```

Groups are collection of users. Each client included in a group is a member of the group. Each group member has access to the group's secrets. 

```javascript
// A user could create a group
let alice_group = await mls.groupCreate(alice, alice_credential);

// Access the group information such as group members: 
let alice_group_details = await alice_group.details().members;

// Invite others to the group: 
// We will talk about commits and context (ctx) later in the explainer
let commit_added_bob = await alice_group.add(bob_key_package);

// Send a message to the group
let ctx = await alice_group.send(message);

// Remove a member from a group
let commit_removed_bob = await alice_group.remove(bob);

```

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

### Open questions

The following questions fall outside the scope of the current explainer; however, we consider it valuable to initiate a discussion on them:

1) Group Storage Strategy: How do the applications plan to store groups? Should they be maintained per client, per origin, or using another approach?

2) API and WebCrypto Integration: Are there any advantages to allowing the API to interact directly with WebCrypto?

3) Performance Considerations: What performance implications should be taken into account, and what potential optimizations exist (e.g., cloning groups or adding/removing multiple users simultaneously)?


### Draft specification

To be udpated

### Incubation and/or standardization destination

To be updated.
