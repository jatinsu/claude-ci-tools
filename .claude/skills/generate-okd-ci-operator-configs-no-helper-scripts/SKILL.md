---
name: generate-okd-ci-operator-configs
description: Generate OKD/SCOS ci-operator configuration YAML files from ART ocp-build-data. Replicates the logic of doozer's `images:okd prs open` command as a standalone tool.
argument-hint: "<ocp-build-data-path> [--okd-version <version>] [--output-dir <path>] [--github-token <token>]"
---

Generate OKD/SCOS ci-operator configuration YAML files for the openshift/release repository from ART image metadata. This skill replicates the core config-generation logic of doozer's `images:okd prs open` command without requiring the doozer runtime, using subagents for parallelization.

## Arguments

Parse from: $ARGUMENTS

- First positional argument: path to the `ocp-build-data` directory (must contain `group.yml`, `streams.yml`, and an `images/` subdirectory)
- `--okd-version <version>`: OKD version string (e.g. `4.18`, `5.0`). If omitted, derived from `group.yml` vars as `{MAJOR}.{MINOR}`
- `--output-dir <path>`: Directory to write ci-operator config files (default: `./okd-ci-configs`). Files are written at `{output-dir}/{org}/{repo}/{org}-{repo}-{branch}__okd-scos.yaml`
- `--github-token <token>`: GitHub personal access token for downloading upstream Dockerfiles. If omitted, uses unauthenticated requests (subject to rate limits)

If no ocp-build-data path is provided, search these locations in order:
- `./tmp-dir/ocp-build-data/`
- `../ocp-build-data/`
- `./ocp-build-data/`

## Overview

This skill generates ci-operator configuration YAML files that tell Prow how to build OKD/SCOS images. The process has three phases:

1. **Data Loading** — Read group.yml, streams.yml, and all image metadata YAMLs. Apply variable substitution. Build a dependency graph. Filter to OKD-relevant images and resolve their upstream URLs, branches, and OKD pullspecs.
2. **Dockerfile Analysis** — Download each image's upstream Dockerfile from GitHub. Parse FROM statements to identify parent images and stage names.
3. **Config Generation** — For each unique (org, repo, branch) group, assemble a ci-operator config YAML with the correct base_images, build_root, images, promotion, releases, resources, and optional raw_steps.

## Execution

### Step 1: Validate inputs and read group config

Read `{ocp-build-data}/group.yml` and extract:
- `vars.MAJOR` and `vars.MINOR` (integers)
- `public_upstreams` (list of `{private, public, public_branch?}` mappings)

Derive `okd_version` as `"{MAJOR}.{MINOR}"` if not provided.

Read `{ocp-build-data}/streams.yml` and apply variable substitution (replace all `{VAR_NAME}` patterns with values from `group.yml` vars). Build a stream lookup table.

### Step 2: Launch Subagent 1 — Data Analysis

Launch one subagent with the Agent tool using the following prompt (substitute the actual paths and values):

---

**Subagent 1 prompt:**

You are analyzing ART ocp-build-data to determine which images need OKD ci-operator configurations.

**Input:** ocp-build-data at `OCP_BUILD_DATA_PATH`, OKD version `OKD_VERSION`

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

   c. **public_url and public_branch**: Map the private source git URL to its public equivalent using `public_upstreams`:
      - For each mapping in `public_upstreams`, check if the image's HTTPS-normalized source URL starts with the HTTPS-normalized private URL
      - Use the longest matching private prefix
      - Replace the matched prefix with the corresponding public URL
      - If the mapping has a `public_branch`, use it; otherwise fall back to the source branch

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

**Input:** Read the manifest at `MANIFEST_WITH_DOCKERFILES_PATH`. Output directory: `OUTPUT_DIR`.

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
   dockerfile_path: {dockerfile_path}  # relative to context_dir if set
   from: {ci_operator_name_of_last_desired_parent}
   to: {payload_tag}
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
     {ci_operator_name_of_desired_parent}:
       as:
         - {stage_name}        # if present
         - {original_pullspec} # from the Dockerfile
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
                 
           from: {ci_operator_name_of_base_image}
           to: pre-repo-{payload_tag}
     ```
   - Set the image entry's `from` to the intermediate tag instead of the base image name

5. **Assemble the complete config:**

   ```yaml
   base_images:
     {unique_key}:
       namespace: {namespace}
       name: {name}
       tag: {tag}
   build_root:
     image_stream_tag:
       namespace: {namespace}
       name: {name}
       tag: {tag}
   images:
     - {image entries from step 4}
   promotion:
     to:
       - namespace: origin
         name: scos-{okd_version}
   raw_steps:    # only if any images have inject_rpm_repositories
     - {raw_step entries}
   releases:
     latest:
       integration:
         namespace: origin
         name: scos-{okd_version}
   resources:
     '*':
       requests:
         cpu: 100m
         memory: 200Mi
   ```

   Key ordering in the output: `base_images`, `build_root`, `images`, `promotion`, `raw_steps`, `releases`, `resources`. Only include sections that have content (e.g. omit `base_images` if there are none, omit `raw_steps` if none exist).

6. **Write the YAML file** using `yaml.safe_dump()` with `default_flow_style=False`.

7. **Report** the total number of files written and their paths.

---

### Step 5: Report results

After all subagents complete, report:
1. Total number of ci-operator config files generated
2. List of output file paths
3. Number of images skipped and summary of reasons
4. Any errors encountered

## Key Data Structures

### ImageCoordinate

A tuple of `(namespace, name, tag)` parsed from a `registry.ci.openshift.org/namespace/name:tag` pullspec. The unique_key is `"{namespace}_{name}_{tag}"`.

### ci-operator config structure

The final YAML file follows the ci-operator configuration schema. Key sections:
- `base_images`: External image dependencies pulled from CI imagestreams
- `build_root`: The root image used for building (typically a golang builder)
- `images`: List of images to build, each with FROM, inputs, Dockerfile path, and build args
- `promotion`: Where built images are promoted to (origin/scos-{version} imagestream)
- `raw_steps`: Optional pipeline steps (used for RPM repo injection)
- `releases`: CI release integration configuration
- `resources`: Default resource requests for build pods
