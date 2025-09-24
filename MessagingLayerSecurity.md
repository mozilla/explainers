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
// We will talk about commits later in the explainer
let commit_added_bob = await alice_group.add(bob_key_package);

// Send a message to the group
let ctx = await alice_group.send(message);

// Remove a member from the group
let commit_removed_bob = await alice_group.remove(bob);

```

A group view represents a group from the perspective of an individual member. A group view reflects the current members of the group, the epoch (the current version of the group) as well as metadata about the group. It should be noted, however, that group views may temporarily diverge across clients if some members have not yet processed the same set of messages.

Application developers, including those working with the browser API, are not required to manage the low-level details of MLS directly. Instead, they interact with group views, which provide a consistent abstraction of both the group’s membership and its current state. 

```javascript

// Bob joins the group
let bob_group = await mls.groupJoin(bob, commit_added_bob.welcome);

// Bob group view and Alice group view are consistent
is (alice_group.members, bob_group.members);

```

The changes that a member makes to its group view need to be communicated to other clients. This occurs two forms:

1) Commits: Operations that alter the state of the group, resulting in creating a message for other group participants.

2) Proposals: Proposal are suggestioned modifications that don't immediately change the group state. Proposals can be used when a client wants to make a change, but does not want to force the change. Maybe the client doesn't have the necessary permissions or maybe it wants to suggest something.

```javascript
// As in our example above that fact that Alice added a new member to the group created a commit
let commit_added_charlie = await alice_group.add(charlie_key_package);

// This commit is then propagated to all the existing members to the group
let alice_group_added_charlie = await alice_group.receive(commit_added_charlie);
let bob_group_added_charlie = await bob_group.receive(commit_added_charlie);

// Bob does not have a right to remove Charlie, so he proposes the removal
let propose_remove_charlie = await bob_group.proposeRemove(charlie);

// Alice does have a right to remove Charlie, so she generates a commit
let commit_remove_charlie = await alice_group.receive(propose_remove_charlie);
// This commit (commit_remove_charlie) will be send to all the participants

```

Each modification to the group results in an increment of the epoch.


```javascript
// The group view epochs are constistent between the members
is (alice_group.epoch, bob_group.epoch);

// All the participants received the same commit
let alice_group_after_commit = await alice_group.receive(some_commit);
let bob_group_after_commit = await bob_group.receive(some_commit);

// The group views are consistent after receiving the same commit
is (alice_group_after_commit.epoch, bob_group_after_commit.epoch)

// The epoch got incremented after receiving the commit. 
is (inc(alice_group.epoch), alice_group_after_commit)

```

Since MLS is not solely a protocol for confidential group messaging but also serves as a foundation for secure group communication more broadly, certain applications may require additional cryptographic keys beyond the one used for message encryption. For example, secure file sharing would require a separate encryption key. To address this requirement, MLS provides a mechanism for deriving (exporting) keys.

```javascript
// The group views are constistent between the members
is (alice_group, bob_group);

// Alice and Bob derive a key to be used for file sharing
// For learning more about context, please check RFC 9420
let exporter_alice = await alice_group.exportSecret("file sharing key", context_for_file_id_and_epoch, len);
let exporter_bob = await bob_group.exportSecret("file sharing key", context_for_file_id_and_epoch, len);

// The keys are equal for consistent groups
is (exporter_alice, exporter_bob)

// And the keys for different label ("file sharing key") or context would not be the same
// This ensures FS and PCS even for derived keys
```

### Open questions

The following questions fall outside the scope of the current explainer; however, we consider it valuable to initiate a discussion on them:

1) Group Storage Strategy: How do the applications plan to store groups? Should they be maintained per client, per origin, or using another approach?

2) API and WebCrypto Integration: Are there any advantages to allowing the API to interact directly with WebCrypto?

3) Performance Considerations: What performance implications should be taken into account, and what potential optimizations exist (e.g., cloning groups or adding/removing multiple users simultaneously)?


### Draft specification

To be updated.

### Incubation and/or standardization destination

To be updated.
