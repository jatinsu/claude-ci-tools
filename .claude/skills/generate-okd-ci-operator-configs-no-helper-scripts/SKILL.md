---
name: generate-okd-ci-operator-configs-no-helper-scripts
description: Generate OKD/SCOS ci-operator configuration YAML files from ART ocp-build-data. Replicates the logic of doozer's `images:okd prs open` command as a standalone tool.
argument-hint: "<ocp-build-data-path> [--okd-version <version>] [--output-dir <path>] [--release-repo <path>] [--base-stream <9|10>] [--github-token <token>]"
---

Generate OKD/SCOS ci-operator configuration YAML files for the openshift/release repository from ART image metadata. This skill replicates the core config-generation logic of doozer's `images:okd prs open` command without requiring the doozer runtime, using subagents for parallelization.

## Arguments

Parse from: $ARGUMENTS

- First positional argument: path to the `ocp-build-data` directory (must contain `group.yml`, `streams.yml`, and an `images/` subdirectory)
- `--okd-version <version>`: OKD version string (e.g. `4.18`, `5.0`). If omitted, derived from `group.yml` vars as `{MAJOR}.{MINOR}`
- `--output-dir <path>`: Directory to write ci-operator config files (default: `./okd-ci-configs`). Files are written at `{output-dir}/{org}/{repo}/{org}-{repo}-{branch}__okd-scos.yaml`
- `--github-token <token>`: GitHub personal access token for downloading upstream Dockerfiles. If omitted, uses unauthenticated requests (subject to rate limits)
- `--release-repo <path>`: Path to an openshift/release checkout. Use it whenever one is available. Subagents 1 and 3 read each repo's OCP config from `{release-repo}/ci-operator/config/{org}/{repo}/{org}-{repo}-{branch}.yaml` to check the branch name, image `capabilities` and builder versions. Subagent 3 also reads the content mirror's RPM repo names from `{release-repo}/core-services/release-controller/_repos/`. Without it, skip those checks, use the hardcoded repo-name rule, and say so in the report.
- `--base-stream <9|10>`: CentOS Stream for the scos base images. Defaults to 10 for OKD 5.1 and later and 9 for earlier versions.

If no ocp-build-data path is provided, search these locations in order:

- `./tmp-dir/ocp-build-data/`
- `../ocp-build-data/`
- `./ocp-build-data/`

## Overview

This skill generates ci-operator configuration YAML files that tell Prow how to build OKD/SCOS images. The process has three phases:

1. **Data Loading** — Read group.yml, streams.yml, and all image metadata YAMLs. Apply variable substitution. Build a dependency graph. Filter to OKD-relevant images and resolve their upstream URLs, branches, and OKD pullspecs.
2. **Dockerfile Analysis** — Download each image's upstream Dockerfile from GitHub. Parse FROM statements to identify parent images and stage names.
3. **Config Generation** — For each unique (org, repo, branch) group, assemble a ci-operator config YAML with the correct base_images, build_root, images, promotion, releases, resources, and optional raw_steps.

## Recording fixes

Many rules below exist because a generated 5.0 or 5.1 config was later fixed by hand in openshift/release (PR numbers are in parentheses). Fixes made only to 4.22 configs are deliberately left out. Each subagent must keep two lists and return them with its output:

- `fixes`: every change a rule made, with the image or repo, the old and new value, and the PR number.
- `checks`: everything a person needs to look at.

Write them to `OCP_BUILD_DATA_PATH/../okd_fixups.yaml` (append, don't overwrite) as `{fixes: [...], checks: [...]}`.

## Execution

### Step 1: Validate inputs and read group config

Read `{ocp-build-data}/group.yml` and extract:

- `vars.MAJOR` and `vars.MINOR` (integers)
- `public_upstreams` (list of `{private, public, public_branch?}` mappings)

Derive `okd_version` as `"{MAJOR}.{MINOR}"` if not provided. Set `base_stream` from `--base-stream`, or else to 10 for OKD 5.1 and later and 9 for earlier versions. Set `release_repo` from `--release-repo` and check that `{release_repo}/ci-operator/config` exists. Pass `BASE_STREAM` and `RELEASE_REPO` into every subagent prompt along with the paths.

Read `{ocp-build-data}/streams.yml` and apply variable substitution (replace all `{VAR_NAME}` patterns with values from `group.yml` vars). Build a stream lookup table.

### Step 2: Launch Subagent 1 — Data Analysis

Launch one subagent with the Agent tool using the following prompt (substitute the actual paths and values):

---

**Subagent 1 prompt:**

You are analyzing ART ocp-build-data to determine which images need OKD ci-operator configurations.

**Input:** ocp-build-data at `OCP_BUILD_DATA_PATH`, OKD version `OKD_VERSION`, base stream `BASE_STREAM`, openshift/release checkout `RELEASE_REPO` (may be empty)

**Task:** Read all YAML files, apply variable substitution, filter to OKD-relevant images, and produce a manifest file.

Follow these steps exactly:

1. **Read group.yml** at `OCP_BUILD_DATA_PATH/group.yml`. Extract `vars` (a dict of variable names to values, e.g. MAJOR=5, MINOR=0, GO_LATEST="1.26"). Extract `public_upstreams` (a list of private-to-public URL mappings).

2. **Read streams.yml** at `OCP_BUILD_DATA_PATH/streams.yml`. For each stream entry, apply variable substitution: replace all occurrences of `{VARNAME}` in string values with the corresponding value from `vars`. Record each stream's `upstream_image` and `image` fields (after substitution).

3. **Read every `.yml` file** in `OCP_BUILD_DATA_PATH/images/`. For each file:
   - Parse the YAML
   - Apply variable substitution recursively through all string values
   - The distgit_key is the filename without `.yml` extension

4. **Determine which images are needed for OKD payload construction.** An image is needed if ANY of these are true:
   - Its `for_payload` field is `true`
   - It is referenced as `from.member` by any image that is needed for OKD payload
   - It is referenced as `from.builder[].member` by any image that is needed for OKD payload
     This requires building a dependency graph and recursively marking images as needed.

5. **Filter images.** For each image, skip it if ANY of these conditions apply:
   - `content.source.okd_alignment.enabled` is explicitly `false` (if the key is absent, the image is NOT skipped)
   - The image has no `from` config at all
   - `content.source.okd_alignment.resolve_as` is set (these images are resolved, not built)
   - The image is not needed for OKD payload construction (from step 4) AND does not have `content.source.okd_alignment.resolve_as.tag_name` set (note: this checks `resolve_as.tag_name`, NOT the top-level `tag_name` — images that only have a top-level `tag_name` but are not needed for payload ARE skipped)
   - The image has no `content.source.git.url` field, or the URL does not contain `github.com`
   - Its payload_tag (computed in step 6a) is `upi-installer`. UPI isn't tested on OKD (openshift/release#87242). Record the reason in the skipped list.

6. **For each remaining image, compute:**

   a. **payload_tag**: The tag name for this image in the CI imagestream.
   - If `content.source.okd_alignment.tag_name` is set, use it
   - Else if `for_payload` is true: use `payload_name` if set, otherwise take the last path component of `name` and strip the `ose-` prefix if present
   - Else (non-payload builder/base image): take the last path component of `name` and strip the `ose-` prefix

   b. **desired_parents**: A list of OKD pullspecs for all FROM stages.
   - If `content.source.okd_alignment.from` exists: use it directly as the list of pullspecs
   - Otherwise, resolve each `from.builder[]` entry and then the `from.member`/`from.stream`/`from.image` entry using these resolution rules:
     - For `member` entries: find the referenced image's metadata and resolve its OKD pullspec (see Image OKD Pullspec Resolution below)
     - For `stream` entries: resolve via Stream Resolution rules below
     - For `image` entries: use the literal pullspec as-is

   b2. **Adjust desired_parents** (skip this step when `okd_alignment.from` was used, since that list is ART's decision):
   - **dhclient pin (#84562).** For OKD 5.0 and later, if payload_tag is `agent-installer-node-agent`, set the last parent to `registry.ci.openshift.org/origin/scos-4.22:base-stream9`. CS10 has no `dhclient`. Record a fix and skip the stream rule for this image.
   - **Base stream (#85990).** If BASE_STREAM is 10, replace every `registry.ci.openshift.org/origin/scos-OKD_VERSION:base-stream9` entry with `...:base-stream10`, builder stages included. Record a fix.
   - **Hand-maintained images.** If payload_tag is `haproxy-router-base`, `haproxy-router-haproxy32` or `haproxy-router-haproxy28`, record a check. These were hand-edited upstream (stream9 pins in #84562, an EL9 compat image for haproxy32 in #86771, haproxy28 retired for 5.1+ in #87955), so a person must compare with the existing config.

   c. **public_url and public_branch**: Map the private source git URL to its public equivalent using `public_upstreams`:
   - For each mapping in `public_upstreams`, check if the image's HTTPS-normalized source URL starts with the HTTPS-normalized private URL
   - Use the longest matching private prefix
   - Replace the matched prefix with the corresponding public URL
   - If the mapping has a `public_branch`, use it; otherwise fall back to the source branch
   - **Branch check (#86173, #85816).** If RELEASE_REPO is set and `RELEASE_REPO/ci-operator/config/{org}/{repo}/{org}-{repo}-{branch}.yaml` doesn't exist, try the alternate name: `release-X.Y` and `openshift-X.Y` swap, as do `master` and `main`. If the alternate's OCP config exists, use that branch and record a fix. If neither exists, keep the branch and record a check. (etcd moved from `openshift-X.Y` to `release-X.Y`, and router moved from `master` to `main`.)

   d. **dockerfile_path**:
   - If `content.source.okd_alignment.dockerfile` is set, use it
   - Else if `content.source.dockerfile` is set, use it
   - Else default to `"Dockerfile"`
   - Then prepend the path prefix: if `content.source.okd_alignment.path` is set, join it; else if `content.source.path` is set, join it

   e. **build_root**:
   - If `content.source.okd_alignment.ci_build_root` is set, resolve it to an OKD pullspec
   - Otherwise, default to the `rhel-9-golang` stream's upstream_image

   f. **inject_rpm_repositories**: Copy from `content.source.okd_alignment.inject_rpm_repositories` if present

   g. **build_args**: Copy from `content.source.okd_alignment.build_args` if present

   h. **context_dir**: Copy from `content.source.okd_alignment.context_dir` if present

   i. **org and repo_name**: Split the public_url to extract the GitHub org and repo name

7. **Write the manifest** to `OCP_BUILD_DATA_PATH/../okd_manifest.yaml` with this structure:

```yaml
okd_version: "5.0"
images:
  - distgit_key: "marketplace-operator"
    payload_tag: "operator-marketplace"
    for_payload: true
    desired_parents:
      - "registry.ci.openshift.org/ocp/builder:rhel-9-golang-1.26-openshift-5.0"
      - "registry.ci.openshift.org/origin/scos-5.0:base-stream9"
    public_url: "https://github.com/operator-framework/operator-marketplace"
    public_branch: "master"
    dockerfile_path: "Dockerfile.okd"
    build_root: "registry.ci.openshift.org/openshift/release:rhel-9-release-golang-1.26-openshift-5.0"
    org: "operator-framework"
    repo_name: "operator-marketplace"
    inject_rpm_repositories: null
    build_args: null
    context_dir: null
    from_builders:
      - member: null
        stream: "rhel-9-golang"
        image: null
      # ... etc
    from_base:
      member: "openshift-enterprise-base-rhel9"
```

Also write a summary of skipped images to `OCP_BUILD_DATA_PATH/../okd_skipped.yaml`.

### Stream Resolution Rules

To resolve a stream name to an OKD pullspec:

1. Look up the stream in streams.yml (after variable substitution)
2. If the stream has `okd.resolve_as.image` → use that value
3. If the stream has `upstream_image` → use that value
4. Otherwise → use the stream's `image` field

### Image OKD Pullspec Resolution

To resolve an image metadata entry to an OKD pullspec:

1. If `content.source.okd_alignment.resolve_as.stream` is set → resolve via Stream Resolution
2. If `content.source.okd_alignment.resolve_as.image` is set → use that literal pullspec
3. If `content.source.okd_alignment.tag_name` is set → `registry.ci.openshift.org/origin/scos-{okd_version}:{tag_name}`
4. Otherwise → take `payload_name` (or last component of `name`), strip `ose-` prefix, use as tag: `registry.ci.openshift.org/origin/scos-{okd_version}:{stripped_name}`

### URL Normalization

To normalize a git URL to HTTPS for comparison:

- Strip leading `git@`, `ssh://`, `http://`, `https://`, `git://`
- Strip trailing `.git`
- Replace `:` between host and path with `/`
- Prepend `https://`

---

### Step 3: Launch Subagent 2 — Dockerfile Fetching and Parsing

After Subagent 1 completes, read the manifest file it produced. Then launch a subagent to download and parse all upstream Dockerfiles.

**Subagent 2 prompt:**

You are downloading and parsing upstream Dockerfiles for OKD image builds.

**Input:** Read the manifest at `MANIFEST_PATH`.

**Task:** For each image in the manifest, download its Dockerfile from GitHub, parse FROM statements, and update the manifest.

For each image entry:

1. **Download the Dockerfile** from the public GitHub repo:

   ```
   curl -sL -H "Authorization: token GITHUB_TOKEN" \
     "https://raw.githubusercontent.com/{org}/{repo_name}/{public_branch}/{dockerfile_path}"
   ```

   (Omit the Authorization header if no token is provided.)

2. **Handle symlink-like files**: If the downloaded content is a single line that does NOT start with `FROM` (case-insensitive), treat it as a relative path to the real Dockerfile. Resolve the path relative to the current dockerfile's directory (using `os.path.join(os.path.dirname(dockerfile_path), content.strip())`), download again, AND UPDATE the image's `dockerfile_path` to the resolved path. Repeat until you get a real Dockerfile. This is critical — the final `dockerfile_path` must reflect the resolved location, not the original symlink.

3. **Parse FROM statements**: For each `FROM` line in the Dockerfile, extract:
   - The image pullspec (everything after `FROM` up to `AS` or end of line)
   - The stage name (the name after `AS`, if present; null otherwise)

   Record these as a list: `[{image: "...", stage_name: "..." or null}, ...]`

4. **Validate parent count**: The number of FROM statements in the Dockerfile MUST equal the number of entries in `desired_parents`. If they don't match, mark the image as skipped with reason "parent count mismatch".

5. **Write updated manifest** to `MANIFEST_DIR/okd_manifest_with_dockerfiles.yaml`, adding a `dockerfile_froms` field to each image entry:

   ```yaml
   dockerfile_froms:
     - image: "registry.ci.openshift.org/ocp/builder:rhel-9-golang-1.25-openshift-4.22"
       stage_name: "builder"
     - image: "registry.ci.openshift.org/ocp/4.16:base-rhel9"
       stage_name: null
   ```

---

### Step 4: Launch Subagent 3 — Config Generation

After Subagent 2 completes, read the updated manifest. Then launch a subagent to generate the ci-operator YAML config files.

**Subagent 3 prompt:**

You are generating ci-operator configuration YAML files for OKD/SCOS builds.

**Input:** Read the manifest at `MANIFEST_WITH_DOCKERFILES_PATH`. Output directory: `OUTPUT_DIR`. openshift/release checkout: `RELEASE_REPO` (may be empty). OKD version: `OKD_VERSION`.

**Task:** Group images by (org, repo_name, public_branch). For each group, generate one ci-operator config YAML file.

For each group:

1. **Compute the output path**: `{OUTPUT_DIR}/{org}/{repo_name}/{org}-{repo_name}-{public_branch}__okd-scos.yaml`

2. **Build the ci-operator image name mapping.** Every image reference (base images, builder replacements, built images) needs a ci-operator-internal name:
   - For external dependencies (base images from other imagestreams): use `{namespace}_{name}_{tag}` (underscores joining the ImageCoordinate fields). An ImageCoordinate is parsed from a `registry.ci.openshift.org/namespace/name:tag` pullspec.
   - For images built within this config: use the image's `payload_tag`

3. **Determine base_images vs built images.** Start by assuming all image dependencies are base images (externally sourced). Then for any image that is actually built by this config, remove it from base_images. An image is built by this config if its ImageCoordinate's namespace matches the promotion namespace (`origin`) and name matches the promotion imagestream (`scos-{okd_version}`).

4. **For each image in the group, build the image entry:**

   ```yaml
   build_args:
     - name: TAGS
       value: scos
     # Plus any additional build_args from okd_alignment.build_args
   dockerfile_path: { dockerfile_path } # relative to context_dir if set
   from: { ci_operator_name_of_last_desired_parent }
   to: { payload_tag }
   ```

   **Inputs (builder replacements):** For each FROM stage EXCEPT the last one (i.e., for index 0 through len(dockerfile_froms)-2):
   - Get the corresponding builder entry from `from_builders[index]` in the image metadata
   - **SKIP this replacement entirely** if ANY of:
     - The builder entry has an `image` key (it's a literal image pullspec, not a member or stream reference)
     - The index is beyond the length of `from_builders` (the Dockerfile has more stages than the metadata declares builders)
   - Build a replacement list containing:
     - The stage name from the Dockerfile (if it has one via `AS name`)
     - The original pullspec from the Dockerfile's FROM line
   - Map these to the ci-operator name of the corresponding `desired_parents[index]` entry

   ```yaml
   inputs:
     { ci_operator_name_of_desired_parent }:
       as:
         - { stage_name } # if present
         - { original_pullspec } # from the Dockerfile
   ```

   **Context dir:** If `context_dir` is set on the image, add it to the image entry. If the `dockerfile_path` starts with the `context_dir`, strip the context_dir prefix from the dockerfile_path.

   **inject_rpm_repositories (raw_steps):** If the image has `inject_rpm_repositories`, generate a `raw_steps` entry:
   - Create an intermediate tag name: `pre-repo-{payload_tag}`
   - Build a `pipeline_image_cache_step` that writes a yum repo file:

     ```yaml
     raw_steps:
       - pipeline_image_cache_step:
           commands: |

             cat << EOF > /etc/yum.repos.d/art.repo
             [{repo_id}]
             id = {repo_id}
             name = {repo_id}
             baseurl = {baseurl}
             enabled = 1
             gpgcheck = 0
             sslverify = false
             skip_if_unavailable = true

             EOF

           from: { ci_operator_name_of_base_image }
           to: pre-repo-{payload_tag}
     ```

   - Set the image entry's `from` to the intermediate tag instead of the base image name
   - **Fix the mirror host and repo path before writing the repo file** (#84562, #86112). Record each change as a fix.
     - Rewrite any `base-A-B-rhelN.ocp.svc` host in `baseurl` to use the OKD version (`base-5-0-rhel10` for 5.0). A 5.0 config once pointed at `base-5-1-*`.
     - The mirror serves each repo at a path equal to its repo id, so the id must be one the mirror defines. For ids of the form `rhel-N-server-ose` (with or without a `-X.Y` suffix):
       - If RELEASE_REPO is set and `RELEASE_REPO/core-services/release-controller/_repos/ocp-X.Y-rhelN.repo` exists (`rhelN` taken from the mirror host, for example `rhel10` from `base-5-1-rhel10`), read its `[section]` names. Keep the id if it's listed. Otherwise use `rhel-N-server-ose-X.Y` if listed, or else `rhel-N-server-ose` if listed. If none are listed, keep the id and record a check, because the URL will probably return 404.
       - Otherwise, use the hardcoded rule: RHEL 10 and later get `-X.Y` and RHEL 9 doesn't. This matches the mirror files as of 2026-11. If RELEASE_REPO was set but the file was missing, record a check.
       - Change the last path segment of `baseurl` to the final id.

   **Capabilities (#84305).** Every image in an `openshift/installer` config gets `capabilities: [intranet]`. Those builds pull RPMs from intranet hosts and fail on build01.

5. **Cross-check against the OCP config** when RELEASE_REPO is set. Read `RELEASE_REPO/ci-operator/config/{org}/{repo}/{org}-{repo}-{branch}.yaml`. If it doesn't exist, skip this step.
   - **Capabilities.** For each image, find the OCP image with the same `dockerfile_path`. If it has `capabilities` and the generated image doesn't, copy them and record a fix.
   - **Builder versions (#84589, #88570).** Compare `rhel-N-golang-A.B-openshift-X.Y` builder tags in `base_images`. If the OCP config uses a newer golang or openshift version for the same RHEL major, record a check.
   - **`.rhel10` Dockerfiles (#88570).** If the OCP config builds a `.rhel10` Dockerfile in the same directory as a generated image's Dockerfile, and the generated one isn't `.rhel10`, record a check. driver-toolkit needed `Dockerfile.rhel10` for 5.1.

6. **Assemble the complete config:**

   ```yaml
   base_images:
     { unique_key }:
       namespace: { namespace }
       name: { name }
       tag: { tag }
   build_root:
     image_stream_tag:
       namespace: { namespace }
       name: { name }
       tag: { tag }
   images:
     items:
       - { image entries from step 4 }
   promotion:
     to:
       - namespace: origin
         name: scos-{okd_version}
   raw_steps: # only if any images have inject_rpm_repositories
     - { raw_step entries }
   releases:
     latest:
       integration:
         namespace: origin
         name: scos-{okd_version}
   resources:
     "*":
       requests:
         cpu: 100m
         memory: 200Mi
   ```

   Key ordering in the output: `base_images`, `build_root`, `images`, `promotion`, `raw_steps`, `releases`, `resources`. Only include sections that have content (e.g. omit `base_images` if there are none, omit `raw_steps` if none exist). `images` must be an `items` list, because determinize-ci-operator rejects a flat list. `promotion.to` holds only `origin/scos-{okd_version}`, never a `-art` stream (#87110, #87847).

7. **Write the YAML file** using `yaml.safe_dump()` with `default_flow_style=False`.

8. **Report** the total number of files written and their paths.

---

### Step 5: Report results

After all subagents complete, report:

1. Total number of ci-operator config files generated
2. List of output file paths
3. Number of images skipped and summary of reasons
4. Any errors encountered
5. The `fixes` from `okd_fixups.yaml`, summarized by rule (for example, "160 images: base-stream9 changed to base-stream10")
6. Every `checks` entry, word for word
7. If RELEASE_REPO was not given, a note that the branch, capabilities and builder checks didn't run

### Step 6: Post-generation checklist

The rules above can't decide these from ART metadata. Go through them before calling the output ready, and report each one that applies:

1. **Missing payload components.** Compare the generated repos with the payload's components. New components (for example kube-vip, karpenter-operator and agentic-skills in 5.1) were left out before and had to be added in openshift/release#85803. Every skipped `for_payload` image belongs in the report.
2. **openshift/os.** Its okd-scos config must promote only to `origin/scos-X.Y`, never to `scos-X.Y-art` (#87110, #87847). `OPENSHIFT_VERSION` and the `server-ose-X.Y` entries in `YUM_REPO_NAMES` must match the target version (#84660).
3. **openshift/images.** When the target builds on stream10, the images config must build `base-stream10` from a stream10 RPM cache step, and the mirror mapping needs an `origin_scos-X.Y_base-stream10` entry (#84562).
4. **Hand-maintained images.** For the router images and any other image named in a check, diff the generated entry against the existing okd-scos config. If the existing config holds a deliberate hand edit, keep it.
5. **Bumping the dev tip.** Never move main/master configs to a new version with a find-and-replace on version strings. #84431 did that, which kept stream9 and stale `ocp/4.22` aliases that #85990 and #85940 had to fix. Regenerate from build data for the new version.
6. **Release controller configs.** `openshift-release-main__okd-scos-X.Y.yaml` isn't generated here. Upgrade jobs can't use a `X-scos-stable` stream until it exists, so point them at the nightly stream until then (#85137).

When a new hand fix to generated configs lands in openshift/release, add a rule to the matching subagent prompt with its PR number, and add it to the helper-script skill's `fixups.py` too.

## Key Data Structures

### ImageCoordinate

A tuple of `(namespace, name, tag)` parsed from a `registry.ci.openshift.org/namespace/name:tag` pullspec. The unique*key is `"{namespace}*{name}\_{tag}"`.

### ci-operator config structure

The final YAML file follows the ci-operator configuration schema. Key sections:

- `base_images`: External image dependencies pulled from CI imagestreams
- `build_root`: The root image used for building (typically a golang builder)
- `images`: List of images to build, each with FROM, inputs, Dockerfile path, and build args
- `promotion`: Where built images are promoted to (origin/scos-{version} imagestream)
- `raw_steps`: Optional pipeline steps (used for RPM repo injection)
- `releases`: CI release integration configuration
- `resources`: Default resource requests for build pods
