# KIP-21 Proving Spec

Status: Reserved companion document.

This companion document is reserved for proving and witness algorithms that consume KIP-21 commitments. It is not itself a consensus specification unless explicitly incorporated by the main KIP.

It will guide provers on what Kaspa L1 commits to long term, and how to avoid making zk guests depend on commitment-structure details that may change unless the application explicitly needs those details.

Reserved scope:

* lane activity proofs
* inactive-lane and reactivation proofs
* inactivity shortcut usage
* guest algorithms
* witness formats and node-serving expectations
