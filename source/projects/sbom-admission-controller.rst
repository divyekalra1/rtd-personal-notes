Supply Chain Security on a Kubernetes Cluster I Definitely Broke Multiple Times
================================================================================

Software supply chain attacks have been a known problem since at least SolarWinds and
Log4Shell. But in the age of AI, the blast radius has gotten significantly larger.

Take the `LiteLLM compromise in March 2026
<https://www.trendmicro.com/en_us/research/26/c/inside-litellm-supply-chain-compromise.html>`_.
LiteLLM is a Python package downloaded 3.4 million times per day -- it acts as a unified
gateway to OpenAI, Anthropic, Azure AI, and every other major LLM provider. Two versions
(1.82.7 and 1.82.8) were trojanized through a cascading attack that started with a
compromised Trivy GitHub Action. The malicious payload did three things: harvested
credentials from over 50 categories (SSH keys, AWS/GCP/Azure tokens, Kubernetes secrets,
LLM API keys), performed lateral movement across Kubernetes clusters by spinning up
privileged pods on every node, and installed a persistent backdoor that polled a C2
server every 50 minutes. Because LiteLLM sits between application code and every
model API, compromising it handed attackers the API keys for multiple LLM providers
simultaneously -- one package, all keys.

A month later, in April 2026, the `Bitwarden CLI npm package was compromised
<https://www.securityweek.com/bitwarden-npm-package-hit-in-supply-chain-attack/>`_.
250,000 monthly downloads. The malicious version systematically collected secrets from
Azure, AWS, GitHub, GCP, npm, and SSH credentials, then weaponized stolen GitHub
tokens to extract more secrets from repositories.

What both attacks have in common: the victim organizations had no reliable way to know
what was actually running inside their dependencies. If you're not generating SBOMs,
signing them, and verifying them at deployment time -- you're flying blind.

That's what this project is. An end-to-end supply chain security pipeline for a
Kubernetes cluster -- admission control, automated SBOM generation, cryptographic
attestation with Cosign, and a vulnerability enrichment module that pulls CVE data
from three different sources to give you a real risk profile per package. Built it
from scratch. Broke things constantly. Had fun.

One thing worth mentioning upfront: the README and the occasional inline comment were
written with LLM assistance. Everything else -- the architecture decisions, the code,
the debugging, the tooling choices -- was done entirely by me. No "what's the best
practice here?" prompts, no asking an AI to design the system. The whole point was
to think through it myself, make my own calls, and actually understand what I was
building. I wanted to be the one connecting the dots, not just shipping whatever
Claude recommended.

What's an SBOM and Why Should You Care?
----------------------------------------

SBOM stands for Software Bill of Materials. It's exactly what it sounds like -- a
manifest of every library, package, and dependency that went into building a piece
of software. Think of it like a nutrition label, but for your container image.

Why does this matter? Because container images are black boxes by default. You pull
``nginx:latest``, deploy it, and pray. An SBOM tells you:

- What exact packages are inside
- What versions they're running
- Which ones have known CVEs

The **CycloneDX** format is what I used here -- it's one of the two main SBOM
standards (the other being SPDX). JSON output, machine-readable, and Trivy generates
it natively.

The Architecture (High Level)
------------------------------

There are three layers to this project:

1. **Admission Control** -- A Kubernetes validating webhook that intercepts pod and
   deployment creation requests and enforces requirements before resources are admitted
   to the cluster.

2. **Supply Chain Pipeline** -- A GitHub Actions CI/CD pipeline that builds the
   webhook container image, generates a CycloneDX SBOM using Trivy, cryptographically
   signs and attaches the SBOM to the image using Cosign, and auto-commits the artifact
   back to the repository.

3. **Vulnerability Intelligence** -- A Python module that enriches SBOM package data
   with CVE information from OSV, exploit probability scores from FIRST EPSS, and
   base severity scores from NVD CVSS -- producing a consolidated risk profile per package.

Part 1: The Admission Webhook
------------------------------

The webhook is a Python Flask app served by Gunicorn over TLS on port 443. Kubernetes
sends it ``AdmissionReview`` JSON objects whenever a pod or deployment creation request
hits the API server. The webhook reads it, applies the validation logic, and returns
``allowed: true`` or ``allowed: false`` with a rejection message.

Here's the shape of a real request the API server sends (simplified):

.. code-block:: json

    {
      "kind": "AdmissionReview",
      "apiVersion": "admission.k8s.io/v1",
      "request": {
        "uid": "7e8214a0-b076-4aeb-88d0-c677912f6a1a",
        "kind": { "group": "apps", "version": "v1", "kind": "Deployment" },
        "operation": "CREATE",
        "object": {
          "spec": {
            "template": {
              "spec": {
                "containers": [{
                  "name": "my-app",
                  "image": "my-image:latest",
                  "env": [{ "name": "LABEL", "value": "deployment" }]
                }]
              }
            }
          }
        }
      }
    }

And the response the webhook sends back:

.. code-block:: json

    {
      "apiVersion": "admission.k8s.io/v1",
      "kind": "AdmissionReview",
      "response": {
        "uid": "7e8214a0-b076-4aeb-88d0-c677912f6a1a",
        "allowed": true,
        "status": { "message": "deployment label exists" }
      }
    }

The ``uid`` in the response must match the ``uid`` from the request -- that's how the
API server knows which request the decision applies to. If you get this wrong, the API
server discards the response.

How Kubernetes Routes to the Webhook
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The magic happens in a ``ValidatingWebhookConfiguration`` resource. You register your
webhook server with the API server, tell it which resource types to intercept (pods,
deployments), and give it a URL and a CA bundle so it can verify the TLS cert.

Here's the actual manifest:

.. code-block:: yaml

    apiVersion: admissionregistration.k8s.io/v1
    kind: ValidatingWebhookConfiguration
    metadata:
      name: validating-webhook-config
    webhooks:
      - admissionReviewVersions: ["v1", "v1beta1"]
        name: validate.default.svc.cluster.local
        failurePolicy: Fail
        sideEffects: None
        clientConfig:
          caBundle: <base64-encoded-CA-cert>
          service:
            name: validate
            namespace: default
            path: /validate
            port: 443
        rules:
          - operations: ["CREATE"]
            apiGroups: ["apps", ""]
            apiVersions: ["v1beta", "v1"]
            resources: ["deployments", "pods"]

A few things worth noting here. ``failurePolicy: Fail`` means if the webhook is
unreachable, the admission request is denied -- so if your webhook pod crashes, nothing
gets deployed until it comes back up. ``caBundle`` is the base64-encoded CA cert that
the API server uses to verify your webhook's TLS cert. And ``rules`` is where you
declare what operations and resource types trigger the webhook -- in this case, CREATE
operations on deployments and pods.

When a user runs ``kubectl apply -f my-deployment.yaml``, the request hits the API
server, gets routed to your webhook before it's written to etcd, and only lands in
the cluster if your webhook signs off on it.

::

    User applies Pod or Deployment manifest
              |
              v
    Kubernetes API Server intercepts CREATE request
              |
              v
    Routes to ValidatingWebhookConfiguration
              |
              v
    HTTPS POST -> validate.default.svc:443/validate
              |
              v
      Webhook Pod (Flask + Gunicorn)
      - Extracts env variables from first container
      - Checks for required LABEL value
      - Returns AdmissionReview response
              |
         _____|_____
        |           |
     Allowed      Denied
        |           |
    Resource     Request
     created    rejected

The validation logic itself is simple on purpose -- the webhook checks for a required
``LABEL`` environment variable in the submitted pod spec. That's the hook. In a real
deployment you'd swap this out for image signature verification, SBOM attestation
checks, whatever policy you care about. The infrastructure is the point.

The TLS Problem
^^^^^^^^^^^^^^^

Here's where things got annoying. The Kubernetes API server will only call your webhook
over HTTPS, and it verifies the cert against the ``caBundle`` you provide in the
``ValidatingWebhookConfiguration``. So you need:

- A CA keypair
- A server cert signed by that CA
- The CA cert base64-encoded and embedded in your webhook config
- The server cert and key stored in a Kubernetes Secret and mounted into the webhook pod

I generated all of this with OpenSSL. The important part is getting the Subject
Alternative Names right in ``csr.conf`` -- they need to match the Kubernetes service
DNS name (``validate.default.svc``) exactly, or the API server will refuse to talk to
your webhook and everything will break silently.

Here's the ``csr.conf``:

.. code-block:: ini

    [ req ]
    default_bits = 2048
    prompt = no
    default_md = sha256
    req_extensions = req_ext
    distinguished_name = dn

    [ dn ]
    C = US
    ST = NY
    L = NY
    CN = 192.168.0.109

    [ req_ext ]
    subjectAltName = @alt_names

    [ alt_names ]
    DNS.1 = validate
    DNS.2 = validate.default
    DNS.3 = validate.default.svc
    DNS.4 = validate.default.svc.cluster
    DNS.5 = validate.default.svc.cluster.local
    IP.1 = 192.168.0.109

    [ v3_ext ]
    authorityKeyIdentifier=keyid,issuer:always
    basicConstraints=CA:FALSE
    keyUsage=keyEncipherment,dataEncipherment
    extendedKeyUsage=serverAuth,clientAuth
    subjectAltName=@alt_names

You need all five DNS entries because Kubernetes resolves services with progressively
shorter names depending on the namespace context of the caller. ``validate.default.svc``
is the minimum for cross-namespace calls, but covering all five means you won't be
chasing a SAN mismatch if the API server resolves to a different form. Replace the IP
with your node's actual IP or remove it if you don't need direct IP access.

I know this because it broke silently the first time. Resources were being admitted
without going through the webhook at all, and nothing in ``kubectl`` output told me
why. I had to ``exec`` into the webhook pod and manually send a test ``AdmissionReview``
request with ``curl`` to confirm whether the server was even receiving anything. It
wasn't. Then I pulled the pod logs -- no incoming requests, no TLS errors, nothing.
The API server had simply stopped routing to the webhook because the cert SAN
(Subject Alternative Name -- the field in a TLS certificate that lists the DNS names
and IPs the cert is valid for) didn't match the service DNS name, and it failed open
rather than loudly.

The private keys and certs are in ``.gitignore``. The ``webhook-secret.yaml`` is also
not committed -- it contains the base64-encoded TLS private key and gets generated
locally at deploy time. Don't commit your TLS private keys. Don't do it.

Part 2: The CI/CD Supply Chain Pipeline
-----------------------------------------

This is where it gets more interesting. Every push to ``admissions-controller/**``
triggers a three-job GitHub Actions pipeline:

**Job 1: Build**

Docker Buildx builds the webhook image and pushes it to Docker Hub. The key detail
is capturing the immutable image digest as a job output so downstream jobs can
reference the exact image that was built:

.. code-block:: yaml

    jobs:
      docker-build-and-push:
        runs-on: ubuntu-latest
        outputs:
          digest: ${{ steps.build-and-push.outputs.digest }}
        steps:
          - uses: docker/build-push-action@v6
            id: build-and-push
            with:
              push: true
              tags: liquidbread0/sbom-validating-webhook:validatingWebhookImage

Why digest instead of tag? Because tags are mutable. ``latest`` today is not
``latest`` tomorrow. The digest is a SHA256 hash of the image content -- it's
pinned forever.

**Job 2: SBOM Generation**

Trivy scans the image and outputs a CycloneDX SBOM in JSON format. The job then
auto-commits the SBOM back to the repo and uploads it as a workflow artifact for
the next job to consume:

.. code-block:: yaml

      sbom-generation-docker-image:
        runs-on: ubuntu-latest
        needs: docker-build-and-push
        permissions:
          contents: write
        steps:
          - uses: aquasecurity/trivy-action@v0.36.0
            with:
              image-ref: "liquidbread0/sbom-validating-webhook:validatingWebhookImage"
              scan-type: image
              format: cyclonedx
              output: data/validatingWebhookImage-sbom.json
          - uses: actions/upload-artifact@v4
            with:
              name: sbom
              path: data/validatingWebhookImage-sbom.json
          - uses: stefanzweifel/git-auto-commit-action@v5
            with:
              commit_message: "Auto-update webhook sbom file"
              file_pattern: data/validatingWebhookImage-sbom.json

The ``permissions: contents: write`` is required for the auto-commit step -- without
it the job silently fails to push. Took me longer than I'd like to admit to figure
out why the commit wasn't landing.

**Job 3: Cosign Attestation**

Cosign signs the SBOM and attaches the attestation to the image at its exact digest
in the Docker Hub registry. The digest from Job 1 is passed in via ``needs``:

.. code-block:: yaml

      sbom-attest-attach:
        runs-on: ubuntu-latest
        needs: [docker-build-and-push, sbom-generation-docker-image]
        steps:
          - uses: actions/download-artifact@v4
            with:
              name: sbom
          - uses: sigstore/cosign-installer@v4.1.0
          - name: Sign the SBOM and attach it to the image
            env:
              TAGS: liquidbread0/sbom-validating-webhook:validatingWebhookImage
              DIGEST: ${{ needs.docker-build-and-push.outputs.digest }}
              COSIGN_PRIV_KEY: ${{ secrets.COSIGN_PRIV_KEY }}
              COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
            run: |
              cosign attest --key env://COSIGN_PRIV_KEY \
                --type cyclonedx \
                --predicate validatingWebhookImage-sbom.json \
                $TAGS@$DIGEST

What does "attaching an attestation" actually mean? Cosign pushes a separate artifact
to the registry alongside the image -- a signed JSON envelope containing the SBOM.
Anyone with the public key can verify it:

.. code-block:: bash

    cosign verify-attestation \
      --key cosign-keys/cosign.pub \
      --type cyclonedx \
      liquidbread0/sbom-validating-webhook:validatingWebhookImage@<digest>

If the signature doesn't verify, you know the SBOM was tampered with or the image
wasn't signed by the expected key.

The full pipeline looks like this::

    Push to admissions-controller/**
              |
              v
    Job 1: Build and push Docker image -> capture digest
              |
              v
    Job 2: Trivy scans image -> CycloneDX SBOM (JSON)
           SBOM auto-committed to data/ in repo
              |
              v
    Job 3: Cosign signs SBOM with private key
           Attestation attached to image digest in registry

Part 3: SBOM + Vulnerability Enrichment
-----------------------------------------

Having an SBOM is step one. Actually understanding what's in it is step two.

The enrichment module reads the CycloneDX SBOM and, for each package, queries three
external sources:

.. list-table::
   :widths: 25 75
   :header-rows: 1

   * - Source
     - What It Gives You
   * - **OSV** (api.osv.dev)
     - CVEs and security advisories matched by Package URL (PURL -- a standardized
       identifier for packages across ecosystems, e.g.
       ``pkg:pypi/requests@2.28.0`` or ``pkg:npm/lodash@4.17.21``)
   * - **FIRST EPSS** (api.first.org)
     - A probability score (0–1) for how likely a CVE is to be exploited in the wild
   * - **NVD CVSS** (services.nvd.nist.gov)
     - Base severity score for each CVE

The output is an ``EnrichedCVEwithEPSS`` record per vulnerability -- package identity,
CVE ID, EPSS probability, and CVSS base score in a single data model.

Why EPSS? Because CVSS alone is a terrible triage tool. A CVSS 9.8 vulnerability in
a package that nobody has ever actually exploited is not the same threat as a CVSS 7.2
with active exploit code in the wild. EPSS gives you the "is anyone actually using this?"
signal. Pair it with CVSS and you get something useful.

Right now this module is a work in progress. It'll get there.

What I Learned
---------------

A few things worth calling out:

**Immutable image digests matter.** You should never be deploying by tag in a
security-sensitive environment. A tag can be retagged. A digest is a cryptographic
commitment. Build your pipelines around digests.

**TLS in Kubernetes is annoying.** The SAN matching requirement trips up basically
everyone the first time. Get the service DNS name right (``<service>.<namespace>.svc``)
and base64-encode the CA cert correctly. When in doubt, ``openssl verify`` before you
ever try to deploy.

**Signing without a verification step at admission is half a solution.** Right now
the attestation gets generated and pushed to the registry, but the admission webhook
doesn't actually verify it before letting pods in. The logical next step is wiring
Cosign verification into the webhook itself -- reject any image that doesn't have a
valid SBOM attestation from the expected key. That closes the loop.

**Auto-committing artifacts from CI is surprisingly useful.** Having the SBOM auto-committed
to the repo on every build means you always know what was in each image, without having
to dig through workflow artifacts. Free audit trail.

What's Next
-----------

A few things I want to add:

- **Cosign verification in the webhook** -- reject images without valid SBOM attestations
  at admission time. This is the thing that actually enforces supply chain policy.
- **Finish the enrichment module** -- wire up the full CVE enrichment pipeline end to end.
- **Policy-as-code** -- integrate something like OPA Gatekeeper so the validation
  logic is declarative and auditable rather than hardcoded in Flask.

The full repo is here: `sbom-admission-controller <https://github.com/divyekalra1/sbom-admission-controller>`_

If you're building anything on Kubernetes and you're not thinking about what's inside
your images -- what packages, what versions, what CVEs -- you're flying blind. SBOMs
aren't a silver bullet, but they're the baseline. Start there.
