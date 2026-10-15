# Day 13: Docker Security and Optimization

Took an intentionally unsafe image (`my-app:bad`) and turned it into a
smaller, safer `my-app:v2`. Scanned before and after with Trivy, removed
unsafe patterns, and ran the result with runtime restrictions.

Repo: https://github.com/YOUR-USERNAME/YOUR-REPO

All numbers below come from my own machine. The token used in this lab is fake.

## Project structure

```
app.py
requirements.txt
Dockerfile.bad      (intentionally unsafe, for comparison only)
Dockerfile.good     (hardened image)
.dockerignore
README.md
reports/before.json
reports/after.json
evidence/
```

`token.txt` is a local-only fake token. It is not committed.

## Risks found in the bad image

| Risk | Where | Why it matters |
|---|---|---|
| Floating base `python:latest` | `FROM python:latest` | Builds can change unexpectedly |
| Hard-coded token | `ENV API_TOKEN=abc123-super-secret` | Visible in image metadata and `docker history` |
| Build tools left in image | `apt-get install build-essential git curl vim` | More packages, more patches, bigger attack surface |
| Runs as root | No `USER` instruction | A compromised app has unnecessary privileges |
| Broad copy | `COPY . .` | Can pull unwanted or secret files into the image |
| Shell-form `CMD` | `CMD python app.py` | An extra shell sits between Docker and the app |

## Remediations in `Dockerfile.good`

| Change | Reason |
|---|---|
| `python:3.12-slim` instead of `python:latest` | Pinned, smaller base |
| Multi-stage build | Dependencies installed in a build stage, runtime stage stays clean |
| Removed `build-essential`, `git`, `curl`, `vim` | Fewer packages and less attack surface |
| `useradd -r -u 10001 app` and `USER app` | Runs as non-root |
| Token removed from the Dockerfile | Passed at runtime with `-e API_TOKEN=...` |
| `COPY --chown=app:app app.py .` | Copies only the needed file |
| Exec-form `CMD ["python", "app.py"]` | Python is the main process |

## Commands used

### Before image

```bash
docker build -f Dockerfile.bad -t my-app:bad .
docker images my-app
docker history --no-trunc my-app:bad
mkdir -p reports
trivy image --severity HIGH,CRITICAL -f json -o reports/before.json my-app:bad
trivy image --severity HIGH,CRITICAL my-app:bad
trivy config .
```

### After image

```bash
docker build --pull --no-cache -f Dockerfile.good -t my-app:v2 .
docker images my-app
docker history --no-trunc my-app:v2
trivy image --severity HIGH,CRITICAL -f json -o reports/after.json my-app:v2
trivy image --exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed my-app:v2
trivy config .
```

Trivy version: [output of `trivy --version`]
Scan date: [date]
Filters: `--severity HIGH,CRITICAL` (the second scan also used `--ignore-unfixed`)

## Before and after comparison

| Measure | my-app:bad | my-app:v2 |
|---|---|---|
| Image size | [MB from `docker images`] | [MB from `docker images`] |
| HIGH findings | [count from before.json] | [count from after.json] |
| CRITICAL findings | [count from before.json] | [count from after.json] |
| Runtime user | root | app (UID 10001) |
| Hard-coded secret | Present (`docker history` shows it) | Not present |
| Base image | python:latest | python:3.12-slim |

Size reduction: [(bad - v2) / bad x 100]% (calculated from my measured sizes)

## Scan results

![scan before](evidence/scan-before.png)
![scan after](evidence/scan-after.png)

Remaining findings: [list what is left, whether a fix exists, and the next step.
If Trivy still returned exit code 1, say so and explain why.
A clean scan is not a guarantee of security.]

## Non-root check

```bash
docker run --rm --entrypoint id my-app:v2
docker image inspect my-app:v2 --format '{{.Config.User}}'
```

Observed: [paste output, expected UID 10001 and not 0]

## Hardened run

Linux, with a local fake token:

```bash
printf '%s' 'training-token-only' > token.txt
chmod 600 token.txt

docker run -d --name safe -p 127.0.0.1:5000:5000 \
  --memory 256m --cpus 0.5 --pids-limit 100 \
  --read-only --tmpfs /tmp \
  --cap-drop ALL --security-opt no-new-privileges \
  -e APP_ENV=lab -e API_TOKEN="$(cat token.txt)" my-app:v2
```

| Option | What it does |
|---|---|
| `-p 127.0.0.1:5000:5000` | Port reachable only from this machine |
| `--memory 256m`, `--cpus 0.5` | Resource limits |
| `--pids-limit 100` | Limits the number of processes |
| `--read-only`, `--tmpfs /tmp` | Read-only root filesystem, writable `/tmp` only |
| `--cap-drop ALL` | Removes Linux capabilities the app does not need |
| `no-new-privileges` | Blocks privilege escalation |
| `-e API_TOKEN=...` | Secret supplied at runtime, not stored in the image |

### Validation

```bash
curl http://127.0.0.1:5000/
curl http://127.0.0.1:5000/health
docker stats --no-stream safe
docker inspect safe --format '{{.HostConfig.ReadonlyRootfs}}'
docker inspect safe --format '{{.HostConfig.Memory}}'
docker inspect safe --format '{{.HostConfig.NanoCpus}}'
docker inspect safe --format '{{.HostConfig.CapDrop}}'
docker inspect safe --format '{{.HostConfig.SecurityOpt}}'
docker logs safe
```

Observed: [paste real output. Expect JSON from `/`, `ok` from `/health`,
ReadonlyRootfs `true`, memory and CPU limits set, capabilities dropped,
no-new-privileges present]

![hardened container](evidence/hardened-container.png)

## Troubleshooting log

| Problem | Evidence | Root cause | Fix |
|---|---|---|---|
| `curl` could not connect to port 5000 after `docker run` | `curl: (7) Failed to connect to 127.0.0.1 port 5000` | [what `docker logs safe` showed] | [what you changed] |

Add any other problems you hit with: symptom, evidence, root cause, fix, retest.

## Cleanup

```bash
docker rm -f safe
rm -f token.txt
```

`reports/before.json` and `reports/after.json` are kept for submission.
Never commit `token.txt` or `.env`. If a real secret were ever built into an
image or pushed to a registry, rotate it immediately. Removing it from the
current Dockerfile does not remove it from old layers or images.

## Interview answers

1. **How do you secure a Docker image?** Small maintained base, pinned tags, multi-stage builds, non-root user, regular scans and rebuilds, no secrets in layers, and runtime restrictions.
2. **How should secrets be handled?** Not in the Dockerfile, ENV, ARG or image. Inject them at runtime or use a secrets manager. Use BuildKit secrets for build-time needs.
3. **What does Trivy scan?** Known vulnerabilities in images, and misconfigurations in supported config files.
4. **Why a slim base image?** Fewer packages, smaller size, less to patch.
5. **What does `--read-only` do?** Makes the root filesystem read-only. Use tmpfs for specific temporary write paths.
6. **Why `--cap-drop ALL`?** Removes privileges a compromised process could abuse.
7. **What do `--memory` and `--cpus` do?** Cap how much memory and CPU the container can use.
8. **Why is removing a secret from the Dockerfile not enough?** It can remain in old layers, history, caches or registries. Rotate it and rebuild cleanly.
9. **Why `--exit-code 1`?** The scan fails when findings match the severity policy, so CI can stop the build.

## Reflection

- **What did I fix?** I used token to make my security strong and compose the images for slim
- **What is still unresolved?** First I run docker my-app in port 5000 but that was not working then I use 5001.
- **What would I add next?**  a Trivy CI gate, BuildKit secrets, pinning the base image by digest
