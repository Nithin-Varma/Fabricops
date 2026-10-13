# CLAUDE.md — FabricOps

Context for AI coding assistants working in this repo. Read this fully before writing code.

## What this project is

**FabricOps** is a Go CLI toolkit that reproduces Hyperledger Fabric transaction failures on a local test network, collects evidence about them, and produces a deterministic, explainable diagnosis. Automated tests verify that each diagnosis is correct.

It is a **learning-first portfolio project**, not a startup. The owner knows Solidity/EVM, TypeScript and distributed systems well. They are new to Fabric internals (MSPs, endorsement, validation) and to Go in real systems. The repo should show hiring engineers real Go, real integration testing, and a correct understanding of Fabric's transaction lifecycle.

**Not building:** a block explorer (Hyperledger Explorer exists), a benchmarking tool (Caliper exists), a monitoring stack (use Prometheus), a deployment platform, an AI that guesses failures, or a dashboard (not until v1 is done).

## How to work with the owner

- Explain the relevant Fabric concept briefly **before** writing code that depends on it.
- Work in small, runnable increments. Never generate the whole system at once.
- **Never invent APIs.** Check Fabric Gateway client, qscc and protobuf names against official docs or the source on pkg.go.dev before using them. If unsure, say so and check.
- Every diagnostic rule gets a unit test.
- For every new dependency, say why it is needed. Prefer the standard library.
- Keep the architecture simple: one binary, no microservices, no Kubernetes, no database (JSON files are enough).
- Target platform is **macOS on Apple Silicon** (Docker Desktop, arm64 images). Keep setup steps reproducible there.

## Versions

- Pin all versions in `VERSIONS`. Target the **Fabric 3.1.x** line (v3.1.5 was the latest found in Oct 2026; confirm the current patch) and the latest **Fabric CA 1.5.x**.
- Use the **Fabric Gateway client API for Go** (`github.com/hyperledger/fabric-gateway`). The legacy Fabric SDKs are unmaintained and must not be used.
- Fabric v3 removed the system channel, the consortium concept, and Solo/Kafka ordering. Ignore tutorials that assume them.
- Go version: whatever the Gateway client and fabric-samples `go.mod` require.

## Fabric transaction lifecycle (the core mental model)

| Stage | What happens | Example failures | Evidence source | On ledger? |
|---|---|---|---|---|
| Endorse | Gateway peer sends proposal to endorsers; they simulate chaincode and sign read/write sets | chaincode error, endorser down, mismatched results, access denied | Client-side Gateway error, peer logs | No |
| Submit / order | Gateway sends endorsed tx to orderer, which batches it into a block | orderer unreachable, tx rejected | Client-side error, orderer logs | No |
| Validate & commit | Every peer checks endorsement policy and MVCC read versions | `ENDORSEMENT_POLICY_FAILURE`, `MVCC_READ_CONFLICT`, `PHANTOM_READ_CONFLICT` | Ledger (qscc, validation code), client commit error | Yes, in the block, flagged invalid |
| Commit status wait | Client waits for commit event | timeout | Client only | Unknown |

### Design consequences (non-negotiable)

1. **The tx ID is generated client-side.** Failures before ordering leave no ledger record. FabricOps therefore has a **capture side**: the scenario runner (and later a client wrapper) records tx ID, timestamp, stage and Gateway error to a JSON evidence file. `inspect` merges the ledger lookup with captured evidence.
2. **"Not found on ledger" is Unknown, not a failure diagnosis.** Possible reasons to list: never submitted, failed before ordering, not committed yet, wrong channel, caller lacks read access.
3. **The Gateway picks endorsers that satisfy the policy** (via discovery). So a validation-stage `ENDORSEMENT_POLICY_FAILURE` must be triggered deliberately, by restricting endorsing organizations to one org under a policy that needs more.
4. **The Go Gateway client returns a distinct error type per stage** (endorse, submit, commit status, commit). gRPC status details carry per-endpoint info (endpoint, MSP ID, message). This is the primary pre-commit evidence. Verify exact type and field names in the package docs before coding against them.

## Evidence labelling

Every statement in a report carries exactly one label:

- **Observed:** directly supported by a ledger record, client error, log line or metric.
- **Inferred:** a likely explanation from correlated evidence. Example: a validation code saying the policy failed is Observed; *which* org's endorsement was missing is Inferred unless the endorsements were inspected.
- **Unknown:** not available to the tool or not supported by the evidence.
- **Simulated:** produced by a controlled scenario.

No confidence scores unless a tested methodology exists.

## Architecture

```
Fabric test network
   │
   ├─ ledger/   (qscc queries, block events)  ─┐
   └─ capture/  (gateway errors, tx IDs)       ├─► evidence.Evidence ─► diagnose (pure rules) ─► report (text + JSON)
      logs / metrics (later)                  ─┘
                                                     ▲
                              test/integration: run scenario → assert report == expected
```

**Key rule:** `internal/diagnose` is **pure**. It takes `Evidence` and returns `Report`, with no network, Docker or Fabric calls. All Fabric I/O lives in `capture/` and `ledger/`. This keeps the rules unit-testable without a network.

## Repo layout

```
fabricops/
├── CLAUDE.md
├── VERSIONS                # pinned Fabric, CA, Go versions
├── Makefile                # network-up, network-down, test, test-integration, scenario S=<id>
├── go.mod
├── cmd/fabricops/          # CLI entrypoint: parse flags, call packages, nothing else
├── internal/
│   ├── evidence/           # Evidence types (ledger record, gateway error, timestamps, source)
│   ├── capture/            # Submit via Gateway step by step; record tx ID + stage + error to JSON
│   ├── ledger/             # qscc lookups: GetTransactionByID → validation code, block number
│   ├── diagnose/           # Rules: Evidence → Report. Pure. Table-driven tests.
│   └── report/             # Render text + JSON with Observed/Inferred/Unknown/Simulated labels
├── chaincode/claims/       # Small Go chaincode used by the scenarios
├── scenarios/
│   ├── s1-success/
│   ├── s2-mvcc-conflict/
│   ├── s3-policy-failure/
│   └── s4-endorser-down/
├── network/                # scripts wrapping fabric-samples test-network; profiles/local-dev.yaml
├── test/integration/       # build tag `integration`; needs Docker + running network
├── docs/
│   ├── SETUP.md            # exact macOS arm64 steps + every gotcha hit
│   ├── ARCHITECTURE.md
│   ├── decisions/          # short ADRs
│   └── examples/           # real generated reports (never hand-written)
└── .github/workflows/      # unit tests on every push; integration manual/nightly
```

Start everything in `internal/`. Promote `diagnose` to `pkg/` only if there is a real external consumer.

## CLI (target surface for v1)

```
fabricops tx inspect --network local-dev --channel <ch> --tx-id <id> [--evidence <file>] [--json]
fabricops scenario list
fabricops scenario run <scenario-id> [--json]
```

Use stdlib `flag` first. Add a CLI library (e.g. cobra) only if subcommands become painful.

## Scenarios (v1)

Each `scenarios/<id>/README.md` must specify: preconditions, how the failure is introduced, expected Fabric behavior, evidence available, expected classification and failure stage, what confirms or refutes the diagnosis, and recovery steps.

| ID | Failure introduced | Expected stage | Expected classification |
|---|---|---|---|
| s1-success | none | committed | VALID |
| s2-mvcc-conflict | Using the step-by-step proposal flow, endorse two updates to the same key **before** submitting either, then submit both | validation | `MVCC_READ_CONFLICT` on one tx (deterministic, no timing race) |
| s3-policy-failure | Endorse with Org1 only under the default majority policy (both orgs on a 2-org network) | validation | `ENDORSEMENT_POLICY_FAILURE` |
| s4-endorser-down | `docker stop peer0.org2.example.com`, then submit | endorse (pre-ledger) | endorsement failure; ledger lookup = Unknown |

Later, only after v1: TLS/certificate failures, orderer unavailable, private data collection access, chaincode non-determinism.

Destructive scenarios (stopping containers, changing config) are opt-in and only run against the local test network.

## Testing

- **Unit:** `go test ./...`. Table-driven tests for every rule in `diagnose`. No Docker needed.
- **Integration:** `go test -tags integration ./test/integration/...`. Brings the network up, runs each scenario, asserts that the report's stage and classification match the scenario spec, and records missing evidence.
- Every integration result records: scenario ID, failure stage, expected vs actual outcome, classification, evidence collected, pass/fail, and unknowns.

## Security rules

- Never commit private keys, certs or generated `organizations/` material. Keep these in `.gitignore`.
- Never log secrets or full certificate private material.
- Use the network's authorized identities only. Never bypass ACLs or private data permissions.
- Metadata access does not imply payload access.
- Read-only diagnosis is separate from anything that changes the network.

## Milestones

- [ ] **M0 Setup:** pinned versions; test network up on macOS arm64; asset-transfer-basic Go chaincode deployed; Go Gateway sample transaction committed; `docs/SETUP.md` written. *(Current milestone.)*
- [ ] **M1 Inspect:** `fabricops tx inspect` queries qscc for a committed tx and prints ledger status, validation code and stage, with labelled evidence and JSON output.
- [ ] **M2 First scenario:** capture package plus s2-mvcc-conflict end to end, with an integration test asserting the diagnosis.
- [ ] **M3 v1:** s1, s3 and s4 added; CI; README with real example reports; write-up on telling Fabric failure modes apart.

Stop after M3 and reassess before adding logs/metrics correlation (Prometheus, peer operations endpoints), more scenarios, or any UI.

## Definition of done (v1)

Another engineer can: clone, follow `SETUP.md`, start the network, deploy the chaincode, run each scenario, run `fabricops` against it, read a labelled report, and run the integration tests, which pass.

## References

- Fabric docs: https://hyperledger-fabric.readthedocs.io/en/latest/
- Test network: https://hyperledger-fabric.readthedocs.io/en/latest/test_network.html
- Transaction flow: https://hyperledger-fabric.readthedocs.io/en/latest/txflow.html
- Private data: https://hyperledger-fabric.readthedocs.io/en/latest/private-data/private-data.html
- Operations service: https://hyperledger-fabric.readthedocs.io/en/latest/operations_service.html
- Fabric Gateway client: https://hyperledger.github.io/fabric-gateway/ and https://github.com/hyperledger/fabric-gateway
- fabric-samples: https://github.com/hyperledger/fabric-samples (start with `asset-transfer-basic/application-gateway-go`, including its error-handling example)
