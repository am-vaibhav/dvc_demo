DVC Data Versioning — Basics
============================

This project demonstrates DVC's core capability: tracking and versioning data files that are too large for Git. It uses a simple data ingestion script that downloads, preprocesses, and saves an e-commerce customer dataset, then tracks the output with DVC.

## How DVC Versioning Works

```text
Git tracks:    code, .dvc files (small pointer files), .dvc/config
DVC tracks:    actual data files (stored in .dvc/cache, pushed to remote)
```

```text
                     .dvc pointer file
dataset.py  ──→  data/  ──→  data.dvc  ──→  Git repo
                   │
                   └──→  .dvc/cache/  ──→  DVC remote (S3 / local)
```

The `.dvc` file is tiny — it stores only a hash (fingerprint) of the data. Git tracks this pointer. DVC tracks the actual data.

---

## What the Script Does

**File:** `src/dataset.py`

```text
1. Downloads e-commerce customer CSV from GitHub
2. Drops first 2 columns (Email, Address — not useful for ML)
3. Filters to customers with Membership > 1 year
4. Saves to data/data_versions/df_v1.csv
```

### Step-by-step through the code

#### Loading data

```python
df = pd.read_csv(data_url)
```

Downloads the CSV directly from a public GitHub URL. No local copy needed.

#### Preprocessing

```python
df = df.iloc[:, 2:]                          # drop first 2 columns
final_df = df[df["Length of Membership"] > 1] # filter short memberships
```

What this means:
- `iloc[:, 2:]` keeps only columns from index 2 onward (drops Email, Address)
- Filters out customers with less than 1 year of membership — they have less data to learn from

#### Saving

```python
raw_data_path = os.path.join(data_path, 'data_versions')
os.makedirs(raw_data_path, exist_ok=True)
df.to_csv(os.path.join(raw_data_path, "df_v1.csv"), index=False)
```

What this means:
- Creates `data/data_versions/` if it doesn't exist
- Saves as `df_v1.csv` — the `v1` naming is manual versioning; DVC handles the real versioning via hashes

### Logging

The script uses Python's `logging` module with two handlers:

| Handler | Level | Output |
|---------|-------|--------|
| Console (`StreamHandler`) | DEBUG | Prints everything to terminal |
| File (`FileHandler`) | ERROR | Only writes errors to `errors.log` |

What this means:
- During normal runs, you see debug messages on screen but `errors.log` stays empty
- If something breaks (bad URL, missing column), the error gets saved to `errors.log` for later review

---

## DVC Remotes

**File:** `.dvc/config`

```ini
[core]
    remote = dvcstore
['remote "local_remote"']
    url = ../temp
['remote "dvcstore"']
    url = s3://dvcdatapipe
```

Two remotes are configured:

| Remote | Type | URL | Purpose |
|--------|------|-----|---------|
| `local_remote` | Local folder | `../temp` | Testing — no cloud needed |
| `dvcstore` | AWS S3 | `s3://dvcdatapipe` | Production — shared storage |

- `remote = dvcstore` sets S3 as the default
- `local_remote` points to `../temp` — a folder one level up, used as a local DVC cache for testing without S3 access
- To switch: `dvc remote default local_remote`

---

## The .dvc Pointer File

**File:** `src/data.dvc`

```yaml
outs:
- md5: 1d8fe066fd68572b5e265943d7f2fb20.dir
  size: 45257
  nfiles: 1
  hash: md5
  path: data
```

What each field means:
- `md5` — hash of the data folder contents. If any file inside changes, this hash changes
- `size` — total size in bytes
- `nfiles` — number of files in the folder (1 = just `df_v1.csv`)
- `path` — the folder being tracked (`data/`)
- `.dir` suffix on the hash means it's tracking a directory, not a single file

This file is what Git tracks. When you `git checkout` to a different commit, the `.dvc` file changes to point to a different hash, and `dvc checkout` restores the matching data.

---

## Full Workflow

```bash
# 1. Run the data script
python src/dataset.py

# 2. Track the output with DVC
dvc add src/data

#    This creates:
#    src/data.dvc       ← pointer file (Git tracks this)
#    src/.gitignore      ← auto-updated to exclude data/ from Git

# 3. Commit the pointer to Git
git add src/data.dvc src/.gitignore
git commit -m "v1: initial dataset"

# 4. Push data to remote storage
dvc push

# 5. Push code to GitHub
git push
```

---

## Versioning Data — When Data Changes

```bash
# 1. Modify the script (e.g., change filter threshold)
# 2. Re-run
python src/dataset.py

# 3. Re-track — DVC detects the data changed and updates the hash
dvc add src/data

# 4. Commit the new pointer
git add src/data.dvc
git commit -m "v2: changed membership filter"

# 5. Push both
dvc push
git push
```

What happens under the hood:
- DVC computes a new hash for the changed data
- The old version is still in `.dvc/cache/` (nothing is deleted)
- The `.dvc` file now points to the new hash
- Git history has both versions — you can go back anytime

---

## Switching Between Versions

```bash
# See all versions
git log --oneline

# Go back to v1
git checkout <commit-hash> -- src/data.dvc
dvc checkout       # restores the data that matches the old .dvc pointer

# Come back to latest
git checkout main -- src/data.dvc
dvc checkout
```

What this means:
- `git checkout` changes the pointer file to an older version
- `dvc checkout` reads the pointer and restores the matching data from cache
- Both commands are needed — git switches the pointer, DVC switches the actual files

---

## Pull Data on Another Machine

```bash
git clone <repo-url>
cd dvc_demo

# Download all tracked data from remote
dvc pull
```

What this means:
- `git clone` gives you code + `.dvc` pointer files (but no data)
- `dvc pull` reads the pointers and downloads the actual data from the remote (S3 or local)
- After `dvc pull`, the data folder is fully restored

---

## Local Remote vs S3 Remote

| | Local Remote (`../temp`) | S3 Remote (`s3://dvcdatapipe`) |
|---|---|---|
| **Setup** | No config needed | Needs `aws configure` |
| **Sharing** | Only works on your machine | Anyone with S3 access can pull |
| **Cost** | Free | S3 storage costs |
| **Use case** | Learning, testing | Team collaboration, production |
| **Speed** | Instant (local copy) | Depends on network |

Switch between them:
```bash
dvc remote default local_remote    # use local
dvc remote default dvcstore        # use S3
```

---

## Common Commands

```bash
dvc add <file/folder>     # start tracking with DVC
dvc push                  # upload data to remote
dvc pull                  # download data from remote
dvc checkout              # restore data to match current .dvc pointers
dvc status                # check if data changed since last dvc add
dvc remote list           # show configured remotes
dvc remove <file.dvc>     # stop tracking a file with DVC
dvc diff                  # show what changed in tracked data
```
