# Short Explainer: Web API for Messaging Layer Security (MLS)

[Anna Weine](https://github.com/Frosne)

Last updated: November 6, 2025

### User problems to be solved

Many web-based applications use groups to organize users and enable collaboration. Examples include Slack Web, Microsoft Teams Web, Google Workspace, and GitHub, which use groups to manage permissions and communication. Although these platforms generally do not employ end-to-end encryption for group messaging, adopting such protection could greatly enhance user privacy and provide stronger security guarantees. One promising approach to enabling secure and scalable group communication is the Messaging Layer Security ([MLS](https://www.rfc-editor.org/rfc/rfc9420.html)) protocol. MLS handles the complex tasks of establishing, updating, and transparently tracking shared cryptographic keys, giving developers a foundation to build scalable and user-friendly applications with strong end-to-end guarantees.

Exchanging cryptographic keys is tricky even in simple one-to-one communication. In MLS, the challenge grows: the protocol has to manage keys for the entire group, handling members joining, leaving, or being removed, all while keeping the keys secure and up to date. A MLS Web API eliminates the need to re-invent the wheel in every web application, offering a vetted and tested implementation that works out of the box with minimal effort for developers. The MLS API handles all cryptographic operations internally, so developers don’t need to implement or worry about security primitives themselves. By offering a single, consistent API across browsers, developers can dedicate their time to building real application features rather than wrestling with security primitives, interoperability quirks, and integration overhead.

While it is technically possible to integrate WebAssembly (WASM) into an existing application to provide MLS functionality in the browser, a native implementation can offer advantages such as better performance and tighter integration with Web APIs like WebCrypto and WebRTC, giving developers more flexibility when building secure group messaging.

Making MLS available in the browser allows for better transparency. Although the browser itself server (in this context, the browser)  cannot access the content of messages due to the end-to-end security guarantees of the group, it still remains a critical component of the communication system, as it is responsible for the storage and distribution of the messages and updates. Nonetheless, it is possible to augment the browser with additional functionality. For example, the browser could enable public auditing and transparency mechanisms on top of MLS.
