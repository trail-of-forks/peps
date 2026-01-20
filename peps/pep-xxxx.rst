PEP: XXXX
Title: Index support for transparency log proofs
Author: TODO <TODO@TODO.com>
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 01-Jan-2026


Abstract
========

This PEP proposes a collection of changes related to the distribution of
transparency log proofs and metadata used to verify them on a Python package
repository, such as PyPI.

More specifically, this PEP proposes:

* A format for transparency log entries that record distribution metadata.
* Backwards-compatible updatesChanges to the HTML and JSON "Simple" APIs,
  allowing clients to discover transparency endpoints for individual
  distribution files.
* A new transparency APIendpoint for retrieving inclusion proofs and signed
  checkpoints.
* A standardized verification procedure that clients should follow to confirm
  a distribution file was included in the transparency log.


Rationale and Motivation
========================

The Python packaging ecosystem has made significant security improvements in
recent years. :pep:`458` (TUF) protects against compromised mirrors and CDNs.
:pep:`740` (Attestations) provides verifiable build provenance. However, these
mechanisms share a common assumption: that PyPI itself is honest and not
compromised.

Today, PyPI operates as a trusted but unauditable intermediary. There is no
public record of what packages were available at any given time, no mechanism
to detect if different users were served different content, and no way to
verify after the fact what PyPI actually served. A sophisticated attacker who
compromises PyPI's infrastructure could serve malicious packages to targeted
users while showing legitimate packages to everyone else, then remove all
evidence of the attack.

This PEP addresses this gap by requiring PyPI to maintain a transparency
log—a public, append-only, cryptographically verifiable ledger of every
distribution file it serves. Transparency logs are Merkle trees where each
entry is cryptographically chained to all previous entries, making tampering
detectable. Independent witnesses verify log consistency and cosign
checkpoints, preventing the index from showing different states to different
parties.

With a transparency log in place:

- Every package served has a corresponding public log entry, eliminating
  undetectable serving of malicious packages. Indices, even if compromised,
  can't serve a package to a client without publicly logging the distribution
  file in the transparency log.
- Clients verify inclusion proofs against witnessed checkpoints, ensuring all
  users see the same log state.
- The append-only property prevents evidence destruction—entries cannot be
  modified or deleted without detection.
- Monitors can cross-reference log contents against packages actually served,
  detecting discrepancies or possible attacks. This discourages bad practices
  and insider attacks and exploits due to the much higher risk of being
  detected.

This approach is proven at scale. Certificate Transparency has protected the
web PKI since 2013, with major browsers now rejecting certificates not present
in CT logs. Go's checksum database has secured module downloads since 2019.
Sigstore's Rekor provides transparency logs for software signatures across
multiple ecosystems including npm. This PEP brings the same accountability
guarantees to Python package distribution.


Concepts and Terminology
========================

This section defines the core concepts used throughout this PEP. Readers
familiar with transparency log systems (such as Certificate Transparency or
Go's checksum database) may skip this section.

Transparency Log
----------------

A transparency log is a public, append-only data structure that records
entries in a way that makes tampering detectable. It provides three key
properties:

- **Append-only:** New entries can be added, but existing entries cannot be
  modified or deleted without detection.
- **Publicly auditable:** Anyone can verify the log's contents and consistency.
- **Efficient verification:** Clients can verify that a specific entry exists
  without downloading the entire log.

Merkle Tree
-----------

A Merkle tree is the cryptographic data structure underlying a transparency
log. It is a binary tree where:

- Each **leaf node** contains the hash of a log entry.
- Each **internal node** contains the hash of its two children.
- The **root hash** is a single value that cryptographically commits to the
  entire tree contents.

Changing any entry changes its leaf hash, which propagates up to change the
root hash. This makes tampering detectable to anyone who has seen a previous
root hash.

Log Entry
---------

A log entry is a single record in the transparency log. For PyPI's
transparency log, each entry is a canonical JSON object containing metadata
about a distribution file: the checksum, filename, and trusted publisher
identity (when available).

Checkpoint
----------

A checkpoint is a signed statement of the log's current state, containing an
identifier for the log (its origin), the number of entries (tree size), and
the Merkle tree root hash. Checkpoints are signed by the log and optionally
cosigned by witnesses.

Inclusion Proof
---------------

An inclusion proof is a compact cryptographic proof that a specific entry
exists in the log at a given checkpoint. It consists of a sequence of hashes
(the Merkle path) that allows a verifier to recompute the root hash from the
entry. For a log with millions of entries, the amount of data necessary to
prove a given entry exists in the logan inclusion proof is only requires a
few hundred bytes.

Consistency Proof
-----------------

A consistency proof demonstrates that one checkpoint is an extension of
another—that the log has only appended entries, not modified or deleted any.
It consists of a sequence of hashes that prove the older tree is a prefix of
the newer tree.

Witness
-------

A witness is an independent third party that verifies log consistency and
cosigns checkpoints. Witnesses ensure the log cannot show different log states
to different parties (a split-view attack) without detection.

Monitor
-------

A monitor is a service that watches the transparency log and cross-references
it against other sources to detect anomalies. Unlike witnesses (which verify
log consistency), monitors verify log contents—for example, detecting packages
served without corresponding log entries or notifying maintainers of new
releases of their software.


Design Considerations
=====================

TODO


Specification
=============

Log-Side Specification
----------------------

Log public key
~~~~~~~~~~~~~~

The Ed25519 public key of the transparency log MUST be available at the
following well-known URL: ``$LOG_URL/.well-known/tlog-key``.

Index-Side Specification
------------------------

Package Upload Behavior
~~~~~~~~~~~~~~~~~~~~~~~

The index MUST insert an entry for a distribution file into the transparency
log upon upload, ensuring that every distribution served by the index is
logged. The distribution should not be made available for download until this
log entry is created.

Furthermore, when a client uploads a distribution file, the index MUST wait
until the entry is successfully recorded in the transparency log before
confirming a successful upload back to the client. After the upload is
successful, the client MUST verify that the uploaded artifact was correctly
added to the transparency log.

Log Entry Format
~~~~~~~~~~~~~~~~

Each log entry represents metadata about a distribution, such as the filename,
the checksum, optionally the identity, etc. The ``checksum`` and ``filename``
properties are mandatory, while ``identity`` is optional. When inserting data
in the transparency log, these metadata are serialized with a text-based
format where each line contains a key/value pair. Each line starts with a
``-`` followed by the ``key`` string and a base64-encoded value. Clients
verifying inclusion proofs must be aware of this serialization scheme, as they
need to canonicalize the entry data themselves to recompute the leaf hash
during verification.

Schema:

.. code-block:: json

    {
      "checksum": "sha256:<lowercase hex digest>",
      "filename": "<distribution filename>",
      "publisher": { ... }
    }

Fields:

- **checksum**: The SHA-256 hash of the distribution file, prefixed with
  ``sha256:``.
- **filename**: The distribution filename (e.g.,
  ``requests-2.31.0-py3-none-any.whl``).
- **publisher**: (Optional) The trusted publishing identity, if the upload
  used Trusted Publishing. The structure depends on the publisher kind. If the
  upload did not use Trusted Publishing, this field is omitted. See Appendix A
  for examples.

Serialization format
^^^^^^^^^^^^^^^^^^^^

::

    pypi-transparency/v1
    sha256:<lowercase hex digest>
    <distribution filename>
    <kind>
    <kind-specific fields>

The ``kind-specific fields`` are defined in Appendix A.

Example Log Entry
^^^^^^^^^^^^^^^^^

Example metadata with its text-based representation as included in the
transparency log:

.. code-block:: json

    {
      "checksum": "sha256:942c5a758f98d790eaed1a29cb6eefc7ffb0d1cf7af05c3d2791656dbd6ad1e1",
      "filename": "requests-2.31.0-py3-none-any.whl",
      "publisher": {
        "kind": "GitHub",
        "repository": "psf/requests",
        "workflow": "release.yml"
      }
    }

::

    sha256:942c5a758f98d790eaed1a29cb6eefc7ffb0d1cf7af05c3d2791656dbd6ad1e1
    requests-2.31.0-py3-none-any.whl
    GitHub
    psf/requests
    release.yml

Checkpoint Format
~~~~~~~~~~~~~~~~~

The log state is captured in a checkpoint following the C2SP tlog-checkpoint
specification:

::

    bt-log.pypi.org
    1234567
    SssviGxgLpKjoCiBhzOvGwrP+okDxwCv/pF8MnHxKa8=

    — bt-log.pypi.org Az3grlgtzOB6MwPjQh...
    — witness.example.org/1 Abcd1234...
    — witness.example.org/2 Efgh5678...

The checkpoint consists of:

- **Line 1:** Log origin identifier (e.g., ``bt-log.pypi.org``)
- **Line 2:** Tree size (number of entries in the log)
- **Line 3:** Root hash (Base64-encoded SHA-256 Merkle root)
- **Blank line:** Separator between body and signatures
- **Remaining lines:** Ed25519 signatures from the log and witnesses, each
  prefixed with ``—``

The signed portion is lines 1–3.
Signature lines follow the format specified in C2SP tlog-checkpoint: each line
begins with ``-`` followed by a key name (which MUST NOT contain spaces), a
space, and the Base64-encoded signature.

Simple API Extension
~~~~~~~~~~~~~~~~~~~~

The Simple API is extended to include a reference to the transparency endpoint
for each distribution.

HTML Format (:pep:`503`)
^^^^^^^^^^^^^^^^^^^^^^^^

A ``data-inclusion-proof`` attribute is added to distribution links:

.. code-block:: html

    <a href="https://files.pythonhosted.org/.../requests-2.31.0.tar.gz#sha256=942c5a..."
       data-dist-info-metadata="sha256=abc123..."
       data-provenance="https://...provenance"
       data-inclusion-proof="https://pypi.org/transparency/requests/2.31.0/requests-2.31.0.tar.gz/proof">
       requests-2.31.0.tar.gz
    </a>

JSON Format (:pep:`691`)
^^^^^^^^^^^^^^^^^^^^^^^^

An ``inclusion-proof`` field is added to file objects:

.. code-block:: json

    {
      "filename": "requests-2.31.0.tar.gz",
      "url": "https://files.pythonhosted.org/.../requests-2.31.0.tar.gz",
      "hashes": {"sha256": "942c5a..."},
      "provenance": "https://...provenance",
      "inclusion-proof": "https://bt-log.pypi.org/transparency/requests/2.31.0/requests-2.31.0.tar.gz/proof"
    }

The ``data-inclusion-proof`` attribute (HTML) and ``inclusion-proof`` field
(JSON) MUST be present if the index supports this PEP. If present, the value
MUST be an absolute URL to the transparency endpoint for that distribution.

Indices that do not support this PEP MUST NOT include these fields.

Transparency Endpoint
~~~~~~~~~~~~~~~~~~~~~

The transparency endpoint provides inclusion proofs for distributions.

URL Structure
^^^^^^^^^^^^^

Following the pattern established by :pep:`740`'s Integrity API:

::

    GET /transparency/{project}/{version}/{filename}/proof

Where:

- **project**: Normalized project name (e.g., ``requests``)
- **version**: Normalized version (e.g., ``2.31.0``)
- **filename**: Distribution filename (e.g., ``requests-2.31.0.tar.gz``)

Response Format
^^^^^^^^^^^^^^^

The endpoint returns a JSON object:

.. code-block:: json

    {
      "version": 1,
      "log_origin": "bt-log.pypi.org",
      "entry_index": 12345,
      "entry": {
        "checksum": "sha256:942c5a758f98d790eaed1a29cb6eefc7ffb0d1cf7af05c3d2791656dbd6ad1e1",
        "filename": "requests-2.31.0-py3-none-any.whl",
        "publisher": {
          "kind": "GitHub",
          "repository_owner": "psf",
          "repository_name": "requests",
          "workflow_filename": "release.yml",
          "environment": "pypi"
        }
      },
      "checkpoint": "<base64-encoded signed checkpoint>",
      "inclusion_proof": ["<base64 hash>", "<base64 hash>", "..."]
    }

Fields:

- **version**: Schema version (currently ``1``)
- **log_origin**: Identifier for the transparency log
- **entry_index**: Position of this entry in the log (zero-indexed)
- **entry**: The log entry object (same format as stored in the log)
- **checkpoint**: Base64-encoded signed checkpoint
- **inclusion_proof**: Array of Base64-encoded hashes forming the Merkle path

Response Codes
^^^^^^^^^^^^^^

- **200 OK**: Success, returns inclusion proof JSON
- **404 Not Found**: Distribution not found, or index does not support this
  PEP.

Client-Side Specification
-------------------------

Client Configuration
~~~~~~~~~~~~~~~~~~~~

Clients implementing inclusion proof verification MUST be able to retrieve:

* ``log_public_key``: The Ed25519 public key of the transparency log, MUST be
  retrieved from the well-known URL specified in the Log-side specification
  (e.g. ``bt-log.pypi.org/.well-known/tlog-key``).
* ``witness_keys``: A mapping of witness key names to their Ed25519 public
  keys.

  * This can be configured directly in the client or retrieved from a list of
    Witnesses URLs (e.g. ``witness.example.org/.well-known/public-key``)

* ``witness_threshold``: The minimum number of valid witness cosignatures
  required.

Verification Algorithm
~~~~~~~~~~~~~~~~~~~~~~

The library ``pypi-transparency`` has the reference implementation for log
verification. Clients MUST perform the following steps to verify a
distribution:

Step 1: Compute Distribution Checksum
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    checksum = sha256(distribution_file).hexdigest()

Step 2: Verify Entry Matches Distribution
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    entry = proof["entry"]
    assert entry["filename"] == distribution_file
    assert entry["checksum"] == f"sha256:{checksum}"

Step 3: Verify Checkpoint Signatures
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    from pypi_transparency import verify_checkpoint, InclusionProof

    log_public_key = # retrieve from well-known URL
    witness_keys = # retrieve from well-known URL or configuration
    witness_threshold = # retrieve from configuration or use default

    checkpoint = base64_decode(proof["checkpoint"])
    # verify_checkpoint gets the latest checkpoint from the log and verifies the consistency between it and the checkpoint in the proof
    verify_checkpoint(checkpoint, log_public_key, witness_keys, witness_threshold)

Step 4: Verify Inclusion Proof
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    from pypi_transparency import verify_inclusion, Entry

    log_public_key = # retrieve from well-known URL

    entry = Entry(data=proof["entry"])
    index = proof["entry_index"]
    checkpoint = proof["checkpoint"]

    verify_inclusion(e, index, proof["inclusion_proof"], checkpoint)

If all steps pass, the distribution and its hash are present in the
transparency log. If any step fails, the client MUST NOT install the
distribution.


Backwards Compatibility
=======================

This PEP is fully backwards compatible with existing clients and indices.

API Compatibility
-----------------

The changes to the Simple API are additive and optional:

- The ``data-inclusion-proof`` attribute (HTML) and ``inclusion-proof`` field
  (JSON) are optional for indices. Indices that do not support binary
  transparency simply omit these fields. Indices that support binary
  transparency MUST include these fields.
- Existing clients that do not implement this PEP will ignore the new
  attribute and field, continuing to function as before.
- No changes are required to the package upload format or existing upload
  APIs.

Clients configured to require inclusion proofs from an index will not be able
to download packages if the index does not provide their inclusion proofs.
This is intentional—such clients are explicitly opting into stricter security
guarantees.

No Transition Period
--------------------

Clients implementing this PEP MUST require inclusion proofs for all packages.
A missing proof indicates one of:

- A misconfigured index that claims to support binary transparency but failed
  to log an entry
- A potential attack where a package is being served without being logged

In either case, the client MUST refuse to install the package.

Index Adoption
--------------

Indices other than PyPI may adopt this specification at their discretion.

Clients MUST NOT rely on the presence or absence of ``data-inclusion-proof``
or ``inclusion-proof`` fields to determine whether an index supports binary
transparency. A malicious index could omit these fields to bypass
verification.

Instead, clients MUST maintain an out-of-band list of indices that are
expected to provide inclusion proofs. This list may be hardcoded (e.g., PyPI
is included by default) or user-configured. For any index on this list, the
client MUST require a valid inclusion proof and refuse to install packages
without one.


Security Implications
=====================

Security Model
--------------

Trust Assumptions
~~~~~~~~~~~~~~~~~

For binary transparency to provide its security guarantees:

- **Clients should trust a threshold of third party witnesses:** That at least
  some configured witnesses are honest and operating correctly. If all
  witnesses collude with the index, split-view attacks become possible.
- **There are third party monitors continuously monitoring the index:**
  monitors should check indices with regards to, at least, a unique mapping
  between distribution filenames and hashes, to ensure the same distribution
  file is never duplicated in the log.
- **Cryptographic primitives:** SHA-256 collision resistance and Ed25519
  signature security.

Notably, clients do NOT need to trust that the index is honest—dishonesty is
detectable through the mechanisms described in this PEP.

This PEP guarantees:

* Protection against artifact tampering by the index or mirrors: If client
  verification succeeds, then the downloaded artifact is the same as the one
  the index added to the transparency log during upload.
* Protection against split-view attacks: An index cannot provide different
  views for the same artifact to different clients.
* Packages upload events are auditable and tamper-evident, so compromised
  indices cannot deny having provided distributions that were logged in the
  binary transparency log.

Role of Witnesses
-----------------

Witnesses are independent third parties that verify the log's consistency and
cosign checkpoints. They prevent the index from showing different log states
to different parties (split-view attacks).

The witness `protocol <https://github.com/C2SP/C2SP/blob/main/tlog-witness.md>`_
works as follows:

1. The log produces a new checkpoint and submits it to witnesses requesting
   cosignatures.
2. Each witness verifies that the new checkpoint is consistent with its
   previously-seen checkpoint (i.e., the log is append-only).
3. If verification succeeds, the witness cosigns the checkpoint and returns
   its signature.
4. If verification fails, the witness refuses to cosign and may publish an
   alert.
5. The log collects cosignatures and publishes the fully-signed checkpoint.

Clients verify that checkpoints carry signatures from a threshold of trusted
witnesses before accepting them. This ensures that even if the index attempts
to provide divergent log states, it cannot obtain valid witness signatures for
inconsistent checkpoints.

For the witness ecosystem to be effective:

- Witnesses must be operated by organizations independent of the index.
- The witness threshold should balance security (more witnesses required)
  against availability (some witnesses may be temporarily unreachable).

Role of Monitors
----------------

Monitors are services that watch the transparency log and detect anomalies
that witnesses cannot. While witnesses verify log *consistency* (e.g.
append-only property), monitors verify log *contents*.

Monitors can perform several functions:

- **Check distribution uniqueness in the log:** Ensure that there are not
  multiple entries in the log for the same distribution. The same distribution
  file should not appear within multiple log entries.
- **Maintainer alerting:** Notify package maintainers when new versions of
  their packages are logged, or when their identities are used to upload a
  package, helping them detect unauthorized uploads.

Unlike witnesses, monitors do not need to be trusted by clients. They provide
an additional layer of detection that benefits the ecosystem as a whole.
Monitors can be operated by security organizations, package maintainers
watching their own packages, or community volunteers.

Attack Scenarios
----------------

Binary transparency addresses several classes of attacks:

**Serving artifacts different from the ones logged:** If the index serves a
package that doesn't have a corresponding entry in the log, client
verification fails immediately (no valid inclusion proof). Additionally,
monitors comparing the Simple API against the log will detect the discrepancy.

**Split-view attacks:** If the index attempts to show different packages to
different users by maintaining divergent log states, witnesses will observe
inconsistent checkpoints when they attempt to verify consistency proofs. They
will refuse to cosign, and clients will reject the unwitnessed checkpoints.

**Evidence destruction:** If an attacker compromises the index and later
attempts to delete evidence, the append-only property makes this detectable.
Any modification to the log changes the root hash, which is immediately
visible to anyone comparing checkpoints. Furthermore, clients, witnesses and
monitors maintain independent copies of the latest seen checkpoint.

Limitations
-----------

Binary transparency does not protect against all threats:

- **Malicious packages:** A package can be both present in the transparency
  log and contain malicious code. Binary transparency records what was
  uploaded, not whether its contents are safe.
- **Immediate prevention:** Upload events are auditable, but not blocked in
  real-time. A malicious package may be installed before monitors detect
  anomalies. Compromised indices might manipulate the log (e.g. add duplicated
  entries) and that will only be detected by monitors, not by clients.
- TODO: index says latest version of package X is P but actually the latest is
  P2.
- **Privacy:** Log entries are public. While this is necessary for
  auditability, it means package upload patterns are visible to anyone.

Complementary Defenses
----------------------


How to Teach This
=================


Reference Implementation
========================


Appendices
==========

Appendix A
----------

Publisher Schemas
~~~~~~~~~~~~~~~~~

The ``publisher`` field structure varies by provider. Field names align with
those used in PyPI's Trusted Publishing configuration.

**GitHub Actions:**

.. code-block:: json

    {
      "kind": "GitHub",
      "repository": "<owner>/<repo>",
      "workflow": "<filename>",
      "environment": "<environment or null>"
    }

**GitLab CI/CD:**

.. code-block:: json

    {
      "kind": "GitLab",
      "repository": "<owner>/<repo>",
      "workflow_filepath": "<filepath>",
      "environment": "<environment or null>"
    }

**Google Cloud:**

.. code-block:: json

    {
      "kind": "Google",
      "email": "<serviceaccount email>"
    }

**ActiveState:**

.. code-block:: json

    {
      "kind": "ActiveState",
      "repository": "<organization>/<project>"
    }

Appendix B
----------

Bulk Ingestion
~~~~~~~~~~~~~~

Before enabling binary transparency verification in clients, the index (e.g.,
PyPI) will perform a **bulk ingestion** of all existing distribution files
into the transparency log. This ensures that every package—past and
present—has an inclusion proof available.

The bulk ingestion process will:

- Create log entries for all existing distributions using their current
  metadata (checksum, filename).
- For distributions uploaded with a :pep:`740` attestation, include the
  publisher identity in the log entry.
- For distributions uploaded without an attestation, omit the ``publisher``
  field.

The bulk ingestion must complete before clients begin requiring inclusion
proofs.


Copyright
=========

This document is placed in the public domain or under the CC0-1.0-Universal
license, whichever is more permissive.
