---
layout: default
css_id: ecus
---

## Preparing an ECU for Uptane

At the highest level, the basic requirement for an ECU to be capable of supporting Uptane is that it must be able to perform either full or partial verification, and access a secure source of time. (See the [Uptane Standard](https://uptane.github.io/uptane-standard/uptane-standard.html#build-time-prerequisite-requirements-for-ecus) for official requirements.)

To bootstrap an Uptane-capable ECU, a few things need to be provisioned into the unit:

* **A current set of Uptane metadata**, so that the ECU is able to verify the first set of metadata it gets from the server. The exact metadata files required depend on whether the ECU performs full or partial verification. Full verification ECUs need a complete set of metadata from both repositories, while partial verification ECUs only need the Targets metadata from the Director repository.
* **A secure way to know what time it is**, so the ECU can not be tricked into accepting expired metadata. If a [time server](https://uptane.github.io/deployment-considerations/repositories.html#time-server) is in use, each ECU would need a recent time downloaded from this, and the public key of the time server to verify it. If another source of time is being used, the ECU must receive a fairly recent time as soon as it is powered on (or reset to factory settings), to prevent the possibility of freeze attacks.
* **ECU key(s)**, to sign the ECU's [version reports](https://uptane.github.io/papers/ieee-isto-6100.1.0.0.uptane-standard.html#version_report), and optionally to decrypt images. These keys should be unique to the ECU, and the public keys will need to be stored in the Director repository's inventory database.
* **Information about repository locations**, generally in the form of a [repository mapping file](https://uptane.github.io/papers/ieee-isto-6100.1.0.0.uptane-standard.html#repo_mapping_meta). This is a metadata file that tells the ECU the URIs of the repositories (if it is a primary ECU), as well as which images should be fetched from which repository. (Images that are encrypted or customized per-device would generally come from the Director repository, and all others from the Image repository.)

## ECU implementation choices

There are three big decisions to make about each Uptane ECU: first, whether it will perform full or partial verification, second, whether it will use an asymmetric or symmetric ECU key, and third, whether it will use encrypted or unencrypted update images. Here, we offer some advice on making those choices.

### Full vs. partial verification

Uptane is designed with automotive requirements in mind, and one of the difficulties in that space is that ECUs requiring OTA updates might have very slow and or memory-limited microcontrollers. To accommodate those ECUs, Uptane includes the option of partial verification. So, how do you choose between full and partial verification for a particular ECU?

Firstly, if the ECU is a Primary ECU, partial verification is not an option. Primaries need to perform full verification. For other ECUs, full verification is preferable when possible, for at least two reasons:

1. Full verification is more secure. Because they don't check Image repository metadata, partial verification ECUS could be instructed to install malicious software by an attacker in possession of the Director repository's Targets key (and, of course, a way to send traffic on the relevant in-vehicle bus).
2. Full verification ECUs can rotate keys. Because partial verification is designed for ECUs that can only reasonably check a single signature, they do not download or process Root metadata. Since the Root metadata is the mechanism for revoking and rotating signing keys for all other metadata, a partial verification ECU has no truly secure way to invalidate a signing key.

Partial verification ECUs are expected to have the Root and Targets metadata present at the time of manufacturing or installation in the vehicle. To update the Root metadata, the ECU SHOULD install a new image containing the metadata. To update the Targets metadata, the ECU SHOULD follow the steps described in the [Uptane standard](https://uptane.github.io/papers/ieee-isto-6100.1.0.0.uptane-standard.html#partial_verification). Partial verification Secondaries MAY additionally fetch and check metadata from other roles or the Image repository if the ECU is powerful enough to process them and you wish to take advantage of their respective security benefits.

### Symmetric vs. asymmetric ECU keys

<img align="center" src="assets/images/ECU_1_sym_asym.png" width="500" style="margin: 0px 20px"/>

**Figure 1.** *An arrangement that an OEM SHOULD use when using symmetric ECU keys.*

ECUs are permitted to use either symmetric or asymmetric keys. This choice is effectively a performance vs. security trade-off. Symmetric keys allow for faster cryptographic operations, but expose a larger attack surface because the Director will need online access to the key. Asymmetric ECU keys are not affected by this problem, because the Director only needs access to the ECU's public key.

Basically, choosing symmetric keys increases the performance of the common case (checking signatures and decrypting images), but makes disaster recovery harder, because a compromised key server could require updating ECU keys on every vehicle.

#### Symmetric key server

If you choose to use symmetric ECU keys, it would be a good idea to store the keys on an isolated, separate key server, rather than in the inventory database. This separate key server can then expose only two very simple operations to the Director:

1. Check the signature on an ECU version report.
2. Given an ECU identifier and an image identifier, encrypt the image for that ECU.

Unencrypted images should be loaded onto the symmetric key server by some out-of-band physical channel (for example, via USB stick).

## Encryption of images on ECUs

The Director repository may encrypt images if required (see [Section 5.3.2](https://uptane.github.io/papers/ieee-isto-6100.1.0.0.uptane-standard.html#director_repository) of the Uptane Standard). However, no Uptane implementation should support interactive requests from an ECU for encryption.  Allowing the Target ECU to explicitly request an encrypted image at download time would not only increase the attack surface, but could also be used to turn off encryption. This would make it easy for attackers to reverse engineer unencrypted firmware and steal intellectual property. Only the OEM and its suppliers should determine policy on encrypting particular binaries, and this policy should be configured for use by the Director repository, rather than being toggled by the Target ECU.
