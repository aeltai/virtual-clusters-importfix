# Virtual Clusters UI — import job image fix

Prebuilt Rancher UI extension package that fixes the Virtual Clusters (K3K) **cluster import job** image for air-gapped and production environments.

Upstream PR: [rancher/virtual-clusters-ui#150](https://github.com/rancher/virtual-clusters-ui/pull/150)  
Upstream issue: [rancher/virtual-clusters-ui#151](https://github.com/rancher/virtual-clusters-ui/issues/151)

## Problem

The import job created by Virtual Clusters UI **1.1.0** hardcodes `rancher/shell:head`. That image:

- ignores Rancher’s `system-default-registry` setting
- uses a mutable `head` tag that air-gap image lists normally do not ship

## Fix

At job creation time the extension resolves the container image as:

1. Rancher setting `shell-image`, if set
2. otherwise pinned `rancher/shell:v0.7.0`
3. then prefixes `system-default-registry` when the image has no registry host

Fully-qualified `shell-image` values are used unchanged.

## Package

| Artifact | Path |
|----------|------|
| Release tarball | [`virtual-clusters-1.1.0-importfix.tar.gz`](./virtual-clusters-1.1.0-importfix.tar.gz) |
| Unpacked bundle | [`dist/virtual-clusters-1.1.0/`](./dist/virtual-clusters-1.1.0/) |
| Entry script | `virtual-clusters-1.1.0.umd.min.js` |

Built from Virtual Clusters UI **1.1.0** with the import-job image fix applied.

A Helm chart of the same fix is also published from the fork at
[aeltai/virtual-clusters-ui](https://github.com/aeltai/virtual-clusters-ui) (`gh-pages`, version `1.1.1-importfix.1`).

## Prerequisite

Ensure `rancher/shell:v0.7.0` is available in your private registry (included in Rancher air-gap image lists for recent releases), **or** set the Rancher management setting `shell-image` to an image you already mirror (for example `<registry>/rancher/shell:v0.6.3`).

## Install options

### A — Extension repository (recommended)

Install or update through the Rancher **Extensions** UI using the Helm chart repo on `gh-pages`.

1. In Rancher: **Extensions → ⋮ → Manage Repositories → Create**
2. Set:
   - **Target:** Git repository  
   - **Git Repo URL:** `https://github.com/aeltai/virtual-clusters-ui`  
   - **Branch:** `gh-pages`
3. Save, then open **Extensions → Available** / **Updates**
4. Install or update **Virtual Clusters** to version **`1.1.1-importfix.1`**
5. Create a test virtual cluster via **Cluster Management → Create → K3K**

Rollback: on the extension card, choose the previous version (e.g. `1.1.0`).

> Air-gap note: the Rancher management cluster must be able to reach `github.com` (or mirror this Git repo internally and point the repository URL at your mirror).

### B — Developer Load (non-invasive)

Loads the patched extension only in your browser session. Stock 1.1.0 stays installed. Reload the page to roll back.

1. Host `dist/virtual-clusters-1.1.0/` over HTTPS so the browser can reach every file under that path.
2. In Rancher: avatar → **Preferences** → enable **Extension developer features**.
3. **Extensions** → ⋮ → **Developer Load**.
4. Extension URL:

   ```text
   https://<your-host>/virtual-clusters-1.1.0/virtual-clusters-1.1.0.umd.min.js
   ```

5. Create a test virtual cluster via **Cluster Management → Create → K3K**.

### C — Replace files in `ui-plugin-catalog` (persistent)

```bash
NS=cattle-ui-plugin-system
POD=$(kubectl -n "$NS" get pod -o jsonpath='{.items[0].metadata.name}')

# Backup
kubectl -n "$NS" exec "$POD" -- cp -r \
  /home/plugin-server/plugin-contents/plugin/virtual-clusters-1.1.0 \
  /tmp/virtual-clusters-1.1.0.bak

# Install patched files
cd dist/virtual-clusters-1.1.0
for f in *.js package.json; do
  kubectl -n "$NS" cp "$f" \
    "$POD:/home/plugin-server/plugin-contents/plugin/virtual-clusters-1.1.0/plugin/$f"
done

# Rebuild Rancher plugin cache
kubectl -n "$NS" patch uiplugin virtual-clusters --type merge \
  -p '{"spec":{"plugin":{"noCache":true}}}'
sleep 10
kubectl -n "$NS" patch uiplugin virtual-clusters --type merge \
  -p '{"spec":{"plugin":{"noCache":false}}}'
kubectl -n "$NS" get uiplugin virtual-clusters -w
```

Hard-reload the browser, then create a test virtual cluster.

## Verify

On the host / downstream cluster where the import job runs:

```bash
kubectl get job -n <namespace> -l job-name -o wide
# or by name, e.g. import-<cluster-name>
kubectl get job import-<cluster-name> -n <namespace> \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Expected image (example):

```text
<system-default-registry>/rancher/shell:v0.7.0
```

The job pod should pull successfully and reach `Completed`, and the virtual cluster should register in Rancher.

Jobs created **before** the fix still show `rancher/shell:head`.

## Rollback

- **Extension repository:** roll the extension back to `1.1.0` in the Extensions UI.
- **Developer Load:** reload the page (or remove the developer-loaded extension).
- **Catalog pod swap:** restore the backup and toggle `noCache` again:

```bash
kubectl -n "$NS" exec "$POD" -- sh -c \
  'cp /tmp/virtual-clusters-1.1.0.bak/plugin/* \
   /home/plugin-server/plugin-contents/plugin/virtual-clusters-1.1.0/plugin/'
```

## License

Same license as upstream [rancher/virtual-clusters-ui](https://github.com/rancher/virtual-clusters-ui) (Apache-2.0).
