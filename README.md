# oci-capacity-watcher

Retries launching an Oracle Cloud **Always Free** ARM server (`VM.Standard.A1.Flex`) every ~15 min until capacity frees up. Stops once the instance exists.

## Setup (about 20 min)

### 1. Create your own copy

The workflow must run in **your** GitHub account, with your own secrets.

1. Click **Use this template → Create a new repository** (top right).
2. Name it, choose **Private**, click **Create repository**.

Actions are enabled by default in the new repo.

### 2. Get an Oracle Cloud account

1. Sign up at <https://www.oracle.com/cloud/free/> (card needed for verification, not charged).
2. Pick your home region carefully. It can't be changed. The workflows default to `eu-frankfurt-1`. To use another region, keep `region=` in your `OCI_CONFIG` set to it, and it takes priority.

### 3. Add secrets

Your new repo → **Settings → Secrets and variables → Actions → New repository secret**.

| Secret | Where to get it |
|---|---|
| `OCI_CONFIG` | OCI Console → profile → **My profile → API keys → Add API key → Generate**. Paste the config preview, with `key_file=~/.oci/oci_api_key.pem`. |
| `OCI_PRIVATE_KEY` | The `.pem` file downloaded in the step above (full contents). |
| `OCI_COMPARTMENT_ID` | **Identity → Compartments** → copy OCID. Root compartment = your `tenancy` OCID. |
| `OCI_SUBNET_ID` | **Networking → VCNs** → create one with the wizard if needed → public subnet → copy OCID. Allow inbound TCP 22. |
| `OCI_IMAGE_ID` | Run `oci-lookup` (see step 4). Copy the newest `id` from its output. Needs the three secrets above first. |
| `OCI_SSH_PUBLIC_KEY` | Output of `ssh-keygen -t ed25519 -f ~/.ssh/oci_key` then `cat ~/.ssh/oci_key.pub`. |
| `OCI_INSTANCE_NAME` | Optional. Default `my-server`. |
| `NTFY_TOPIC` | Optional. Random string for [ntfy.sh](https://ntfy.sh) alerts, e.g. `oci-$(openssl rand -hex 8)`. Subscribe in the ntfy app. |

Example `OCI_CONFIG`:

```ini
[DEFAULT]
user=ocid1.user.oc1..xxxx
fingerprint=aa:bb:cc:...
tenancy=ocid1.tenancy.oc1..xxxx
region=eu-frankfurt-1
key_file=~/.oci/oci_api_key.pem
```

### 4. Find your image ID

Do this before the first test. It gives you `OCI_IMAGE_ID`.

1. Add the secrets `OCI_CONFIG`, `OCI_PRIVATE_KEY` and `OCI_COMPARTMENT_ID`.
2. **Actions → oci-lookup → Run workflow**.
3. Defaults list Ubuntu 24.04 ARM images. Optional: type another OS or version in the form, e.g. `Oracle Linux` / `9`.
4. Open the finished run, copy the newest `id`, save it as secret `OCI_IMAGE_ID`.

### 5. Test

1. Open the **Actions** tab.
2. Run **oci-capacity-retry** manually.

### 6. Enable the schedule

Off by default. In `.github/workflows/oci-capacity-retry.yml`, uncomment:

```yaml
  schedule:
    - cron: '13,28,43,58 * * * *'
```

### 7. After the server exists

1. Find the public IP in **Compute → Instances**.
2. Connect: `ssh -i ~/.ssh/oci_key ubuntu@<public-ip>`
3. Re-comment the cron.

## Notes

- **Size:** 1 OCPU / 6 GB, the size most likely to be available. Free limit is 2 OCPU / 12 GB. Edit `--shape-config` in the workflow to change it. Bigger requests fail more often.
- **Availability domains:** looked up automatically.
- **Logs are safe:** only error codes are printed, never IDs or names.
- **Free Actions minutes:** private repos get about 2,000 min/month. A run every 15 min uses roughly 100 min/day, so the limit lasts ~3 weeks. To stretch it, use `'13,43 * * * *'` (every 30 min).
- **Failed runs are normal.** "No capacity" counts as a failed run, and GitHub may email you. Mute it under GitHub **Settings → Notifications → Actions**.
- **GitHub limits:** scheduled runs can lag 5-30 min. Public repos get the schedule disabled after 60 days of no activity. Re-enable it in the Actions tab.
- **Pinned versions:** `oci-cli` 3.96.3 and `python:3.12.16-slim`. Bump them yourself when you want updates.

## Errors

| Error | Cause |
|---|---|
| `Out of host capacity` | Expected. Keeps retrying. Can take hours to days. |
| Timeout, no reply, `ConnectTimeout`, 500, 502, 503, `TooManyRequests` (429) | Oracle's API sometimes doesn't answer or is overloaded. Normal. Ignore it. The next scheduled run retries. |
| `NotAuthenticated` / 401 | `OCI_CONFIG` fingerprint, user or tenancy doesn't match the uploaded key. |
| `LimitExceeded` | Free A1 quota already used. |
| `NotAuthorizedOrNotFound` | Wrong compartment, subnet or image OCID. Images are per region. |

## License

MIT, see [LICENSE](LICENSE).
