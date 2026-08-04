---
name: generate-okd-ci-operator-configs-with-helper-script
description: Generate OKD/SCOS ci-operator configuration YAML files from ART ocp-build-data. Replicates the logic of doozer's `images:okd prs open` command as a standalone tool.
argument-hint: "<ocp-build-data-path> <okd-version> [--output-dir <path>] [--github-token <token>] [--dry-run]"
---

Generate OKD/SCOS ci-operator configuration YAML files for the openshift/release repository from ART image metadata.

## Arguments

Parse from: $ARGUMENTS

- First argument: path to the ocp-build-data directory (must contain `group.yml`, `streams.yml`, `images/`)
- Second argument: OKD version string (e.g. `4.18`, `5.0`)
- `--output-dir <path>`: Directory to write ci-operator configs (default: `./okd-ci-configs`)
- `--github-token <token>`: GitHub token for downloading upstream Dockerfiles
- `--dry-run`: Print what would be generated without writing files

If no arguments provided, look for ocp-build-data in common locations:

- `./tmp-dir/ocp-build-data/`
- `../ocp-build-data/`
  And derive the OKD version from `group.yml` vars (MAJOR.MINOR).

## What This Skill Does

This skill replicates the `doozer images:okd prs open` command logic. It:

1. Reads ART ocp-build-data (group.yml, streams.yml, image metadata YAMLs)
2. For each image, determines the OKD/SCOS equivalent pullspecs
3. Downloads upstream Dockerfiles from GitHub to analyze FROM statements
4. Generates ci-operator configuration YAML files that tell Prow how to build OKD images
5. Writes configs to `{output-dir}/{org}/{repo}/{org}-{repo}-{branch}__okd-scos.yaml`

## Execution

### Step 0: Locate the helper script

The Python helper script is at: `~/.claude/skills/generate-okd-ci-operator-configs/generate_okd_ci_configs.py`

Check if it exists. If not, inform the user it needs to be created first.

### Step 1: Validate inputs

Verify the ocp-build-data path exists and contains the required files:

```bash
ls {ocp_build_data_path}/group.yml {ocp_build_data_path}/streams.yml {ocp_build_data_path}/images/
```

### Step 2: Run the generator with subagents

Launch **3 subagents in parallel** via the Agent tool:

#### Subagent 1: Analyze image metadata and resolve dependencies

Prompt (fill in OCP_BUILD_DATA_PATH and OKD_VERSION):

---

Read and analyze the ART ocp-build-data at OCP_BUILD_DATA_PATH to build a dependency graph of OKD images.

1. Read `group.yml` — extract `vars.MAJOR`, `vars.MINOR`, and the `public_upstreams` list.
2. Read `streams.yml` — build a lookup of stream name to `upstream_image` value (and `okd.resolve_as` if present).
3. List all YAML files in the `images/` directory.
4. For each image YAML, read it and extract:
   - `for_payload` (boolean)
   - `content.source.okd_alignment` config (if any)
   - `from` config (builders and base image)
   - `content.source.git.url` and `content.source.git.branch.target`
   - `name`, `payload_name`
   - `content.source.dockerfile`
   - `content.source.path`

5. Build two lists:
   a. **payload_images**: images where `for_payload: true`
   b. **builder_images**: images referenced as `from.builder[].member` or `from.member` by payload images

6. For each image in both lists, determine:
   - The OKD payload tag name (from `okd_alignment.tag_name`, or `payload_name`/`name` with `ose-` prefix stripped)
   - The public upstream repo URL (map private URLs using `public_upstreams`)
   - The upstream branch
   - The dockerfile path

Return the results as a structured YAML document written to `OCP_BUILD_DATA_PATH/../okd_image_analysis.yaml` with this schema:

```yaml
major: 5
minor: 0
okd_version: "5.0"
streams:
  rhel-9-golang:
    upstream_image: "registry.ci.openshift.org/..."
  ...
public_upstreams:
  - private: "..."
    public: "..."
images:
  - distgit_key: "cluster-node-tuning-operator"
    payload_tag: "cluster-node-tuning-operator"
    for_payload: true
    okd_alignment: { ... }
    from_config: { ... }
    source_url: "git@github.com:openshift-priv/..."
    public_url: "https://github.com/openshift/..."
    branch: "release-5.0"
    dockerfile_path: "Dockerfile"
    name: "openshift/ose-cluster-node-tuning-rhel9-operator"
    payload_name: "cluster-node-tuning-operator"
```

---

#### Subagent 2: Download and analyze upstream Dockerfiles

**Wait for Subagent 1 to complete first**, then read the `okd_image_analysis.yaml` it produced. For each image entry that has a `public_url`:

1. Download the Dockerfile from the public GitHub URL using:

   ```bash
   curl -sL "https://raw.githubusercontent.com/{org}/{repo}/{branch}/{dockerfile_path}"
   ```

   If a GitHub token is available, add `-H "Authorization: token {TOKEN}"`.

2. Parse each Dockerfile's FROM statements to extract:
   - The parent image pullspec
   - The stage name (from `AS <name>`)

3. Write the parsed Dockerfile info back to `okd_dockerfile_analysis.yaml`:

   ```yaml
   - distgit_key: "cluster-node-tuning-operator"
     parent_images:
       - image: "registry.ci.openshift.org/ocp/builder:..."
         stage_name: "builder"
       - image: "registry.ci.openshift.org/ocp/4.16:base-rhel9"
         stage_name: null
   ```

#### Subagent 3: Generate ci-operator config YAML files

**Wait for Subagents 1 and 2 to complete**, then read both analysis files and generate the ci-operator configurations.

For each unique (org, repo, branch) combination:

1. Create the output directory: `{OUTPUT_DIR}/{org}/{repo}/`

2. Build the ci-operator config following this structure:

   ```yaml
   base_images:
     {namespace}_{name}_{tag}:
       namespace: {namespace}
       name: {name}
       tag: {tag}
   build_root:
     image_stream_tag:
       namespace: {ns}
       name: {is_name}
       tag: {tag}
   images:
     - build_args:
       - name: TAGS
         value: scos
       dockerfile_path: {path}
       from: {base_image_ref}
       inputs:
         {builder_ref}:
           as:
           - {stage_name}
           - {original_pullspec}
       to: {payload_tag}
   promotion:
     to:
     - namespace: origin
       name: scos-{okd_version}
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

3. Handle special cases:
   - `inject_rpm_repositories`: Add `raw_steps` with `pipeline_image_cache_step`
   - `okd_alignment.build_args`: Append to the default TAGS=scos arg
   - `okd_alignment.context_dir`: Set context_dir on the image entry
   - `okd_alignment.ci_build_root`: Use specified build root instead of default

4. Write to `{OUTPUT_DIR}/{org}/{repo}/{org}-{repo}-{branch}__okd-scos.yaml`

### Step 3: Alternatively, use the Python script directly

If the Python helper script exists and has all dependencies, run it:

```bash
python3 ~/.claude/skills/generate-okd-ci-operator-configs/generate_okd_ci_configs.py \
    --ocp-build-data OCP_BUILD_DATA_PATH \
    --okd-version OKD_VERSION \
    --output-dir OUTPUT_DIR \
    [--github-token TOKEN] \
    [--dry-run]
```

### Step 4: Report results

After generation completes, report:

1. Total number of ci-operator configs generated
2. List of output files created
3. Any images that were skipped and why
4. Any errors encountered (e.g. Dockerfiles that couldn't be downloaded)

## Key Resolution Rules

These rules determine how ART image metadata maps to OKD CI pullspecs:

### Stream Resolution

- If stream has `okd.resolve_as.image` -> use that
- If stream has `upstream_image` -> use that
- Otherwise -> use stream's `image` field

### Image OKD Pullspec Resolution

- If `okd_alignment.resolve_as.stream` -> resolve via stream rules above
- If `okd_alignment.resolve_as.image` -> use literal pullspec
- If `okd_alignment.tag_name` set -> `registry.ci.openshift.org/origin/scos-{version}:{tag_name}`
- Otherwise -> strip `ose-` from image name, use as tag: `registry.ci.openshift.org/origin/scos-{version}:{stripped_name}`

### Public Upstream URL Resolution

Apply `public_upstreams` mappings from group.yml:

- `https://github.com/openshift-priv/X` -> `https://github.com/openshift/X` (most common)
- Some repos have specific overrides (e.g. operator-marketplace -> operator-framework/operator-marketplace)

### Branch Resolution for PRs

- If branch starts with `release-` and we're targeting the master/main version: use `main` or `master` (whichever exists)
- If branch starts with `release-` and non-master: use `release-{MAJOR}.{MINOR}` or `openshift-{MAJOR}.{MINOR}`
- If branch starts with `openshift-`: use as-is
