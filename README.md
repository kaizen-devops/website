# kaizendevops 
Website for the business kaizendevops

## Colors used for this website and logo

![kaizendevops.png](docs/images/kaizendevops.png)

| Purpose / Element           | Hex Code  | RGB               | Notes                                |
| --------------------------- | --------- | ----------------- | ------------------------------------ |
| Cloud outline (top right)   | `#1D74B7` | rgb(29, 116, 183) | Dominant outer cloud blue            |
| Kubernetes logo background  | `#326CE5` | rgb(50, 108, 229) | Blue hexagon behind K8s ship wheel   |
| Middle square (DevOps logo) | `#4CB2E2` | rgb(76, 178, 226) | Light cyan-blue square outline       |
| "DEVOPS" text               | `#3B7ABD` | rgb(59, 122, 189) | Slightly muted blue used in the text |

## Local development

This site is built with [MkDocs](https://www.mkdocs.org/) + the [Material](https://squidfunk.github.io/mkdocs-material/) theme.

1. Create and activate a virtualenv (optional but recommended):

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

2. Install the dependencies (same plugins installed in the `Containerfile`):

   ```bash
   pip install mkdocs-material mkdocs-macros-plugin mkdocs-glightbox mkdocs-video mkdocs-static-i18n
   ```

3. Run the live-reloading dev server:

   ```bash
   mkdocs serve
   ```

   The site is now available at http://127.0.0.1:8000 and reloads automatically as you edit files under `docs/`.

4. Build the static site (outputs to `site/`):

   ```bash
   mkdocs build
   ```

## Building the container image

The `Containerfile` does a multi-stage build: it builds the static site with MkDocs, then serves it with `nginx-unprivileged`.

Build the image locally:

```bash
docker build -t rene007/kaizendevops:tagname -f Containerfile .
```

Run it locally to check the result (served on port 8080 by default for the unprivileged nginx image):

```bash
docker run --rm -p 8080:8080 rene007/kaizendevops:tagname
```

Then open http://localhost:8080.

## Pushing to Docker Hub

1. Log in to your private Docker Hub repo (only needed once per session/machine):

   ```bash
   docker login
   ```

2. Tag the image if you haven't already baked the tag into the build (replace `tagname`, e.g. with a version like `v1.0.0` or `latest`):

   ```bash
   docker tag rene007/kaizendevops:tagname rene007/kaizendevops:tagname
   ```

3. Push it:

   ```bash
   docker push rene007/kaizendevops:tagname
   ```

> Tip: build and push in one go for a new tag:
>
> ```bash
> docker build -t rene007/kaizendevops:tagname -f Containerfile . && docker push rene007/kaizendevops:tagname
> ```

## CI: build, push and sign

`.github/workflows/build-image.yml` builds the image from the `Containerfile`, pushes it to Docker Hub and signs it with [cosign](https://docs.sigstore.dev/) using our own key pair. The signature is stored next to the image on Docker Hub only, so nothing is written to the public Sigstore transparency log.

- **Push to `main`**: pushes `main`, `latest` and `sha-<short>` tags, then signs the image.
- **Tag `v1.2.3`**: pushes `1.2.3`, `1.2` and `sha-<short>` tags, then signs the image.
- **Pull request**: builds only, without pushing or signing.

### Create the signing key pair (once)

```bash
cosign generate-key-pair
```

cosign asks for a password and writes two files:

- `cosign.key`: the encrypted private key. **Never commit it** (it is in `.gitignore`). Keep a copy in your password manager.
- `cosign.pub`: the public key. Safe to share and commit; it is what you verify with.

### Repository secrets

Add these under Settings → Secrets and variables → Actions:

| Secret               | Value                                                     |
| -------------------- | --------------------------------------------------------- |
| `DOCKERHUB_USERNAME` | `rene007`                                                 |
| `DOCKERHUB_TOKEN`    | A Docker Hub access token with *Read & Write* scope       |
| `COSIGN_PRIVATE_KEY` | The full contents of `cosign.key`                         |
| `COSIGN_PASSWORD`    | The password you entered during `cosign generate-key-pair` |

With the GitHub CLI:

```bash
gh secret set COSIGN_PRIVATE_KEY < cosign.key
gh secret set COSIGN_PASSWORD        # prompts for the value
gh secret set DOCKERHUB_USERNAME --body rene007
gh secret set DOCKERHUB_TOKEN        # prompts for the value
```

### Verify a signature

Because there is no transparency log entry, verification needs `--insecure-ignore-tlog`:

```bash
cosign verify --key cosign.pub --insecure-ignore-tlog \
  docker.io/rene007/kaizendevops:latest
```
