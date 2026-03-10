# Image Mirroring in AirGapped Environments — Security and Compliance Notes

<!-- Notes for the Security and Compliance subtopic of Image Mirroring in AirGapped Environments. Organized by question, not source. -->
<!-- See .windsurf/skills/internet-research/SKILL.MD for research workflow and guidelines. -->
<!-- Citations are stored in whitepaper/Image-Mirroring-in-AirGapped-Envs-references.md -->

## Questions

### Q1: What security considerations apply to image mirroring in air-gapped environments?

**Supply chain security concerns:**
- Container images are often made up of layers controlled by multiple people and teams [11]
- Difficult to ascertain what exactly was included in each layer without cryptographic verification [11]
- When images are shared, they are essentially anonymous—hard to verify content or source [11]
- End-users must verify the chain of custody (CoC) and establish trust with all teams that added files to the image [11]

**Image signing and verification:**
- Container image signing adds a digital signature to ensure authenticity and integrity [11]
- Signature verifies the image has not been tampered with and is the same as originally signed [11]
- Most common method: private/public key pairs where private key signs, public key verifies [11]
- Images can have multiple signatures from same or different signers for different security requirements [11]

**Cryptographic signatures:**
- Use asymmetric encryption with private key (secret) and public key (shared) [11]
- Private key creates unique digital signature based on document content [11]
- Signature verification uses hashing—hash of document compared to hash in signature [11]
- Provides: identity authentication, integrity verification, non-repudiation [11]

**Air-gap specific security:**
- Physical isolation provides strong protection from network-based attacks [4]
- Manual updating requirement means systems can become outdated and vulnerable [4]
- USB drives and portable media can introduce malware (human error vector) [4]
- Supply chain attacks (like Stuxnet) can target air-gapped systems through pre-installed software [4]

**Hauler's security features:**
- Stores attestations (atts), SBOMs, and signatures (sigs) alongside images [3]
- Example output shows separate entries for image, attestations, SBOM, and signatures [3]
- Enables verification of image provenance even in disconnected environments

### Q2: How do organizations ensure image integrity and provenance in air-gapped deployments?

**Image signing workflow:**
1. Create private/public key pair [11]
2. Store private key in secure location [11]
3. Use private key to sign the image [11]
4. Add signature and public key to the image [11]
5. Container runtime verifies signature using public key at runtime [11]

**Tools for signing:**
- **Cosign** (Sigstore project): Modern container signing tool
- **Notation** (Notary Project): Open-source supply chain security tool backed by Microsoft
- **Docker Content Trust (DCT)**: Docker's native signing mechanism

**Verification in Kubernetes:**
- Policy engines can enforce signature verification before allowing image deployment
- Admission controllers reject unsigned or improperly signed images
- Enables "trust but verify" model even in air-gapped environments

**Provenance tracking:**
- SBOMs (Software Bill of Materials) document image contents [3]
- Attestations provide cryptographic proof of build process
- Signatures prove image hasn't been modified since signing

**Best practices for air-gapped integrity:**
1. Sign images before transfer across air gap
2. Verify signatures after loading into air-gapped registry
3. Configure Kubernetes to require signature verification
4. Maintain audit logs of all image transfers
5. Use vulnerability scanning before images enter air-gapped environment
6. Implement approval workflows for new images

**Offline verification challenges:**
- Signature verification typically requires access to public key infrastructure
- Air-gapped environments must have local copies of public keys and trust anchors
- Certificate revocation checking may not be possible—use time-limited certificates or local CRL copies

---

## Summary

Security in air-gapped image mirroring centers on two key concerns: ensuring images haven't been tampered with during transfer (integrity) and verifying images come from trusted sources (provenance). Container image signing using cryptographic signatures addresses both concerns—private keys sign images, public keys verify them, and the signature proves the image hasn't been modified [11].

Organizations should implement signing workflows before images cross the air gap, verify signatures after loading into the air-gapped registry, and configure Kubernetes admission controllers to enforce signature requirements. Tools like Hauler support storing attestations, SBOMs, and signatures alongside images, enabling comprehensive provenance tracking even in disconnected environments [3]. The main challenge is maintaining the necessary cryptographic infrastructure (public keys, trust anchors) locally within the air-gapped environment for offline verification [11].
